+++
date = 2025-06-09 
title = "Interval-trees in Uiua"
+++

As part of the course [Advanced Data Structures](https://www.fib.upc.edu/en/studies/masters/master-innovation-and-research-informatics/curriculum/syllabus/ADS-MIRI) at [UPC](https://www.fib.upc.edu/en) in Barcelona, we were asked to implement so-called [interval-trees](https://en.wikipedia.org/wiki/Interval_tree) to solve the line-stabbing problem. I felt it would be interesting to see what it was like to implement a datastructure in [Uiua](https://www.uiua.org/). In this post, I will show and compare three Uiua implementations.

## The Problem

To illustrate the motivating problem, consider the following. Imagine you work at an office and everybody there works during certain windows of time. Those shifts are characterized by starting points and end points; they are intervals.
Now suppose you would like to
know which of your `n`
co-workers is present at the
office at any given point(s) in time, as illustrated below. We call one such question a _query_ here.

```
                     ?
Fred │   ────────────|──────────
Jaya │           ────|──────────────────────
Susi │       ────────|────────
Maya │               |     ──────────────────
Nina │              ─|─────────────────────
                     ?
        8am   10am  12pm  14pm  16pm  18pm  20pm  22pm
```

The naive way to answer a query is to check whether each
of your co-workers is present
individually; this approach is exactly linear in the
number of coworkers `n`, that is, `O(n)`.

Astute readers will see that this is optimal - after all, it is possible that there is some time during the day at which everyone is at the office; reporting that would require at least `n` operations.

However, if we consider the size of the output `m` as its own parameter, we can do better, namely there is a data-structure for intervals which can be queried in `O(log n + m)` time, i.e., it provides an [output-sensitive algorithm](https://en.wikipedia.org/wiki/Output-sensitive_algorithm).

## Interval-trees

### The Basic Idea

As so often in algorithmics,
the answer is to [divide-and-conquer](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm). Let `M` be the median of all the `2n` endpoints of the intervals. The basic principles
of the datastructure are the
following.

- An interval can either contain `M`, be to its left or to its right.
- When querying for a point, if it is to the left of the `M`, no interval which lies on the right of the median will contain it, and vice versa.

As such, the first step to an interval tree is as follows. Store for a given set of intervals its median and the intervals that contain the median. In addition, recursively constructing an interval-tree `L` to the left (`R` respectively to the right) with the intervals that lie to the left (respectively to the right) of `M`.

```
 Intervals:  │ Corresponding tree:
    [0  1]   │            ┌─
    [2  5]   │            │ [2 5] │ M = 3.5
    [3  4]   │            │ [3 4] │
    [6  7]   │                    ┘
             │          /          \
             │     ┌─              ┌─
             │   L │ [0 1] │       │ [6 7] │ R
             │             ┘               ┘
```

To query, check
the intervals that contain the
median and recurse either into the left or
the right subtree. As the median
always halves the number of
intervals, the tree will have `O(log n)` levels.

However, consider what would happen if we query which of the intervals
`[[1 6] [2 5] [3 4]]` contains the number `1.1`. As all the intervals contain the median of `2.5`, our current idea of an interval-tree would have only one level. Clearly, only `[1 6]` contains `1.1`, yet, all intervals on that level are being checked --- the worst-case complexity remains `O(n)`.

### The Refinement

To combat the above issue, interval-trees employ another useful optimization. Consider the example from before of querying `[[1 6] [2 5] [3 4]]`. Mentally, it is natural to visualize the intervals and take everything that starts to the left of `1.1` and then stop. As such, I will call this trick _early stopping_. Like this, only as many operations are performed for reporting intervals as are needed. Formally what is done is that each level stores its intervals in two forms: once sorted by the left endpoints, and once by the right endpoints.

```
          ┌─
          │ [2 5] | [3 4] │ M = 3.5
          │ [3 4] | [2 5] │
                          ┘
        /                  \
      L                      R
```

The query is similar as before, however if the required point is to the left of `M`, one iterates through the intervals sorted by the left and stops once an interval with a greater starting point than the query point is found, and otherwise one does the analogue on the right side. Finally, this achieves a query time of `O(log n + m)`.

## Uiua

Lately, I have found great joy in using Uiua, which is a programming language that is

- _array based_: fundamentally, everything is an array,
- _stack-based_: inputs and outputs are put on a stack, and
- _point-free_: there are no
  parameters or variable
  names within functions.

Since I was curious about the experience of implementing a data-structure in Uiua, I got my hands dirty and went at it. However, before looking at the implementations, let me introduce you to some basic principles of Uiua via examples, which are somewhat self-explanatory if one hovers over the symbols to see their meaning.

{{ uiua(id="0_17_0-dev_1__MSAxICMgRGF0YSBpcyBwdXQgb24gdGhlIHN0YWNrCgorIDEgMSAjIEZ1bmN0aW9ucyBjYW4gYmUgYXBwbGllZCB0byB0aGUgc3RhY2sgCgpb4oqD4oyK4oyIIDAuNV0gICAgICAjIOKKgyBhcHBsaWVzIHR3byBmdW5jdGlvbnMgdG8gdGhlIHRvcCwKW-KKg-KKg-KMiuKMiCgrMSkgMC41XSAjIGFuZCDiioPiioMgYXBwbGllcyB0aHJlZS4KClsrIDEgMV0gICMgRnVuY3Rpb25zIG5vcm1hbGx5IGNvbnN1bWUgaW5wdXRzLCBidXQKW-KfnCsgMSAxXSAjIOKfnCBrZWVwcyBmaXJzdCBhcmd1bWVudCBvbiBvdXRwdXQgYW5kClvipJorIDEgMV0gIyDipJoga2VlcHMgZmlyc3QgYXJndW1lbnQgdW5kZXIgb3V0cHV0Lgo=" height="525") }}

For a full introduction to the language, the [tutorial](https://www.uiua.org/tutorial/introduction) is expansive and provides a good entry point to this wonderful language, whereas the [language tour](https://www.uiua.org/tour) is a rather quick read.

### The Hypothesis

Uiua is an interpreted language. In practice, this means that operations on arrays are fast, while more granular operations which happen on the Uiua level are slower. Just as an example, consider the following two ways to compute the sum the sine values of the first fifty thousand numbers, and their corresponding times. The first one operates largely on the Uiua level, whereas the latter operates more on the rust level. When I ran it, there was a speedup of approximately 300 when opting for the array-based code rather than the one using repeat (`⍥`), which is like a for-loop. As a bonus, the latter is in my opinion quite readable.

{{ uiua(id="0_17_0-dev_1__IyAiTWFudWFsIgpb4o2cbm93KOKXjOKNpSjiioMo4oqZ4peMKzEpKCviiL8pKTVlNCAwIDApXQojIEFycmF5LWJhc2VkClvijZxub3coLyviiL_ih6E1ZTQpXQojIENvbXB1dGUgc3BlZWR1cAril6Eo4oGF4oqiw7cpCg==", height=300) }}

This leads me to one major problem with regard to implementing an interval tree in Uiua: there is no array-based way to "stop" when iterating over the sorted intervals as is done in interval-trees. Thus, the question I would like to address in this work is the following:

> How do different implementations of interval-trees compare in terms of actual execution speed?

## The Implementations

I will make some assumptions. Namely, I will assume that different "non-array-based" ways to do early stopping are similarly performant. That is why I will present only one of them.

As such, I will consider three different ways to solve the line-stabbing problem. First, I will consider the naive solution, that is, just checking every interval. As this can be done using array-based operations, this is still a viable option in Uiua.

Then, I will consider the solution in which something like an interval-tree is queried, however, each level of the tree is checked completely without stopping. As there are only logarithmically many operations on the Uiua side of things, this might still be quite fast.

Lastly, I will consider the textbook implementation of an interval-tree in Uiua, which includes checking intervals on the Uiua level. Asymptotically, this will be the best option, yet I (correctly) conjectured that practically, this will be the slowest.

I list the full code right here, and will elaborate on the individual bindings later in more detail. Recall that hovering over them explains the symbols.

{{ uiua(id="https://www.uiua.org/pad?src=0_17_0-dev_1__4pSM4pSA4pW0SVQKICB-IHtMIENMIENSIFIgTSBBbGx9CiAgTWVkaWFuIOKGkCDDtzIgLysg4oqP4oqfIOKKg-KMiuKMiCDDtzItMeKnuy4g4o2GCiAgSXNMICAgIOKGkCA-4oqj4o2JCiAgSXNSICAgIOKGkCA84oqi4o2JCiAgSXNDcm9zIOKGkCDDl-KKgyjCrElzTCkowqxJc1IpCiAgU0wgICAgIOKGkCDijYYKICBTUiAgICAg4oaQIOKNnOKJoeKHjOKNhgogIEVtcHR5ICDihpAgPTDip7sKCiAgSW5pdCDihpAgfDEgKAogICAgRW1wdHkuCiAgICDiqKwo4p-cKE1lZGlhbuKZrSnijYYKICAgICAgSVTiioPiioPiioPiioMoSW5pdOKWveKkmklzTCko4oqDU0wgU1Lilr3ipJpJc0Nyb3MpKEluaXTilr3ipJpJc1IpKOKLheKImCko4oqZ4peMKQogICAgfCBbXSkKICApCgogIFFOYWl2ZSDihpAg4pa94qSaSVR-SXNDcm9zIElUfkFsbAoKICBRU2ltcGxlIOKGkCB8MiAoCiAgICBFbXB0eS4KICAgIOKorCjiioLiioMo4pa94qSaSXNDcm9zIENMKSjiqKwoUVNpbXBsZSBMfFFTaW1wbGUgUinil6EoPk0pKQogICAgfCBbXSkKICApCgogIEZpbHRlckNMIOKGkCDiipnil4wg4o2i4oaY4oKBIOKNoyjil6EoPOKKouKKoil8MCkg4oeMIENMCiAgRmlsdGVyQ1Ig4oaQIOKKmeKXjCDijaLihpjigoEg4o2jKOKXoSg-4oqj4oqiKXwwKSBDUgogIFFDbGV2ZXIg4oaQIHwyICgKICAgID0w4qe7LgogICAg4qisKOKXoSg-TSkKICAgICAg4qisKOKKguKKgyhGaWx0ZXJDTCkoUUNsZXZlciBMKQogICAgICB8IOKKguKKgyhGaWx0ZXJDUikoUUNsZXZlciBSKQogICAgICApCiAgICB8IFtdCiAgICApCiAgKQrilJTilIDilbQKPyBJVH5Jbml0IFtbMCAxXSBbMiA1XSBbMyA0XSBbNiA3XV0K4oqD4oqDSVR-UU5haXZlIElUflFTaW1wbGUgSVR-UUNsZXZlciA6IDIK", height=1600) }}

The implementation is models interval-trees as a Uiua object with the fields `L,CL, CR, R, M, All`, which stands for _left, crossing sorted by left, crossing sorted by right, right, and all intervals_ respectively.

#### Initialization

The code that initializes an interval-tree is the following.

```
  Init ← |1 (
    Empty.
    ⨬(⟜(Median♭)⍆
      IT⊃⊃⊃⊃(Init▽⤚IsL)(⊃SL SR▽⤚IsCros)(Init▽⤚IsR)(⋅∘)(⊙◌)
    | [])
  )
```

It reads as follows, from top down, right to left: if the given
array is empty, return an empty array. This is
checked using [`⨬ switch`](https://www.uiua.org/docs/switch). Else flatten the array with [`♭ deshape`](https://www.uiua.org/docs/deshape) and compute its median, whilst keeping the
array on top.
At this point in time, the stack has the intervals on it, followed by the median. To the intervals and the median, apply six different functions using a chain of [`⊃ fork`](https://www.uiua.org/docs/fork)s:

- `Init▽⤚IsL`: recursively initialize an interval-tree from those intervals lying on the left of the median, where [`▽ keep`](https://www.uiua.org/docs/keep) selects elements from an array using some binary mask,
- `⊃SL SR▽⤚IsCros` get the intervals that crossing the median, and both sort them by the left and the right using [`⊃ fork`](https://www.uiua.org/docs/fork),
- `Init▽⤚IsL`: recursively initialize an interval-tree from those intervals lying on the right of the median,
- `⋅∘`: keep only the median, and
- `⊙◌`: keep only the intervals using [planet notation](https://www.uiua.org/tutorial/morestack#planet-notation).

The computed values are then stored as the corresponding fields.

### The Naive Query

The naive query `QNaive ← ▽⤚IsCros All` checks for all intervals whether they cross the query point and correspondingly returns them.

### The Simple Query

The simple query `QSimple` employs divide-and-conquer, but does not consider early stopping when scanning for through the sorted intervals.

```
  QSimple ← |2 (
    Empty.
    ⨬(⊂⊃(▽⤚IsCros CL|⨬(QSimple L|QSimple R)◡(>M))
    | [])
  )
```

Again, if the provided array is empty, it returns an empty array. Else, it performs a [`⊂ join`](https://www.uiua.org/docs/join) on the output of two functions which are applied via [`⊃ fork`](https://www.uiua.org/docs/fork):

- `▽⤚IsCros CL` does the same as the naive query, but only with the intervals stored at the corresponding level of the tree sorted by their left point, and
- `⨬(QSimple L|QSimple R)◡(>M)` queries either the left or the right subtree, depending on whether the query point is greater than the median `M`.

### The "Clever" Query

The clever query employs both divide-and-conquer and early stopping.

```
  FilterCL ← ⊙◌ ⍢↘₁ ⍣(◡(<⊢⊢)|0) ⇌ CL
  FilterCR ← ⊙◌ ⍢↘₁ ⍣(◡(>⊣⊢)|0) CR
  QClever ← |2 (
    Empty.
    ⨬(◡(>M)
      ⨬(⊂⊃(FilterCL)(QClever L)
      | ⊂⊃(FilterCR)(QClever R)
      )
    | []
    )
  )
```

The basic code is the very similar, so I will only explain `FilterCL`. Admittedly, it is a complicated function and I do not quite like it. The insight is that instead of taking intervals until some point is exceeded, one can drop intervals to the same point from the other side of the sorted array. As such, first, the function first uses [`⇌ reverse`](https://www.uiua.org/docs/reverse) the array storing the intervals sorted by the left point. Then, [`⍢ do`](https://www.uiua.org/docs/do) executes [`↘₁ take`](https://www.uiua.org/docs/take), which just pops the first element of an array, while the median is less than the first interval's first element with `<⊢⊢`, which uses [`⊢ first`](https://www.uiua.org/docs/first). The condition of the defacto while loop is wrapped in a [`⍣ try`](https://www.uiua.org/docs/try), which defaults to `0` if the array is empty and an exception is raised. Then lastly, the provided query point is just being dropped with planet notation `⊙◌`.

### Experiment and Results

As we will consider execution speeds, let me preface this with a small table with a system specification:

| Spec         | Detail                    |
| ------------ | ------------------------- |
| OS           | macOS Sequoia 15.5 arm64  |
| Host         | MacBook Air (M1, 2020)    |
| Kernel       | Darwin 24.5.0             |
| CPU          | Apple M1 (8) @ 3.20 GHz   |
| GPU          | Apple M1 (7) [Integrated] |
| Memory       | 8.00 GiB                  |
| Uiua version | 0.16.0                    |

The experiment is very simple. Essentially, for each number of intervals (size) in `Sizes`, `PerSize` many trees are generated from a set of intervals uniformly distributed in `[0, 1]`, along with similarly distributed query points. Note here that this randomness is not seeded, as in Uiua, [seeded randomness](https://www.uiua.org/docs/gen) is in my opinion complicated to use, because for each invocation of [`gen`](https://www.uiua.org/docs/gen) (the seeded randomness generator), a different seed must be used.

Then, on each combination of tree and query, the time of executing each of the different queries is measured, and averaged for each size. The output shows the average microseconds for the different kinds of queries. A simplified version that does not keep track of the data for reproducibility and does not handle inputs that are too large is shown below.

{{ uiua(id="0_17_0-dev_1__IyBFeHBlcmltZW50YWwhCgrilIzilIDilbRJVAogIH4ge0wgQ0wgQ1IgUiBNIEFsbH0KICBNZWRpYW4g4oaQIMO3MiAvKyDiio_iip8g4oqD4oyK4oyIIMO3Mi0x4qe7LiDijYYKICBJc0wgICAg4oaQID7iiqPijYkKICBJc1IgICAg4oaQIDziiqLijYkKICBJc0Nyb3Mg4oaQIMOX4oqDKMKsSXNMKSjCrElzUikKICBTTCAgICAg4oaQIOKNhgogIFNSICAgICDihpAg4o2c4omh4oeM4o2GCiAgRW1wdHkgIOKGkCA9MOKnuwoKICBJbml0IOKGkCB8MSAoCiAgICBFbXB0eS4KICAgIOKorCjin5woTWVkaWFu4pmtKeKNhgogICAgICBJVOKKg-KKg-KKg-KKgyhJbml04pa94qSaSXNMKSjiioNTTCBTUuKWveKkmklzQ3JvcykoSW5pdOKWveKkmklzUiko4ouF4oiYKSjiipnil4wpCiAgICB8IFtdKQogICkKCiAgUU5haXZlIOKGkCDilr3ipJpJVH5Jc0Nyb3MgSVR-QWxsCgogIFFTaW1wbGUg4oaQIHwyICgKICAgIEVtcHR5LgogICAg4qisKOKKguKKgyjilr3ipJpJc0Nyb3MgQ0wpKOKorChRU2ltcGxlIEx8UVNpbXBsZSBSKeKXoSg-TSkpCiAgICB8IFtdKQogICkKCiAgRmlsdGVyQ0wg4oaQIOKKmeKXjCDijaLihpjigoEg4o2jKOKXoSg84oqi4oqiKXwwKSDih4wgQ0wKICBGaWx0ZXJDUiDihpAg4oqZ4peMIOKNouKGmOKCgSDijaMo4pehKD7iiqPiiqIpfDApIENSCiAgUUNsZXZlciDihpAgfDIgKAogICAgRW1wdHkuCiAgICDiqKwo4pehKD5NKQogICAgICDiqKwo4oqC4oqDKEZpbHRlckNMKShRQ2xldmVyIEwpCiAgICAgIHwg4oqC4oqDKEZpbHRlckNSKShRQ2xldmVyIFIpCiAgICAgICkKICAgIHwgW10KICAgICkKICApCuKUlOKUgOKVtAoKUmFuZFZlYyAgICAgICDihpAg4omh4ouF4pqC4oehClJhbmRMZW4gICAgICAg4oaQIMOX4oqD4pqCKC06MSkKUmFuZEludGVydmFscyDihpAg4o2J4oqfOuKKuCviirhSYW5kTGVuIFJhbmRWZWMKClNpemVzICAgICAgICDihpAgWzEgMjAgMzAgNTAgMTAwXQpUcmVlc1BlclNpemUg4oaQIDEwClF1ZXJpZXMgICAgICDihpAgUmFuZFZlYyAxMApUcmVlcyAgICAgICAg4oaQIOKZreKCguKKnihJVH5Jbml0IFJhbmRJbnRlcnZhbHMg4oqZ4peMKVNpemVz4oehVHJlZXNQZXJTaXplCgp-U2FtcGxlIHtOIFMgQyBRIFNpemV9CgrOvHMg4oaQIOKBhcOXMWU2CgpCZW5jaG1hcmsg4oaQICgKICBJVCEo4oqD4oqDKAogICAgICDijZxub3dRTmFpdmUKICAgIHwg4o2cbm93UVNpbXBsZQogICAgfCDijZxub3dRQ2xldmVyCiAgICApKQogIOKIqeKCgyjiipnil4zOvHMpCikKCkV4cGVyaW1lbnQg4oaQIOKZreKCgiDiip4oU2FtcGxlIOKKg-KKg0JlbmNobWFya-KLheKImCjip7tJVH5BbGwpKQp-T3V0cHV0IHtRTmFpdmUgUVNpbXBsZSBRQ2xldmVyIFNpemV9CkZtdEV4cCDihpAgU2FtcGxlIShPdXRwdXQg4oqD4oqD4oqDKOKJoU4pKOKJoVMpKOKJoUMpKOKJoVNpemUpKQpBdmcgICAg4oaQIMO3IOKKgyjip7spKC8rKQpSZXBvcnQg4oaQIEZtdEV4cCDiipxBdmcg4oq44omhU2FtcGxlflNpemUKUmVwb3J0IEV4cGVyaW1lbnQgVHJlZXMgUXVlcmllcwo=", height=2200) }}

The actual experiments and code (find them on [GitHub](https://github.com/tmpr/ituiua)) include tests, try to be a little more reproducible and run with larger input parameters, namely the following:

```
Sizes   ← [1 20 30 50 100 1000 5000 10000 50000 100000]
PerSize ← 20
Queries ← RandVec 100
```

That is, the execution time of two-thousand queries are being averaged for each size. Additionally, I have disabled `QClever` for large inputs, as it is evidently too slow. The results are the following:

```
┌─
  QNaive   │QSimple  │QClever │Size
  ─────────┼─────────┼────────┼──────
  6.955    │17.6015  │20.651  │1
  6.4035   │33.249   │47.4755 │20
  5.669    │34.536   │50.9865 │30
  5.8285   │31.1215  │58.21   │50
  5.9675   │32.9605  │87.046  │100
  108.922  │104.7035 │895.5505│1000
  191.6165 │230.797  │NaN     │5000
  259.0495 │336.595  │NaN     │10000
  1078.8005│912.488  │NaN     │50000
  3106.4595│2384.619 │NaN     │100000
  8603.283 │5317.4415│NaN     │500000
                                      ┘
```

It turns out that for at least fifty thousand intervals, it makes some sense to use `QSimple` instead of `QNaive`. Not quite the massive improvement it should be, but this is probably due to the fact that `QSimple` is still an `O(n)` algorithm.

### Conclusion

Interval-trees are a
beautiful and very simple
data-structure. I believe
they are one of the
data-structures which are
very easy to grasp quite
intuitive. It was fun to
implement them in Uiua,
albeit that the language does not yet allow for an idiomatic implementation of proper interval-trees. However, I believe that for this exact task, Uiua might not be the language. Recall that the problem is
that there is no fast way to check the sorted intervals up to a certain point and then stop.

First, a solution that I considered was to propose for `<` to benefit from the [sortedness
flag](https://www.uiua.org/docs/optimizations#sortedness-flags),
which has been introduced recently; if an element in a sorted array is greater than some number, then so are all the elements that follow it. This would allow for some optimization. The mask output by `<` could then be used to filter the array. However, as demonstrated below, `<` is already roughly as fast as it is to merely generate a range; the sortedness optimization would thus not help much. The only thing that actually works quickly in this regard is to use the [`↙ take`](https://www.uiua.org/docs/take) operator, which takes a given number of elements from the start of an array.

{{ uiua(id="0_17_0-dev_1__4oqZ4peM4o2cbm93KDwyMDApIMKw4o2GIOKHoSAxZTUgIyBDaGVja2luZyBzaHVmZmxlZAriipnil4zijZxub3coPDIwMCkg4o2GIOKHoSAxZTUgICMgQ2hlY2tpbmcgc29ydGVkCuKKmeKXjOKNnG5vdyjih6EgMWU1KSAgICAgICAgICMgR2VuZXJhdGluZyBhcnJheQriipnil4zijZxub3co4oaZIDIwMCkg4oehIDFlNSAgICMgVGFraW5nIAo=", height=300) }}

How to go about this? To me, it seems as if the only
sensible addition to the language to solve this specific issue would
be some way to bisect a sorted array.
By that, I mean some primitive to perform a binary search for the
index at which some element should be
inserted. Providing the found index to [`↙ take`](https://www.uiua.org/docs/take) on
every level of an interval-tree would
yield an algorithm that is `O(log^2 n +
m)`, which is not as good as
interval-trees should be in theory, but
probably good enough in practice. I have
created a corresponding issue for that;
let us see if there will be an update to
the language.
