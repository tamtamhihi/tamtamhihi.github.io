# README

## Installation

When cloning this repo on a new device:
1. Install Hugo v0.128.0. _(For some reason, the current usage of PaperMod in this site fails to compile with later Hugo versions)_

Determine OS and CPU information.
```
uname -a
```

Download the corresponding tarball from https://github.com/gohugoio/hugo/releases/tag/v0.128.0. for example:
```
curl -LO https://github.com/gohugoio/hugo/releases/download/v0.128.0/hugo_0.128.0_darwin-universal.tar.gz
```

Extract tarball, which will yield a binary `hugo`.
```
tar -xvzf hugo_0.128.0_darwin-universal.tar.gz
```

Move binary to `usr/local/bin` to give access to all users:
```
sudo mv hugo /usr/local/bin/hugo
```

Double check version:
```
hugo version
```


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
