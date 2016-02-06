
The **m**acro**m**olecular **t**ransmission **f**ormat (MMTF) is a binary encoding of biological structures.

This repository holds the [specification](spec.md) of the format.

Various encoding/decoding libaries and services are available:

- [Java de/coding library](https://github.com/rcsb/codec-devel)
- [JavaScript decoding library](https://git.rcsb.org/Research/MMTF-Decoder-JavaScript)
- REST services (replace XXXX with a PDB ID)
	- http://132.249.213.68:8080/servemessagepack/XXXX
	- http://132.249.213.68:8080/servemessagecalpha/XXXX
	- http://132.249.213.68:8080/servemultimessage/
		- wget --post-data="1AQ1" http://132.249.213.68:8080/servemultimessage/
