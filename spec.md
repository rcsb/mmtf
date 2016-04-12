

# MMTF Specification

*INITIAL DRAFT* (to be replaced by the version number)

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Specifically, a large subset of the data in mmCIF or PDB files can be represented. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally the format aims to be easy to understand and implement to facilitates its dissemination.


## Table of contents

* [Overview](#overview)
* [Container](#container)
* [Types](#types)
* [Encodings](#encodings)
* [Fields](#fields)
    * [Format data](#format-data)
    * [Structure data](#structure-data)
    * [Model data](#model-data)
    * [Chain data](#chain-data)
    * [Group data](#group-data)
    * [Atom data](#atom-data)
* [Traversal](#traversal)
* [mmCIF](#mmcif)


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
* `Array` A list of elements that may be of different type.
* `Binary` A list of unsigned 8-bit integer numbers representing binary data.

The `Binary` type is used here to store arrays with values of the same type. Such "typed arrays" are not directly supported by the `msgpack` format. However it is straightforward to work with arrays of simple numeric types by re-interpreting the data in a `Binary` field:

* List of 8-bit unsigned integers.
* List of 8-bit signed integers.
* List of 16-bit unsigned integers in big-endian format.
* List of 16-bit signed integers in big-endian format.
* List of 32-bit unsigned integers in big-endian format.
* List of 32-bit signed integers in big-endian format.
* List of 32-bit floating-point numbers in big-endian format.

For example, for an array of 32-bit integers groups of 4 bytes are interpreted as 32-bit integers. All such multi-byte types must be represented in big-endian format.

Note that the MessagePack format limits the `String`, `Map`, `Array` and `Binary` type to (2^32)-1 entries per instance.


## Encodings

The following encoding strategies are used to compress the data contained in MMTF files.


### Run-length encoding

Run-length decoding can generally be used to compress lists that contain stretches of equal values. Instead of storing each value itself, stretches of equal values are represented by the value itself and the occurrence count, that is a value/count pair.

*Examples*:

Starting with the encoded list of value/count pairs. In the following example there are three pairs `1, 10`, `2, 1` and `1, 4`. The first entry in a pair is the value to be repeated and the second entry denotes how often the value must be repeated.

```JSON
[ 1, 10, 2, 1, 1, 4 ]
```

Applying run-length decoding by repeating, for each pair, the value as often as denoted by the count entry.

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1 ]
```

Another example that has strings instead of numbers as values:

```JSON
[ "A", 5, "B", 2 ]
```

Applying run-length decoding:

```JSON
[ "A", "A", "A", "A", "A", "B", "B" ]
```


### Delta encoding

Delta encoding is used to store lists of numbers. Instead of storing the numbers themselves, the differences (deltas) between the numbers are stored. When the values of the deltas are smaller than the numbers themselves they can be more efficiently packed to require less space.

Note that lists which change by an identical amount for a range of consecutive values lend themselves to subsequent run-length encoding.

*Example*:

Starting with the encoded list of delta values:

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1 ]
```

Applying delta decoding. The first entry in the list is left as is, the second is calculated as the sum of the first and the second (not decoded) value, the third as the sum of the second (decoded) and third (not decoded) value and so forth.

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 13, 14, 15, 16 ]
```


#### Split-list delta encoding

Split-list delta encoding is an adjusted delta encoding to handle lists with some intermittent large delta values. The list is split into two arrays, called "big" and "small". The "big" array of 32-bit signed integers holds pairs of a large delta value (>=2^15) and the number of subsequent small delta values (<2^15). The "small" array of 16-bit signed integers holds the small values, that is, the values fitting into a 16-bit signed integer.

*Example*:

Starting with the "big" and the "small" arrays:

```JavaScript
[ 1200, 3, 100, 2 ]  // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create an array of 32-bit signed integers:

```JSON
[ 1200, 1200, 1202, 1201, 1301, 1298, 1303 ]
```


### Integer encoding

In integer encoding, floating point numbers are converted to integer values by multiplying with a factor and discard everything after the decimal point. Depending on the multiplication factor this can change the precision but with a sufficiently large factor it is lossless. The integer values can then often be compressed with delta encoding which is the main motivation for it.

*Example*:

Starting with the list of integer values:

```JSON
[ 100, 100, 100, 100, 50, 50 ]
```

Applying integer decoding with a divisor of `100`:

```JSON
[ 1.00, 1.00, 1.00, 1.00, 0.50, 0.50 ]
```


### Dictionary encoding

For dictionary encoding a `Map` is created to store key-value pairs. References to the keys can then be used instead of repeating the values over and over again. Lists of keys can afterwards be compressed with delta and run-length encoding.

*Example*:

First create a `Map` to hold values that are referable by keys. In the following example the are two keys, `1` and `2` with some values associated.

```JSON
{
    "1": {
        "A": [ 1, 2, 3 ],
        "B": "HELLO"
    },
    "2": {
        "A": [ 4, 5, 6 ],
        "B": "HEY"
    }
}
```

The keys can then be used to reference the values as often as needed:

```JSON
[ 1, 1, 1, 1, 2, 2, 2, 2, 1, 1, 1 ]
```

Note that in the above example the list of keys can also be efficiently run-length encoded:

```JSON
[ 1, 4, 2, 4, 1, 3 ]
```


## Fields

The following table lists all top level fields, including their [type](#types) and whether they are required or optional. The top-level fields themselves are stores as a `Map`.

| Name                                        | Type                 | Required |
|---------------------------------------------|----------------------|:--------:|
| [mmtfVersion](#mmtfversion)                 | [String](#string)    |    Y     |
| [mmtfProducer](#mmtfproducer)               | [String](#string)    |    Y     |
| [unitCell](#unitcell)                       | [Array](#array)      |          |
| [spaceGroup](#spacegroup)                   | [String](#string)    |          |
| [structureId](#structureid)                 | [String](#string)    |          |
| [title](#title)                             | [String](#string)    |          |
| [depositionDate](#depositiondate)           | [String](#string)    |          |
| [releaseDate](#releasedate)                 | [String](#string)    |          |
| [bioAssemblyList](#bioassemblylist)         | [Array](#array)      |          |
| [entityList](#entitylist)                   | [Array](#array)      |          |
| [experimentalMethods](#experimentalmethods) | [Array](#array)      |          |
| [resolution](#resolution)                   | [Float](#float)      |          |
| [rFree](#rfree)                             | [Float](#float)      |          |
| [rWork](#rwork)                             | [Float](#float)      |          |
| [numBonds](#numbonds)                       | [Integer](#integer)  |    Y     |
| [numAtoms](#numatoms)                       | [Integer](#integer)  |    Y     |
| [groupList](#grouplist)                     | [Array](#Array)      |    Y     |
| [bondAtomList](#bondatomlist)               | [Binary](#binary)    |          |
| [bondOrderList](#bondorderlist)             | [Binary](#binary)    |          |
| [xCoordBig](#xcoordbig-xcoordsmall)         | [Binary](#binary)    |    Y     |
| [xCoordSmall](#xcoordbig-xcoordsmall)       | [Binary](#binary)    |    Y     |
| [yCoordBig](#ycoordbig-ycoordsmall)         | [Binary](#binary)    |    Y     |
| [yCoordSmall](#ycoordbig-ycoordsmall)       | [Binary](#binary)    |    Y     |
| [zCoordBig](#zcoordbig-zcoordsmall)         | [Binary](#binary)    |    Y     |
| [zCoordSmall](#zcoordbig-zcoordsmall)       | [Binary](#binary)    |    Y     |
| [bFactorBig](#bfactorbig-bfactorsmall)      | [Binary](#binary)    |          |
| [bFactorSmall](#bfactorbig-bfactorsmall)    | [Binary](#binary)    |          |
| [atomIdList](#atomidlist)                   | [Binary](#binary)    |          |
| [altLocList](#altloclist)                   | [Array](#array)      |          |
| [occupancyList](#occupancylist)             | [Binary](#binary)    |          |
| [groupIdList](#groupidlist)                 | [Binary](#binary)    |    Y     |
| [groupTypeList](#grouptypelist)             | [Binary](#binary)    |    Y     |
| [secStructList](#secstructlist)             | [Binary](#binary)    |          |
| [insCodeList](#inscodelist)                 | [Array](#array)      |          |
| [sequenceIdList](#sequenceidlist)           | [Binary](#binary)    |          |
| [chainIdList](#chainidlist)                 | [Binary](#binary)    |    Y     |
| [chainNameList](#chainnamelist)             | [Binary](#binary)    |          |
| [groupsPerChain](#groupsperchain)           | [Array](#array)      |    Y     |
| [chainsPerModel](#chainspermodel)           | [Array](#array)      |    Y     |


### Format data

#### mmtfVersion

*Required field*

*Type*: `String`

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

*Type*: `String`.

*Description*: The name and version of the software used to produce the file. For development versions it can be useful to also include the checksum of the commit. If the producer is not available set to `"NA"`. The main purpose of this field is to identify the software that has written a file, for instance because it has format errors.

*Examples*:

A software name and the checksum of a commit:

```JSON
"RCSB PDB mmtf-java-encoder---version: 6b8635f8d319beea9cd7cc7f5dd2649578ac01a0"
```

Another software name and its version number:

```JSON
"NGL mmtf exporter v1.2"
```

No software name available:

```JSON
"NA"
```


### Structure data

#### title

*Optional field*

*Type*: `String`.

*Description*: A short description of the structural data included in the file.

*Example*:

```JSON
"CRAMBIN"
```


#### structureId

*Optional field*

*Type*: `String`

*Description*: An ID for the structure, for example the PDB ID if applicable.

*Example*:

```JSON
"1CRN"
```


#### depositionDate

*Optional field*

*Type*: `String` with the format `YYYY-MM-DD`, where `YYYY` stands for the year in the Gregorian calendar, `MM` is the month of the year between 01 (January) and 12 (December), and `DD` is the day of the month between 01 and 31.

*Description*: A date that relates to the deposition of the structure in a database, e.g. the wwPDB archive.

*Example*:

For example, the second day of October in the year 2005 is written as:

```JSON
"2005-10-02"
```


#### releaseDate

*Optional field*

*Type*: `String` with the format `YYYY-MM-DD`, where `YYYY` stands for the year in the Gregorian calendar, `MM` is the month of the year between 01 (January) and 12 (December), and `DD` is the day of the month between 01 and 31.

*Description*: A date that relates to the release of the structure in a database, e.g. the wwPDB archive.

*Example*:

For example, the third day of December in the year 2013 is written as:

```JSON
"2013-12-03"
```


#### numAtoms

*Required field*

*Type*: `Integer`.

*Description*: The overall number of atoms in the structure. This also includes atoms at alternate locations.

*Example*:

```JSON
1023
```


#### numBonds

*Required field*

*Type*: `Integer`.

*Description*: The overall number of bonds. This number must reflect both the bonds given in `bondAtomList` and the bonds given in the `groupType` entries in `groupList`.

*Example*:

```JSON
1142
```


#### spaceGroup

*Optional field*

*Type*: `String`.

*Description*: The Hermann-Mauguin space-group symbol.

*Example*:

```JSON
"P 1 21 1"
```


#### unitCell

*Optional field*

*Type*: `Array` of six `Float` values.

*Description*: List of six values defining the unit cell. The first three entries are the length of the sides `a`, `b`, and `c` in Å. The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.

*Example*:

```JSON
[ 10.0, 12.0, 30.0, 90.0, 90.0, 120.0 ]
```


#### bioAssemblyList

*Optional field*

*Type*: `Array` of assembly objects with the following fields:

| Name                      | Type             | Description                       |
|---------------------------|------------------|-----------------------------------|
| transforms                | [Array](#array)  | List of transform objects         |

Fields in a `transform` object:

| Name                      | Type             | Description                       |
|---------------------------|------------------|-----------------------------------|
| chainIndexList            | [Array](#array)  | Pointers into chain data fields   |
| transformation            | [Array](#array)  | 4x4 transformation matrix         |

The entries of `chainIndexList` are indices into the [chainIdList](#chainidlist) and [chainNameList](#chainnamelist) fields.

The elements of the 4x4 `transformation` matrix are stored linearly in row major order. Thus, the translational component comprises the 4th, 8th, and 12th element.

*Description*: List of instructions on how to transform coordinates for a list of chains to create (biological) assemblies.

*Example*:

The following example shows a single transform object from PDB ID [4opj](http://www.rcsb.org/pdb/explore.do?structureId=4OPJ). The transformation matrix performs no rotation and a translation of -42.387 in dimension x.

```JSON
[
    {
        "transforms": [
            {
                "chainIndexList": [ 0, 4, 6 ],
                "transformation": [
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

*Type*: `Array` of entity objects with the following fields:

| Name             | Type               | Description                       |
|------------------|--------------------|-----------------------------------|
| chainIndexList   | [Array](#array)    | Pointers into chain data fields   |
| description      | [String](#string)  | Description of the entity         |
| type             | [String](#string)  | Name of the entity type           |
| sequence         | [String](#string)  | Sequence in one-letter-code       |

The entries of `chainIndexList` are indices into the [chainIdList](#chainidlist) and [chainNameList](#chainnamelist) fields.

The characters of the `sequence` string are referenced by the entries of the [sequenceIdList](#sequenceidlist) field. 

*Description*: List of unique molecular entities within the structure. Each entry in `chainIndexList` represents an instance of that entity in the structure.

*Vocabulary*: Known values for the entity field `type` from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx.dic/Items/_entity.type.html) are `macrolide`, `non-polymer`, `polymer`, `water`.

*Example*:

```JSON
[
    {
        "chainIndexList": [ 0, 1, 2, 3 ],
        "description": "some polymer",
        "type": "polymer",
        "sequence": "MNTYASDE"
    },
    {
        "chainIndexList": [ 4 ],
        "description": "some ligand",
        "type": "non-polymer",
        "sequence": ""
    }
]
```


#### resolution

*Optional field*

*Type*: `Float`.

*Description*: The experimental resolution in Angstrom. If not applicable the field must be omitted.

*Examples*:

```JSON
2.3
```


#### rFree

*Optional field*

*Type*: `Float`.

*Description*: The R-free value. If not applicable the field must be omitted.

*Examples*:

```JSON
0.203
```


#### rWork

*Optional field*

*Type*: `Float`.

*Description*: The R-work value. If not applicable the field must be omitted.

*Examples*:

```JSON
0.176
```


#### experimentalMethods

*Optional field*

*Type*: `Array` of `String`s.

*Description*: The list of experimental methods employed for structure determination.

*Vocabulary*: Known values from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx_v40.dic/Items/_exptl.method.html) are `ELECTRON CRYSTALLOGRAPHY`, `ELECTRON MICROSCOPY`, `EPR`, `FIBER DIFFRACTION`, `FLUORESCENCE TRANSFER`, `INFRARED SPECTROSCOPY`, `NEUTRON DIFFRACTION`, `POWDER DIFFRACTION`, `SOLID-STATE NMR`, `SOLUTION NMR`, `SOLUTION SCATTERING`, `THEORETICAL MODEL`, `X-RAY DIFFRACTION`.

*Example*:

```JSON
[ "X-RAY DIFFRACTION" ]
```


#### bondAtomList

*Optional field*

*Type*: `Binary` data that is interpreted as an array of 32-bit unsigned integers.

*Description*: Pairs of values represent indices of covalently bonded atoms. The indices point to the [Atom data](#atom-data) arrays. Only covalent bonds may be given.

*Example*:

In the following example there are three bonds, one between the atoms with the indices 0 and 1, one between the atoms with the indices 0 and 2, as well as one between the atoms with the indices 2 and 4.

```JSON
[ 0, 1, 0, 2, 2, 4 ]
```


#### bondOrderList

*Optional field* If it exists [bondAtomList](#bondatomlist) must also be present. However `bondAtomList` may exist without `bondOrderList`.

*Type*: `Binary` data that is interpreted as an array of 8-bit unsigned integers, i.e. take as is.

*Description*: List of bond orders for bonds in `bondAtomList`. Must be values between 1 and 3.

*Example*:

In the following example there are bond orders given for three bonds. The first and third bond have a bond order of 1 while the second bond has a bond order of 2.

```JSON
[ 1, 2, 1 ]
```


### Model data

The number of models in a structure is equal to the length of the [chainsPerModel](chainspermodel) field. The `chainsPerModel` field also defines which chains belong to each model.


#### chainsPerModel

*Required field*

*Type*: `Array` of `Integer` numbers. The number of models is thus equal to the length of the `chainsPerModel` field.

*Description*: List of the number of chains in each model. The list allows looping over all models:

```Python
# initialize index counter
set modelIndex to 0

# traverse models
for modelChainCount in chainsPerModel
    print modelIndex
    increment modelIndex by one
```

*Example*:

In the following example there are 2 models. The first model has 5 chains and the second model has 8 chains. This also means that the chains with indices 0 to 4 belong to the first model and that the chains with indices 5 to 12 belong to the second model.

```JSON
[ 5, 8 ]
```


### Chain data

The number of chains in a structure is equal to the length of the [groupsPerChain](#groupsperchain) field. The `groupsPerChain` field also defines which groups belong to each chain.


#### groupsPerChain

*Required field*

*Type*: `Array` of `Integer` numbers.

*Description*: List of the number of groups (aka residues) in each chain. The number of chains is thus equal to the length of the `groupsPerChain` field. In conjunction with `chainsPerModel`, the list allows looping over all chains:

```Python
# initialize index counters
set chainIndex to 0
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

*Type*: `Binary` data that is interpreted as an array of 8-bit unsigned integers representing ASCII characters.

*Decoding*: Groups of four consecutive ASCII characters create the list of chain IDs. Note that the ASCII decoding here is optional, a decoding library may choose to pass the array of 8-bit unsigned integers on for performance reasons. Nevertheless we describe all the steps for complete decoding here as an illustration.

*Description*: List of chain IDs.

*Example*:

Starting with the array of 8-bit unsigned integers:

```JSON
[ 0, 0, 0, 65, 0, 0, 0, 66, 0, 0, 89, 67 ]
```

Decoding the ASCII characters:

```JSON
[ "", "", "", "A", "", "", "", "B", "", "", "Y", "C" ]
```

Creating the list of chain IDs:

```JSON
[ "A", "B", "YC" ]
```


#### chainNameList

*Optional field*

*Type*: `Binary` data that is interpreted as an array of 8-bit unsigned integers representing ASCII characters.

*Decoding*: Same as for the `chainIdList` field.

*Description*: List of chain names. This field allows to specify an additional set of labels for chains. For example, it can be used to store both, the `label_asym_id` (in `chainIdList`) and the `auth_asym_id` (in `chainNameList`) from `mmCIF` files.


### Group data

The fields in the following sections hold group-related data.

The mmCIF format allows for so-called micro-heterogeneity on the group-level. For groups (residues) with micro-heterogeneity there are two or more entries given that have the same [sequence id](#sequenceidlist), [group id](#groupidlist) (and [insertion code](#inscodelist)) but are of a different [group type](#grouptypelist). The defining property is their identical sequence id.


#### groupList

*Required field*

*Type*: `List` of `groupType` objects with the following fields:

| Name             | Type              | Description                                                 |
|------------------|-------------------|-------------------------------------------------------------|
| atomChargeList   | [Array](#array)   | List of formal charges as [Integers](#integer)              |
| atomNameList     | [Array](#array)   | List of atom names, 0 to 5 character [Strings](#string)     |
| elementList      | [Array](#array)   | List of elements, 0 to 3 character [Strings](#string)       |
| bondIndexList    | [Array](#array)   | List of bonded atom indices, [Integers](#integer)           |
| bondOrderList    | [Array](#array)   | List of bond orders as [Integers](#integer) between 1 and 3 |
| groupName        | [String](#string) | The name of the group, 0 to 5 characters                    |
| singleLetterCode | [String](#string) | The single letter code, 1 character                         |
| chemCompType     | [String](#string) | The chemical component type                                 |


The element name must follow the IUPAC standard where only the first character is capitalized and the remaining ones are lower case, for instance `Cd` for Cadmium.

Two consecutive entries in `bondIndexList` representing indices of covalently bound atoms. The indices point into the `atomChargeList`, `atomNameList`, and `elementList` fields.

The `singleLetterCode` is the IUPAC single letter code for protein or DNA/RNA residues, otherwise the character 'X'.

*Description*: Common group (residue) data that is referenced via the `groupType` key by group entries.

*Vocabulary*: Known values for the groupType field `chemCompType` from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx_v40.dic/Items/_chem_comp.type.html) are `D-beta-peptide, C-gamma linking`, `D-gamma-peptide, C-delta linking`, `D-peptide COOH carboxy terminus`, `D-peptide NH3 amino terminus`, `D-peptide linking`, `D-saccharide`, `D-saccharide 1,4 and 1,4 linking`, `D-saccharide 1,4 and 1,6 linking`, `DNA OH 3 prime terminus`, `DNA OH 5 prime terminus`, `DNA linking`, `L-DNA linking`, `L-RNA linking`, `L-beta-peptide, C-gamma linking`, `L-gamma-peptide, C-delta linking`, `L-peptide COOH carboxy terminus`, `L-peptide NH3 amino terminus`, `L-peptide linking`, `L-saccharide`, `L-saccharide 1,4 and 1,4 linking`, `L-saccharide 1,4 and 1,6 linking`, `RNA OH 3 prime terminus`, `RNA OH 5 prime terminus`, `RNA linking`, `non-polymer`, `other`, `peptide linking`, `peptide-like`, `saccharide`.

*Example*:

```JSON
{
    "0": {
        "atomChargeList": [ 0, 0, 0, 0 ],
        "atomNameList": [ "N", "CA", "C", "O" ],
        "elementList": [ "N", "C", "C", "O" ],
        "bondIndexList": [ 1, 0, 2, 1, 3, 2 ],
        "bondOrderList": [ 1, 1, 2 ],
        "chemCompType": "PEPTIDE LINKING",
        "groupName": "GLY",
        "singleLetterCode": "G"
    }
}
```


#### groupTypeList

*Required field*

*Type*: `Binary` data that is interpreted as an array of 32-bit signed integers.

*Description*: List of pointers to `groupType` entries in `groupList` by their keys. One entry for each residue, thus the number of residues is equal to the length of the `groupTypeId` field.

*Example*:

In the following example there are 5 groups. The 1st, 4th and 5th reference the `groupType` with key `2`, the 2nd references key `0` and the third references key `1`.

```JSON
[ 2, 0, 1, 2, 2 ]
```


#### groupIdList

*Required field*

*Type*: `Binary` data that is interpreted as an array of 32-bit signed integers.

*Decoding*: First, run-length decode the input array of 32-bit signed integers into a second array of 32-bit signed integers. Finally apply delta decoding to the second array, which can be done in-place, to create the output array of 32-bit signed integers.

*Description*: List of group (residue) numbers. One entry for each group/residue.

*Example*:

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

*Type*: `Binary` data that is interpreted as an array of 8-bit signed integers.

*Description*: List of secondary structure assignments coded according to the following table, which shows the eight different types of secondary structure the [DSSP](https://dx.doi.org/10.1002%2Fbip.360221211) algorithm distinguishes. If the field is included there must be an entry for each group (residue) either in all models or only in the first model.

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

Starting with the array of 8-bit signed integers:

```JSON
[ 7, 7, 2, 2, 2, 2, 2, 2, 2, 7 ]
```


#### insCodeList

*Optional field*

*Type*: `Array` of `Integer` values.

*Decoding*: Run-length decode the input `Array` into an array of 8-bit unsigned integers representing ASCII characters.

*Description*: List of insertion codes, one for each group (residue).

*Example*:

Starting with the `Array`:

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


#### sequenceIdList

*Required field*

*Type*: `Binary` data that is interpreted as an array of 32-bit signed integers.

*Decoding*: First, run-length decode the input array of 32-bit signed integers into a second array of 32-bit signed integers. Finally apply delta decoding to the second array, which can be done in-place, to create the output array of 32-bit signed integers.

*Description*: List of indices that point into the `sequence` property of an entity object in the [entityList](entitylist) field that is associated with the chain the group belongs to (i.e. the index of the chain is included in the `chainIndexList` of the entity). There is one entry for each group (residue). It must be set to `-1` when a group entry has no associated entity (and thus no sequence), for example water molecules.

*Example*:

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


### Atom data

The fields in the following sections hold atom-related data.

The mmCIF format allows for alternate locations of atoms. Such atoms have multiple entries in the atom-level fields (including the fields in the [groupList](grouplist) entries). They can be identified and distinguished by their distinct values in the [altLocList](altloclist) field.


#### atomIdList

*Optional field*

*Type*: `Binary` data that is interpreted as an array of 32-bit signed integers.

*Decoding*: First, run-length decode the input array of 32-bit signed integers into a second array of 32-bit signed integers. Finally apply delta decoding to the second array, which can be done in-place, to create the output array of 32-bit signed integers.

*Description*: List of atom serial numbers. One entry for each atom.

*Example*:

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

*Type*: `Array` of `Integer` values.

*Decoding*: Run-length decode the input `Array` into an array of 8-bit unsigned integers representing ASCII characters.

*Description*: List of alternate location labels, one for each atom.

*Example*:

Starting with the `Array`:

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


#### bFactorBig bFactorSmall

*Optional fields*

*Type*: Two `Binary` data fields that are interpreted as array of 32-bit signed integers and array of 16-bit signed integers.

*Decoding*: First split-list delta decode the input array of 32-bit signed integers and the array of 16-bit signed integers into a second array of 32-bit signed integers. Finally integer decode the second array using `100` as the divisor to create an array of 32-bit floating-point values.

*Description*: List of atom B-factors in in Å^2. One entry for each atom.

*Example*:

Starting with the "big" array of 32-bit signed integers and the "small" array of 16-bit signed integers:

```JavaScript
[ 200, 3, 100, 2 ]   // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create an array of 32-bit signed integers:

```JSON
[ 200, 200, 202, 201, 301, 298, 303 ]
```

Applying integer decoding with a divisor of `100` to create an array of 32-bit floating-point values:

```JSON
[ 2.00, 2.00, 2.02, 2.01, 3.01, 2.98, 3.03 ]
```


#### xCoordBig xCoordSmall
#### yCoordBig yCoordSmall
#### zCoordBig zCoordSmall

*Required fields*

*Type*: Two `Binary` data fields that are interpreted as array of 32-bit signed integers and array of 16-bit signed integers.

*Decoding*: First split-list delta decode the input array of 32-bit signed integers and the array of 16-bit signed integers into a second array of 32-bit signed integers. Finally integer decode the second array using `1000` as the divisor to create an array of 32-bit floating-point values.

*Description*: List of x, y, and z atom coordinates, respectively, in Å. One entry for each atom and coordinate.

*Note*: To clarify, the data for each coordinate is stored in a separate pair of arrays.

*Example*:

Starting with the "big" array of 32-bit signed integers and the "small" array of 16-bit signed integers:

```JavaScript
[ 1200, 3, 100, 2 ]  // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create an array of 32-bit signed integers:

```JSON
[ 1200, 1200, 1202, 1201, 1301, 1298, 1303 ]
```

Applying integer decoding with a divisor of `1000` to create an array of 32-bit floating-point values:

```JSON
[ 1.000, 1.200, 1.202, 1.201, 1.301, 1.298, 1.303 ]
```


#### occupancyList

*Optional field*

*Description*: List of atom occupancies, one for each atom.

*Type*: `Binary` data that is interpreted as an array of 32-bit signed integers.

*Decoding*: First, run-length decode the input array of 32-bit signed integers into a second array of 32-bit signed integers. Finally apply integer decoding using `100` as the divisor to the second array to create a array of 32-bit floating-point values.

*Example*:

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

The following traversal pseudo code assumes that all fields have been decoded and specifically that the split-list delta encoded fields are decoded into fields named like in the following example: `xCoordBig`/`xCoordSmall` decode into `xCoordList`.

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
            print sequenceIdList[ groupIndex ]
            print groupTypeList[ groupIndex ]
            set group to groupList[ groupTypeList[ groupIndex ] ]
            print group.groupName
            print group.singleLetterCode
            print group.chemCompType
            set atomOffset to atomIndex
            set groupBondCount to group.bondIndexList.length / 2
            for i in 1 to groupBondCount
                print atomOffset + group.bondIndexList[ i * 2 ]      # atomIndex1
                print atomOffset + group.bondIndexList[ i * 2 + 1 ]  # atomIndex2
                print group.bondOrderList[ i ]
            set groupAtomCount to group.atomChargeList.length
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
                print group.atomChargeList[ i ]
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
