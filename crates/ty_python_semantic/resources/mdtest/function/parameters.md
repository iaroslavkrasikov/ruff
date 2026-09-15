# Function parameter types

## Basic

Within a function scope, the declared type of each parameter is its annotated type (or Unknown if
not annotated). The initial inferred type is the annotated type of the parameter, if any:

```py
def f(declared: int, unannotated):
    reveal_type(declared)  # revealed: int
    reveal_type(unannotated)  # revealed: Unknown
```

The variadic parameter is a variadic tuple of its annotated type; the variadic-keywords parameter is
a dictionary from strings to its annotated type:

```py
def g(*args: int, **kwargs: int):
    reveal_type(args)  # revealed: tuple[int, ...]
    reveal_type(kwargs)  # revealed: dict[str, int]
```

## Unannotated parameters with defaults

If there is no annotation but there is a default value, the inferred parameter type is the union of
the inferred type of the default value and `Unknown`:

```py
def f(a="foo", b=0, c=True, d=None):
    reveal_type(a)  # revealed: Unknown | Literal["foo"]
    reveal_type(b)  # revealed: Unknown | Literal[0]
    reveal_type(c)  # revealed: Unknown | Literal[True]
    reveal_type(d)  # revealed: Unknown | None
```

This means that the code in the function body needs to handle the case where the parameter is set to
the default value:

```py
def g(x=0):
    print(x + 1)

    # error: [unsupported-operator] "Operator `+` is not supported between objects of type `Unknown | Literal[0]` and `Literal["foo"]`"
    x + "foo"
```

But it still allows callers to pass in arguments of a wider type:

```py
g(1.5)  # fine
```

## Parameter kinds

```py
from typing import Literal

def f(a, b: int, c=1, d: int = 2, /, e=3, f: Literal[4] = 4, *args: object, g=5, h: Literal[6] = 6, **kwargs: str):
    reveal_type(a)  # revealed: Unknown
    reveal_type(b)  # revealed: int
    reveal_type(c)  # revealed: Unknown | Literal[1]
    reveal_type(d)  # revealed: int
    reveal_type(e)  # revealed: Unknown | Literal[3]
    reveal_type(f)  # revealed: Literal[4]
    reveal_type(g)  # revealed: Unknown | Literal[5]
    reveal_type(h)  # revealed: Literal[6]
    reveal_type(args)  # revealed: tuple[object, ...]
    reveal_type(kwargs)  # revealed: dict[str, str]
```

## Unannotated variadic parameters

...are inferred as tuple of Unknown or dict from string to Unknown.

```py
def g(*args, **kwargs):
    reveal_type(args)  # revealed: tuple[Unknown, ...]
    reveal_type(kwargs)  # revealed: dict[str, Unknown]
```

## Annotation is present but not a fully static type

If there is an annotation, we respect it fully and don't union in the default value type.

```py
from typing import Any

def f(x: Any = 1):
    reveal_type(x)  # revealed: Any
```

## Default value type must be assignable to annotated type

The default value type must be assignable to the annotated type. If not, we emit a diagnostic, and
fall back to inferring the annotated type, ignoring the default value type.

```py
# error: [invalid-parameter-default]
def f(x: int = "foo"):
    reveal_type(x)  # revealed: int

# The check is assignable-to, not subtype-of, so this is fine:
from typing import Any

def g(x: Any = "foo"):
    reveal_type(x)  # revealed: Any
```

## Implicit None values disabled by default

A literal `None` default does not make an annotated parameter optional by default. Both the default
and an explicit `None` argument are rejected, and the parameter keeps its annotated type in the
body.

```py
# error: [invalid-parameter-default]
def f(x: int = None):
    reveal_type(x)  # revealed: int

f()
f(1)
f(None)  # error: [invalid-argument-type]
```

## Implicit None values explicitly disabled

Explicitly disabling `implicit-none-value` preserves the default behavior.

```toml
[analysis]
implicit-none-value = false
```

```py
# error: [invalid-parameter-default]
def f(x: int = None):
    reveal_type(x)  # revealed: int

f()
f(1)
f(None)  # error: [invalid-argument-type]
```

## Implicit None values enabled

```toml
[analysis]
implicit-none-value = true
```

### Parameter kinds

With `implicit-none-value` enabled, a literal `None` default widens an annotated parameter to
include `None` in both the body and the call signature. This applies to positional-only,
positional-or-keyword, and keyword-only parameters.

```py
def f(a: int = None, /, b: int = None, *, c: int = None):
    reveal_type(a)  # revealed: int | None
    reveal_type(b)  # revealed: int | None
    reveal_type(c)  # revealed: int | None

f()
f(1, 2, c=3)
f(None, 2, c=3)
f(1, None, c=3)
f(1, b=None, c=None)
f("wrong")  # error: [invalid-argument-type]
f(b="wrong")  # error: [invalid-argument-type]
f(c="wrong")  # error: [invalid-argument-type]
```

### Narrowing and assignment in the body

The widened type can be narrowed like an explicit union. The parameter's declared type also includes
`None`, so assigning `None` after narrowing is valid, but assigning an unrelated type is not.

```py
def f(x: int = None) -> int:
    reveal_type(x)  # revealed: int | None
    if x is None:
        reveal_type(x)  # revealed: None
        return 0

    reveal_type(x)  # revealed: int
    result = x + 1
    x = None
    reveal_type(x)  # revealed: None
    x = "wrong"  # error: [invalid-assignment]
    return result
```

### Compatible defaults and required parameters

A compatible non-`None` default does not widen the annotation. Parameters without defaults also keep
their annotated types and remain required.

```py
def defaulted(x: int = 0):
    reveal_type(x)  # revealed: int

def required(x: int):
    reveal_type(x)  # revealed: int

defaulted()
defaulted(1)
required(1)
defaulted(None)  # error: [invalid-argument-type]
required(None)  # error: [invalid-argument-type]
required()  # error: [missing-argument]
```

### Other incompatible defaults

The setting does not permit arbitrary incompatible defaults or widen annotations to include them.

```py
# error: [invalid-parameter-default]
def text_default(x: int = "wrong"):
    reveal_type(x)  # revealed: int

text_default("wrong")  # error: [invalid-argument-type]
text_default(None)  # error: [invalid-argument-type]
```

Only a literal `None` default triggers widening. A name whose inferred type is `None` does not.

```py
none_value = None

# error: [invalid-parameter-default]
def named_default(x: int = none_value):
    reveal_type(x)  # revealed: int

named_default(None)  # error: [invalid-argument-type]
```

A call returning `None` does not trigger widening either.

```py
def get_none() -> None:
    return None

# error: [invalid-parameter-default]
def call_default(x: int = get_none()):
    reveal_type(x)  # revealed: int

call_default(None)  # error: [invalid-argument-type]
```

### Ellipsis defaults

An ellipsis default remains incompatible with `int` in an ordinary function, even when implicit
`None` values are enabled. Neither `None` nor ellipsis becomes a valid argument.

```py
# error: [invalid-parameter-default]
def f(x: int = ...):
    reveal_type(x)  # revealed: int

f(None)  # error: [invalid-argument-type]
f(...)  # error: [invalid-argument-type]
```

### Quoted and already optional annotations

Quoted annotations are widened after resolving the annotation. An annotation that already includes
`None` keeps the same union rather than adding a duplicate member.

```py
def quoted(x: "int" = None):
    reveal_type(x)  # revealed: int | None

def optional(x: int | None = None):
    reveal_type(x)  # revealed: int | None

quoted()
quoted(None)
quoted(1)
quoted("wrong")  # error: [invalid-argument-type]
optional()
optional(None)
optional(1)
optional("wrong")  # error: [invalid-argument-type]
```

### Any, object, and unannotated parameters

Widening `Any` preserves an explicit `None` member, while `object` already includes `None`.
Unannotated parameters retain the existing union of `Unknown` and the default's type. All three
continue to accept arbitrary arguments.

```py
from typing import Any

def f(dynamic: Any = None, broad: object = None, unannotated=None):
    reveal_type(dynamic)  # revealed: Any | None
    reveal_type(broad)  # revealed: object
    reveal_type(unannotated)  # revealed: Unknown | None

f()
f(None, None, None)
f("text", 1, False)
```

### Methods

Instance methods, class methods, and static methods widen literal `None` defaults in their bodies
and bound call signatures just like ordinary functions.

```py
class C:
    def method(self, x: int = None):
        reveal_type(x)  # revealed: int | None

    @classmethod
    def class_method(cls, x: int = None):
        reveal_type(x)  # revealed: int | None

    @staticmethod
    def static_method(x: int = None):
        reveal_type(x)  # revealed: int | None

c = C()
c.method()
c.method(None)
C.method(c, None)
C.class_method()
C.class_method(None)
C.static_method()
C.static_method(None)
c.method("wrong")  # error: [invalid-argument-type]
C.class_method("wrong")  # error: [invalid-argument-type]
C.static_method("wrong")  # error: [invalid-argument-type]
```

### Overloads

A literal `None` default widens the corresponding overload signature, not just the implementation.
Calls with omitted or `None` arguments select that overload; unrelated overloads are unchanged.

```py
from typing import overload

@overload
def f(x: int = None) -> int: ...
@overload
def f(x: str) -> str: ...
def f(x: int | str = None) -> int | str:
    reveal_type(x)  # revealed: int | str | None
    return 0 if x is None else x

reveal_type(f())  # revealed: int
reveal_type(f(None))  # revealed: int
reveal_type(f(1))  # revealed: int
reveal_type(f("text"))  # revealed: str
```

### Callable assignability

A function with an implicit `None` default is assignable to a `Callable` accepting `int | None`,
just like one with an explicit optional annotation. A function accepting only `int` is not.

```py
from typing import Callable

def implicit(x: int = None) -> None: ...
def explicit(x: int | None = None) -> None: ...
def non_optional(x: int = 0) -> None: ...

a: Callable[[int | None], None] = implicit
b: Callable[[int | None], None] = explicit
c: Callable[[int | None], None] = non_optional  # error: [invalid-assignment]
d: Callable[[int], None] = implicit
```

### Imported stub signatures

Imported stub functions also expose widened signatures for literal `None` defaults. Ellipsis remains
a permitted placeholder default in stubs, but it does not widen the parameter type or become a valid
argument itself.

`defaults.pyi`:

```pyi
def implicit(x: int = None) -> int: ...
def placeholder(x: int = ...) -> int: ...
```

```py
from typing import Callable
from defaults import implicit, placeholder

reveal_type(implicit())  # revealed: int
reveal_type(implicit(None))  # revealed: int
implicit(1)
implicit("wrong")  # error: [invalid-argument-type]

reveal_type(placeholder())  # revealed: int
placeholder(1)
placeholder(None)  # error: [invalid-argument-type]
placeholder(...)  # error: [invalid-argument-type]

accepts_none: Callable[[int | None], int] = implicit
rejects_none: Callable[[int | None], int] = placeholder  # error: [invalid-assignment]
```

## TypedDict defaults use annotation context

```py
from typing import TypedDict

class Foo(TypedDict):
    x: int

def x(a: Foo = {"x": 42}): ...
def y(a: Foo = dict(x=42)): ...
```

## TypedDict defaults still validate keys and value types

```py
from typing import TypedDict

class Foo(TypedDict):
    x: int
    y: int

# error: [missing-typed-dict-key]
def missing_key(a: Foo = {"x": 42}): ...

# error: [invalid-argument-type]
def wrong_type(a: Foo = {"x": "s", "y": 1}): ...

# error: [invalid-key]
def extra_key(a: Foo = {"x": 1, "y": 2, "z": 3}): ...
```

## Stub functions

```toml
[environment]
python-version = "3.12"
```

### In Protocol

```py
from typing import Protocol

class Foo(Protocol):
    def x(self, y: bool = ...): ...
    def y[T](self, y: T = ...) -> T: ...

class GenericFoo[T](Protocol):
    def x(self, y: bool = ...) -> T: ...
```

### In abstract method

```py
from abc import abstractmethod

class Bar:
    @abstractmethod
    def x(self, y: bool = ...): ...
    @abstractmethod
    def y[T](self, y: T = ...) -> T: ...
```

### In function overload

```py
from typing import overload

@overload
def x(y: None = ...) -> None: ...
@overload
def x(y: int) -> str: ...
def x(y: int | None = None) -> str | None: ...
```

### In `if TYPE_CHECKING` blocks

We generally view code in `if TYPE_CHECKING` blocks as having the same semantics and exemptions to
code in stub files:

```py
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    def foo(x: bool = ...): ...  # fine
```
