

# MMTF Specification

*INITIAL DRAFT* (to be replaced by the version number)

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally the format aims to be easy to understand and implement to facilitates its dissemination.


## Remarks


- Fields are either optional or required. Decoding libraries must handle both presence and absence of optional fields.
- Spatial data (e.g. coordinates, unit cell lengths) are given in angstrom.
- TODO debate custom fields



## Table of contents

* [Container](#Container)
* [Encodings](#encodings)
* [Types](#types)
* [Fields](#fields)
    * [Structure data](#structure-data)
    * [Model data](#model-data)
    * [Chain data](#chain-data)
    * [Group data](#group-data)
    * [Atom data](#atom-data)
* [Extra](#extra)
* [mmCIF](#mmCIF)


## Container

The [fields](#fields) in MMTF are stored in a binary container format. The top-level of the container contains the field names as keys and field data as values. To describe the layout of data in MMTF we use [JSON](http://www.json.org/) throughout this document. The following table lists all top level [fields](#fields), including their [type](#types) and whether they are required or optional.

| Name                     | Type                      | Required |
| -------------------------|---------------------------|:--------:|
| mmtfVersion              | String                    |    Y     |
| mmtfProducer             | String                    |    Y     |
| unitCell                 | Array                     |          |
| spaceGroup               | String                    |          |
| pdbId                    | String                    |          |
| title                    | String                    |          |
| bioAssembly              | Map                       |          |
| entityList               | Array                     |          |
| experimentalMethods      | Array                     |          |
| resolution               | 32-bit Float              |          |
| rFree                    | 32-bit Float              |          |
| rWork                    | 32-bit Float              |          |
| numBonds                 | 32-bit Unsigned Integer   |    Y     |
| numAtoms                 | 32-bit Unsigned Integer   |    Y     |
| groupMap                 | Map                       |    Y     |
| bondAtomList             | Uint32Array               |          |
| bondOrderList            | Uint8Array                |          |
| xCoordBig                | Int32Array                |    Y     |
| xCoordSmall              | Int16Array                |    Y     |
| yCoordBig                | Int32Array                |    Y     |
| yCoordSmall              | Int16Array                |    Y     |
| zCoordBig                | Int32Array                |    Y     |
| zCoordSmall              | Int16Array                |    Y     |
| bFactorBig               | Int32Array                |          |
| bFactorSmall             | Int16Array                |          |
| atomIdList               | Int32Array                |          |
| altLabelList             | Array                     |          |
| insCodeList              | Array                     |          |
| occList                  | Int32Array                |          |
| groupIdList              | Int32Array                |    Y     |
| groupTypeList            | Int32Array                |    Y     |
| secStructList            | Int8Array                 |          |
| chainIdList              | Uint8Array                |    Y     |
| chainNameList            | Uint8Array                |          |
| groupsPerChain           | Array                     |    Y     |
| chainsPerModel           | Array                     |    Y     |

The MessagePack format (version 5) is used as the binary container format of MMTF. The MessagePack [specification](https://github.com/msgpack/msgpack/blob/master/spec.md) describes the data types and the data layout. Encoding and decoding libraries for MessagePack are available in many languages, see the MessagePack [website](http://msgpack.org/).

While MessagePack supports `64-bit Unsigned/Signed Integers` they are forbidden in MMTF as they can not be represented natively in JavaScript. This is especially important for `userData` fields otherwise all MessagePack types are allowed (Note that the current version of the MMTF specification does not include `userData` fields but they may be added in the future).

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

- Special handling of lists with intermittent large delta values
- The list is split into two arrays to store the values
    - An array of 32-bit signed integers holds pairs of a large delta value and the number of subsequent small delta values
    - An array of 16-bit signed integers holds the small values

*Example*:

```
TODO
```


### Integer encoding

- Convert floating point numbers to integers by multiplying with a factor and discard everything after the decimal point.
- Depending on the multiplication factor this can change the precision but with a sufficiently large factor it is lossless.
- The Integers can then often be compressed with delta encoding which is the main motivation for it.

*Example*:

```
TODO
```


### Dictionary encoding

- Create a key-value store and use the key instead of repeating the value over and over again.
- Lists of keys can afterwards be compressed with delta and run-length encoding.

*Example*:

```
TODO
```


## Types

### String

An ASCII encoded 8-bit string of up to (2^32)-1 characters.


### 32-bit Float

A 32-bit floating point number.


### 32-bit Unsigned Integer

An unsigned 32-bit integer number.


### 32-bit Signed Integer

A signed 32-bit integer number.


### Map

A data structure of key-value pairs where each key is unique. Also known as "dictionary", "map", "hash" or "object". There can be up to (2^32)-1 pairs.


### Array

A list of elements that may be of different type. There can be up to (2^32)-1 elements.


### BinaryArray

A list of unsigned 8-bit integer numbers representing binary data. There can be up to (2^32)-1 numbers.


### TypedArray

Typed arrays are not directly supported by the `msgpack` format. However it is straightforward to work with arrays of simple numeric types by re-interpreting the binary data in a `BinaryArray`. For example, for a `Float32Array` groups of 4 bytes of a `BinaryArray` are interpreted as a 32-bit floating point numbers. Multi-byte types are always represented in big-endian format.


#### Uint8Array

A list of unsigned 8-bit integer numbers. There can be up to (2^32)-1 numbers.


#### Int8Array

A list of signed 8-bit integer numbers. There can be up to (2^32)-1 numbers.


#### Uint16Array

A list of unsigned 16-bit integer numbers. There can be up to (2^31)-2 numbers. Represented in big-endian format.


#### Int16Array

A list of signed 16-bit integer numbers. There can be up to (2^31)-2 numbers. Represented in big-endian format.


#### Uint32Array

A list of unsigned 32-bit integer numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.


#### Int32Array

A list of signed 32-bit integer numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.


#### Float32Array

A list of 32-bit floating point numbers. There can be up to (2^30)-4 numbers. Represented in big-endian format.



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


#### pdbId

*Optional field*

*Type*: `String` of four characters.

*Description*: The four character PDB ID.

*Examples*:

```JSON
"1CRN"
```

```JSON
"3pqr"
```


#### numAtoms

*Required field*

*Type*: `32-bit Unsigned Integer`.

*Description*: The overall number of atoms in the structure. This also includes atoms at alternate locations.

*Example*:

```JSON
1023
```


#### numBonds

*Required field*

*Type*: `32-bit Unsigned Integer`.

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

*Type*: `Array` of six `32-bit Float` values.

*Description*: List of six values defining the unit cell. The first three entries are the length of the sides `a`, `b`, and `c` in angstrom. The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.

*Example*:

```JSON
[ 10, 12, 30, 90, 90, 120 ]
```


#### bioAssembly

*Optional field*

*Type*: `Map` of `assembly` entries. An `assembly` entry holds
    - an array of transforms that contain a `chainIdList` and a 4x4 `transformation` matrix.
    - a `32-bit Signed Integer` named `macroMolecularSize`

*Description*: List of instructions on how to transform a list of chains to create (biological) assemblies.

*Example*:

```JSON
{
    "1": {
        "macroMolecularSize": 1,
        "transforms": [
            {
                "chainIdList": [ "A", "B", "..." ],
                "transformation": [
                    1, 0, 0, 0,
                    0, 1, 0, 0,
                    0, 0, 1, 0,
                    0, 0, 0, 1
                ]
            }
        ]
    }
}
```


#### entityList

*Optional field*

*Type*: `Array` of `entity` entries. An `entity` object has three fields, `chainIndexList` of type `Array`, `description` of type `String` and `type` of type `String`.

*Description*: List of unique molecular entities within the structure. The values in `chainIdList` must correspond with each of the chains in the top-level `chainIdList` field that represent instances of that entity. The entity `type` must be `polymer`, `non-polymer` or `water`. A `description` text can be included for each `entity`.

*Example*:

```JSON
[
    {
        "chainIndexList": [ 1, 2, 3 ],
        "description": "some polymer",
        "type": "polymer"
    },
    {
        "chainIndexList": [ 4 ],
        "description": "some ligand",
        "type": "non-polymer"
    }
]
```


#### resolution

*Optional field*

*Type*: `32-bit Float`.

*Description*: The experimental resolution in Angstrom. If not applicable either do not include the field or set to `-1`.

*Examples*:

```JSON
2.3
```

```JSON
-1
```


#### rFree

*Optional field*

*Type*: `32-bit Float`.

*Description*: The R-free value. If not applicable either do not include the field or set to `-1`.

*Examples*:

```JSON
0.203
```

```JSON
-1
```


#### rWork

*Optional field*

*Type*: `32-bit Float`.

*Description*: The R-work value. If not applicable either do not include the field or set to `-1`.

*Examples*:

```JSON
0.176
```

```JSON
-1
```


#### experimentalMethods

*Optional field*

*Type*: `Array` of `String`s.

*Description*: The list of experimental methods employed for structure determination.

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

*Type*: `Array` of `Unsigned 32-bit integer` numbers. The number of models is thus equal to the length of the `chainsPerModel` field.

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

*Type*: `Array` of `Unsigned 32-bit integer` numbers.

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

Starting with the `BinaryArray`/`Uint8Array`

```JSON
[ 0, 0, 0, 65, 0, 0, 0, 66, 0, 0, 89, 67 ]
```

Decoding the ASCII characters

```JSON
[ "", "", "", "A", "", "", "", "B", "", "", "Y", "C" ]
```

Creating the list of chain IDs

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
    - `atomCharges` is an `Array`  of `32-bit Signed Integers` holding the formal charges of each atom.
    - `atomInfo` is an `Array` of `String` pairs representing the element (0 to 3 characters) and the atom name (0 to 4 characters).
    - `bondIndices` is an `Array` of `32-bit Signed Integers` pairs representing indices of bonded atoms. The indices point to the `atomInfo`/`atomCharges` lists.
    - `bondOrders` is an `Array` of `32-bit Signed Integers` denoting the number of chemical bonds for each bond in `bondIndices`.
    - `chemCompType` is a `String` .
    - `groupName` is a `String` holding the of the name of the group (0 to 5 characters).
    - `singleLetterCode` is a `String` of length one representing the single letter code of protein or DNA/RNA residue, otherwise a question mark.

*Description*: Common group (residue) data that is referenced via the `groupType` key by group entries.

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


#### groupNumList

*Required field*

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply delta decoding to the second `Int32Array`, which can be done in-place, to create the output `Int32Array`.

*Description*: List of group (residue) numbers. One entry for each group/residue.

*Example*:

Starting with the `Int32Array`

```JSON
[ 1, 10, 2, 1, 1, 4 ]
```

Applying run-length decoding

```JSON
[ 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 1, 1, 1 ]
```

Applying delta decoding

```JSON
[ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 13, 14, 15, 16 ]
```


#### secStructList

*Optional field*

*Type*: `BinaryArray` that is interpreted as an `Int8Array`.

*Description*: List of secondary structure assignments coded according to the following table. If the field is included there must be an entry for each group (residue) either in all models or only in the first model.

| Code | Name         | Shorthand |
|-----:|--------------|:---------:|
|    0 | pi helix     |     i     |
|    1 | bend         |     s     |
|    2 | alpha helix  |     h     |
|    3 | extended     |     e     |
|    4 | 3-10 helix   |     g     |
|    5 | bridge       |     b     |
|    6 | turn         |     t     |
|    7 | coil         |     l     |
|   -1 | undefined    |     -     |

*Example*:

Starting with the `Int8Array`:

```JSON
[ 7, 7, 2, 2, 2, 2, 2, 2, 2, 7 ]
```

A decoding library may decide to provide the secondary structure assignments using the one letter shorthand:

```JSON
[ "l", "l", "h", "h", "h", "h", "h", "h", "h", "l" ]
```


### Atom data

#### atomIdList

*Optional field*

*Type*: `BinaryArray` that is interpreted as an `Int32Array`.

*Decoding*: First, run-length decode the input `Int32Array` into a second `Int32Array`. Finally apply delta decoding to the second `Int32Array`, which can be done in-place, to create the output `Int32Array`.

*Description*: List of atom serial numbers. One entry for each atom.

*Example*:

```
TODO
```


#### altLabelList

*Optional field*

*Type*: `Array` of alternating `String` and `32-bit Unsigned Integer` values.

*Decoding*: Run-length decode the input `Array` into an `Uint8Array` representing ASCII characters.

*Description*: List of alternate location identifiers, one for each atom.

*Example*:

```
TODO
```


#### insCodeList

*Optional field*

*Type*: `Array` of alternating `String` and `32-bit Unsigned Integer` values.

*Decoding*: Run-length decode the input `Array` into an `Uint8Array` representing ASCII characters.

*Description*: List of insertion codes, one for each atom.

*Example*:

```
TODO
```


#### bFactorBig, bFactorSmall

*Optional field*

- List of atom b-factors.
- One entry for each atom.
- Split-list delta encoded.
- Integer encoded with a multiplier of 100.
- Decodes into an array of 32-bit floats.
- Decoding instructions:
    1. Get 32-bit signed integers from 8-bit unsigned integers input.
    2. Apply split-list delta decoding to output from step 1.
    3. Apply integer decoding with a divisor of 100 to output from step 2.
    4. Return output from step 3.

*Example*:

```
TODO
```


#### xCoordBig & xCoordSmall, yCoordBig & yCoordSmall, zCoordBig & zCoordSmall

*Required field*

- List of x, y, and z atom coordinates.
- One entry for each atom and coordinate.
- Split-list delta encoded.
- Integer encoded with a multiplier of 1000.
- Decode into arrays of 32-bit floats representing coordinates in angstrom.
- Decoding instructions:
    1. Get 32-bit signed integers from 8-bit unsigned integers input.
    2. Apply split-list delta decoding to output from step 1.
    3. Apply integer decoding with a divisor of 1000 to output from step 2.
    4. Return output from step 3.

*Example*:

```
TODO
```


#### occList

*Optional field*

- Delta and run-length encoded.
- Integer encoded with a multiplier of 100.
- Decodes into an array of 32-bit floats.
- Decoding instructions:
    1. Get 32-bit signed integers from 8-bit unsigned integers input.
    2. Apply run-length decoding to output from step 1.
    3. Apply delta decoding to output from step 2.
    4. Apply integer decoding with a divisor of 100 to output from step 3.
    5. Return output from step 4.

*Example*:

```
TODO
```


## mmCIF

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
