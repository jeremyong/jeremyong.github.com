---
layout: post
title: The C++20 Naughty and Nice List for Game Devs
date: 2023-12-24 00:00
categories:
  - C++
---

The goal here is to compile a list of C++20 features I think game devs should probably
use (possibly with caveats), and features I think game devs will probably want to avoid.
You're probably curious "why now?" given that it's 2023 and C++20 was standardized
several years ago. Well, mainly it's because it took me several years of trying and retrying
various features to get a sense for what I liked and what I didn't.

I'm going to start with a disclaimer, because whenever feedback regarding C++
matters is concerned, things can get touchy, but please do read the following
before reacting too strongly to anything that follows. The main discalimer is
that I'm assuming that the interested reader is a developer with similar codebase
requirements:

- Large codebase (10M+ LOC) that is routinely compiled on your machine
- Heavy DLL usage for plugins and general code structure modularization
- Broad platform compatibility
- Sensitivity to disk usage and binary bloat
- Runtime efficiency is of utmost importance (`-O0` speed is also important)

To the extent that your experience as a developer (or game developer) matches the
requirements above, you may be more inclined to agree with my naughty-and-nice
classification. I think without such constraints, some of the "naughty features" here
are honestly quite nice.

The second disclaimer is that even if your requirements seem to match mine,
you can still come up with diametrically opposite conclusions, and that's OK also.
Even if codebases "rhyme," coding culture and practices differ greatly from team
to team, so you'll need to apply the _reasoning_ used to classify features as "naughty"
or "nice" to your own use cases to determine what applies and to what extent.

The next disclaimer is that even if a feature is on the nice list, it doesn't mean
the feature can't be abused, and it doesn't mean the feature isn't naughty in some
contexts. Similarly, features in the naughty list may not be naughty in all
circumstances.

The last disclaimer is that this post is not even remotely exhaustive. There is far
too much material to really cover in detail in a single post, so this just covers
some of the broad strokes. If I miss your favorite feature (or most hated feature),
it's either because there wasn't room to cover it, or because I don't have enough
personal experience with the feature to draw a conclusion one way or another.

With that out of the way, let's start with the nice items, followed immediately
by the naughty items.

## (Nice) Default comparison and the three-way comparison operator (aka the spaceship: <=>)

New in C++20 are [default comparison](https://en.cppreference.com/w/cpp/language/default_comparisons)
rules for structured types that perform an automatic lexicographic comparison when requested.

```c++
struct Date
{
    int year;
    int month;
    int day;

    auto operator<=>(Date const&) const = default;
    bool operator==(Date const&) const  = default;
};
```

With the declaration above, instances of `Date` are comparable with any of the binary comparison
operators, with the behavior you expect (don't actually define a `Date` class this way of course).
The [three-way comparison operator](https://en.cppreference.com/w/cpp/language/operator_comparison#Three-way_comparison)
there defines a single binary operator which then enables a family of binary comparison operators
on a given object. This operator can of course be overloaded.

These features are a win in my book because combined, they cut down on code duplication,
reducing the surface area for bugs as code changes. Lexicographic comparison is a great
default, because what other behavior would have been a more sensible default?

The main caveat is that I would not use the 3-way comparison operator for custom numeric types
where maximum efficiency is needed, because there may be slight overhead in debug builds,
but chances are, such types are likely small or change infrequently (such that the advantages
of the 3-way comparison operator with respect to maintainability are not felt). As an additional
note to be aware of, in most cases, you probably want to at least implement `operator==`, which
may be faster than invoking `operator<=>` to do equality checks.

## (Nice) Signed integers are 2's complement and arithmetic shifts

Signed overflow/underflow remain UB (and it's understandable that changing this behavior would have
dramatic consequences), but it's nice to know that we can at least expect sign extension when using
`operator>>` and `operator<<` on signed operands now.

## (Nice) Coroutines (with heavy caveats)

C++20 [coroutines](https://en.cppreference.com/w/cpp/language/coroutines) are one of my favorite
features. It's been also wildly criticized for its complexity (or because they aren't stackful, etc.).
However, while I do think the criticisms are understandable, coroutines can work remarkably well
_as a user_, with similar and often better overhead than what we'd see in a typical task-graph
implementation.

The main downside is that there's a heavy upfront setup cost (and possibly maintenance cost) to
bootstrap your coroutine-based frontend API for task scheduling.
I recommend checking out [concurrencpp](https://github.com/David-Haim/concurrencpp)
if you're in the market for frameworks that provide a coroutine interface, or if you want to see
the type of constructs that are possible.

Another downside to coroutines are that the dependency graph is implicit as opposed to explicit,
so if you have tooling to visualize the dependency graph and such, this viz is going to have to
operate using runtime data.

## (Nice) Constraints and Concepts

Sure there is the dreaded [`requires requires`](https://stackoverflow.com/questions/54200988/why-do-we-require-requires-requires)
syntactic oddity that shows up, but on the whole, [constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints)
should show up pretty much anywhere you have a template (which is hopefully used in your codebase
only where needed already). In exchange for the effort of writing constraints and concepts (all
of I recommend naming as opposed to writing inline concepts), you are rewarded with nicer error
feedback from the compiler _forevermore_ which seems pretty worth if you ask me. In addition, the
template type signatures, when decorated with constraints, now inform the user what types are usable
as valid type parameters. This is a strict win in my book.

The concepts that ship with the compiler are also immediately usable. I wouldn't recommend writing these yourself,
since you may be missing out on optimizations available through the use of compiler intrinsics.

## (Nice) `<bit>`

The [`<bit>` header](https://en.cppreference.com/w/cpp/header/bit) is nice and standardizes a lot
of operations we arguably should have had years ago (like `popcount` and bit rotations and such).
The _very_ important addition here is the inclusion of [`std::bit_cast`](https://en.cppreference.com/w/cpp/numeric/bit_cast)
which lets us cast unrelated types safely without relying on the `memcpy` pattern. In MSVC for example,
this is implemented using the MSVC `__builtin_bit_cast` intrinsic.

## (Nice) `<numbers>`

You now have `pi` and other constants available for use in a [header](https://en.cppreference.com/w/cpp/header/numbers).

## (Nice) New synchronization primitives

The new [`<barrier>`](https://en.cppreference.com/w/cpp/header/barrier),
[`<latch>`](https://en.cppreference.com/w/cpp/header/latch), and [`<semaphore>`](https://en.cppreference.com/w/cpp/header/semaphore)
primitives have been working for me with the caveat that if you already have well-tested cross-platform
implementations of these primitives, I would keep using those. A potential disadvantage is that using
these primitives requires you to buy into the `<chrono>` way of spelling time points and durations,
which I've grown accustomed to but isn't for everyone.

## (Nice) `<span>`

Passing non-owning views is a good idea in general, so it's nice that the notion of
[`std::span`](https://en.cppreference.com/w/cpp/container/span) is now officially codified. This isn't going
to convince me to use STL containers anytime soon (for other reasons I won't get into), but the implementation
of `std::span` as a pointer plus size is unobjectionable enough and easy-to-use (interoperability with functions
that take iterator pairs, range for-loops, etc.).

## (Nice-ish) Designated initializers

[Designated initializers](https://en.cppreference.com/w/cpp/language/aggregate_initialization#Designated_initializers) are
a new form of initialization that initializes structured variable members by name.

```c++
struct Point
{
    float x;
    float y;
    float z;
};

Point origin{.x = 0.f, .y = 0.f, .z = 0.f};
```

I consider this feature "nice-ish." On the one hand, the feature allows us to initialize structured variables in
a self-documenting manner, and I'm always a fan of optimizing for the reader. However, the main caveat regarding
designated initializer usage is that initialized members _must_ appear in declaration order. In other words,
this code won't compile:

```c++
struct Point
{
    float x;
    float y;
    float z;
};

// Oops, field order is incorrect.
Point origin{.y = 0.f, .z = 0.f, .x = 0.f};
```

This behavior makes sense for structured types with non-trivial layout, but feels unnecessarily restrictive
for POD types. I often work with types that have dozens of fields or more, so honoring the initialization
order to match the original declaration is quite tedious indeed. Furthermore, in game dev, we often move
structure fields around in order to optimize layout (coalescing hot data in cache lines, promoting true
sharing, avoiding false sharing). This type of optimization can't be done without cascading into compilation
failures throughout the codebase. As a result, designated initializers have the paradoxical effect of making a
codebase _less maintainable_ in a sense.

This one is still _barely_ in the nice category because it does make life better for the reader (who we
all prioritize over the writer), but I do hope that restriction is relaxed for trivial types some day. Flexibility
with ordering was the entire point of the "named parameters" feature in many other languages after all.

## (Naughty) char8_t

MSVC added `/Zc:char8_t-` to disable this type entirely, and GCC did the same.
The main reason `char8_t` was introduced was to provide a dedicated
type for unicode data. Unfortunately, as specified, unicode conversions to `char*` were disallowed,
thereby breaking a lot of compatibility with `char*` interfaces that expected users to pass `u8""` strings.
This famously caused a bit of drama by breaking a number of
[Dear ImGui](https://github.com/ocornut/imgui) APIs ([wayback twitter thread](https://web.archive.org/web/20220104084024/https://twitter.com/ocornut/status/1377986353498095618)).
I will continue to abide by the guidelines summarized
[here](https://utf8everywhere.org/) for UTF-8 matters (unless I'm forced to live in a `TCHAR` world).
That said, the `char8_t` proposal in general has my sympathy, since it _would be nice_ to have the unicode
story fully straightened out, but c'est la vie.

## (Naughty-ish) [[no_unique_address]]

This is naughty _specifically_ because of MSVC, where you continue to need to use `[[msvc::no_unique_address]]`
for [reasons](https://devblogs.microsoft.com/cppblog/msvc-cpp20-and-the-std-cpp20-switch/#c20-no_unique_address).

Otherwise, you probably want this for things like stateless allocators that are used as data structure
members but shouldn't occupy actual memory.

## (Naughty) Modules

C++20 [modules](https://en.cppreference.com/w/cpp/language/modules) in its current state are
unfortunately a big disappointment. I do hope they eventually get to a better place, but this may be difficult
for as long as the C++ standard continues to ignore shared linkage and DLLs. Here's a
[StackOverflow answer](https://stackoverflow.com/questions/52286991/what-is-the-expected-relation-of-c-modules-and-dynamic-linkage/74444920#74444920)
I wrote some time ago to describe the problem. Essentially, module and DLL linkages are completely orthogonal
to each other, so if you're in the business of maintaining a codebase where you'd need both (as most game
devs are), expect there to be a lot of build complexity juggling symbol visibilty and linkage types.
While the problematic `jmp` indirection mentioned in that StackOverflow answer can be eliminated with full LTO
(and possibly thin LTO), I suspect most game devs find LTO to be a bit too heavy for frequent usage anyways.

## (Naughty) `<format>`

The [`<format>`](https://en.cppreference.com/w/cpp/header/format) header is probably great in many contexts,
but I would not personally use it for a game engine. Once you get in the business of making an API that
encourages custom formatter specification to live in a template, IMO, you've already lost. Using `std::format`
and `fmtlib` which the standard is based on, I saw compile times and binary sizes increase dramatically
in a way that does not scale across a gigantic codebase (to my satisfication). Relying on the linker to
drop duplicate symbols is "an approach" but not one I'm a fan of. It's _incredibly wasteful_ to ask your
compiler to compile copies of your code and write them to disk for every translation unit that requires
a custom formatter, only to then request the linker to read it all back and drop duplicates (not to mention
PDB sizes). Of course, it's possible to declare custom formatter templates and implement them in a source file,
but again, this isn't encouraged by the API, and I don't believe it's possible to rely on raw discipline to prevent
that type of code bloat from accruing across a large team.

A preferable interface (I use, but also others AFAIK) is to check the type in a template (no choice there),
and dispatch the formatting routine to somewhere that lives in a single translation unit.

## (Naughty) `<ranges>`

I tried [`<ranges>`](https://en.cppreference.com/w/cpp/header/ranges) in a medium-ish codebase and benchmarked
the compiler about a year ago. I wasn't remotely happy with the results and reverted the changes, YMMV.
An incredible amount of work has gone into this, but it sort of reinforces my belief that for some constructs,
templates are a valuable prototyping tool, but not the endgame. Personally, I find code that leverages
ranges _harder_ to read, not easier, because lambdas inlined in functions introduce new scopes that have a
strong non-linearizing effect on the code. This isn't a criticism of ranges per se, but certainly is a stylistic
preference.

## (Naughty) `<source_location>`

The new [`<source_location>`](https://en.cppreference.com/w/cpp/utility/source_location) feature lets
you remove a macro which would have historically expanded `__FILE__` and `__LINE__` directives and replace
it with a `std::source_location` which is populated by a default argument (spelled `std::source_location::current`).
This isn't overwhelmingly naughty, but I don't see it as a dramatic improvement either. Chances are,
your log statements need to remain macros since you need a way to strip those statements at build time
depending on the build type anyways. The main drawback of `std::source_location` is that compared to
`__FILE__`, where the file length is statically knowable, `std::source_location::file_name` returns
a `const char*` string with a file length that _isn't_ statically knowable.

IMO, the new `std::source_location` "pattern" makes function signatures a lot uglier, and would be
better served with a separate mechanism for querying PDBs and stack frame data. Luckily,
[stacktrace](https://en.cppreference.com/w/cpp/header/stacktrace) is on the horizon, but I doubt
we'll get any cross-platform interfaces for querying DWARF/PDB files anytime soon (wouldn't that be
lovely).

# Conclusion

On the whole, I view C++20 as an impactful and positive change, albeit with a few hiccups here
and there (understandable given the sheer scale of the standardization effort). It's also likely
the case that some features I view as misses, might be viewed as absolute godsends by other engineers
with different requirements or coding practices to my own. Finally, as a reminder, there were plenty
of changes in C++20 that weren't covered here, either because I lack personal experience with the feature,
or because it's a feature unlikely to be relevant to game devs specifically. With that, I encourage
readers to take this list with a grain of salt, try things out, and draw your own conclusions.
