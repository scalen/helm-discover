# helm-discover
A Helm downloader plugin that supports discovery of values files.

This plugin provides only a "downloader" protocol.  Despite the name,
this protocol is used to discover/collect all values files in a given
_local_ directory.

Various options are available to refine which files are selected, and
whether to recurse into subdirecories.

![Plugin Tests](https://github.com/scalen/helm-discover/actions/workflows/test.yml/badge.svg)

## Installation

After installing Helm, simply run the following:
```bash
helm plugin install https://github.com/scalen/helm-discover
```

## Usage

This is only applicable to the `-f` or `--values` option of a Helm
command (e.g. `install`, `upgrade` or `template`).  The basic usage
is to reference a directory (either absolutely, or relative to the
PWD) from which to collect all non-hidden files with the extension
`.yaml` or `.yml`, not including sub-directories:

```bash
helm upgrade -f discover://path/to/values path/to/chart
```

## Options

Options may be used to change which files are discovered.  These are
specified in the format of URL query parameters. For example, in order
to collect all files that start `0_` or `1_` and end `.yml` from the given
directory and all subdirectories:

```bash
... -f discover://path/to/values?recursive&prefix=0_&prefix=1_&suffix=.yml
```

The available options are:

`recursive`: Discover files from subdirectories recursively.

`hidden`: Discover hidden files in addition to non-hidden files.

`hidden-only`: Discover hidden files, and do not discover non-hidden files.

`suffix=<suffix>`: Select only files named with the given suffix.  Can be
specified multiple times, in which case files named with any of the given
suffixes will be selected.

`prefix=<prefix>`: Like `suffix=`, but for files named with the given
prefix(es).

`suffix_protocol=<suffix>/<protocol>`: Like `suffix=`, in that it selects
files named with the given suffix(es); In addition, it uses the given
protocol to process these specific files.  This effectively expands the
file names as follows:
```
path/to/values/filename<suffix> -> <protocol>://path/to/values/filename<suffix>
```
This is mostly to facilitate interactions with other plugins.  For
example, in order to discover values files with the suffix `.enc` that
have been encrypted with
[`helm-secrets`](https://github.com/jkroepke/helm-secrets), use the
following query parameter:
```
...?suffix_protocol=.enc/secrets
```

`prefix_protocol=<prefix>/<protocol>`: Like `suffix_protocol=`, but for files named with the given prefix(es).

`union`: Discovers all files with any of the given suffixes _or_ the given
prefixes.

### Default suffixes

It is important to note that the following suffix options are applied by
default:
```
suffix=.yml&suffix=.yaml
```
This will apply regardless of the `prefix=...` or `prefix_protocol=.../...`
options specified.  This can lead to some unexpected behaviour where
values files are unexpectedly excluded (or included, when using the
`union` option).

However, these default suffix options will be removed if any other
`suffix=...` and/or `suffix_protocol=.../...` options are specified.

If you wish to disable suffix filtering entirely, the null-suffix
option `suffix=` (e.g. `...?suffix=&...`) may be used; this will also
disable any suffix filtering instituted by use of the `suffix_protocol`
option.  The same is true of using the null `prefix=` to disable prefix
filtering instituted by the use of `prefix_protocol`.

Let us clarify with an example. With the following two files in a
directory, the described queries will produce the given results:
```
foo.yaml bar.txt
```
```
(default behaviour filtering to .yaml and .yml, no query string)
foo.yaml
```
```
?suffix_protocol=.txt/baz
baz://bar.txt
```
```
?suffix=
foo.yaml,bar.txt
```
```
?suffix=&suffix_protocol=.txt/baz
foo.yaml,baz://bar.txt
```

### Usecase: Encrypted values

Imagine a case where we have an arbitrary organisation of values files
nested in a `config` directory, describing the configuration for an
application. _Some_ of these values files at the top level contain
sensitive information and/or configuration that may only be set by
developers with certain clearance; this restricted information is
encrypted-at-rest with SOPS, and identified with filename extension
`.enc`. The remainder of the configuration is not encrypted, and those
files use the standard YAML extensions (`.yml` and `.yaml`).

Our requirements here are that:
* Encrypted files are decrypted on the fly during `helm upgrade`/`helm
  install`.  Assuming the Helm-secrets plugin is installed (and that the
  deployer has access to the relevant decryption keys), this can be
  achieved with the downloader plugin `secrets://`.
* The values in encrypted files should always override values in
  unencrypted files, to prevent injection of malicious configuration by
  those without the necessary clearance.
* All values files are otherwise expected to be orthogonal, regardless of
  how they are nested.

This can be achieved by performing two rounds of discovery, as
demonstrated in the below `helm upgrade` command:
```bash
helm upgrade -f discover://config?recursive -f discover://config?suffix_protocol=.enc/secrets repo/chart
```
This will:
1. Discover all non-hidden files with the extension `.yaml` or `.yml` (the
   default behaviour) in all nested directories (`recursive` argument)
   under `config`.
2. Discover all non-hidden files with the extension `.enc` and decrypt them
  (`suffix_protocol=` argument) in the root (default behaviour) of
  `config`, overriding any duplicate config from the values established
  by 1.
