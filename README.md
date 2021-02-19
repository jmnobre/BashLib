# BashLib
Several bash utilities

The idea behind this repo is to aggregate multiple scripts and modules, created to cover different administration tasks but without enought weight to justify a repo by itself.

## Existing Helper Scripts
Currently, under the scripts folder, the following utilities exist:
 * create-password, to generate a random password
 * rsaenc, creates a RSA encypted package
 * to-phonetic-alphabet, cobverts an ASCII string, composed by printable, non-blank characters into its phonetic representation

All these utilities support the ```--help``` and ```--version``` options.

## Existing Modules
The following modules are available:
 * xlog.module, which contains a collection of functions that help the creation of a structured log for bash scripts
 * chkp.module, which offers the possibility for a script to preserve context across multiple executions (e.g. across system restarts)
 * ersc.module, which offers the possibility for a bash script to include any resource in the script body, which can be used at
   any time.
