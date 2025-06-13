# Retrospectives
Retrospectives across public and open source DSE projects.

## Purpose
This repository centralizes retrospectives for public Schmdit DSE projects. This includes our open source work. These documents serve as a historical record for reference and offers centralization as some projects span multiple repositories.

## Background
The [Schmidt Center for Data Science and Environment](https://dse.berkeley.edu/) at the University of California Berkeley produces open source software where the organization serves as the maintainer of public code. While routine bugs or improvements do not require this level of analysis, we may have certain security or reliability incidents that benefit from documentation and debrief subject to the project-specific license. This information is captured in "retrospectives" which are files detailing those events, vulnerabilities, or bugs. We use the term retrospective instead of post-mortem to reflect that this may include documentation of events which did not lead to an actual incident like near misses.

## Usage
After making a report public, repositories should link to the report from relevant pull requests and commits. If necessary, these can be included as comments included after merge.

## Standards
Reports should be written in markdown. Different reports may have different needs so the format may be adapted as required. However, see prior reports for examples. At minimum, please ensure each report mentions the relevant CVEs, author(s), and project name. File names should follow the format `YYYY-MM-DD_Project_Incident.md`.

## Deployment
Reports merged to the `main` branch should be considered published. At this time, the GitHub repository itself serves as the location publication.

## License
All documents in this repository are released under the [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/) license.
