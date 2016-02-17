

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



## Fields

### Format data

#### mmtfVersion

- Required field.
- String holding the version number of the specification the file adheres to.


#### mmtfProducer

- Required field.
- String holding the name and version of the software used to produce the file.
- For development versions it can be useful to also include the checksum of the commit.


### Structure data

#### numAtoms

- Required field.
- 32-bit unsigned integer holding the number of atoms.


#### numBonds

- Required field.
- 32-bit unsigned integer holding the number of bonds.


#### bioAssembly

- Optional field.
- The `bioAssembly` field is a dictionary/object of `assembly` entries.
- An `assembly` entry holds an array of transforms that contain a `chainId` list and a 4x4 `transformation` matrix.
- Layout example:
	```
	{
		1: {
			id: 1,
			macromolecularSize: 1,
			transforms: [
				{
					id: 1,
					chainId: [ "A", "B", "..." ],
					transformation: [
						1, 0, 0, 0,
						0, 1, 0, 0,
						0, 0, 1, 0,
						0, 0, 0, 1
					]
				}
			]
		},
		2: {
			// ...
		}
	}
	```
- TODO simplify layout


#### spaceGroup

- Optional field.
- String containing the Hermann-Mauguin space-group symbol.


#### unitCell

- Optional field.
- Array of six 32-bit float values defining the unit cell.
- The first three entries are the length of the sides `a`, `b`, and `c` in angstrom.
- The last three angles are the `alpha`, `beta`, and `gamma` angles in degree.
- Layout example:
	```
	[ 10, 12, 30, 90, 90, 120 ]
	```


#### bondAtomList

- Optional field.
- Array of integer pairs representing indices of bonded atoms. The indices point to the [Atom data](#atom-data) arrays.


#### bondOrderList

- Optional field.
- Array of integers denoting bond orders for bonds in `bondAtomList`.
- Run-length encoded.


### Model data

#### chainsPerModel

- Required field.
- List of number of chains in each model.


### Chain data

#### groupsPerChain

- Required field.
- List of number of groups/residues in each chain.


#### chainList

- Required field.
- List of chain names.
- Array of 8-bit unsigned integers with four bytes for each chain name.


### Group data

#### groupMap

- Required field.
- Dictionary of per-residue/group data.
- The `groupMap` is a dictionary/object of `groupType` entries.
- Each `groupType` entry contains the following fields:
	- `atomCharges` is an array of integers holding the formal charges of each atom.
	- `atomInfo` is an array of string pairs representing the element (0 to 3 characters) and the atom name (0 to 4 characters).
	- `bondIndices` is an array of integer pairs representing indices of bonded atoms. The indices point to the `atomInfo`/`atomCharges` arrays.
	- `bondOrders` is an array of integers denoting the number of chemical bonds for each bond in `bondIndices`.
	- `hetFlag` is a boolean denoting if the group is a not a standard protein, DNA or RNA residue.
	- `groupName` is string of the name of the group (0 to 5 characters).
- Layout example:
	```
	{
		0: {
			atomCharges: [ 0, 0, 0, 0, 0, 0, 0, 0 ],
			atomInfo: [ "N", "N", "C", "CA", "C", "C", "O", "O", "C", "CB", "C", "CG", "S", "SD", "C", "CE" ],
			bondIndices: [ 1, 0, 2, 1, 4, 1, 3, 2, 5, 4, 6, 5, 7, 6 ],
			bondOrders: [ 1, 1, 1, 2, 1, 1, 1 ],
			hetFlag: false,
			groupName: "MET"
		},
		1: {
			// ...
		}
	}
	```


#### groupTypeList

- Required field.
- List of pointers to the groupMap dictionary.
- One entry for each residue.
- Currently an array of 32-bit signed integers.
- TODO can probably be an array of 16-bit unsigned integers.
- Decoding instructions:
	1. Get 32-bit signed integers from 8-bit unsigned integers input.
	2. Return output from step 1.


#### groupNumList

- Required field.
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

- Optional field.
- List of secondary structure codes.
- One entry per residue.
- Array of 8-bit signed integers.
- TODO how to handle multi-model structures?
- Decoding instructions:
	1. Get 8-bit signed integers from 8-bit unsigned integers input.
	2. Return output from step 1.


### Atom data

#### atomIdList

- Optional field.
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

- Optional field.
- List of atom alternate location identifier.
- One entry for each atom.
- Run-length encoded.
- Decodes into an array of 8-bit unsigned integers representing ASCII characters.
- Decoding instructions:
	1. Apply run-length decoding to string list input.
	2. Return output from step 1.


#### insCodeList

- Optional field.
- List of atom insertion codes.
- One entry for each atom.
- Run-length encoded.
- Decodes into an array of 8-bit unsigned integers representing the insertion code character or null.
- Decoding instructions:
	1. Apply run-length decoding to string list input.
	2. Return output from step 1.


#### bFactorBig, bFactorSmall

- Optional field.
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

- Required field.
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

- Optional field.
- Delta and run-length encoded.
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
