[![Website](https://img.shields.io/badge/sqr--099-lsst.io-brightgreen.svg)](https://sqr-099.lsst.io)
[![CI](https://github.com/lsst-sqre/sqr-099/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-sqre/sqr-099/actions/workflows/ci.yaml)

# Breakdown of adaptations to the CADC TAP Service for the RSP

## SQR-099

This technote details the modifications made to the CADC TAP service for the Rubin Science Platform, covering both QServ and PostgreSQL implementations. 

We highlight which components of the upstream codebase we adapt, add implementations to, or replace, as well as breakdown  of whether these would be required assuming we move towards a more event-based architecture.

**Links:**

- Publication URL: https://sqr-099.lsst.io
- Alternative editions: https://sqr-099.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-099
- Build system: https://github.com/lsst-sqre/sqr-099/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-sqre/sqr-099
cd sqr-099
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://sqr-099.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-099.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
