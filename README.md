# DVLA Engioneering

[https://dvla.github.io/](https://dvla.github.io/)

## Installing Hugo

### MacOS

```bash
brew install hugo
```

## Instructions for viewing

This module contains a theme that is a submodule so cloning is recommended with this flag

```bash
git clone --recurse-submodules $repo-url
```

If the repo is already cloned then you will need to execute the following command to populate the themes/book dir

```bash
git submodule update --init --recursive
```

After cloning the repository to your local machine, navigate to the repository root (dvla-reference-architecture).

on command line:

```bash
git checkout master                   'ensure you are on the master branch'
git pull                              'get latest changes from master branch'
hugo server -D                        'starts site on localhost:1313'
```
