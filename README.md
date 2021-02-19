# BashLib
Several bash utilities

The idea behind this repo is to aggregate multiple scripts and modules, created to cover different administration tasks but without enought weight to justify a repo by itself.

## Existing Helper Scripts
Currently, under the [scripts](https://github.com/jmnobre/BashLib/edit/main/scripts) folder, the following utilities exist:
 * create-password, to generate a random password
 * rsaenc, creates a RSA encypted package
 * to-phonetic-alphabet, cobverts an ASCII string, composed by printable, non-blank characters into its phonetic representation

All these utilities support the ```--help``` and ```--version``` options.

## Existing Modules
The following modules are available under the [modules](https://github.com/jmnobre/BashLib/edit/main/modules) folder:
 * [xlog.module](https://github.com/jmnobre/BashLib/blob/main/modules/xlog.md), which contains a collection of functions that help the creation of a structured log for bash scripts
