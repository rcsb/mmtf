
# FAQ

### What is MMTF?
The [**M**acro**M**olecular **T**ransmission **F**ormat][spec] (MMTF) is an extensible, compact, self-contained binary format to efficiently transmit, load, and process 3D biomolecular structural data.

### Why another file format?
Traditional file formats for 3D structures, the legacy PDB format and the current PDBx/mmCIF format represent [archival formats][file-format] for the Protein Data Bank (PDB). [PDBx/mmCIF][wwpdb] is a flexible, extensible, and verbose format with rich metadata, well suited for archival purposes. MMTF on the other hand is optimized for the efficient transmission over networks, and high-speed and memory efficient parsing. MMTF is the equivalent of a JPEG file for images or MPEG-4 file for multimedia, just that MMTF is designed for 3D structural data. MMTF is not intended to be human-readable; it’s a binary custom compressed data format for machine processing.

### What are the design considerations for MMTF?

#### Compactness.
The interactive visualization of very large macromolecular complexes on the web and mobile devices is becoming a challenging problem as experimental techniques advance at an unprecedented rate and deliver structures of increasing size. The small file size of MMTF files enables the download of even the largest structures in the PDB in seconds. In addition, it reduces client-side memory requirements significantly. Furthermore, the entire PDB can be stored in less than 7 GB (~6 bytes/atom), thus enables large-scale distributed in-memory processing of the PDB at unprecedented speeds.

#### Fast loading and parsing.
File loading is sped up by the small file size. Data are stored as primitive types (bytes, integers, strings, floating point numbers) in a compressed binary format. Therefore, they can be loaded into typed arrays by simply copying the data; no parsing or further processing is required. This reduces both client side cpu and memory consumption.
Complete self-contained data model.

We augment MMTF with complete and accurate bond and bond order information for all molecular entities, as well as, consistently calculated secondary structure (DSSP). These data are crucial for fast rendering, as these data do not have to be inferred during runtime.

#### Interoperability.
The MessagePack format (version 5) is used as the binary container format for MMTF. MessagePack is an efficient binary serialization format, similar to JSON, but faster and more compact. The MessagePack [specification][msgpack-spec] describes the data types and the data layout. Encoding and decoding libraries for MessagePack are available in many languages, see the [MessagePack website][msgpack-web].

#### Extensibility.
As structural biology evolves, so do the requirements for storing new data. Similarly, an application may want to add custom data. MMTF, with the MessagePack as its container format enables extensions without affecting the core [MMTF format][mmtf].

### Is there an MMTF decoder available for my favorite programming language?
We currently support Java and JavaScript decoders for MMTF. A Python version is under development. Contact us if you need a decoder for your favorite programming language. We will be happy to collaborate with you in the development of a parser. Official decoders that we adopt must conform to strict coding and documentation standards and must pass code review and extensive testing.

### How do I access individual MMTF files?
See the description how to download gzipped [MMTF files][mmtf-download].

### How do I access the entire PDB in MMTF format?
The entire PDB archive is available as a [Hadoop Sequence file][hadoop] (~7GB). This file can be efficiently processed using [Apache Spark][spark], and other big data platforms. Each record in the Hadoop Sequence file is a key-value pair, where the key is the PDB ID and the value is an MMTF-encoded and gzipped binary array. See the description how to [download MMTF files][mmtf-download].

[spec]: <https://github.com/rcsb/mmtf/blob/v0.1/spec.md>
[file-format]: <http://www.wwpdb.org/documentation/file-format>
[wwpdb]: <http://mmcif.wwpdb.org/>
[msgpack-spec]: <https://github.com/msgpack/msgpack/blob/master/spec.md>
[msgpack-web]: <http://msgpack.org/index.html>
[mmtf]: <https://github.com/rcsb/mmtf>
[mmtf-download]: <http://mmtf.rcsb.org/download.html>
[hadoop]: <https://hadoop.apache.org/docs/r2.6.2/api/org/apache/hadoop/io/SequenceFile.html>
[spark]: <http://spark.apache.org>
