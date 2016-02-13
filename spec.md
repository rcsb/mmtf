

# MMTF Sepcification

The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures. It includes the coordinates, the topology and associated data. Pronounced goals are a reduced file size for efficient transmission over the Internet or from hard disk to memory and fast decoding/parsing speed. Additionally the format aims to be easy to understand and implement to facilitates its dissemination.



## Remarks

- Msgpack (v5) as container format, see [msgpack spec](https://github.com/msgpack/msgpack/blob/master/spec.md).
- Encoded, binary fields are stored as big-endian when applicable (16/32-bit un/signed integers, 16/32/64 floats).
- 64-bit un/signed integers in custom fields are forbidden as they can not represented natively in JavaScript.
- Some fields are optional.



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

### Structure data

#### numAtoms

- Integer with the number of atoms


#### numBonds - not for now

- Integer with the number of bonds
- TODO not available yet, currently calculated from data in `resOrder` and `groupMap`.



#### bioAssembly

- Biological assembly data dumped from `biojava`
- TODO evaluate layout
- TODO describe layout


#### spaceGroup

- String containing the space group name


#### unitCell

- Array of six values defining the unit cell
- The first three entries are the length of the sides a, b, and c
- The last three angles are the alpha, beta, and gamma angles



### Model data

#### chainsPerModel

- List of number of chains in each model


### Chain data

#### groupsPerChain

- List of number of groups/residues in each chain


#### chainList

- List of chain names
- 8-bit unsigned integer array with four bytes for each chain name


### Group data

#### groupMap

- Dictionary of per-residue/group data
- TODO describe layout


#### resOrder => groupTypeList

- List of pointers to the groupMap dictionary
- One entry for each residue
- Currently a 32-bit signed integer array
- TODO can probably be a 8-bit or 16-bit unsigned integer array


#### _atom_site_auth_seq_id => resnumList

- List of group/residue numbers
- One entry for each group/residue
- Delta and run-length encoded
- Decodes into 32-bit signed integer array


#### secStruct => secStructList

- List of secondary structure codes
- One entry per residue
- 8-bit signed integer array
- TODO how to handle multi-model structures?


#### _atom_site_label_entity_poly_seq_num => remove

- ??? identical to label_seq_id mmcif field?
- TODO maybe remove or optional
- TODO currently not decoded in JS decoder


### Atom data

#### _atom_site_id => atomIdList

- List of atom serial numbers
- One entry for each atom
- Delta and run-length encoded
- Decodes into 32-bit signed integer array


#### _atom_site_label_alt_id => altLabelList

- List of atom alternate location identifier
- One entry for each atom
- Run-length encoded
- Decodes into 8-bit unsigned integer array representing ASCII characters


#### _atom_site_pdbx_PDB_ins_code => insCodeList

- List of atom insertion codes
- One entry for each atom
- Run-length encoded
- Decodes into a 8-bit unsigned integer array representing the insertion code character or null
- TODO currently not decoded


#### b_factor_big & b_factor_small => bFactorBig, bFactorSmall

- List of atom b-factors
- One entry for each atom
- Split-list delta encoded


#### cartn_x_big & cartn_x_small, cartn_y_big & cartn_y_small, cartn_z_big & cartn_z_small => xCoordSmall, xCoordBig

- List of x, y, and z atom coordinates
- One entry for each atom and coordinate
- Split-list delta encoded
- Decode into 32-bit float arrays


#### occupancy => occList

- Delta and run-length encoded
- Decodes into 32-bit float array
- TODO currently not decoded



## Extra

Some useful values are not explicitly included as a field in the `msgpack` but can easily derived from other fields.

- Residue count is length of `resOrder` field.
- Chain count is length of `groupsPerChain` field.
- Model count is length of `chainsPerModel` field.
