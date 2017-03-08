

# MMTF Specification

*Version*: v1.0

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Specifically, a large subset of the data in mmCIF or PDB files can be represented. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally, the format aims to be easily understood and implemented to facilitate its wide dissemination. For testing encoder and decoder implementations a [test suite](test-suite/) is available.


## Table of contents

* [Overview](#overview)
* [Container](#container)
* [Types](#types)
* [Codecs](#codecs)
    * [Header](#header)
    * [Strategies](#strategies)
* [Encodings](#encodings)
* [Fields](#fields)
    * [Format data](#format-data)
    * [Structure data](#structure-data)
    * [Model data](#model-data)
    * [Chain data](#chain-data)
    * [Group data](#group-data)
    * [Atom data](#atom-data)
* [Traversal](#traversal)


## Overview

This specification describes a set of required and optional [fields](#fields) representing molecular structures and associated data. The fields are limited to six primitive [types](#types) for efficient serialization and deserialization using the binary [MessagePack](http://msgpack.org/) format. The [fields](#fields) in MMTF are stored in a binary [container](#container) format. The top-level of the container contains the field names as keys and field data as values. To describe the layout of data in MMTF we use the [JSON](http://www.json.org/) notation throughout this document.

The first step of decoding MMTF is decoding the MessagePack-encoded container. Many of the resulting MMTF fields do not need to be decoded any further. However, to allow for custom compression some fields are given as binary data and must be decoded using the [strategies](#encodings) described below. For maximal size savings the binary MMTF data can be compressed using general purpose algorithms like [gzip](https://www.gnu.org/software/gzip/) or [brotli](https://github.com/google/brotli).

The fields in the MMTF format group data of the same type together to create a flat data-structure, for instance, the coordinates of all atoms are stored together, instead of in atom objects with other atom-related data. This avoids imposing a deeply-nested hierarchical structure on consuming programs, while still allowing efficient [traversal](traversal) of models, chains, groups, and atoms.


## Container

In principle any serialization format that supports the [types](#types) described below can be used to store the above [fields](#fields). MMTF files (specifically files with the `.mmtf` extension) use the binary [MessagePack](http://msgpack.org/) serialization format.


### MessagePack

The MessagePack format (version 5) is used as the binary container format of MMTF. The MessagePack [specification](https://github.com/msgpack/msgpack/blob/master/spec.md) describes the data types and the data layout. Encoding and decoding libraries for MessagePack are available in many languages, see the MessagePack [website](http://msgpack.org/).


### JSON

The test suite will additionally provide files representing the MMTF [fields](#fields) as [JSON](http://www.json.org/) to help validating implementations of this specification.


## Types

The following types are used for the fields in this specification.

* `String` An UTF-8 encoded string.
* `Float` A 32-bit floating-point number.
* `Integer` A 32-bit signed integer.
* `Map` A data structure of key-value pairs where each key is unique. Also known as "dictionary", "hash".
* `Array` A sequence of elements that have the same type.
* `Binary` An array of unsigned 8-bit integer numbers representing binary data.

The `Binary` type is used here to store encoded data as described in the [Codecs](#codecs) section. When the encoded data is to be interpreted as a multi-byte type (e.g. 32-bit integers) it must be represented in big-endian format.

Note that the MessagePack format limits the `String`, `Map`, `Array` and `Binary` type to (2^32)-1 entries per instance.


## Codecs

This section describes the binary layout of the header and the encoded data as well as the available en/decoding strategies.


### Header

* Bytes  0 to  3: 32-bit signed integer specifying the codec type
* Bytes  4 to  7: 32-bit signed integer specifying the length of the resulting array
* Bytes  8 to 11: 4 bytes containing codec-specific parameter data
* Bytes 12 to  N: bytes containing the encoded array data


### Strategies

#### Pass-through: 32-bit floating-point number array

*Type* 1

*Signature* `byte[] -> float32[]`

*Description* Interpret bytes as array of 32-bit floating-point numbers.


#### Pass-through: 8-bit signed integer array

*Type* 2

*Signature* `byte[] -> int8[]`

*Description* Interpret bytes as array of 8-bit signed integers.


#### Pass-through: 16-bit signed integer array

*Type* 3

*Signature* `byte[] -> int16[]`

*Description* Interpret bytes as array of 16-bit signed integers.


#### Pass-through: 32-bit signed integer array

*Type* 4

*Signature* `byte[] -> int32[]`

*Description* Interpret bytes as array of 32-bit signed integers.


#### UTF8/ASCII fixed-length string array

*Type* 5

*Parameter* `byte[4] -> int32` denoting the string length

*Signature* `byte[] -> uint8[] -> string<length>[]`

*Description* Interpret bytes as array of 8-bit unsigned integers, then iteratively consume `length` many bytes to form a string array.


#### Run-length encoded character array

*Type* 6

*Signature* `byte[] -> int32[] -> char[]`

*Description* Interpret bytes as array of 32-bit signed integers, then run-length decode into array of characters.


#### Run-length encoded 32-bit signed integer array

*Type* 7

*Signature* `byte[] -> int32[] -> int32[]`

*Description* Interpret bytes as array of 32-bit signed integers, then run-length decode into array of 32-bit signed integers.


#### Delta & run-length encoded 32-bit signed integer array

*Type* 8

*Signature* `byte[] -> int32[] -> int32[] -> int32[]`

*Description* Interpret bytes as array of 32-bit signed integers, then run-length decode into array of 32-bit signed integers, then delta decode into array of 32-bit signed integers.


#### Integer & run-length encoded 32-bit floating-point number array

*Type* 9

*Parameter* `byte[4] -> int32` denoting the divisor

*Signature* `byte[] -> int32[] -> int32[] -> float32[]`

*Description* Interpret bytes as array of 32-bit signed integers, then run-length decode into array of 32-bit signed integers, then integer decode into array of 32-bit floating-point numbers using the `divisor` parameter.


#### Integer & delta encoded & two-byte-packed 32-bit floating-point number array

*Type* 10

*Parameter* `byte[4] -> int32` denoting the divisor

*Signature* `byte[] -> int16[] -> int32[] -> int32[] -> float32[]`

*Description* Interpret bytes as array of 16-bit signed integers, then unpack into array of 32-bit integers, then delta decode into array of 32-bit integers, then integer decode into array of 32-bit floating-point numbers using the `divisor` parameter.


#### Integer encoded 32-bit floating-point number array

*Type* 11

*Parameter* `byte[4] -> int32` denoting the divisor

*Signature* `byte[] -> int16[] -> float32[]`

*Description* Interpret bytes as array of 16-bit signed integers, then integer decode into array of 32-bit floating-point numbers using the `divisor` parameter.


#### Integer & two-byte-packed 32-bit floating-point number array

*Type* 12

*Parameter* `byte[4] -> int32` denoting the divisor

*Signature* `byte[] -> int16[] -> int32[] -> float32[]`

*Description* Interpret bytes as array of 16-bit signed integers, then unpack into array of 32-bit signed integers, then integer decode into array of 32-bit floating-point numbers using the `divisor` parameter.

*Note* Useful for arrays where a small amount of values may be slightly larger than two bytes. However, note that with many values larger than that the packing becomes inefficient.


#### Integer & one-byte-packed 32-bit floating-point number array

*Type* 13

*Parameter* `byte[4] -> int32` denoting the divisor

*Signature* `byte[] -> int8[] -> int32[] -> float32[]`

*Description* Interpret array of bytes as array of 8-bit signed integers, then unpack into array of 32-bit signed integers, then integer decode into array of 32-bit floating-point numbers using the `divisor` parameter.

*Note* Useful for arrays where a small amount of values may be slightly larger than one bytes. However, note that with many values larger than that the packing becomes inefficient.


#### Two-byte-packed 32-bit signed integer array

*Type* 14

*Signature* `byte[] -> int16[] -> int32[]`

*Description* Interpret bytes as array of 16-bit signed integers, then unpack into array of 32-bit signed integers.

*Note* Useful for arrays where a small amount of values may be slightly larger than two bytes. However, note that with many values larger than that the packing becomes inefficient.


#### One-byte-packed 32-bit signed integer array

*Type* 15

*Signature* `byte[] -> int8[] -> int32[]`

*Description* Interpret bytes as array of 8-bit signed integers, then unpack into array of 32-bit signed integers.

*Note* Useful for arrays where a small amount of values may be slightly larger than one bytes. However, note that with many values larger than that the packing becomes inefficient.


## Encodings

The following general encoding strategies are used to compress the data contained in MMTF files.


### Run-length encoding

Run-length encoding can generally be used to compress arrays that contain stretches of equal values. Instead of storing each value itself, stretches of equal values are represented by the value itself and the occurrence count, that is a value/count pair.

*Example*:

Starting with the encoded array of value/count pairs. In the following example there are three pairs `1, 10`, `2, 1` and `1, 4`. The first entry in a pair is the value to be repeated and the second entry denotes how often the value must be repeated.

```JSON
[ 1, 10, 2, 1, 1, 4 ]
```

Applying run-length decoding by repeating, for each pair, the value as often as denoted by the count entry.

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1 ]
```


### Delta encoding

Delta encoding is used to store an array of numbers. Instead of storing the numbers themselves, the differences (deltas) between the numbers are stored. When the values of the deltas are smaller than the numbers themselves they can be more efficiently packed to require less space.

Note that arrays in which the values change by an identical amount for a range of consecutive values lend themselves to subsequent run-length encoding.

*Example*:

Starting with the encoded array of delta values:

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1 ]
```

Applying delta decoding. The first entry in the array is left as is, the second is calculated as the sum of the first and the second (not decoded) value, the third as the sum of the second (decoded) and third (not decoded) value and so forth.

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 13, 14, 15, 16 ]
```


### Packing/Recursive indexing encoding

Packing/Recursive indexing encodes values such that the encoded values lie within the open interval (MIN, MAX). This allows to create a more compact representation of a 32-bit signed integer array when the majority of values in the array fit into 16-bit (or 8-bit). To encode each value in the input array the method stores the value itself if it lies within the open interval (MIN, MAX), otherwise the MAX (or MIN if the number is negative) interval endpoint is stored and subtracted from the input value. This process of storing and subtracting is repeated recursively until the remainder lies within the interval.

*Example*:

Starting with the array of 8-bit integer values, so the open interval is (127, -128):

```JSON
[ 127, 41, 34, 1, 0, -50, -128, 0, 7, 127, 0, 127, 127, 14 ]
```

Unpacking/Applying recursive indexing decoding. Values that lie within the interval are copied over to the output array. Values that are equal to an interval endpoint are added to the subsequent value while the subsequent value is equal to an interval endpoint, e.g. the sequence `127, 127, 14` becomes `268`:

```JSON
[ 168, 34, 1, 0, -50, -128, 7, 127, 268 ]
```


### Integer encoding

In integer encoding, floating point numbers are converted to integer values by multiplying with a factor and discard everything after the decimal point. Depending on the multiplication factor this can change the precision but with a sufficiently large factor it is lossless. The integer values can then often be compressed with delta encoding which is the main motivation for it.

*Example*:

Starting with the array of integer values:

```JSON
[ 100, 100, 100, 100, 50, 50 ]
```

Applying integer decoding with a divisor of `100`:

```JSON
[ 1.00, 1.00, 1.00, 1.00, 0.50, 0.50 ]
```


### Dictionary encoding

For dictionary encoding an `Array` is created to store values. Indices as references to the values can then be used instead of repeating the values over and over again. Arrays of indices can afterwards be compressed with delta and run-length encoding.

*Example*:

First create a `Array` to hold values that are referable by indices. In the following example the are two indices, `0` and `1` with some values associated.

```JSON
[
    {
        "groupName": "ASP",
        "singleLetterCode": "D",
        "chemCompType": "L-PEPTIDE LINKING",
        "atomNameList": [ "N", "CA", "C", "O", "CB", "CG", "OD1", "OD2" ],
        "elementList": [ "N", "C", "C", "O", "C", "C", "O", "O" ],
        "formalChargeList": [ 0, 0, 0, 0, 0, 0, 0, 0 ],
        "bondAtomList": [ 1, 0, 2, 1, 3, 2, 4, 1, 5, 4, 6, 5, 7, 5 ],
        "bondOrderList": [ 1, 1, 2, 1, 1, 2, 1 ]
    },
    {
        "groupName": "SER",
        "singleLetterCode": "S",
        "chemCompType": "L-PEPTIDE LINKING",
        "atomNameList": [ "N", "CA", "C", "O", "CB", "OG" ],
        "elementList": [ "N", "C", "C", "O", "C", "O" ],
        "formalChargeList": [ 0, 0, 0, 0, 0, 0 ],
        "bondAtomList": [ 1, 0, 2, 1, 3, 2, 4, 1, 5, 4 ],
        "bondOrderList": [ 1, 1, 2, 1, 1 ]
    }
]
```

The indices can then be used to reference the values as often as needed:

```JSON
[ 0, 1, 1, 0, 1 ]
```


## Fields

The following table lists all top level fields, including their [type](#types) and whether they are required or optional. The top-level fields themselves are stores as a `Map`.

| Name                                        | Type                | Required |
|---------------------------------------------|---------------------|:--------:|
| [mmtfVersion](#mmtfversion)                 | [String](#types)    |    Y     |
| [mmtfProducer](#mmtfproducer)               | [String](#types)    |    Y     |
| [unitCell](#unitcell)                       | [Array](#types)     |          |
| [spaceGroup](#spacegroup)                   | [String](#types)    |          |
| [structureId](#structureid)                 | [String](#types)    |          |
| [title](#title)                             | [String](#types)    |          |
| [depositionDate](#depositiondate)           | [String](#types)    |          |
| [releaseDate](#releasedate)                 | [String](#types)    |          |
| [ncsOperatorList](#ncsoperatorlist)         | [Array](#types)     |          |
| [bioAssemblyList](#bioassemblylist)         | [Array](#types)     |          |
| [entityList](#entitylist)                   | [Array](#types)     |          |
| [experimentalMethods](#experimentalmethods) | [Array](#types)     |          |
| [resolution](#resolution)                   | [Float](#types)     |          |
| [rFree](#rfree)                             | [Float](#types)     |          |
| [rWork](#rwork)                             | [Float](#types)     |          |
| [numBonds](#numbonds)                       | [Integer](#types)   |    Y     |
| [numAtoms](#numatoms)                       | [Integer](#types)   |    Y     |
| [numGroups](#numgroups)                     | [Integer](#types)   |    Y     |
| [numChains](#numchains)                     | [Integer](#types)   |    Y     |
| [numModels](#nummodels)                     | [Integer](#types)   |    Y     |
| [groupList](#grouplist)                     | [Array](#types)     |    Y     |
| [bondAtomList](#bondatomlist)               | [Binary](#types)    |          |
| [bondOrderList](#bondorderlist)             | [Binary](#types)    |          |
| [xCoordList](#xcoordlist)                   | [Binary](#types)    |    Y     |
| [yCoordList](#ycoordlist)                   | [Binary](#types)    |    Y     |
| [zCoordList](#zcoordlist)                   | [Binary](#types)    |    Y     |
| [bFactorList](#bfactorlist)                 | [Binary](#types)    |          |
| [atomIdList](#atomidlist)                   | [Binary](#types)    |          |
| [altLocList](#altloclist)                   | [Binary](#types)    |          |
| [occupancyList](#occupancylist)             | [Binary](#types)    |          |
| [groupIdList](#groupidlist)                 | [Binary](#types)    |    Y     |
| [groupTypeList](#grouptypelist)             | [Binary](#types)    |    Y     |
| [secStructList](#secstructlist)             | [Binary](#types)    |          |
| [insCodeList](#inscodelist)                 | [Binary](#types)    |          |
| [sequenceIndexList](#sequenceindexlist)     | [Binary](#types)    |          |
| [chainIdList](#chainidlist)                 | [Binary](#types)    |    Y     |
| [chainNameList](#chainnamelist)             | [Binary](#types)    |          |
| [groupsPerChain](#groupsperchain)           | [Array](#types)     |    Y     |
| [chainsPerModel](#chainspermodel)           | [Array](#types)     |    Y     |


### Format data

#### mmtfVersion

*Required field*

*Type*: [String](#types).

*Description*: The version number of the specification the file adheres to. The specification follows a [semantic versioning](http://semver.org/) scheme. In a version number `MAJOR.MINOR`, the `MAJOR` part is incremented when specification changes are incompatible with previous versions. The `MINOR` part is changed for additions to the specification that are backwards compatible.

*Examples*:

The current, unreleased, in development specification:

```JSON
"0.1"
```

A future version with additions backwards compatible to versions "1.0" and "1.1":

```JSON
"1.2"
```


#### mmtfProducer

*Required field*

*Type*: [String](#types).

*Description*: The name and version of the software used to produce the file. For development versions it can be useful to also include the checksum of the commit. The main purpose of this field is to identify the software that has written a file, for instance because it has format errors.

*Examples*:

A software name and the checksum of a commit:

```JSON
"RCSB PDB mmtf-java-encoder---version: 6b8635f8d319beea9cd7cc7f5dd2649578ac01a0"
```

Another software name and its version number:

```JSON
"NGL mmtf exporter v1.2"
```


### Structure data

#### title

*Optional field*

*Type*: [String](#types).

*Description*: A short description of the structural data included in the file.

*Example*:

```JSON
"CRAMBIN"
```


#### structureId

*Optional field*

*Type*: [String](#types).

*Description*: An ID for the structure, for example the PDB ID if applicable. If not in conflict with the format of the ID, it must be given in uppercase.

*Example*:

```JSON
"1CRN"
```


#### depositionDate

*Optional field*

*Type*: [String](#types) with the format `YYYY-MM-DD`, where `YYYY` stands for the year in the Gregorian calendar, `MM` is the month of the year between 01 (January) and 12 (December), and `DD` is the day of the month between 01 and 31.

*Description*: A date that relates to the deposition of the structure in a database, e.g. the wwPDB archive.

*Example*:

For example, the second day of October in the year 2005 is written as:

```JSON
"2005-10-02"
```


#### releaseDate

*Optional field*

*Type*: [String](#types) with the format `YYYY-MM-DD`, where `YYYY` stands for the year in the Gregorian calendar, `MM` is the month of the year between 01 (January) and 12 (December), and `DD` is the day of the month between 01 and 31.

*Description*: A date that relates to the release of the structure in a database, e.g. the wwPDB archive.

*Example*:

For example, the third day of December in the year 2013 is written as:

```JSON
"2013-12-03"
```


#### numBonds

*Required field*

*Type*: [Integer](#types).

*Description*: The overall number of bonds. This number must reflect both the bonds given in `bondAtomList` and the bonds given in the `groupType` entries in `groupList`.

*Example*:

```JSON
1142
```


#### numAtoms

*Required field*

*Type*: [Integer](#types).

*Description*: The overall number of atoms in the structure. This also includes atoms at alternate locations.

*Example*:

```JSON
1023
```


#### numGroups

*Required field*

*Type*: [Integer](#types).

*Description*: The overall number of groups in the structure. This also includes extra groups due to micro-heterogeneity.

*Example*:

```JSON
302
```


#### numChains

*Required field*

*Type*: [Integer](#types).

*Description*: The overall number of chains in the structure.

*Example*:

```JSON
4
```


#### numModels

*Required field*

*Type*: [Integer](#types).

*Description*: The overall number of models in the structure.

*Example*:

```JSON
1
```


#### spaceGroup

*Optional field*

*Type*: [String](#types).

*Description*: The Hermann-Mauguin space-group symbol.

*Example*:

```JSON
"P 1 21 1"
```


#### unitCell

*Optional field*

*Type*: [Array](#types) of six [Float](#types) values.

*Description*: Array of six values defining the unit cell. The first three entries are the length of the sides `a`, `b`, and `c` in Å. The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.

*Example*:

```JSON
[ 80.37, 96.12, 57.67, 90.00, 90.00, 90.00 ]
```


#### ncsOperatorList

*Optional field*

*Type*: [Array](#types) of [Array](#types)s of 16 [Float](#types) values.

*Description*: Array of arrays representing 4x4 transformation matrices that are stored linearly in row major order. Thus, the translational component comprises the 4th, 8th, and 12th element. The transformation matrices describe noncrystallographic symmetry operations needed to create all molecules in the unit cell.

*Example*:

```JSON
[
    [
         0.5,   -0.809, -0.309,  128.875,
         0.809,  0.309,  0.5,   -208.524,
        -0.309, -0.5,    0.809,   79.649,
         0.0,    0.0,    0.0,      1.0
    ],
    [
        -0.5,    0.809, -0.309,  386.625,
         0.809,  0.309, -0.5,   -208.524,
        -0.309, -0.5,   -0.809,   79.649,
         0.0,    0.0,    0.0,      1.0
    ]
]
```


#### bioAssemblyList

*Optional field*

*Type*: `Array` of assembly objects with the following fields:

| Name             | Type             | Description                       |
|------------------|------------------|-----------------------------------|
| transformList    | [Array](#types)  | Array of transform objects         |
| name             | [String](#types) | Name of the biological assembly   |

Fields in a `transform` object:

| Name             | Type             | Description                                          |
|------------------|------------------|------------------------------------------------------|
| chainIndexList   | [Array](#types)  | Pointers into chain data fields, [Integers](#types)  |
| matrix           | [Array](#types)  | 4x4 transformation matrix, [Floats](#types)          |

The entries of `chainIndexList` are indices into the [chainIdList](#chainidlist) and [chainNameList](#chainnamelist) fields.

The elements of the 4x4 transformation `matrix` are stored linearly in row major order. Thus, the translational component comprises the 4th, 8th, and 12th element.

*Description*: Array of instructions on how to transform coordinates for an array of chains to create (biological) assemblies. The translational component is given in Å.

*Example*:

The following example shows two transform objects from PDB ID [4OPJ](http://www.rcsb.org/pdb/explore.do?structureId=4OPJ). The transformation matrix of the first object performs no rotation and a translation of 42.387 Å in dimension x. The second one translates -42.387 Å in dimension x.

```JSON
[
    {
        "transformList": [
            {
                "chainIndexList": [ 0, 4, 6 ],
                "matrix": [
                    1.0, 0.0, 0.0,  42.387,
                    0.0, 1.0, 0.0,   0.000,
                    0.0, 0.0, 1.0,   0.000,
                    0.0, 0.0, 0.0,   1.000
                ]
            }
        ]
    },
    {
        "transformList": [
            {
                "chainIndexList": [ 0, 4, 6 ],
                "matrix": [
                    1.0, 0.0, 0.0, -42.387,
                    0.0, 1.0, 0.0,   0.000,
                    0.0, 0.0, 1.0,   0.000,
                    0.0, 0.0, 0.0,   1.000
                ]
            }
        ]
    }
]
```


#### entityList

*Optional field*

*Type*: [Array](#types) of entity objects with the following fields:

| Name             | Type               | Description                                          |
|------------------|--------------------|------------------------------------------------------|
| chainIndexList   | [Array](#array)    | Pointers into chain data fields, [Integers](#types)  |
| description      | [String](#string)  | Description of the entity                            |
| type             | [String](#string)  | Name of the entity type                              |
| sequence         | [String](#string)  | Sequence of the full construct in one-letter-code    |

The entries of `chainIndexList` are indices into the [chainIdList](#chainidlist) and [chainNameList](#chainnamelist) fields.

The `sequence` string contains the full construct, not just the resolved residues. Its characters are referenced by the entries of the [sequenceIndexList](#sequenceindexlist) field. Further, characters follow the IUPAC single letter code for [protein](https://dx.doi.org/10.1111/j.1432-1033.1984.tb07877.x) or [DNA/RNA](https://dx.doi.org/10.1093/nar/13.9.3021) residues, otherwise the character 'X'.

*Description*: Array of unique molecular entities within the structure. Each entry in `chainIndexList` represents an instance of that entity in the structure.

*Vocabulary*: Known values for the entity field `type` from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx.dic/Items/_entity.type.html) are `macrolide`, `non-polymer`, `polymer`, `water`.

*Example*:

```JSON
[
    {
        "description": "BROMODOMAIN ADJACENT TO ZINC FINGER DOMAIN PROTEIN 2B",
        "type": "polymer",
        "chainIndexList": [ 0 ],
        "sequence": "SMSVKKPKRDDSKDLALCSMILTEMETHEDAWPFLLPVNLKLVPGYKKVIKKPMDFSTIREKLSSGQYPNLETFALDVRLVFDNCETFNEDDSDIGRAGHNMRKYFEKKWTDTFKVS"
    },
    {
        "description": "4-FLUOROBENZAMIDOXIME",
        "type": "non-polymer",
        "chainIndexList": [ 1 ],
        "sequence": ""
    },
    {
        "description": "METHANOL",
        "type": "non-polymer",
        "chainIndexList": [ 2, 3, 4 ],
        "sequence": ""
    },
    {
        "description": "water",
        "type": "water",
        "chainIndexList": [ 5 ],
        "sequence": ""
    }
]
```


#### resolution

*Optional field*

*Type*: [Float](#types).

*Description*: The experimental resolution in Angstrom. If not applicable the field must be omitted.

*Examples*:

```JSON
2.3
```


#### rFree

*Optional field*

*Type*: [Float](#types).

*Description*: The R-free value. If not applicable the field must be omitted.

*Examples*:

```JSON
0.203
```


#### rWork

*Optional field*

*Type*: [Float](#types).

*Description*: The R-work value. If not applicable the field must be omitted.

*Examples*:

```JSON
0.176
```


#### experimentalMethods

*Optional field*

*Type*: [Array](#types) of [String](#types)s.

*Description*: The array of experimental methods employed for structure determination.

*Vocabulary*: Known values from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx_v40.dic/Items/_exptl.method.html) are `ELECTRON CRYSTALLOGRAPHY`, `ELECTRON MICROSCOPY`, `EPR`, `FIBER DIFFRACTION`, `FLUORESCENCE TRANSFER`, `INFRARED SPECTROSCOPY`, `NEUTRON DIFFRACTION`, `POWDER DIFFRACTION`, `SOLID-STATE NMR`, `SOLUTION NMR`, `SOLUTION SCATTERING`, `THEORETICAL MODEL`, `X-RAY DIFFRACTION`.

*Example*:

```JSON
[ "X-RAY DIFFRACTION" ]
```


#### bondAtomList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of 32-bit signed integers.

*Description*: Pairs of values represent indices of covalently bonded atoms. The indices point to the [Atom data](#atom-data) arrays. Only covalent bonds may be given.

*Example*:

Using the 'Pass-through: 32-bit signed integer array' encoding strategy (type 4).

In the following example there are three bonds, one between the atoms with the indices 0 and 61, one between the atoms with the indices 2 and 4, as well as one between the atoms with the indices 6 and 12.

```JSON
[ 0, 61, 2, 4, 6, 12 ]
```


#### bondOrderList

*Optional field* If it exists [bondAtomList](#bondatomlist) must also be present. However `bondAtomList` may exist without `bondOrderList`.

*Type*: [Binary](#types) data that decodes into an array of 8-bit signed integers.

*Description*: Array of bond orders for bonds in `bondAtomList`. Must be values between 1 and 4, defining single, double, triple, and quadruple bonds.

*Example*:

Using the 'Pass-through: 8-bit signed integer array' encoding strategy (type 2).

In the following example there are bond orders given for three bonds. The first and third bond have a bond order of 1 while the second bond has a bond order of 2.

```JSON
[ 1, 2, 1 ]
```


### Model data

The number of models in a structure is equal to the length of the [chainsPerModel](chainspermodel) field. The `chainsPerModel` field also defines which chains belong to each model.


#### chainsPerModel

*Required field*

*Type*: [Array](#types) of [Integer](#types) numbers. The number of models is thus equal to the length of the `chainsPerModel` field.

*Description*: Array of the number of chains in each model. The array allows looping over all models:

```Python
# initialize index counter
set modelIndex to 0

# traverse models
for modelChainCount in chainsPerModel
    print modelIndex
    increment modelIndex by one
```

*Examples*:

In the following example there are 2 models. The first model has 5 chains and the second model has 8 chains. This also means that the chains with indices 0 to 4 belong to the first model and that the chains with indices 5 to 12 belong to the second model.

```JSON
[ 5, 8 ]
```

For structures with homogeneous models the number of chains per model is identical for all models. In the following example there are five models, each with four chains.

```JSON
[ 4, 4, 4, 4, 4 ]
```


### Chain data

The number of chains in a structure is equal to the length of the [groupsPerChain](#groupsperchain) field. The `groupsPerChain` field also defines which groups belong to each chain.


#### groupsPerChain

*Required field*

*Type*: [Array](#types) of [Integer](#types) numbers.

*Description*: Array of the number of groups (aka residues) in each chain. The number of chains is thus equal to the length of the `groupsPerChain` field. In conjunction with `chainsPerModel`, the array allows looping over all chains:

```Python
# initialize index counters
set modelIndex to 0
set chainIndex to 0

# traverse models
for modelChainCount in chainsPerModel
    print modelIndex
    # traverse chains
    for 1 to modelChainCount
        print chainIndex
        set offset to chainIndex * 4
        print chainIdList[ offset : offset + 4 ]
        print chainNameList[ offset : offset + 4 ]
        increment chainIndex by 1
    increment modelIndex by 1
```

*Example*:

In the following example there are 3 chains. The first chain has 73 groups, the second 59 and the third 1. This also means that the groups with indices 0 to 72 belong to the first chain, groups with indices 73 to 131 to the second chain and the group with index 132 to the third chain.

```JSON
[ 73, 59, 1 ]
```


#### chainIdList

*Required field*

*Type*: [Binary](#types) data that decodes into an array of 4-character strings.

*Description*: Array of chain IDs. For storing data from mmCIF files the `chainIdList` field should contain the value from the `label_asym_id` mmCIF data item and the `chainNameList` the `auth_asym_id` mmCIF data item. In PDB files there is only a single name/identifier for chains that corresponds to the `auth_asym_id` item. When there is only a single chain identifier available it must be stored in the `chainIdList` field.

*Note*: The character strings must be left aligned and unused characters must be represented by 0 bytes.

*Example*:

Using the 'UTF8/ASCII fixed-length string array' encoding strategy (type 5).

Starting with the array of 8-bit unsigned integers:

```JSON
[ 65, 0, 0, 0, 66, 0, 0, 0, 67, 0, 0, 0 ]
```

Decoding the ASCII characters:

```JSON
[ "A", "", "", "", "B", "", "", "", "C", "", "", "" ]
```

Creating the array of chain IDs:

```JSON
[ "A", "B", "C" ]
```


#### chainNameList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of 4-character strings.

*Description*: Array of chain names. This field allows to specify an additional set of labels/names for chains. For example, it can be used to store both, the `label_asym_id` (in `chainIdList`) and the `auth_asym_id` (in `chainNameList`) from mmCIF files.

*Example*:

Using the 'UTF8/ASCII fixed-length string array' encoding strategy (type 5).

Starting with the array of 8-bit unsigned integers:

```JSON
[ 65, 0, 0, 0, 68, 65, 0, 0 ]
```

Decoding the ASCII characters:

```JSON
[ "A", "", "", "", "DA", "", "", "" ]
```

Creating the array of chain IDs:

```JSON
[ "A", "DA" ]
```



### Group data

The fields in the following sections hold group-related data.

The mmCIF format allows for so-called micro-heterogeneity on the group-level. For groups (residues) with micro-heterogeneity there are two or more entries given that have the same [sequence index](#sequenceindexlist), [group id](#groupidlist) (and [insertion code](#inscodelist)) but are of a different [group type](#grouptypelist). The defining property is their identical sequence index.


#### groupList

*Required field*

*Type*: [Array](#types) of `groupType` objects with the following fields:

| Name             | Type              | Description                                                 |
|------------------|-------------------|-------------------------------------------------------------|
| formalChargeList | [Array](#types)   | Array of formal charges as [Integers](#types)               |
| atomNameList     | [Array](#types)   | Array of atom names, 0 to 5 character [Strings](#types)     |
| elementList      | [Array](#types)   | Array of elements, 0 to 3 character [Strings](#types)       |
| bondAtomList     | [Array](#types)   | Array of bonded atom indices, [Integers](#types)            |
| bondOrderList    | [Array](#types)   | Array of bond orders as [Integers](#types) between 1 and 4  |
| groupName        | [String](#types)  | The name of the group, 0 to 5 characters                    |
| singleLetterCode | [String](#types)  | The single letter code, 1 character                         |
| chemCompType     | [String](#types)  | The chemical component type                                 |


The element name must follow the IUPAC [standard](http://dx.doi.org/10.1515/ci.2014.36.4.25) where only the first character is capitalized and the remaining ones are lower case, for instance `Cd` for Cadmium.

Two consecutive entries in `bondAtomList` representing indices of covalently bound atoms. The indices point into the `formalChargeList`, `atomNameList`, and `elementList` fields.

The `singleLetterCode` is the IUPAC single letter code for [protein](https://dx.doi.org/10.1111/j.1432-1033.1984.tb07877.x) or [DNA/RNA](https://dx.doi.org/10.1093/nar/13.9.3021) residues, otherwise the character 'X' for polymer groups or '?' for non-polymer groups.

*Description*: Common group (residue) data that is referenced via the `groupType` key by group entries.

*Vocabulary*: Known values for the groupType field `chemCompType` from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx_v40.dic/Items/_chem_comp.type.html) are `D-beta-peptide, C-gamma linking`, `D-gamma-peptide, C-delta linking`, `D-peptide COOH carboxy terminus`, `D-peptide NH3 amino terminus`, `D-peptide linking`, `D-saccharide`, `D-saccharide 1,4 and 1,4 linking`, `D-saccharide 1,4 and 1,6 linking`, `DNA OH 3 prime terminus`, `DNA OH 5 prime terminus`, `DNA linking`, `L-DNA linking`, `L-RNA linking`, `L-beta-peptide, C-gamma linking`, `L-gamma-peptide, C-delta linking`, `L-peptide COOH carboxy terminus`, `L-peptide NH3 amino terminus`, `L-peptide linking`, `L-saccharide`, `L-saccharide 1,4 and 1,4 linking`, `L-saccharide 1,4 and 1,6 linking`, `RNA OH 3 prime terminus`, `RNA OH 5 prime terminus`, `RNA linking`, `non-polymer`, `other`, `peptide linking`, `peptide-like`, `saccharide`.

*Example*:

```JSON
[
    {
        "groupName": "GLY",
        "singleLetterCode": "G",
        "chemCompType": "PEPTIDE LINKING",
        "atomNameList": [ "N", "CA", "C", "O" ],
        "elementList": [ "N", "C", "C", "O" ],
        "formalChargeList": [ 0, 0, 0, 0 ],
        "bondAtomList": [ 1, 0, 2, 1, 3, 2 ],
        "bondOrderList": [ 1, 1, 2 ],
    },
    {
        "groupName": "ASP",
        "singleLetterCode": "D",
        "chemCompType": "L-PEPTIDE LINKING",
        "atomNameList": [ "N", "CA", "C", "O", "CB", "CG", "OD1", "OD2" ],
        "elementList": [ "N", "C", "C", "O", "C", "C", "O", "O" ],
        "formalChargeList": [ 0, 0, 0, 0, 0, 0, 0, 0 ],
        "bondAtomList": [ 1, 0, 2, 1, 3, 2, 4, 1, 5, 4, 6, 5, 7, 5 ],
        "bondOrderList": [ 1, 1, 2, 1, 1, 2, 1 ]
    },
    {
        "groupName": "SER",
        "singleLetterCode": "S",
        "chemCompType": "L-PEPTIDE LINKING",
        "atomNameList": [ "N", "CA", "C", "O", "CB", "OG" ],
        "elementList": [ "N", "C", "C", "O", "C", "O" ],
        "formalChargeList": [ 0, 0, 0, 0, 0, 0 ],
        "bondAtomList": [ 1, 0, 2, 1, 3, 2, 4, 1, 5, 4 ],
        "bondOrderList": [ 1, 1, 2, 1, 1 ]
    }
]
```


#### groupTypeList

*Required field*

*Type*: [Binary](#types) data that decodes into an array of 32-bit signed integers.

*Description*: Array of pointers to `groupType` entries in `groupList` by their keys. One entry for each residue, thus the number of residues is equal to the length of the `groupTypeList` field.

*Example*:

Using the 'Pass-through: 32-bit signed integer array' encoding strategy (type 4).

In the following example there are 5 groups. The 1st, 4th and 5th reference the `groupType` with index `2`, the 2nd references index `0` and the third references index `1`. So using the data from the `groupList` example this describes the polymer `SER-GLY-ASP-SER-SER`.

```JSON
[ 2, 0, 1, 2, 2 ]
```


#### groupIdList

*Required field*

*Type*: [Binary](#types) data that decodes into an array of 32-bit signed integers.

*Description*: Array of group (residue) numbers. One entry for each group/residue.

*Example*:

Using the 'Delta & run-length encoded 32-bit signed integer array' encoding strategy (type 8).

Starting with the array of 32-bit signed integers:

```JSON
[ 1, 10, -10, 1, 1, 4 ]
```

Applying run-length decoding:

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, -10, 1, 1, 1, 1 ]
```

Applying delta decoding:

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 1, 2, 3, 4, 5 ]
```


#### secStructList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of 8-bit signed integers.

*Description*: Array of secondary structure assignments coded according to the following table, which shows the eight different types of secondary structure the [DSSP](https://dx.doi.org/10.1002%2Fbip.360221211) algorithm distinguishes. If the field is included there must be an entry for each group (residue) either in all models or only in the first model.

| Code | Name         |
|-----:|--------------|
|    0 | pi helix     |
|    1 | bend         |
|    2 | alpha helix  |
|    3 | extended     |
|    4 | 3-10 helix   |
|    5 | bridge       |
|    6 | turn         |
|    7 | coil         |
|   -1 | undefined    |

*Example*:

Using the 'Pass-through: 8-bit signed integer array' encoding strategy (type 2).

Starting with the array of 8-bit signed integers:

```JSON
[ 7, 7, 2, 2, 2, 2, 2, 2, 2, 7 ]
```


#### insCodeList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of characters.

*Description*: Array of insertion codes, one for each group (residue). The lack of an insertion code must be denoted by a 0 byte.

*Example*:

Using the 'Run-length encoded character array' encoding strategy (type 6).

Starting with the array of 32-bit signed integers:

```JSON
[ 0, 5, 65, 3, 66, 2 ]
```

Applying run-length decoding:

```JSON
[ 0, 0, 0, 0, 0, 65, 65, 65, 66, 66 ]
```

If needed the ASCII codes can be converted to an `Array` of `String`s with the zeros as zero-length `String`s:

```JSON
[ "", "", "", "", "", "A", "A", "A", "B", "B" ]
```


#### sequenceIndexList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of 32-bit signed integers.

*Description*: Array of indices that point into the `sequence` property of an entity object in the [entityList](entitylist) field that is associated with the chain the group belongs to (i.e. the index of the chain is included in the `chainIndexList` of the entity). There is one entry for each group (residue). It must be set to `-1` when a group entry has no associated entity (and thus no sequence), for example water molecules.

*Example*:

Using the 'Delta & run-length encoded 32-bit signed integer array' encoding strategy (type 8).

Starting with the array of 32-bit signed integers:

```JSON
[ 1, 10, -10, 1, 1, 4 ]
```

Applying run-length decoding:

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, -10, 1, 1, 1, 1 ]
```

Applying delta decoding:

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 0, 1, 2, 3, 4 ]
```


### Atom data

The fields in the following sections hold atom-related data.

The mmCIF format allows for alternate locations of atoms. Such atoms have multiple entries in the atom-level fields (including the fields in the [groupList](grouplist) entries). They can be identified and distinguished by their distinct values in the [altLocList](altloclist) field.


#### atomIdList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of 32-bit signed integers.

*Description*: Array of atom serial numbers. One entry for each atom.

*Example*:

Using the 'Delta & run-length encoded 32-bit signed integer array' encoding strategy (type 8).

Starting with the array of 32-bit signed integers:

```JSON
[ 1, 7, 2, 1 ]
```

Applying run-length decoding:

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 2 ]
```

Applying delta decoding:

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 9 ]
```


#### altLocList

*Optional field*

*Type*: [Binary](#types) data that decodes into an array of characters.

*Description*: Array of alternate location labels, one for each atom. The lack of an alternate location label must be denoted by a 0 byte.

*Example*:

Using the 'Run-length encoded character array' encoding strategy (type 6).

Starting with the array of 32-bit signed integers:

```JSON
[ 0, 5, 65, 3, 66, 2 ]
```

Applying run-length decoding:

```JSON
[ 0, 0, 0, 0, 0, 65, 65, 65, 66, 66 ]
```

If needed the ASCII codes can be converted to an `Array` of `String`s with the zeros as zero-length `String`s:

```JSON
[ "", "", "", "", "", "A", "A", "A", "B", "B" ]
```


#### bFactorList

*Optional fields*

*Type*: [Binary](#types) data that decodes into an array of 32-bit floating-point numbers.

*Description*: Array of atom B-factors in in Å^2. One entry for each atom.

*Example*:

Using the 'Integer & delta encoded & two-byte-packed 32-bit floating-point number array' encoding strategy (type 10) with a divisor of 100.

Starting with the packed array of 16-bit signed integers:

```JSON
[ 18200, 0, 2, -1, 100, -3, 5 ]
```

Unpacking/applying recursive indexing decoding to create an array of 32-bit signed integers (note, only the array type changed as the values all fitted into 16-bit signed integers):

```JSON
[ 18200, 0, 2, -1, 100, -3, 5 ]
```

Applying delta decoding to create an array of 32-bit signed integers:

```JSON
[ 18200, 18200, 18202, 18201, 18301, 18298, 18303 ]
```

Applying integer decoding with a divisor of `100` to create an array of 32-bit floating-point numbers:

```JSON
[ 182.00, 182.00, 182.02, 182.01, 183.01, 182.98, 183.03 ]
```


#### xCoordList
#### yCoordList
#### zCoordList

*Required fields*

*Type*: [Binary](#types) data that decodes into an array of 32-bit floating-point numbers.

*Description*: Array of x, y, and z atom coordinates, respectively, in Å. One entry for each atom and coordinate.

*Note*: To clarify, the data for each coordinate is stored in a separate array.

*Example*:

Using the 'Integer & delta encoded & two-byte-packed 32-bit floating-point number array' encoding strategy (type 10) with a divisor of 1000.

Starting with the packed array of 16-bit signed integers:

```JSON
[ 32767, 32767, 32767, 6899, 0, 2, -1, 100, -3, 5 ]
```

Unpacking/Applying recursive indexing decoding to create an array of 32-bit signed integers:

```JSON
[ 105200, 0, 2, -1, 100, -3, 5 ]
```

Applying delta decoding to create an array of 32-bit signed integers:

```JSON
[ 105200, 105200, 105202, 105201, 105301, 105298, 105303 ]
```

Applying integer decoding with a divisor of `1000` to create an array of 32-bit floating-point values:

```JSON
[ 100.000, 105.200, 105.202, 105.201, 105.301, 105.298, 105.303 ]
```


#### occupancyList

*Optional field*

*Description*: Array of atom occupancies, one for each atom.

*Type*: [Binary](#types) data that decodes into an array of 32-bit floating-point numbers.

*Example*:

Using the 'Integer & run-length encoded 32-bit floating-point number array' encoding strategy (type 9) with a divisor of 100.

Starting with the array of 32-bit signed integers:

```JSON
[ 100, 4, 50, 2 ]
```

Applying run-length decoding:

```JSON
[ 100, 100, 100, 100, 50, 50 ]
```

Applying integer decoding with a divisor of `100` to create an array of 32-bit floating-point values:

```JSON
[ 1.00, 1.00, 1.00, 1.00, 0.50, 0.50 ]
```


## Traversal

The following traversal pseudo code assumes that all fields have been decoded.

```Python
# initialize index counters
set modelIndex to 0
set chainIndex to 0
set groupIndex to 0
set atomIndex to 0

# traverse models
for modelChainCount in chainsPerModel
    print modelIndex
    # traverse chains
    for 1 to modelChainCount
        print chainIndex
        set offset to chainIndex * 4
        print chainIdList[ offset : offset + 4 ]
        print chainNameList[ offset : offset + 4 ]
        set chainGroupCount to groupsPerChain[ chainIndex ]
        # traverse groups
        for 1 to chainGroupCount
            print groupIndex
            print groupIdList[ groupIndex ]
            print insCodeList[ groupIndex ]
            print secStructList[ groupIndex ]
            print sequenceIndexList[ groupIndex ]
            print groupTypeList[ groupIndex ]
            set group to groupList[ groupTypeList[ groupIndex ] ]
            print group.groupName
            print group.singleLetterCode
            print group.chemCompType
            set atomOffset to atomIndex
            set groupBondCount to group.bondAtomList.length / 2
            for i in 1 to groupBondCount
                print atomOffset + group.bondAtomList[ i * 2 ]      # atomIndex1
                print atomOffset + group.bondAtomList[ i * 2 + 1 ]  # atomIndex2
                print group.bondOrderList[ i ]
            set groupAtomCount to group.atomNameList.length
            # traverse atoms
            for i in 1 to groupAtomCount
                print atomIndex
                print xCoordList[ atomIndex ]
                print yCoordList[ atomIndex ]
                print zCoordList[ atomIndex ]
                print bFactorList[ atomIndex ]
                print atomIdList[ atomIndex ]
                print altLocList[ atomIndex ]
                print occupancyList[ atomIndex ]
                print group.formalChargeList[ i ]
                print group.atomNameList[ i ]
                print group.elementList[ i ]
                increment atomIndex by 1
            increment groupIndex by 1
        increment chainIndex by 1
    increment modelIndex by 1

# traverse inter-group bonds
for i in 1 to bondAtomList.length / 2
    print bondAtomList[ i * 2 ]      # atomIndex1
    print bondAtomList[ i * 2 + 1 ]  # atomIndex2
    print bondOrderList[ i ]
```
