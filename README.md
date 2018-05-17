# tor-static

This project helps compile Tor into a static lib for use in other projects.

The dependencies are in this repository as submodules so this repository needs to be cloned with `--recursive`. The
dependencies are:

* [OpenSSL](https://github.com/openssl/openssl/) - Checked out at tag `OpenSSL_1_0_2o`
* [Libevent](https://github.com/libevent/libevent) - Checked out at tag `release-2.1.8-stable`
* [zlib](https://github.com/madler/zlib) - Checked out at tag `v1.2.11`
* [XZ Utils](https://git.tukaani.org/?p=xz.git) - Checked out at tag `v5.2.4`
* [Tor](https://github.com/torproject/tor) - Checked out at tag `tor-0.3.3.5-rc`

## Building

TODO: At some point, automate this with a build script per platform.

### Linux

TODO (should be mostly straightforward using some of the Windows commands below)

## macOS

TODO (should be mostly straightforward using some of the Windows commands below)

### Windows

Many many bugs and quirks were hit while deriving these steps. Also many other repos, mailing lists, etc were leveraged
to get some of the pieces right. They are not listed here for brevity reasons.

#### Msys2 and MinGW Setup

Tor is not really designed to work well with MSVC so we use MinGW instead. Since we are statically compiling, this means
we use the MinGW form of Rust too. In order to compile the dependencies, Msys2 + MinGW should be installed.

Download and install the latest [MSYS2 64-bit](http://www.msys2.org/) that uses the `MinGW-w64` toolchains. Once
installed, open the "MSYS MinGW 64-bit" shell link that was created. Once in the shell, run:

    pacman -Syuu

Terminate and restart the shell if asked. Rerun this command as many times as needed until it reports that everything is
up to date. Then in the same mingw-64 shell, run:

    pacman -Sy --needed base-devel mingw-w64-i686-toolchain mingw-w64-x86_64-toolchain \
                        git subversion mercurial \
                        mingw-w64-i686-cmake mingw-w64-x86_64-cmake

This will install all the tools needed for building and will take a while. Once complete, we have to downgrade a couple
of packages due to a bug in the current MinGW libraries. In the same shell, run:

    pacman -U /var/cache/pacman/pkg/mingw-w64-x86_64-crt-git-5.0.0.4745.d2384c2-1-any.pkg.tar.xz
    pacman -U /var/cache/pacman/pkg/mingw-w64-x86_64-headers-git-5.0.0.4747.0f8f626-1-any.pkg.tar.xz
    pacman -U /var/cache/pacman/pkg/mingw-w64-x86_64-winpthreads-git-5.0.0.4741.2c8939a-1-any.pkg.tar.xz \
              /var/cache/pacman/pkg/mingw-w64-x86_64-libwinpthread-git-5.0.0.4741.2c8939a-1-any.pkg.tar.xz

At least these were the cached package names on my install, they may be different on others. Once complete, MinGW is now
setup to build the dependencies.

#### Clone Repo

Inside the mingw-64 shell, clone this repo and submodules:

    git clone --recursive https://github.com/cretz/tor-static.git

Then you can `cd tor-static`. We will assume throughout this guide that you are starting at the cloned root.

#### OpenSSL

Inside the mingw-64 shell, navigate to the OpenSSL folder and build it:

    cd openssl
    ./Configure --prefix=$PWD/dist no-shared no-dso no-zlib mingw64
    make depend
    make
    make install

This will put OpenSSL libs at `dist/lib`.

#### Libevent

Inside the mingw-64 shell, navigate to the Libevent folder and build it:

    cd libevent
    ./autogen.sh
    ./configure --prefix=$PWD/dist --disable-shared --enable-static --with-pic
    make
    make install

This will put Libevent libs at `dist/lib`.

#### zlib

Inside the mingw-64 shell, navigate to the zlib folder and build it:

    cd zlib
    PREFIX=$PWD/dist make -fwin32/Makefile.gcc
    PREFIX=$PWD/dist BINARY_PATH=$PWD/dist/bin INCLUDE_PATH=$PWD/dist/include LIBRARY_PATH=$PWD/dist/lib make install -fwin32/Makefile.gcc

This will put zlib libs at `dist/lib`.

#### XZ Utils

Inside the mingw-64 shell, navigate to the XZ Utils folder and build it:

    cd xz
    ./autogen.sh
    ./configure --prefix=$PWD/dist \
                --disable-shared \
                --enable-static \
                --disable-doc \
                --disable-scripts \
                --disable-xz \
                --disable-xzdec \
                --disable-lzmadec \
                --disable-lzmainfo \
                --disable-lzma-links
    make
    make install

This will put XZ Utils libs at `dist/lib`.

#### Tor

Inside the mingw-64 shell, navigate to the tor folder and build it:

    cd tor
    ./autogen.sh
    LIBS=-lcrypt32 ./configure --prefix=$PWD/dist \
                                --disable-gcc-hardening \
                                --enable-static-tor \
                                --enable-static-libevent \
                                --with-libevent-dir=$PWD/../libevent/dist \
                                --enable-static-openssl \
                                --with-openssl-dir=$PWD/../openssl/dist \
                                --enable-static-zlib \
                                --with-zlib-dir=$PWD/../openssl/dist \
                                --disable-system-torrc \
                                --disable-asciidoc
    ln -s $PWD/../zlib/dist/lib/libz.a $PWD/../openssl/dist/lib/libz.a
    make
    make install

This will put Tor libs throughout the `src` area.

## Using

Once the libs have been compiled, they can be used to link with your program. Below is the list of each dir (relative to
this repository root) and the library names inside the dirs. Library names are just the filenames without the "lib"
prefix of ".a" extension. When using something like `ld` or `LDFLAGS`, the directories (i.e. `-L<dir>`) and the libs
(i.e. `-l<libname>`) must be given in the order below:

* `tor/src/or` - `tor`
* `tor/src/common` - `or`, `or-crypto`, `curve25519_donna`, `or-ctime`, and `or-event`
* `tor/src/trunnel` - `or-trunnel`
* `tor/src/ext/keccak-tiny` - `keccak-tiny`
* `tor/src/ext/ed25519/ref10` - `ed25519_ref10`
* `tor/src/ext/ed25519/donna` - `ed25519_donna`
* `libevent/dist/lib` - `event`
* `xz/dist/lib` - `lzma`
* `zlib/dist/lib` - `z`
* `openssl/dist/lib` - `ssl`, and `crypto`

The OS-specific system libs that have to be referenced (i.e. `-l<libname>`) are:

* Linux - TODO
* macOS - TODO
* Windows (MinGW) - `ws2_32`, `crypt32`, and `gdi32`