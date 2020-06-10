# macOS python3 install keystone

```shell
2h0Ng$ git clone https://github.com/keystone-engine
2h0Ng$ cd keystone-engint
2h0Ng$ mkdir build
2h0Ng$ cd build
2h0Ng$ vim ../make-common.sh
```

delete ```i386``` from ARCH

```vim
BUILDTYPE='Release'

# on MacOS, build universal binaries by default
ARCH='x86_64'
#ARCH='i386;x86_64'

# Linux FHS wants to install x64 libraries in "lib64"
# Examples are Fedora, Redhat, Suse.
LLVM_LIBDIR_SUFFIX=''

# by default we do NOT build 32bit on 64bit system
LLVM_BUILD_32_BITS=0

# by default we build libraries & kstool
# but we can skip kstool & build libraries only
BUILD_LIBS_ONLY=0

while [ "$1" != "" ]; do
  case $1 in
    lib_only)
      BUILD_LIBS_ONLY=1
      ;;
    lib32)
      LLVM_BUILD_32_BITS=1
      ;;
    lib64)
      LLVM_LIBDIR_SUFFIX='64'
      ;;
    debug)
      BUILDTYPE='Debug'
      ;;
    macos-no-universal)
      ARCH=''	# do not build MacOS universal binaries
      ;;
    *)
      echo "ERROR: unknown parameter \"$1\""
      usage
      exit 1
      ;;
  esac
  shift
done
```
```shell
2h0Ng$ ../make-share.sh
2h0Ng$ sudo make install
2h0Ng$ cd ../bindings/python
2h0Ng$ sudo make install3
```

Done~~~

