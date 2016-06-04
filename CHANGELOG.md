# Change Log
All notable changes to this project will be documented in this file, following the suggestions of [Keep a CHANGELOG](http://keepachangelog.com/). This project adheres to [Semantic Versioning](http://semver.org/).


## [Unreleased]
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

### Removed
- split-list delta encoding strategy


## v0.1.0 - 2016-04-26
### Added
- Initial release


[Unreleased]: https://github.com/rcsb/mmtf/compare/v0.1...HEAD