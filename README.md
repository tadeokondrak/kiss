# KISS Alternative Package System

This is an alternative package system I am experimenting with. Instead of the usual `PKGBUILD`, `APKBUILD`, `xbps-template` and `Pkgfile` format, this repository explores a more unixy approach.

Each Package is split into multiple files.

```sh
zlib/            # Package name.
├─ build         # Build script.
├─ depends       # Dependencies (one per line) (optional).
├─ sources       # Sources (one per line).
├─ version       # Package version.
┘

# Files generated by the package manager.
├─ manifest      # The built package's files and directories.
├─ checksums     # The checksums for the source files.
┘

# Optional files.
├─ post_install  # Script to run after package installation.
```

When a built package is installed, this entire directory tree is copied to `/var/db/puke` where it becomes a database entry. Listing the dependencies for a package is a simple as printing the contents of the `depends` file. Searching for which package owns a file is as simple as checking each `manifest` file.

This new structure also allows the package manager to be stupid simple. POSIX `sh` has no arrays. However, they are mimicked by looping over each line of each file. No more insecure `depends="pkg pkg pkg"` and `for pkg in $depends`.

Instead, the following can be done.

```sh
while read -r depend; do
    # do thing.
done < depends
```

This also means anyone can write a tool to manipulate the repository or even their own package manager. It's all plain text files delimited by a new line or a space.

## Table of Contents

<!-- vim-markdown-toc GFM -->

* [Getting started with `puke`](#getting-started-with-puke)
    * [`puke build pkg`](#puke-build-pkg)
    * [`puke checksum pkg`](#puke-checksum-pkg)
    * [`puke install pkg`](#puke-install-pkg)
    * [`puke remove pkg`](#puke-remove-pkg)
    * [`puke list` or `puke list pkg`](#puke-list-or-puke-list-pkg)
    * [`puke update`](#puke-update)
* [The package format](#the-package-format)
    * [`build`](#build)
    * [`manifest`](#manifest)
    * [`sources`](#sources)
    * [`depends`](#depends)
    * [`version`](#version)
    * [`checksums`](#checksums)
    * [`post-install`](#post-install)
* [Frequently asked questions](#frequently-asked-questions)
    * [How do I change compiler options globally?](#how-do-i-change-compiler-options-globally)

<!-- vim-markdown-toc -->


## Getting started with `puke`

Puke is a simple package manager written in POSIX `sh`. The package manager does not need to be added to your `PATH`. Instead it runs inside the packages repository, very similar to Void Linux's `xbps-src`.

Puke has 6 different "operators".

- `build`: Build a package.
- `checksum`: Generate checksums for a package.
- `install`: Install a built package.
- `remove`: Remove an installed package.
- `list`: List installed packages.
- `update`: List packages with available updates.

### `puke build pkg`

Puke's `build` operator handles a package from its source code to the installable `.tar.gz` file. Sources are downloaded, checksums are verified, dependencies are checked and the package is compiled then packaged.

### `puke checksum pkg`

Puke's `checksum` operator generates the initial checksums for a package from every source in the `sources` file.

### `puke install pkg`

Puke's `install` operator takes the built `.tar.gz` file and installs it in the system. This is as simple as removing the old version of the package (*if it exists*) and unpacking the archive at `/`.

### `puke remove pkg`

Puke's `remove` operator uninstalls a package from your system. Files and directories in `/etc` are untouched. Support for exclusions will come as they are needed.

### `puke list` or `puke list pkg`

Puke's `list` operator lists the installed packages and their versions. Giving `list` an argument will check if a singular package is installed.

### `puke update`

Puke's `update` operator compares the repository versions of packages to the installed database versions of packages. Any mismatch in versions is considered a new upgrade from the repository.

The `update` mechanism doesn't do a `git pull` of the repository. This must be done manually beforehand. This is intentional. It allows the user to `git pull` selectively. You can slow down the distribution's package updates by limiting pulling to a week behind master for example.


## The package format

### `build`

The `build` file should contain the necessary steps to patch, configure, build and install the package. The build script is sent a single argument. This argument points to the package directory. Whatever is in this directory will become part of the package's manifest and will be copied to `/` (or `$PUKE_ROOT`). The first argument is frequently used in `make DESTDIR="$1" install` for example.

The `build` file can be written in any language. The only requirement is that the file be executable.

```sh
./configure \
    --prefix=/usr \
    --libdir=/lib \
    --shared

make
make DESTDIR="$pkg_dir" install
```

### `manifest`

The `manifest` file contains the built package's file and directory list. The full paths to files are listed first and the directories (*in reverse*) follow. This allows the package manager to remove the directories if they're empty without needing checks in-between.

The manifest also includes the package's database entry. You can install the package with or without `puke` and it will be recognized.

```
/usr/share/man/man3/zlib.3
/usr/include/zconf.h
/usr/include/zlib.h
/var/db/puke/zlib/sources
/var/db/puke/zlib/manifest
/var/db/puke/zlib/checksums
/var/db/puke/zlib/build
/var/db/puke/zlib/version
/lib/libz.so.1.2.11
/lib/libz.so.1
/lib/libz.so
/lib/libz.a
/lib/pkgconfig/zlib.pc
/var/db/puke/zlib
/var/db/puke
/var/db
/var
/usr/share/man/man3
/usr/share/man
/usr/share
/usr/include
/usr
/lib/pkgconfig
/lib
```

### `sources`

The `sources` file contains the package's sources one per line. Sources can be local or remote.

```
https://www.openssl.org/source/openssl-X.X.X.tar.gz
patches/fix-musl.patch
```

### `depends`

The `depends` file contains the package's dependencies one per line.

```
zlib
binutils
openssl
```

### `version`

The `version` file contains the package's version as well as its release number. The format of this file is `version release`. The `release` portion allows a package upgrade without the modification of the version number.

The version can also be `git 1` to specify that the package is built from the latest `git` head.

```
1.2.11 1
```

### `checksums`

The `checksums` file contains the `sha256` sums of each entry in the `sources` file. This is generated and verified automatically.

```
c3e5e9fdd5004dcb542feda5ee4f0ff0744628baf8ed2dd5d66f8ca1197cb1a1  zlib-1.2.11.tar.gz
```

### `post-install`

The `post-install` file should contain any steps required directly after the package is installed. This includes updating font databases and creating any post-install symlinks which may be required.


## Frequently asked questions

### How do I change compiler options globally?

All you need to do is define `CFLAGS`, `MAKEFLAGS` or equivalent in your environment. Either give it to `puke` directly (`CFLAGS=-O3 MAKEFLAGS=-j4 ./puke build zlib`) or set it in your shell's RC file.
