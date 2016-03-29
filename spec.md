

# MMTF Sepcification

*INITIAL DRAFT* (to be replaced by the version number)

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally the format aims to be easy to understand and implement to facilitates its dissemination.


## Remarks

- Msgpack (v5) is used as the container format, see [msgpack spec](https://github.com/msgpack/msgpack/blob/master/spec.md).
- Encoded, binary fields are stored as big-endian when applicable i.e. for 16/32-bit un/signed integers and for 16/32/64 floats.
- For encoded fields, decoding instructions are provided below.
- 64-bit un/signed integers in custom fields are forbidden as they can not represented natively in JavaScript.
- Fields are either optional or required. Decoding libraries must handle both presence and absence of optional fields.
- Spatial data (e.g. coordinates, unit cell lengths) are given in angstrom.
- TODO debate custom fields



## Table of contents

* [Encodings](#encodings)
* [Types](#types)
* [Fields](#fields)
	* [Structure data](#structure-data)
	* [Model data](#model-data)
	* [Chain data](#chain-data)
	* [Group data](#group-data)
	* [Atom data](#atom-data)
* [Extra](#extra)



## Encodings

### Integer encoding

- Convert floating point numbers to integers by multiplying with a factor and discard everything after the decimal point.
- Depending on the multiplication factor this can change the precision but with a sufficiently large factor it is lossless.
- The Integers can then often be compressed with delta encoding which is the main motivation for it.


### Delta encoding

- For lists of numbers.
- Store differences (deltas) between numbers instead of the numbers themselves.
- When the deltas are smaller than the numbers themselves they can be packed to require less space.
- Lists that change by an identical amount for a range of consecutive values lend themselves to subsequent run-length encoding.


#### Split-list delta encoding

- Special handling of lists with intermittent large delta values
- The list is split into two arrays to store the values
	- An array of 32-bit signed integers holds pairs of a large delta value and the number of subsequent small delta values
	- An array of 16-bit signed integers holds the small values


### Run-length encoding

- For lists of values that support a equality comparison.
- Represent stretches of equal values by the value itself and the occurrence count.


### Dictionary encoding

- Create a key-value store and use the key instead of repeating the value over and over again.
- Lists of keys can afterwards be compressed with delta and run-length encoding.



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

Typed arrays are not directly supported by the `msgpack` format. However it is straightforward to work with arrays of simple numeric types by re-interpreting the binary data in a `BinaryArray`. For example, in a `Float32Array` the first 4 bytes of a `BinaryArray` are interpreted as a 32-bit floating point number.


#### Uint8Array

A list of unsigned 8-bit integer numbers. There can be up to (2^32)-1 numbers.


#### Int8Array

A list of signed 8-bit integer numbers. There can be up to (2^32)-1 numbers.


#### Uint16Array

A list of unsigned 16-bit integer numbers. There can be up to (2^31)-2 numbers.


#### Int16Array

A list of signed 16-bit integer numbers. There can be up to (2^31)-2 numbers.


#### Uint32Array

A list of unsigned 32-bit integer numbers. There can be up to (2^30)-4 numbers.


#### Int32Array

A list of signed 32-bit integer numbers. There can be up to (2^30)-4 numbers.


#### Float32Array

A list of 32-bit floating point numbers. There can be up to (2^30)-4 numbers.



## Fields

Top level fields and their type.

```
{
	"mmtfVersion": String,
    "mmtfProducer": String,

    "unitCell": Array,
    "spaceGroup": String,
    "pdbId": String,
    "title": String,

	"bioAssembly": Map,
    "entityList": Array,

    "experimentalMethods": Array,
    "resolution": 32-bit Float,
    "rFree": 32-bit Float,
    "rWork": 32-bit Float,

    "numBonds": 32-bit Unsigned Integer,
    "numAtoms": 32-bit Unsigned Integer,

    "groupMap": Map,

    "bondAtomList": Uint32Array,
    "bondOrderList": Uint8Array,

    "xCoordBig": Int32Array,
    "xCoordSmall": Int16Array,
    "yCoordBig": Int32Array,
    "yCoordSmall": Int16Array,
    "zCoordBig": Int32Array,
    "zCoordSmall": Int16Array,
    "bFactorBig": Int32Array,
    "bFactorSmall": Int16Array,
    "atomIdList": Int32Array,
    "altLabelList": Array,
    "insCodeList": Array,
    "occList": Int32Array,

    "groupIdList": Int32Array,
    "groupTypeList": Int32Array,
    "secStructList": Int8Array,

    "chainIdList": Uint8Array,
    "chainNameList": Uint8Array,
    "groupsPerChain": Array,

    "chainsPerModel": Array,
}
```


### Format data

#### mmtfVersion

*Required field*

Type: `String`

Description: The version number of the specification the file adheres to.

Examples:

```JSON
"0.1"
```

```JSON
"1.2"
```


#### mmtfProducer

*Required field*

Type: `String`

Description: The name and version of the software used to produce the file. For development versions it can be useful to also include the checksum of the commit. If the producer is not available set to `"NA"`.

Example:

```JSON
"RCSB-PDB Generator---version: 6b8635f8d319beea9cd7cc7f5dd2649578ac01a0"
```


### Structure data

#### title

*Optional field*

Type: `String`

Description: A short description of the structural data included in the file.

Example:

```JSON
"CRAMBIN"
```


#### pdbId

*Optional field*

Type: `String` of four characters

Description: The four character PDB ID.

Examples:

```JSON
"1CRN"
```

```JSON
"3pqr"
```


#### numAtoms

*Required field*

Type: `32-bit Unsigned Integer`

Description: The overall number of atoms in the structure. This also includes atoms at alternate locations.

Example:

```JSON
1023
```


#### numBonds

*Required field*

Type: `32-bit Unsigned Integer`

Description: The overall number of bonds. This number must reflect both the bonds given in `bondAtomList` and the bonds given in the `groupType` entries in `groupMap`.

Example:

```JSON
1142
```


#### spaceGroup

*Optional field*

Type: `String`

Description: The Hermann-Mauguin space-group symbol.

Example:

```JSON
"P 1 21 1"
```


#### unitCell

*Optional field*

Type: `Array` of six `32-bit Float` values

Description: Array of six values defining the unit cell. The first three entries are the length of the sides `a`, `b`, and `c` in angstrom. The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.

Example:

```JSON
[ 10, 12, 30, 90, 90, 120 ]
```


#### bioAssembly

*Optional field*

Type: `Map` of `assembly` entries.

- The `bioAssembly` field is a dictionary/object of `assembly` entries.
- An `assembly` entry holds
	- an array of transforms that contain a `chainIdList` and a 4x4 `transformation` matrix.
	- a 32-bit integer `macroMolecularSize`

Example

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

Type: `Array` of `entity` entries. An `entity` object has three fields, `chainIndexList` of type `Array()`, `description` of type `String` and `type` of type `String`.

Description: List of unique molecular entities within the structure. The values in `chainIdList` must correspond with each of the chains in the top-level `chainIdList` field that represent instances of that entity. The entity `type` must be `polymer`, `non-polymer` or `water`. A `description` text can be included for each `entity`.

Example:

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

Type: `32-bit Float`

Description: The experimental resolution in Angstrom. If not applicable either do not include the field or set to `-1`.

Examples:

```JSON
2.3
```

```JSON
-1
```


#### rFree

*Optional field*

Type: `32-bit Float`

Description: The R-free value. If not applicable either do not include the field or set to `-1`.

Examples:

```JSON
0.203
```

```JSON
-1
```


#### rWork

*Optional field*

Type: `32-bit Float`

Description: The R-work value. If not applicable either do not include the field or set to `-1`.

Examples:

```JSON
0.176
```

```JSON
-1
```


#### spaceGroup

*Optional field*

Type: `String`

Description: The Hermann-Mauguin space-group symbol.

Example:

```JSON
"P 1 21 1"
```


#### bondAtomList

*Optional field*

Type: `BinaryArray` that is interpreted as a `Uint32Array`

Description: Pairs of values represent indices of bonded atoms. The indices point to the [Atom data](#atom-data) arrays.

Example:

In the following example there are three bonds, one between the atoms with the indices 0 and 1, one between the atoms with the indices 0 and 2, as well as one between the atoms with the indices 2 and 4.

```JSON
[
	0, 1,
	0, 2,
	2, 4
]
```


#### bondOrderList

*Optional field* If it exists `bondAtomList` must also be present. However `bondAtomList` may exist without `bondOrderList`.

Type: `BinaryArray` that is interpreted as a `Uint8Array`, i.e. take as is.

Description: List of bond orders for bonds in `bondAtomList`.

Example:

In the following example there are bond orders given for three bonds. The first and third bond have a bond order of 1 while the second bond has a bond order of 2.

```JSON
[
	1, 2, 1
]
```


### Model data

#### chainsPerModel

*Required field*

Type: `Array` of `Unsigned 32-bit integer` numbers

Description: List of the number of chains in each model.

Example:

In the following example there are 2 models. The first model has 5 chains and the second model has 8 chains.

```JSON
[
	5, 8
]
```


### Chain data

#### groupsPerChain

*Required field*

Type: `Array` of `Unsigned 32-bit integer` numbers

Description: List of the number of groups (aka residues) in each chain.

Example:

In the following example there are 3 chains. The first chain has 73 groups, the second 59 and the third 1.

```JSON
[
	73, 59, 1
]
```


#### chainIdList

*Required field*

Type: `BinaryArray` that is interpreted as a `Uint8Array` representing ASCII characters.

Decoding: Groups of four consecutive ASCII characters create the list of chain IDs. Note that the decoding here is optional, a decoding library may choose to pass the `Uint8Array` on for performance reasons. Nevertheless we describe the all the steps for complete decoding here as an illustration.

Description: List of chain IDs.

Example:

Starting with the `BinaryArray`/`Uint8Array`

```JSON
[ 0, 0, 0, 65, 0, 0, 0, 66, 0, 0, 89, 67 ]
```

Decoding the ASCII characters

```JSON
[ "", "", "", "A", "", "", "", "B", "", "", "Y", "C" ]
```

Creating the list of chain IDs

```
[ "A", "B", "YC" ]
```


#### chainNameList

*Optional field*

Type: `BinaryArray` that is interpreted as a `Uint8Array` representing ASCII characters.

- List of chain names.
- Array of 8-bit unsigned integers with four bytes for each chain name.



### Group data

#### groupMap

*Required field*

- Dictionary of per-residue/group data.
- The `groupMap` is a dictionary/object of `groupType` entries.
- Each `groupType` entry contains the following fields:
	- `atomCharges` is a list of integers holding the formal charges of each atom.
	- `atomInfo` is a list of string pairs representing the element (0 to 3 characters) and the atom name (0 to 4 characters).
	- `bondIndices` is a list of integer pairs representing indices of bonded atoms. The indices point to the `atomInfo`/`atomCharges` arrays.
	- `bondOrders` is a list of integers denoting the number of chemical bonds for each bond in `bondIndices`.
	- `hetFlag` is a boolean denoting if the group is a not a standard protein, DNA or RNA residue.
	- `groupName` is string of the name of the group (0 to 5 characters).

Examples
```
{
	0: {
		atomCharges: [ 0, 0, 0, 0, 0, 0, 0, 0 ],
		atomInfo: [ "N", "N", "C", "CA", "C", "C", "O", "O", "C", "CB", "C", "CG", "S", "SD", "C", "CE" ],
		bondIndices: [ 1, 0, 2, 1, 4, 1, 3, 2, 5, 4, 6, 5, 7, 6 ],
		bondOrders: [ 1, 1, 1, 2, 1, 1, 1 ],
		hetFlag: false,
		groupName: "MET"
	}
}
```

Notes

- `groupMap` is a map instead of a list to may allow in the future having a global map within the decoder that is only supplemented with a sparse map in the file.


#### groupTypeList

*Required field*

- List of pointers to the groupMap dictionary.
- One entry for each residue.
- Currently an array of 32-bit signed integers.
- TODO can probably be an array of 16-bit unsigned integers.
- Decoding instructions:
	1. Get 32-bit signed integers from 8-bit unsigned integers input.
	2. Return output from step 1.


#### groupNumList

*Required field*

- List of group/residue numbers.
- One entry for each group/residue.
- Delta and run-length encoded.
- Decodes into an array of 32-bit signed integers.
- Note: must be signed to represent PDB data which includes negative group numbers.
- Decoding instructions:
	1. Get 32-bit signed integers from 8-bit unsigned integers input.
	2. Apply run-length decoding to output from step 1.
	3. Apply delta decoding to output from step 2.
	4. Return output from step 3.


#### secStructList

*Optional field*

- List of secondary structure codes.
- One entry per residue.
- Array of 8-bit signed integers.
- Encoding (DSSP shorthand in parentheses)
    - 0: pi helix (i)
    - 1: bend (s)
    - 2: alpha helix (h)
    - 3: extended (e)
    - 4: 3-10 helix (g)
    - 5: bridge (b)
    - 6: turn (t)
    - 7: coil (l)
    - -1: not defined/not available
- TODO how to handle multi-model structures?
- Decoding instructions:
	1. Get 8-bit signed integers from 8-bit unsigned integers input.
	2. Return output from step 1.


### Atom data

#### atomIdList

*Optional field*

- List of atom serial numbers.
- One entry for each atom.
- Delta and run-length encoded.
- Decodes into an array of 32-bit signed integers.
- Decoding instructions:
	1. Get 32-bit signed integers from 8-bit unsigned integers input.
	2. Apply run-length decoding to output from step 1.
	3. Apply delta decoding to output from step 2.
	4. Return output from step 3.


#### altLabelList

*Optional field*

- List of atom alternate location identifier.
- One entry for each atom.
- Run-length encoded.
- Decodes into an array of 8-bit unsigned integers representing ASCII characters.
- Decoding instructions:
	1. Apply run-length decoding to string list input.
	2. Return output from step 1.


#### insCodeList

*Optional field*

- List of atom insertion codes.
- One entry for each atom.
- Run-length encoded.
- Decodes into an array of 8-bit unsigned integers representing the insertion code character or null.
- Decoding instructions:
	1. Apply run-length decoding to string list input.
	2. Return output from step 1.


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



## Extra

Some useful values are not explicitly included as a field in the `msgpack` but can easily derived from other fields. Implementers of decoders are encouraged to provide the following fields through their API.


#### numResidues

- The number of residues is equal to the length of the `groupTypeId` field.


#### numChains

- The number of chains is equal to the length of the `groupsPerChain` field.


#### numModels

- The number of models is equal to the length of the `chainsPerModel` field.
