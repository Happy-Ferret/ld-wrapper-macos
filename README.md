# ld-wrapper-macos

Wrapper script around the system `ld` linker for macOS Haskellers to work around Mach-O header size limitations imposed by macOS Sierra and later.

## Synopsis

This script wraps the raw "ld" linker to sidestep behavior in macOS Sierra and later where the OS prevents loading dynamic libraries that have a Mach-O header size over a fixed threshold of 32,768. When the size is exceeded and GHC goes to `dlopen` the `.dylib`, we get a GHC panic that looks like this:

```
ghc: panic! (the 'impossible' happened)
  (GHC version 8.2.2 for x86_64-apple-darwin):
    Loading temp shared object failed: dlopen(/var/folders/49/bgbzql7j62j5z2r1r0m2m3rr0000gn/T/ghc763_0/libghc_13.dylib, 5): no suitable image found.  Did find:
    /var/folders/49/bgbzql7j62j5z2r1r0m2m3rr0000gn/T/ghc763_0/libghc_13.dylib: malformed mach-o: load commands size (33208) > 32768
    /var/folders/49/bgbzql7j62j5z2r1r0m2m3rr0000gn/T/ghc763_0/libghc_13.dylib: stat() failed with errno=25
```

This issue occurs most often when GHC is loading its temporary 'libghc_<numbers>.dylib` file that is used as part of Template Haskell codegen. This .dylib file dynamically links in just about all of a project's dependencies - both direct and indirect - and can easily exceed the Mach-O header size limit for medium to large-size projects.

Note that macOS does not impose a restriction on the creation of dynamic libraries with header sizes over the threshold. In the above GHC panic example, the "libghc_13.dylib" file was successfully created. The OS restriction comes into play when the library is attempted to be loaded.

## Example

TODO

## Details

There is a stack issue discussing this problem here:
* https://github.com/commercialhaskell/stack/issues/2577

There are also GHC trac tickets about this here:
* https://ghc.haskell.org/trac/ghc/ticket/12479
* https://ghc.haskell.org/trac/ghc/ticket/14444

A workaround exists for this issue in "cabal new-build" via putting all dependency libraries in a common store directory and shortening names, but it can still hit the limit:
* https://github.com/haskell/cabal/pull/4656
* https://github.com/haskell/cabal/issues/5220

The approach we are taking in this script is _heavily_ influenced by the changes that went into NixOS to handle the issue. It is largely a straight lift with removal/replacement of any Nix-y bits and is simplified a good bit to just handle the GHC use case instead of handling all kinds of linking outside of GHC too:
* https://github.com/NixOS/nixpkgs/pull/27536
* https://github.com/NixOS/nixpkgs/pull/38881
* https://github.com/tpoechtrager/cctools-port/pull/34

This script recursively re-exports the dependencies by subdividing to create a tree of reexporting delegate libraries. For example, here is the contents of "<working_dir>/.ld-wrapper-macos/lib" after this script was run successfully during TH codegen on a project with 750 direct dependencies:

```
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-0-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-0-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-1-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-1-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-0-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-0-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-1.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-1-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1-reexport-delegate-1-reexport-delegate-1.dylib
```

All of the "leaf" dylibs above (the longest file paths) are the libraries that re-export the sub-divided chunks of dependencies.  Each non-"leaf" dylib re-exports its two "children" re-exporting dylibs, i.e.:

```
$ otool -L .ld-wrapper-macos/lib/libghc_13-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0.dylib:
        <working_dir>/.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0.dylib (compatibility version 0.0.0, current version 0.0.0)
        <working_dir>/.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-0.dylib (compatibility version 0.0.0, current version 0.0.0)
        <working_dir>/.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0-reexport-delegate-1.dylib (compatibility version 0.0.0, current version 0.0.0)
```

The actual library GHC is intending to create - in the above example, this would be `libghc_13.dylib` - will only link against the two re-exporting libraries at the top-most level of the tree, i.e.:

```
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-0.dylib
.ld-wrapper-macos/lib/libghc_13-reexport-delegate-1.dylib
```

As each delegate library re-exports its "children" delegate libraries, the actual library GHC is intending to create has full access to all the real Haskell dependencies re-exported by the "leaf" delegate libraries.  Most importantly, none of the generated dylibs will have a mach-o header size over the limit imposed by macOS.
