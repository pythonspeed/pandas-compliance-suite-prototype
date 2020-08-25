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


