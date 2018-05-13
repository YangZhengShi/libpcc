# Library for Point Cloud Compression (libpcc)
libpcc is a C++ framework for configurable 3D grid based point cloud compression. The compression algorithm is based on local quantization within 3D grid cells created by subdividing the point clouds bounding box. The dimensions for grid subdivision as well as a target bit size of compressed voxels can be modified per cell. This helps to increase or decrease visual quality and data size of the compression result. Additionally, the framework utilizes zlib to deflate quantized data using the LZSS & Huffman coding algorithms. As a transfer structure for compressed data, network messages are generated which can be sent over a network using the [ZeroMQ](http://zeromq.org/) (zmq) library. A doxygen documentation of the source code can viewed from `<libpcc-repo>/docs/html/index.html`.

## Setup
libpcc depends on [zlib](https://zlib.net/) & [zmq](http://zeromq.org/). You need to have both libraries installed to use this library. If you fulfill these requirements you can build the library from code. Simply create a folder `lib` in the root directory of libpcc. In the examples folder build the project using the make file contained:
```
$ > mkdir lib
$ > cd examples
$ > make
```
When modifying the source code, make sure to clean any previous library builds:
```
$ > make realclean
$ > make
```

## Usage
The `PointCloudGridEncoder` class defines high level access to the libpcc compression engine.
To encode an arbitrary point cloud simply define a `std::vector<UncompressedVoxel>` and pass it to the `PointCloudGridEncoder::encode(...)` function:
```
// create sample point cloud
std::vector<UncompressedVoxel> pc;
UncompressedVoxel v;
v.rgba[0] = 255, v.rgba[1] = 255, v.rgba[2] = 255, v.rgba[3] = 255;
for(int x = 0; x < 10; ++x) {
    for(int y = 0; y < 10; ++y) {
        for(int z = 0; z < 10; ++z) {
            v.pos[0] = x, v.pos[1] = y, v.pos[2] = z;
            pc.emplace_back(v);
        }
    }    
}

// create encoder and compress point cloud
PointCloudGridEncoder encoder;
zmq::message_t msg = encoder.encode(pc);
```
As mentioned before, the output message can directly be sent over a network using the zmq library.
To decode a message, call `PointCloudGridEncoder::decode(...)` providing as arguments a.) a zmq message (previously created by the encoding process) and b.) a pointer to a target `std::vector<UncompressedVoxel>`:
```
std::vector<UncompressedVoxel> pc;
/*... fill pc with meaningful data ...*/

PointCloudGridEncoder encoder;
zmq::message_t msg = encoder.encode(pc);

std::vector<UncompressedVoxel> pc_decomp;
bool success = encoder.decode(msg, &pc_decomp);
std::cout << “Compression success: ” << success << std::endl;
```
As can be seen in the example above, a boolean value is returned by the decoding process to denote the success of the decoding process.

## Configuring the compression engine
Multiple parameters for configuring the compression process can be set. These are handled by the `PointCloudGridEncoder::EncodingSettings` struct:
```
struct EncodingSettings {
    EncodingSettings()
        : grid_precision()
        , num_threads(24)
        , verbose(false)
        , irrelevance_coding(true)
        , entropy_coding(true)
        , appendix_size(0)
    {}
    
    GridPrecisionDescriptor grid_precision;
    bool verbose;
    int num_threads;
    bool irrelevance_coding;
    bool entropy_coding;
    unsigned long appendix_size;
};
```
Each `PointCloudGridEncoder` instance holds a `settings` object as a public class member to configure compression. Simply modify it prior to calling `PointCloudGridEncoder::encode(...)`.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&mdash; *Note: Configuring the encoder prior to decoding is not necessary, as all configurations are encoded into and read from the zmq message generated by the compression process.*

libpcc uses OpenMP for parallelizing the encode and decode processing steps. Use `num_threads` to set the number of threads used by OpenMP. If `verbose` is set true, the compression and decompression process will generate additional debug output to monitor performance. `irrelevance_coding` enables discarding of redundant information created by quantization while averaging properties of voxels moved to the same quantized location. The `irrelevance_coding` process is currently implemented sequentially, so enabling it might increase encoding time but will in most cases reduce the output data size quite heavily. `grid_precision` is of type `GridPrecisionDescriptor` and can be used to adjust the several properties affecting quantization:
```
struct GridPrecisionDescriptor {
    explicit GridPrecisionDescriptor(const Vec8& t_dimensions=Vec8(4,4,4),
                                     const BoundingBox& t_bounding_box=BoundingBox(),
                                     const Vec<BitCount>& t_point_prec=Vec<BitCount>(BIT_4,BIT_4,BIT_4),
                                     const Vec<BitCount>& t_color_prec=Vec<BitCount>(BIT_4,BIT_4,BIT_4))
        : dimensions(t_dimensions)
        , bounding_box(t_bounding_box)
        , default_point_precision(t_point_prec)
        , default_color_precision(t_color_prec)
        , point_precision()
        , color_precision()
    {
        initCells();
    }

    void resize(const Vec8& dim) {
        point_precision.clear();
        color_precision.clear();
        dimensions = dim;
        initCells();
    }

    Vec8 dimensions;
    BoundingBox bounding_box;
    Vec<BitCount> default_point_precision;
    Vec<BitCount> default_color_precision;
    std::vector<Vec<BitCount>> point_precision;
    std::vector<Vec<BitCount>> color_precision;

private:
    void initCells() {
        unsigned idx_count = dimensions.x * dimensions.y * dimensions.z;
        point_precision.resize(idx_count, default_point_precision);
        color_precision.resize(idx_count, default_color_precision);
    }
};
```
The `bounding_box` property describes the overall x,y,z-bounds of the input point cloud to be compressed. To increase precision of encoding it makes sense to define a tight fitting bounding box prior to encoding.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&mdash;*Note: Points outside the configured `bounding_box` will be discarded within the compression step.*

`dimensions` denotes the number of subdivisions of the `bounding_box` for each axis. Within the bounds of the 3D grid cells computed by set configuration quantization will be performed. Thus, high values lead to a less lossy compression but also increase the overall message size.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&mdash;*Note: The dimensions property should only be changed using the `GridPrecisionDescriptor::resize(...)` function. In that way all resulting cells can be initialized again.*

A `default_point_precision` and `default_color_precision` is defined to denote the target bit size of each position and color property of a compressed voxel. When quantizing, a value range is given for each component of a property. For position values this will for instance be the [min,max] range of the respective axis within a grid cell. The number of linearly distributed sample points along these bounds is given by 2^BitCount, i.e. `default_point_precision.x = BIT_4` will generate 2^4=16 sample points along the x-axis of a cell. Additionally, the `point_precision` and `color_precision` vectors can be used to define a varying property precision for arbitrary grid cells. The index of a cell can be computed by `x_idx + y_idx * dimensions.x + z_idx * dimensions.x * dimensions.y`, where x/y/z_idx denote the respective cell component indexes.

libpcc also offers the ability to encase arbitrary additional contents into the zmq message created by the encoder. This data can be appended to a message using the appendix. To do this define the `appendix_size` as a maximum byte count for the data you wish to insert. `PointCloudGridEncoder::writeToAppendix(...)` can then be used to add content to the message appendix after compression. To extract set content use `PointCloudGridEncoder::readFromAppendix(...)`. This can for instance be useful when sending diagnostic data of the compression process (otherwise lost after compression) along with a compressed point cloud.
```
std::vector<UncompressedVoxel> pc;
/*... fill pc with meaningful data ...*/

PointCloudGridEncoder encoder;
encoder.settings.appendix_size(100); // maximum of 100 characters

zmq::message_t msg = encoder.encode(pc);
encoder.writeToAppendix(msg, "some useful information");

std::string apx_content;
encoder.readFromAppendix(apx_content);
```
The following code snippet illustrates a sample compression process incorporating modified values for all settings.
```
// setup grid quantization settings
GridPrecisionDescriptor descr(
    Vec8(6,8,6),
    BoundingBox(Vec<float>(-1.0f, 0.0f, -1.0f), Vec<float>(1.0f, 2.2f, 1.0f)),
    Vec<BitCount>(BIT_8,BIT_8,BIT_8),
    Vec<BitCount>(BIT_5,BIT_5,BIT_5)
);
descr.point_precision[10] = Vec<BitCount>(Bit_4,Bit_4,Bit_4);
descr.color_precision[10] = Vec<BitCount>(Bit_3,Bit_4,Bit_3);

// create and configure encoder
PointCloudGridEncoder encoder;
encoder.settings.grid_precision = descr;
encoder.settings.irrelevance_coding = true;
encoder.settings.entropy_coding = true;
encoder.settings.appendix_size = 500;
encoder.settings.verbose = true;

// compress point cloud
std::vector<UncompressedVoxel> pc;
/*... fill pc with meaningful data ...*/

zmq::message_t msg = encoder.encode(pc);

// add appendix
encoder.writeToAppendix(msg, "some useful information");
```











