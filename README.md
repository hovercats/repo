# KISS Alternative Package System

This is a package system I am experimenting with. This format is an alternative to the usual `PKGBUILD`, `APKBUILD` and `Pkgfile` systems and explores a more unixy approach.

**NOTE**: The package manager can be found here: [kiss](https://github.com/kissx/kiss)

Each Package is split into multiple files.

```sh
zlib/            # Package name.
├─ build         # Build script.
├─ depends       # Dependencies (one per line) (sometimes optional).
├─ sources       # Sources (one per line).
├─ version       # Package version.
┘

# Files generated by the package manager.
├─ manifest      # The built package's files and directories.
├─ checksums     # The checksums for the source files.
┘

# Optional files.
├─ post_install  # Script to run after package installation.
├─ patches/*     # Directory to store patches.
├─ files/*       # Directory to misc files.
┘
```

When a built package is installed, this entire directory tree is copied to `/var/db/kiss` where it becomes a database entry. Listing the dependencies for a package is a simple as printing the contents of the `depends` file. Searching for which package owns a file is as simple as checking each `manifest` file.

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

* [The package format](#the-package-format)
    * [`build`](#build)
    * [`manifest`](#manifest)
    * [`sources`](#sources)
    * [`depends`](#depends)
    * [`version`](#version)
    * [`checksums`](#checksums)
    * [`post-install`](#post-install)

<!-- vim-markdown-toc -->


## The package format

### `build`

The `build` file should contain the necessary steps to patch, configure, build and install the package. The build script is sent a single argument. This argument points to the package directory. Whatever is in this directory will become part of the package's manifest and will be copied to `/` (or `$kiss_ROOT`). The first argument is frequently used in `make DESTDIR="$1" install` for example.

The `build` file can be written in any language. The only requirement is that the file be executable.

```sh
./configure \
    --prefix=/usr \
    --libdir=/lib \
    --shared

make
make DESTDIR="$1" install
```

### `manifest`

The `manifest` file contains the built package's file and directory list. The full paths to files are listed first and the directories (*in reverse*) follow. This allows the package manager to remove the directories if they are empty without needing checks in-between.

The manifest also includes the package's database entry. You can install the package with or without `kiss` and it will be recognized.

```
/usr/share/man/man3/zlib.3
/usr/include/zconf.h
/usr/include/zlib.h
/var/db/kiss/zlib/sources
/var/db/kiss/zlib/manifest
/var/db/kiss/zlib/checksums
/var/db/kiss/zlib/build
/var/db/kiss/zlib/version
/lib/libz.so.1.2.11
/lib/libz.so.1
/lib/libz.so
/lib/libz.a
/lib/pkgconfig/zlib.pc
/var/db/kiss/zlib
/var/db/kiss
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

An optional destination field can be added to tell the package manager where to extract the source. This is relative to the regular extraction directory. The passed directories are also created.

```
https://www.openssl.org/source/openssl-X.X.X.tar.gz
patches/fix-musl.patch
https://www.openssl.org/source/openssl-X.X.X.tar.gz lib/example/
git:https://github.com/dylanaraps/neofetch
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
