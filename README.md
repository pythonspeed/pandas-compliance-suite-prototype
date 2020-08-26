# Pandas API Compliance: Prototyping and Design

1. There are now five different projects with Pandas-like APIs (maybe more?): Pandas, Modin, Dask, cuDF, Koalas.
2. The Pandas API is huge.
3. It would therefore be great to have some way of saying "you can switch to Modin/Dask/etc. with no trouble".

The goal of this repository is to think through how to do this, and prototype an implementation or implementations.

## Goals

There are multiple goals, for different audiences:

* `USER-GOAL-CAN-SWITCH`: Application users of Pandas might want to switch to another library like Modin.
  Will their code work? Are all the APIs supported? It's not clear.
  Some projects list compatible API calls, but this is still insufficient given how complex some Pandas APIs are in terms of allowed inputs.
* `MAINTAINER-GOAL-ADDRESS-INCOMPAT`: Maintainers of libraries like Modin want to know what edge cases in the Pandas API they aren't supporting at all, or aren't supporting correctly, so they can fix those libraries.

## Non-goals

The Data APIs consortium has a similar but somewhat different project, to figure out the _intersection_ of APIs across multiple libraries.
However:

1. It also includes non-Pandas-compatible dataframes like Vaex.
2. It's about the minimal compatible subset.
3. It's aimed at library users.

The goal here is not "what is intersection of all dataframe APIs", but rather "does Dask/Koalas/Modin/cuDF actually support same API as Pandas", perhaps globally, perhaps on a per-application basis.

## Potential UX

### User experience for `USER-GOAL-CAN-SWITCH`

`UX-JUST-WORK`: The ideal user experience is to take some Pandas code, switch an import or two, and have it Just Work.

The next best outcome is to be told where specifically this particular piece of code is incompatible with the alternative library.
This can be done in multiple ways:

* `UX-STATIC-MISMATCH-NOTIFICATION`: Via static analysis, perhaps aided by type hints.
* `UX-RUNTIME-MISMATCH-NOTIFICATION`: At runtime. For example, getting an `StillUnsupportedError` exception when doing `df[x] = 1` for a certain value of `x`.
  This is more accurate than static analysis, maybe, but also less useful: it matters a lot whether it's 1 API that's missing or 50.
  
The next, lower level of user experience is to be presented with a chart of supported APIs, and the user can then manually validate whether an API is supported:

* `UX-LIST-OF-SUPPORTED-METHODS-DETAILED`: A list of supported methods/APIs, including all the various possible inputs (Pandas has so many!).
* `UX-LIST-OF-SUPPORTED-METHODS`: As above, but just a list of supported methods. This is e.g. the current default in Modin, and it's quite insufficient since sometimes certain input variants don't work, even if others do.

### User experience for `MAINTAINER-GOAL-ADDRESS-INCOMPAT`

As a maintainer wanting to make sure all APIs are supported, the equivalent of `UX-LIST-OF-SUPPORTED-METHODS-DETAILED` should suffice.

### Choosing a UX

`UX-STATIC-MISMATCH-NOTIFICATION` or `UX-RUNTIME-MISMATCH-NOTIFICATION` seem like the best options.
In terms of information they require, they need the same information as `UX-LIST-OF-SUPPORTED-METHODS-DETAILED`, and they just require additional coding work to utilize that information.

## Next steps

Per the above, the initial bottleneck is being able to say in detail which APIs a particular library does or does not support.
Therefore, good next steps would be:

1. Prototype some way to gather the detailed API compatibility, probably for Modin.
2. Then, prototype using that information for static analysis, or perhaps runtime errors.


## Gathering compatibility information

### What information do we need?

The fundamental API atoms, the thing that can be supported or not supported, are specific a function/method being called with _specific types_.
Or rather:

1. For some method arguments, the type isn't meaningful: any type will do.
   For example, `Series.add` will take anything that can be added, and one expects the underyling implementation to devolve to a plain `+`. Put another way, the same logic handles all types.
2. For other arguments, the behavior is different based on the type.
   For example, `df[x] = 1` has different behavior depending on whether `x` is a string, a `DataFrame`, a list, a boolean `Index`, etc..
   In typing terms, this would be `Union[str, DataFrame, Index, list]`, although at the moment I'm  not sure one can express a boolean Index...

The information we need is then:

1. Supported Method/function argument variations for the Pandas API.
2. Supported method/function argument variations for the other library's API.

Combined, one can determine compatibility.

There is also, of course, the semantics: even if the same signatures are supported, the results might be different.

### Sources of information

Information about compatibility can come from multiple sources:

* Type annotations of the Pandas API.
  Insofar as they're missing, they could be expanded to be more complete.
* Type annotations of the emulating API, e.g. Modin.
* Pandas test suite, which to some extent is generic test of the API.
* The emulating library's test suite.

### Some potential approaches

#### COMPARE-TYPE-ANNOTATION

Let's assume complete and _thorough_ type annotation for Pandas is available.
For example:

```python
class DataFrame:
    # ...
    def __setitem__(self, key: Union[DataFrame, Index, ndarray, List[str], str, slice, callable], value: Any):
       # ...
```

That's not actually complete; the callable needs to only return the other items in the Union, and there are more variants.
What's more this _needs_ to be complete: if one just said `Iterable` or something that wouldn't allow good checking.

We can then look at type annotations for e.g. Modin:

```python
class DataFrame:
    # ...
    def __setitem__(self, key: Union[str, List[str]], value: Any):
        # ....
```

Obviously there are inputs that aren't supported, and an automated tool could extract those.

Now, Modin could lie about what supports, of course, either on purpose (unlikely) or more likely by accident.
Some ways of dealing this:

1. Manual self-discipline, adding type annotations if and only if they have a corresponding test.
2. In addition, or as alternative, some validation of the supported types must be done based on the Modin test suite.
   There are tools that generate type annotations from running code, whereas this is... the opposite, but probably some of that could be reused.

The benefits of this approach are that each project can work in isolation, in parallel.
It also should work with less-exact emulations like Dask's lazy approach.

The downsides is that it only validates _signatures_, not semantics.
However, if coupled with good testing discipline on the part of the reimplementation libraries, that's OK (albeit duplication of work).

#### REUSE-PANDAS-TEST-SUITE

Pandas has a test suite.
Some of that test suite is testing the public API.

This test suite could be modified to allow running against multiple libraries, so one could see which tests Modin fails.

Benefits are that you get to see that semantics are identical across libraries.

Some problems:

1. Test failures don't necessarily easily match one-to-one to API calls.
   This would need to be annotated.
2. Easier to solve, but annotations of "expected to fail" would need to be maintained outside the tests by the reimplementing libraries.
3. Would have to figure out how to sync up generic test repo with Pandas repo, unless somehow Pandas tests are made generic.

#### NEW-COMPLIANCE-TEST-SUITE

Instead of reusing Pandas' test suite, one could create a completely new one.

This sounds like a lot of work.

#### What else? There are probably more

## Proposal

### 1. Pandas adds highly-specific full-coverage type annotations

By highly-specific, I mean e.g. using `@override` to specify multiple different input/output pairs, using types like `Index[bool]` if a boolean Index is special, etc..

This is a bunch of work, but it's a good documentation practice and will also benefit people who are just using Pandas.
So seems like an easy sell.

### 2. Modin etc. add highly-specific full-coverage type annotations

1. Modeled on Pandas.
2. Careful to only add type annotations for things that are actually tested.

This is a bunch of work, but it's a good documentation practice and will also benefit people who are just using Modin.

### 3. Users of Pandas can use static analysis (mypy etc.) to validate that switching to Modin will work

Simply by having items 1 and 2, switching import from Pandas to Modin will allow type checking if APIs are compatible.

No additional work needed by maintainers.

### 4. Modin etc. can optionally have a runtime checking mode for users attempting to switch from Pandas to Modin.

Static analysis may be difficult for some users.

So using e.g. [typeguard](https://pypi.org/project/typeguard/), maintainers of Modin etc. can enable runtime checking with some sort of API flag or environment variable, so there's no cost by default.

This is a small amount of work, probably.

### 5. A new tool can generate diffs between type annotations for Pandas and type annotations for Modin etc., for documentation purposes and for `MAINTAINER-GOAL-ADDRESS-INCOMPAT`.

This will require some software development, but seems like a nicely scoped project.
