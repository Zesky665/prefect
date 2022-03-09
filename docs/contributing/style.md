# Code style and practices

Generally, we follow the [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html). This document covers sections where we differ or where additional clarification is necessary.

## Imports

A brief collection of rules and guidelines for how imports should be handled in this repository.

### Imports in `__init__` files

Leave `__init__` files empty unless exposing an interface. If you must expose objects to present a simpler API, please follow these rules.

#### Exposing objects from submodules

If importing objects from submodules, the `__init__` file should use a relative import. This is [required for type checkers](https://github.com/microsoft/pyright/blob/main/docs/typed-libraries.md#library-interface) to understand the exposed interface.

```python
# Correct
from .flows import flow
```

```python
# Wrong
from prefect.flows import flow
```

#### Exposing submodules

Generally, submodules should _not_ be imported in the `__init__` file. Submodules should only be exposed when the module is designed to be imported and used as a namespaced object.

For example, we do this for our schema and model modules because it is important to know if you are working with an API schema or database model, both of which may have similar names.

```python
import prefect.orion.schemas as schemas

# The full module is accessible now
schemas.core.FlowRun
```

If exposing a submodule, use a relative import as you would when exposing an object.

```
# Correct
from . import flows
```

```python
# Wrong
import prefect.flows
```

#### Importing to run side-effects

Another use case for importing submodules is perform global side-effects that occur when they are imported.

Often, global side-effects on import are a dangerous pattern. Avoid them if feasible.

We have a couple acceptable use-cases for this currently:

- To register dispatchable types, e.g. `prefect.serializers`.
- To extend a CLI application e.g. `prefect.cli`.

### Imports in modules

#### Importing other modules

The `from` syntax should be reserved for importing objects from modules. Modules should not be imported using the `from` syntax.

```python
# Correct
import prefect.orion.schemas  # use with the full name
import prefect.orion.schemas as schemas  # use the shorter name
```

```python
# Wrong
from prefect.orion import schemas
```

Unless in an `__init__.py` file, relative imports should not be used.


```python
# Correct
from prefect.utilities.foo import bar
```

```python
# Wrong
from .utilities.foo import bar
```

Imports dependent on file location should never be used without explicit indication it is relative. This avoids confusion about the source of a module.

```python
# Correct
from . import test
```

```python
# Wrong
import test
```

#### Resolving circular dependencies

Sometimes, we must defer an import and perform it _within_ a function to avoid a circular dependency.

```python
## This function in `settings.py` requires a method from the global `context` but the context
## uses settings
def from_context():
    from prefect.context import get_profile_context

    ...
```

Attempt to avoid circular dependencies. This often reveals overentanglement in the design.

When performing deferred imports, they should all be placed at the top of the function.

##### With type annotations

If you are just using the imported object for a type signature, you should use the `TYPE_CHECKING` flag.

```python
# Correct
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from prefect.orion.schemas.states import State

def foo(state: "State"):
    pass
```

Note that usage of the type within the module will need quotes e.g. `"State"` since it is not available at runtime.

#### Importing optional requirements

We do not have a best practice for this yet. See the `kubernetes`, `docker`, and `distributed` implementations for now.

#### Delaying expensive imports

Sometimes, imports are slow. We'd like to keep the `prefect` module import times fast. In these cases, we can lazily import the slow module by deferring import to the relevant function body. For modules that are consumed by many functions, the pattern used for optional requirements may be used instead.