# More notes on comparison approach

Some of this has been moved into the README, but not all since some of it is about future ideas.

```python
import generic_pandas.pandas as pandas
from generic_pandas import record

def f():
   x = pandas.DataFrame(...)
   record("x", x)
   y = lalalala(x)
   record("y", y)

f()
```

And then

```shell-session
$ PANDAS=pandas python mycode.py
$ PANDAS=modin python mycode.py
$ pandas-diff
```

(In practice will be more complex, will need virtualenvs for upgrade case, need runner script.)

Minimal version:

1. support different implementations of pandas, e.g. modin: can I switch?
2. support different versions of pandas: can I upgrade?
3. run same code with multiple versions or variants
4. show a diff
5. runs locally

Next iteration:

Diffing is a hard problem. Maybe user configurable validation, floating point is a pain, etc..

Further features:

Make it a service, with N-dimensional matrix of things to run on, ala godbolt (OS/Python version/library versions/...), run on all of them. Maybe compare performance.

Even further features:

Help user figure out correctness, not just non-variance; diffing/validation strategies start pointing that in directioin. E.g. given good semantic diffing/validation, starts looking like basis for metamorphic testing framework.

Capacity testing.
