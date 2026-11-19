# bindings-dsl

[![Hackage](https://img.shields.io/hackage/v/bindings-DSL.svg)](https://hackage.haskell.org/package/bindings-DSL)
[![CI](https://github.com/rethab/bindings-dsl/actions/workflows/ci.yml/badge.svg)](https://github.com/rethab/bindings-dsl/actions/workflows/ci.yml)
![BSD3 License](https://img.shields.io/github/license/rethab/bindings-dsl?label=license)

`bindings-DSL` is a domain specific language for writing Haskell FFI bindings, on
top of [hsc2hs](https://downloads.haskell.org/ghc/latest/docs/users_guide/utils.html#writing-haskell-interfaces-to-c-code-hsc2hs).
It is a set of C macros that describe a C interface, and from which hsc2hs
extracts the Haskell code that mimics it: `#ccall` for a function, `#starttype`
and `#field` for a struct, `#num` for a constant, and so on.

```haskell
#include <bindings.dsl.h>
#include <gpgme.h>

module Bindings.Gpgme where
#strict_import

#num GPGME_PK_RSA

#starttype struct _gpgme_key
#field revoked , CUInt
#field fpr     , CString
#stoptype

#ccall gpgme_get_key , <gpgme_ctx_t> -> CString -> Ptr <gpgme_key_t> -> CInt -> IO <gpgme_error_t>
```

Haskell names are derived from the C names automatically, so there is no naming
scheme to invent and no chance of the two drifting apart: a constant becomes
`c'GPGME_PK_RSA`, a struct becomes a `Storable` instance with `p'_gpgme_key'fpr`
accessors, and a function becomes `c'gpgme_get_key`. Structs are laid out by the
C compiler rather than by hand, which is what makes the bindings portable across
platforms where sizes and padding differ. There are also macros for binding
inline functions and macro functions, which have no symbol to call.

Full documentation, including a tutorial, is on the
[wiki](https://github.com/rethab/bindings-dsl/wiki). Release notes are in
[CHANGELOG.md](CHANGELOG.md).

## Packages

`bindings-DSL` is the DSL itself. The bindings built on top of it are released as
separate packages from this repository:

| Package | Hackage |
| --- | --- |
| [bindings-directfb](bindings-directfb) | [![Hackage](https://img.shields.io/hackage/v/bindings-directfb.svg?label=)](https://hackage.haskell.org/package/bindings-directfb) |
| [bindings-fann](bindings-fann) | [![Hackage](https://img.shields.io/hackage/v/bindings-fann.svg?label=)](https://hackage.haskell.org/package/bindings-fann) |
| [bindings-glib](bindings-glib) | [![Hackage](https://img.shields.io/hackage/v/bindings-glib.svg?label=)](https://hackage.haskell.org/package/bindings-glib) |
| [bindings-gpgme](bindings-gpgme) | [![Hackage](https://img.shields.io/hackage/v/bindings-gpgme.svg?label=)](https://hackage.haskell.org/package/bindings-gpgme) |
| [bindings-gsl](bindings-gsl) | [![Hackage](https://img.shields.io/hackage/v/bindings-gsl.svg?label=)](https://hackage.haskell.org/package/bindings-gsl) |
| [bindings-hdf5](bindings-hdf5) | [![Hackage](https://img.shields.io/hackage/v/bindings-hdf5.svg?label=)](https://hackage.haskell.org/package/bindings-hdf5) |
| [bindings-libcddb](bindings-libcddb) | [![Hackage](https://img.shields.io/hackage/v/bindings-libcddb.svg?label=)](https://hackage.haskell.org/package/bindings-libcddb) |
| [bindings-libffi](bindings-libffi) | [![Hackage](https://img.shields.io/hackage/v/bindings-libffi.svg?label=)](https://hackage.haskell.org/package/bindings-libffi) |
| [bindings-posix](bindings-posix) | [![Hackage](https://img.shields.io/hackage/v/bindings-posix.svg?label=)](https://hackage.haskell.org/package/bindings-posix) |
| [bindings-sqlite3](bindings-sqlite3) | [![Hackage](https://img.shields.io/hackage/v/bindings-sqlite3.svg?label=)](https://hackage.haskell.org/package/bindings-sqlite3) |

## Contributing

Contributions are welcome, whether to the DSL itself or to any of the bindings.
Bugs, missing coverage of a C API, and build problems can be raised in the
[issue tracker](https://github.com/rethab/bindings-dsl/issues); pull requests are
just as welcome. CI builds the DSL and the bindings across several GHC versions
and library releases, so open a pull request and let it run.

## Building bindings-gpgme on macOS with Homebrew

`gpgme.h` includes `<gpg-error.h>`, and Homebrew ships `libgpg-error` as a keg of
its own, so both prefixes have to be on the include and library paths. Passing
only gpgme's prefix leaves the nested include unresolved and the build fails with
a misleading complaint about a missing `gpgme.h`.

```sh
brew install gpgme libgpg-error

cabal build bindings-gpgme \
  --extra-include-dirs="$(brew --prefix gpgme)/include" \
  --extra-include-dirs="$(brew --prefix libgpg-error)/include" \
  --extra-lib-dirs="$(brew --prefix gpgme)/lib" \
  --extra-lib-dirs="$(brew --prefix libgpg-error)/lib"
```

With stack, list both prefixes in `stack.yaml` (`brew --prefix` is
`/opt/homebrew` on Apple silicon and `/usr/local` on Intel):

```yaml
extra-include-dirs:
  - /opt/homebrew/opt/gpgme/include
  - /opt/homebrew/opt/libgpg-error/include
extra-lib-dirs:
  - /opt/homebrew/opt/gpgme/lib
  - /opt/homebrew/opt/libgpg-error/lib
```

## Cross compilation

`bindings.dsl.h` supports hsc2hs assembly cross mode. When hsc2hs is invoked
with `--cross-compile` or `--cross-safe`, it defines `HSC2HS_ASM` and compiles
the probe with one `cc -S -O2` pass. It does not link or run a target binary.
The header then routes `printf`, `putchar`, and `fputs` through `hsc_printf`,
`hsc_putchar`, and `hsc_fputs`, and builds identifiers with `hsc_mangle` instead
of the runtime `bc_word` / `bc_glue` loops. `bc_typemarkup` sends the type
spelling to `hsc_typemarkup`, and the cross decoder rewrites `<ident>` to
`C'ident`, including `Ptr <gchar>` from `#field` and `#ccall`. `hsc_stoptype` asks the compiler to
unroll its field loops so those identifiers are compile-time constants.

Values that are not compile-time constants still need a real run of the probe:

* `--cross-runner=CMD`, plus repeatable `--cross-runner-arg=ARG`, compiles and
  links the probe as usual and executes it through `CMD` (for example
  `qemu-aarch64` or `wine64`). This overrides `-x`.
* `--no-cross-compile` overrides an earlier `-x` or `--cross-compile` when the
  target binary can already run on the build host (`binfmt_misc`, Rosetta 2).

Cabal passes `-x` whenever the host platform differs from the build platform.
`--no-cross-compile` or `--cross-runner` belong after that `-x`. Older hsc2hs
without `HSC2HS_ASM` still needs a wrapper that drops `-x` and runs the probe
under emulation:

```sh
#!/bin/sh
# hsc2hs-no-cross, used via --with-hsc2hs=/path/to/hsc2hs-no-cross
for a in "$@"; do
  shift
  case "$a" in
    -x|--cross-compile) ;;
    *) set -- "$@" "$a" ;;
  esac
done
exec hsc2hs "$@"
```

Otherwise, write plain `.hsc` without bindings-DSL for the modules you need to
cross compile.
