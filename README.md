# README

## Installation

When cloning this repo on a new device:
1. Install Hugo.

2. Clone PaperMod submodule.
```
git submodule update --init --merge
```

To update submodule to latest version:
```
git submodule update --remote --merge
```

## Build

To build the website locally (available at localhost:1313)
```
hugo server
```

Must do one final build before pushing to master (otherwise hyperlinks will be broken)
```
hugo
```
