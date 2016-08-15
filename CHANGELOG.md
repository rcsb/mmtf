# Change Log
All notable changes to this project will be documented in this file, following the suggestions of [Keep a CHANGELOG](http://keepachangelog.com/). This project adheres to [Semantic Versioning](http://semver.org/).


## [v1.0] - 2016-08-15
### Added
- explanation for when types 12-15 are useful
- ncsOperatorList field
- use of "array" when describing fields of type Array


## [v0.2] - 2016-06-08
### Added
- WIP: faq.md
- numGroups, numChains, numModels fields
- recursive indexing encoding strategy
- 12 byte header for binary fields holding encoding type, encoded list size and encoding parameters
- the encoding strategy is now included in the header of binary fields instead of being hardcoded to the name of a field
- bioAssemblyList[].name field

### Changed
- renamed atomChargeList to formalChargeList in groupType objects
- allow quadruple bonds (bondOrder = 4)
- allow '?' as the singleLetterCode for non-polymer groups

### Removed
- split-list delta encoding strategy


## v0.1.0 - 2016-04-26
### Added
- Initial release


[v1.0]: https://github.com/rcsb/mmtf/compare/v0.2...v1.0
[v0.2]: https://github.com/rcsb/mmtf/compare/v0.1...v0.2