# Workload profiles

## GHC
```bash
tar -xf ~/.cache/ghc-bench/ghc-9.12.4-src.tar.gz
cd ghc-9.12.4
./configure
hadrian/build --help

# MEASURE ghc-9.12.4-build
time hadrian/build -j$JOBS --flavour=quickest
```

## CABAL
```bash
cabal unpack hedgehog-1.7 --index-state=2026-04-15T08:18:05Z
cd hedgehog-1.7
cabal --config-file=ghc-bench-cabal-config --store-dir=store user-config init
cabal --config-file=ghc-bench-cabal-config --store-dir=store configure --with-compiler=ghc-9.12.4 --jobs=$JOBS --enable-tests --enable-benchmarks --index-state=2026-04-15T08:18:05Z
cabal --config-file=ghc-bench-cabal-config --store-dir=store build --only-dependencies --only-download

# MEASURE hedgehog-1.7-dependencies
time cabal --config-file=ghc-bench-cabal-config --store-dir=store build --only-dependencies
```

## devel2
```bash
tar -xf ~/.cache/ghc-bench/ghc-9.12.4-src.tar.gz
cd ghc-9.12.4
./configure
hadrian/build --help

# MEASURE ghc-9.12.4-build
time hadrian/build -j$JOBS --flavour=devel2
```

## release
```bash
tar -xf ~/.cache/ghc-bench/ghc-9.12.4-src.tar.gz
cd ghc-9.12.4
./configure
hadrian/build --help

# MEASURE ghc-9.12.4-build
time hadrian/build -j$JOBS
```

## pinned
```bash
tar -xf ~/.cache/ghc-bench/ghc-9.12.4-src.tar.gz
cd ghc-9.12.4
./configure
hadrian/build --help

# MEASURE ghc-9.12.4-build
time taskset -c 0-9 hadrian/build -j10 --flavour=quickest
```
