# How many CPU cores do you really need as a Haskell developer?

I went on a small side quest to get a better understanding of what the best CPU for Haskell development might be. In this blog post I will share what I learned.

Going into this, I had some long held *assumptions*:

- Haskell development does not scale well over cores
- Single core performance is the most important metric for Haskell developers
- Hyperthreading / efficiency cores are mostly useless

Basing decisions on assumptions never felt quite right. So I tried to get some numbers.

To make things reproducible, I wrote a small benchmarking tool called [`ghc-bench`][ghc-bench].
My goal with this is to collect comparable results for different CPUs over time.

**Benchmarking system:** Intel Core i9-10900K (10 cores / 20 threads), 32 GB RAM

## Impact of `--jobs` on compilation times

I looked at two things:

- Building GHC with `--flavour=quickest`
- Installing all required dependencies for `hedgehog-1.7` with `cabal`

| JOBS  | 1   | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |
| ----- | --- |---|---|---|---|---|---|---|---|---|
| [GHC][ghc] | 1460|820|645|566|536|525|524|526|531|534|
| [CABAL][cabal] | 91  | 51| 36| 30| 27| 27| 27| 28| 29| 29|

(all results in seconds, lower is better)

For GHC builds we get consistent speedups up to ~6 cores, but not much beyond that.

When installing dependencies for `hedgehog-1.7`, `cabal` can effectively utilize up to 5 cores.

(Note that for GHC builds we are hitting a hard limit here.  For `cabal`, however, this is highly workload dependent, see the [final remarks](#final-remarks) for why this is.)

## The impact of hyperthreading
| JOBS  | 11 |12 |13 |14 |15 |16 |17 |18 |19 |20 |
| ----- | ---|---|---|---|---|---|---|---|---|---|
| [GHC][ghc]   | 527|530|532|532|534|535|537|537|540|539|
| [CABAL][cabal] |  29| 29| 29| 29| 29| 29| 29| 29| 29| 29|

Hyperthreading doesn't contribute much.  Given that we didn't see scaling beyond six cores, this makes perfect sense.  On the upside, at least performance didn't degrade massively with increasing concurrency.

**NOTE:** The same is likely true for efficiency cores ([numbers](https://github.com/sol/ghc-bench/issues/63#issuecomment-4346047909)).

## Impact of `--flavour`

How much potential there is for parallelization in the first place might depend on the build flavor.
As such, I looked into whether building GHC with different build flavors changes the overall picture.

| JOBS    | 6 | 8 |10 |
| ------- |---|---|---|
| [devel2][devel2]  |752|752|767|
| [release][release] | 1942| 1917| 1918|

`devel2` lines up with what we saw for `quickest`.

A default (release) build benefits somewhat from two additional cores.

## Does pinning to physical cores have any significant impact

The last thing I looked into, is whether it is beneficial to pin a build to physical cores or if it is fine to rely on the OS scheduler to do the right thing.

| JOBS   |10 |
| ------ |---|
| [GHC][ghc]    |534|
| [pinned][pinned] |527|

With pinning, the results seem to be less noisy, but differences and benefits are rather small.

## Conclusion
So what is the best CPU for Haskell development?

The benchmark results suggest that for Haskell development, six strong performance cores are more beneficial than a large number of total cores.

We are looking for something with:

- At least six full (performance) cores
- High single core performance

I still have an incomplete picture, but I will discuss three CPUs that are likely one of the better options in their respective category.
Take these as educated guesses.  Long-term, what we really want are dependable numbers,
and [`ghc-bench`][ghc-bench] is my attempt to get those numbers.

If you have any of these CPUs, or really any other CPU, please consider submitting [`ghc-bench`][ghc-bench] results.

### Intel 270K for desktop

Last month Intel dropped the ***Intel Core Ultra 7 270K Plus***.  It's basically a top-bin ***Intel Core Ultra 9 285K***, with a much lower price tag.

It has:
- 8 performance cores
- the highest single core performance of any x86-64 CPU (according to synthetic benchmarks, alongside the 285K)

If you don't need mobility then this should make a very capable machine for Haskell development.

### AMD 9700X for a SFF build

In a thermally constrained environment we're looking at:

- Intel 265 (non-K)
- AMD 9700X

Both are 65W TDP, come with 8 full (performance) cores and comparable single core performance.

Everything else equal, I'll go with AMD (likely better thermals, platform longevity).

### Intel 255H for mobile

Going by synthetic benchmarks, Intel Arrow Lake H-series CPUs excel in single core performance.
They come with up to six performance cores, and vPro/non-vPro variants.

The ***Intel Core Ultra 7 255H*** is the lowest bin that has all six performance cores activated, and is likely the sweet spot.

You may want to fallback to Lunar Lake (228V or 258V) for ultra portables (e.g. it is not clear to me at this point whether a ThinkPad X1 Carbon with Intel 255H can handle the thermals; if you have such a machine, then please submit [`ghc-bench`][ghc-bench] results).

Personally, I would pass on Panther Lake (Core Ultra Series 3) and hope that Nova Lake will bring more interesting options to the table.

On the AMD side, I would be really eager to see [`ghc-bench`][ghc-bench] results for ***Ryzen AI Max 300 Series*** CPUs.

### If you are shopping for an Apple device

- The 18-core ***Apple M5 Pro*** with six "super" cores looks like your best option, and at least on paper far outperforms anything in the x86-64 department.
- The 15-core version is cheaper, but has one of the super cores disabled, giving you only five instead of six super cores.
- The fully enabled ***M5 Max*** doesn't give you anything extra in terms of CPU power, and only improves on GPU performance.

## Final remarks
- It is important to distinguish between process level parallelization and in-process parallelization.
- GHC builds utilize all available cores in-process, with diminishing returns.
- `cabal` uses process level parallelization. Each unit is built in a separate process.  This scales better than in-process parallelization and in theory allows you to utilize all available cores.  In practice, however, it is limited by how much opportunity for parallelization a given dependency graph offers. For most install plans this is rather limited, but if you were to e.g. build all of Hackage then more cores will pay off.
- For GHC builds however we are hitting a hard limit at ~6 cores.
- For a hybrid approach, `cabal` and `ghc` can coordinate via a semaphore (`-jsem`).  I did not benchmark this, as it is not generally available due to issues with incompatible semaphore implementations (`glibc` vs `musl`).

[ghc-bench]: https://github.com/sol/ghc-bench
[ghc]:     assets/how-many-cpu-cores-do-you-really-need/workload-profiles.md#ghc
[cabal]:   assets/how-many-cpu-cores-do-you-really-need/workload-profiles.md#cabal
[devel2]:  assets/how-many-cpu-cores-do-you-really-need/workload-profiles.md#devel2
[release]: assets/how-many-cpu-cores-do-you-really-need/workload-profiles.md#release
[pinned]:  assets/how-many-cpu-cores-do-you-really-need/workload-profiles.md#pinned

---

## Comments

---
[Leave a comment](https://github.com/sol/rants/issues/1#comment-composer-heading)
