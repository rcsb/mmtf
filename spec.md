

# MMTF Specification

*INITIAL DRAFT* (to be replaced by the version number)

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally the format aims to be easy to understand and implement to facilitates its dissemination.


## Table of contents

* [Container](#container)
* [Encodings](#encodings)
* [Types](#types)
* [Fields](#fields)
    * [Format data](#format-data)
    * [Structure data](#structure-data)
    * [Model data](#model-data)
    * [Chain data](#chain-data)
    * [Group data](#group-data)
    * [Atom data](#atom-data)
* [Extra](#extra)
* [mmCIF](#mmcif)


## Container

The [fields](#fields) in MMTF are stored in a binary container format. The top-level of the container contains the field names as keys and field data as values. To describe the layout of data in MMTF we use [JSON](http://www.json.org/) throughout this document. The following table lists all top level [fields](#fields), including their [type](#types) and whether they are required or optional.

| Name                                        | Type                        | Required |
|---------------------------------------------|-----------------------------|:--------:|
| [mmtfVersion](#mmtfversion)                 | [String](#string)           |    Y     |
| [mmtfProducer](#mmtfproducer)               | [String](#string)           |    Y     |
| [unitCell](#unitcell)                       | [Array](#array)             |          |
| [spaceGroup](#spacegroup)                   | [String](#string)           |          |
| [structureId](#structureid)                 | [String](#string)           |          |
| [title](#title)                             | [String](#string)           |          |
| [date](#date)                               | [String](#string)           |          |
| [bioAssemblyList](#bioassemblylist)         | [Array](#array)             |          |
| [entityList](#entitylist)                   | [Array](#array)             |          |
| [experimentalMethods](#experimentalmethods) | [Array](#array)             |          |
| [resolution](#resolution)                   | [Float32](#float32)         |          |
| [rFree](#rfree)                             | [Float32](#float32)         |          |
| [rWork](#rwork)                             | [Float32](#float32)         |          |
| [numBonds](#numbonds)                       | [Uint32](#uint32)           |    Y     |
| [numAtoms](#numatoms)                       | [Uint32](#uint32)           |    Y     |
| [groupMap](#groupmap)                       | [Map](#map)                 |    Y     |
| [bondAtomList](#bondatomlist)               | [Uint32Array](#uint32array) |          |
| [bondOrderList](#bondorderlist)             | [Uint8Array](#uint8array)   |          |
| [xCoordBig](#xcoordbig-xcoordsmall)         | [Int32Array](#int32array)   |    Y     |
| [xCoordSmall](#xcoordbig-xcoordsmall)       | [Int16Array](#int16array)   |    Y     |
| [yCoordBig](#ycoordbig-ycoordsmall)         | [Int32Array](#int32array)   |    Y     |
| [yCoordSmall](#ycoordbig-ycoordsmall)       | [Int16Array](#int16array)   |    Y     |
| [zCoordBig](#zcoordbig-zcoordsmall)         | [Int32Array](#int32array)   |    Y     |
| [zCoordSmall](#zcoordbig-zcoordsmall)       | [Int16Array](#int16array)   |    Y     |
| [bFactorBig](#bfactorbig-bfactorsmall)      | [Int32Array](#int32array)   |          |
| [bFactorSmall](#bfactorbig-bfactorsmall)    | [Int16Array](#int16array)   |          |
| [atomIdList](#atomidlist)                   | [Int32Array](#int32array)   |          |
| [altLocList](#altloclist)                   | [Array](#array)             |          |
| [occupancyList](#occupancylist)             | [Int32Array](#int32array)   |          |
| [groupIdList](#groupidlist)                 | [Int32Array](#int32array)   |    Y     |
| [groupTypeList](#grouptypelist)             | [Int32Array](#int32array)   |    Y     |
| [secStructList](#secstructlist)             | [Int8Array](#)              |          |
| [insCodeList](#inscodelist)                 | [Array](#array)             |          |
| [sequenceIdList](#sequenceidlist)           | [Int32Array](#int32array)   |          |
| [chainIdList](#chainidlist)                 | [Uint8Array](#uint8array)   |    Y     |
| [chainNameList](#chainnamelist)             | [Uint8Array](#uint8array)   |          |
| [groupsPerChain](#groupsperchain)           | [Array](#array)             |    Y     |
| [chainsPerModel](#chainspermodel)           | [Array](#array)             |    Y     |

The MessagePack format (version 5) is used as the binary container format of MMTF. The MessagePack [specification](https://github.com/msgpack/msgpack/blob/master/spec.md) describes the data types and the data layout. Encoding and decoding libraries for MessagePack are available in many languages, see the MessagePack [website](http://msgpack.org/).

While MessagePack supports 64-bit Unsigned/Signed Integers they are forbidden in MMTF as they can not be represented natively in JavaScript. This is especially important for `userData` fields otherwise all MessagePack types are allowed (Note that the current version of the MMTF specification does not include `userData` fields but they may be added in the future).

The first step of decoding MMTF is decoding the MessagePack encoded container. Many of the resulting MMTF fields do not need to be decoded any further. However, to allow for custom compression some fields are given as binary data and must be decoded using the strategies described below.


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

Split-list delta encoding is an adjusted delta encoding to handle lists with some intermittent large delta values. The list is split into two arrays, called "big" and "small". The "big" `Int32Array` holds pairs of a large delta value (>=2^15) and the number of subsequent small delta values (<2^15). The "small" `Int16Array` holds the small values, that is, the values fitting into a 16-bit Signed Integer.

*Example*:

Starting with the "big" `Int32Array` and the "small `Int16Array`":

```JavaScript
[ 1200, 3, 100, 2 ]  // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create a `Int32Array`:

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


## Types

### String

An ASCII encoded 8-bit string of up to (2^32)-1 characters.


### Float32

A 32-bit Floating Point number.


### Uint32

An Unsigned 32-bit Integer number.


### Int32

A Signed 32-bit Integer number.


### Map

A data structure of key-value pairs where each key is unique. Also known as "dictionary", "map", "hash" or "object". There can be up to (2^32)-1 pairs.


### Array

A list of elements that may be of different type. There can be up to (2^32)-1 elements.


### BinaryArray

A list of unsigned 8-bit integer numbers representing binary data. There can be up to (2^32)-1 numbers.


### TypedArray

Typed arrays are not directly supported by the `msgpack` format. However it is straightforward to work with arrays of simple numeric types by re-interpreting the binary data in a `BinaryArray`. For example, for a `Float32Array` groups of 4 bytes of a `BinaryArray` are interpreted as a 32-bit floating point numbers. Multi-byte types are always represented in big-endian format.


#### Uint8Array

A list of Unsigned 8-bit Integer numbers. There can be up to (2^32)-1 numbers.


#### Int8Array

A list of Signed 8-bit Integer numbers. There can be up to (2^32)-1 numbers.


#### Uint16Array

A list of Unsigned 16-bit Integer numbers. There can be up to (2^31)-2 numbers. Represented in big-endian format.


#### Int16Array

A list of Signed 16-bit Integer numbers. There can be up to (2^31)-2 numbers. Represented in big-endian format.


#### Uint32Array

A list of Unsigned 32-bit Integer numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.


#### Int32Array

A list of Signed 32-bit Integer numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.


#### Float32Array

A list of 32-bit Floating Point numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.



## Fields

### Format data

#### mmtfVersion

*Required field*

*Type*: `String`

*Description*: The version number of the specification the file adheres to.

*Examples*:

```JSON
"0.1"
```

```JSON
"1.2"
```


#### mmtfProducer

*Required field*

*Type*: `String`.

*Description*: The name and version of the software used to produce the file. For development versions it can be useful to also include the checksum of the commit. If the producer is not available set to `"NA"`.

*Example*:

```JSON
"RCSB-PDB Generator---version: 6b8635f8d319beea9cd7cc7f5dd2649578ac01a0"
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


#### date

*Optional field*

*Type*: `String` with the format `YYYY-MM-DD`, where `YYYY` stands for the year in the Gregorian calendar, `MM` is the month of the year between 01 (January) and 12 (December), and `DD` is the day of the month between 01 and 31.

*Description*: A date that relates to the structure, e.g. the date of release or creation.

*Example*:

For example, the second day of October in the year 2005 is written as:

```JSON
"2005-10-02"
```


#### numAtoms

*Required field*

*Type*: `Uint32`.

*Description*: The overall number of atoms in the structure. This also includes atoms at alternate locations.

*Example*:

```JSON
1023
```


#### numBonds

*Required field*

*Type*: `Uint32`.

*Description*: The overall number of bonds. This number must reflect both the bonds given in `bondAtomList` and the bonds given in the `groupType` entries in `groupMap`.

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

*Type*: `Array` of six `Float32` values.

*Description*: List of six values defining the unit cell. The first three entries are the length of the sides `a`, `b`, and `c` in angstrom. The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.

*Example*:

```JSON
[ 10.0, 12.0, 30.0, 90.0, 90.0, 120.0 ]
```


#### bioAssemblyList

*Optional field*

*Type*: `Array` of `assembly` entries. An `assembly` entry holds
    - an array of transforms that contain a `chainIndexList` and a 4x4 `transformation` matrix.

*Description*: List of instructions on how to transform coordinates for a list of chains to create (biological) assemblies.

*Example*:

```JSON
[
    {
        "transforms": [
            {
                "chainIndexList": [ 0, 1, 2 ],
                "transformation": [
                    1.0, 0.0, 0.0, 0.0,
                    0.0, 1.0, 0.0, 0.0,
                    0.0, 0.0, 1.0, 0.0,
                    0.0, 0.0, 0.0, 1.0
                ]
            }
        ]
    }
]
```


#### entityList

*Optional field*

*Type*: `Array` of `entity` entries. An `entity` object has four fields, `chainIndexList` of type `Array`, `description` of type `String`, `type` of type `String` and `sequence` of type `String`.

*Description*: List of unique molecular entities within the structure. The indices in `chainIndexList` point to chain data in e.g. the top-level `chainIdList` field, each entry represents an instance of that entity. The entity `type` must be `polymer`, `non-polymer` or `water`. A `description` text can be included for each `entity`.

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

*Type*: `Float32`.

*Description*: The experimental resolution in Angstrom. If not applicable do not include the field.

*Examples*:

```JSON
2.3
```


#### rFree

*Optional field*

*Type*: `Float32`.

*Description*: The R-free value. If not applicable do not include the field.

*Examples*:

```JSON
0.203
```


#### rWork

*Optional field*

*Type*: `Float32`.

*Description*: The R-work value. If not applicable do not include the field.

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

*Type*: `BinaryArray` that is interpreted as a `Uint32Array`.

*Description*: Pairs of values represent indices of bonded atoms. The indices point to the [Atom data](#atom-data) arrays.

*Example*:

In the following example there are three bonds, one between the atoms with the indices 0 and 1, one between the atoms with the indices 0 and 2, as well as one between the atoms with the indices 2 and 4.

```JSON
[ 0, 1, 0, 2, 2, 4 ]
```


#### bondOrderList

*Optional field* If it exists `bondAtomList` must also be present. However `bondAtomList` may exist without `bondOrderList`.

*Type*: `BinaryArray` that is interpreted as a `Uint8Array`, i.e. take as is.

*Description*: List of bond orders for bonds in `bondAtomList`.

*Example*:

In the following example there are bond orders given for three bonds. The first and third bond have a bond order of 1 while the second bond has a bond order of 2.

```JSON
[ 1, 2, 1 ]
```


### Model data

The number of models in a structure is equal to the length of the `chainsPerModel` field. The `chainsPerModel` field also defines which chains belong to each model.


#### chainsPerModel

*Required field*

*Type*: `Array` of `Uint32` numbers. The number of models is thus equal to the length of the `chainsPerModel` field.

*Description*: List of the number of chains in each model.

*Example*:

In the following example there are 2 models. The first model has 5 chains and the second model has 8 chains. This also means that the chains with indices 0 to 4 belong to the first model and that the chains with indices 5 to 12 belong to the second model.

```JSON
[ 5, 8 ]
```


### Chain data

The number of chains in a structure is equal to the length of the `groupsPerChain` field. The `groupsPerChain` field also defines which groups belong to each chain.


#### groupsPerChain

*Required field*

*Type*: `Array` of `Uint32` numbers.

*Description*: List of the number of groups (aka residues) in each chain. The number of chains is thus equal to the length of the `groupsPerChain` field.

*Example*:

In the following example there are 3 chains. The first chain has 73 groups, the second 59 and the third 1. This also means that the groups with indices 0 to 72 belong to the first chain, groups with indices 73 to 131 to the second chain and the group with index 132 to the third chain.

```JSON
[ 73, 59, 1 ]
```


#### chainIdList

*Required field*

*Type*: `BinaryArray` that is interpreted as an `Uint8Array` representing ASCII characters.

*Decoding*: Groups of four consecutive ASCII characters create the list of chain IDs. Note that the decoding here is optional, a decoding library may choose to pass the `Uint8Array` on for performance reasons. Nevertheless we describe all the steps for complete decoding here as an illustration.

*Description*: List of chain IDs. The IDs here are used to reference the chains from other fields.

*Example*:

Starting with the `BinaryArray`/`Uint8Array`:

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

*Type*: `BinaryArray` that is interpreted as an `Uint8Array` representing ASCII characters.

*Decoding*: Same as for the `chainIdList` field.

*Description*: List of chain names. This field allows to specify an additional set of labels for chains. For example, it can be used to store both, the `label_asym_id` (in `chainIdList`) and the `auth_asym_id` (in `chainNameList`) from `mmCIF` files.


### Group data

#### groupMap

*Required field*

*Type*: `Map` of `groupType` entries. A `groupType` object has the following fields:

- `atomCharges` is an `Array`  of `Int32` holding the formal charges of each atom.
- `atomInfo` is an `Array` of `String` pairs representing the element (0 to 3 characters) and the atom name (0 to 4 characters).
- `bondIndices` is an `Array` of `Int32` pairs representing indices of bonded atoms. The indices point to the `atomInfo`/`atomCharges` lists.
- `bondOrders` is an `Array` of `Int32` denoting the number of chemical bonds for each bond in `bondIndices`.
- `chemCompType` is a `String` .
- `groupName` is a `String` holding the of the name of the group (0 to 5 characters).
- `singleLetterCode` is a `String` of length one representing the single letter code of protein or DNA/RNA residue, otherwise a question mark.

*Description*: Common group (residue) data that is referenced via the `groupType` key by group entries.

*Vocabulary*: Known values for the groupType field `chemCompType` from the [mmCIF dictionary](http://mmcif.wwpdb.org/dictionaries/mmcif_pdbx_v40.dic/Items/_chem_comp.type.html) are `D-beta-peptide, C-gamma linking`, `D-gamma-peptide, C-delta linking`, `D-peptide COOH carboxy terminus`, `D-peptide NH3 amino terminus`, `D-peptide linking`, `D-saccharide`, `D-saccharide 1,4 and 1,4 linking`, `D-saccharide 1,4 and 1,6 linking`, `DNA OH 3 prime terminus`, `DNA OH 5 prime terminus`, `DNA linking`, `L-DNA linking`, `L-RNA linking`, `L-beta-peptide, C-gamma linking`, `L-gamma-peptide, C-delta linking`, `L-peptide COOH carboxy terminus`, `L-peptide NH3 amino terminus`, `L-peptide linking`, `L-saccharide`, `L-saccharide 1,4 and 1,4 linking`, `L-saccharide 1,4 and 1,6 linking`, `RNA OH 3 prime terminus`, `RNA OH 5 prime terminus`, `RNA linking`, `non-polymer`, `other`, `peptide linking`, `peptide-like`, `saccharide`.

*Note*: The `groupMap` field is a `Map` instead of an `Array` to allow having (in the future) a global `Map` within the decoder that is only supplemented with a sparse map in the file.

*Example*:

```JSON
{
    "0": {
        "atomCharges": [ 0, 0, 0, 0 ],
        "atomInfo": [ "N", "N", "C", "CA", "C", "C", "O", "O" ],
        "bondIndices": [ 1, 0, 2, 1, 3, 2 ],
        "bondOrders": [ 1, 1, 2 ],
        "chemCompType": "PEPTIDE LINKING",
        "groupName": "GLY",
        "singleLetterCode": "G"
    }
}
```


#### groupTypeList

*Required field*

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Description*: List of pointers to `groupType` entries in `groupMap` by their keys. One entry for each residue, thus the number of residues is equal to the length of the `groupTypeId` field.

*Example*:

In the following example there are 5 groups. The 1st, 4th and 5th reference the `groupType` with key `2`, the 2nd references key `0` and the third references key `1`.

```JSON
[ 2, 0, 1, 2, 2 ]
```


#### groupIdList

*Required field*

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply delta decoding to the second `Int32Array`, which can be done in-place, to create the output `Int32Array`.

*Description*: List of group (residue) numbers. One entry for each group/residue.

*Example*:

Starting with the `Int32Array`:

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

*Type*: `BinaryArray` that is interpreted as an `Int8Array`.

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

Starting with the `Int8Array`:

```JSON
[ 7, 7, 2, 2, 2, 2, 2, 2, 2, 7 ]
```


#### insCodeList

*Optional field*

*Type*: `Array` of `Unit32` values.

*Decoding*: Run-length decode the input `Array` into an `Uint8Array` representing ASCII characters.

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

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply delta decoding to the second `Int32Array`, which can be done in-place, to create the output `Int32Array`.

*Description*: List of sequence indices that point into the sequence `String` of the entity (from the `entityList` field) associated with the group. One entry for each group (residue). Set to `-1` when a group entry is has no associated entity/sequence given, for example water molecules.

*Example*:

Starting with the `Int32Array`:

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

#### atomIdList

*Optional field*

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply delta decoding to the second `Int32Array`, which can be done in-place, to create the output `Int32Array`.

*Description*: List of atom serial numbers. One entry for each atom.

*Example*:

Starting with the `Int32Array`:

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

*Type*: `Array` of `Uint32` values.

*Decoding*: Run-length decode the input `Array` into an `Uint8Array` representing ASCII characters.

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

*Type*: Two `BinaryArray`s that are interpreted as `Int32Array` and `Int16Array`.

*Decoding*: First split-list delta decode the input `Int32Array` and `Int16Array` into a second `Int32Array`. Finally integer decode the second `Int32Array` using `100` as the divisor to create a `Float32Array`.

*Description*: List of atom b-factors in in angstrom^2. One entry for each atom.

*Example*:

Starting with the "big" `Int32Array` and the "small `Int16Array`":

```JavaScript
[ 200, 3, 100, 2 ]   // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create a `Int32Array`:

```JSON
[ 200, 200, 202, 201, 301, 298, 303 ]
```

Applying integer decoding with a divisor of `100` to create a `Float32Array`:

```JSON
[ 2.00, 2.00, 2.02, 2.01, 3.01, 2.98, 3.03 ]
```


#### xCoordBig xCoordSmall
#### yCoordBig yCoordSmall
#### zCoordBig zCoordSmall

*Required fields*

*Type*: Two `BinaryArray`s that are interpreted as `Int32Array` and `Int16Array`.

*Decoding*: First split-list delta decode the input `Int32Array` and `Int16Array` into a second `Int32Array`. Finally integer decode the second `Int32Array` using `1000` as the divisor to create a `Float32Array`.

*Description*: List of x, y, and z atom coordinates, respectively, in angstrom. One entry for each atom and coordinate.

*Note*: To clarify, the data for each coordinate is stored in a separate pair of `Int32Array` and `Int16Array` fields.

*Example*:

Starting with the "big" `Int32Array` and the "small `Int16Array`":

```JavaScript
[ 1200, 3, 100, 2 ]  // big
[ 0, 2, -1, -3, 5 ]  // small
```

Applying split-list delta decoding to create a `Int32Array`:

```JSON
[ 1200, 1200, 1202, 1201, 1301, 1298, 1303 ]
```

Applying integer decoding with a divisor of `1000` to create a `Float32Array`:

```JSON
[ 1.000, 1.200, 1.202, 1.201, 1.301, 1.298, 1.303 ]
```


#### occupancyList

*Optional field*

*Description*: List of atom occupancies, one for each atom.

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply integer decoding using `100` as the divisor to the second `Int32Array` to create a `Float32Array`.

*Example*:

Starting with the `Int32Array`:

```JSON
[ 100, 4, 50, 2 ]
```

Applying run-length decoding:

```JSON
[ 100, 100, 100, 100, 50, 50 ]
```

Applying integer decoding with a divisor of `100` to create a `Float32Array`:

```JSON
[ 1.00, 1.00, 1.00, 1.00, 0.50, 0.50 ]
```


## mmCIF

TODO

This section describes how (and what) data from mmCIF files (including associated data from CCD and BIRD files) can be stored in MMTF.


* Entity data
    * Full construct sequences
* Model data
    * ?
* Chain data
    * `label_asym_id` and `label_auth_id`
* Group/Residue data
    * Micro-heterogeneity residues
* Atom data
    * Alternate location atoms
* Bond data
    * Bonds from CCD
