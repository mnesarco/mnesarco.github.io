---
layout: post
title: "Python Metaprogramming - Properties on Steroids"
date: 2020-07-23
categories: [python, metaprogramming, advanced]
---

Metaprogramming is an advanced topic, it requires a good understanding of the language itself. I assume you already know Python well enough even though I include references to complementary resources. If you are interested only in the final result, there is a link to te code at the end of the article.

Python is a very dynamic language, everything is an object, objects are created by classes (usually) but classes can also be created by other
clases or functions (this is amazing). Objects, once created, can be modified (add/remove/replace properties, methods, attributes, etc...) and this
means that you can do a lot of metaprogramming in python. Metaprogramming is a key that can open the doors to heaven or hell.

Ok but what is metaprogramming? here is a definition from wikipedia:

<div class="card">
  <p class="card-body">
    Metaprogramming is a programming technique in which computer programs have the ability to treat other programs as their data. It means that a program can be designed > to read, generate, analyze or transform other programs, and even modify itself while running.  
    <br />
    <br />
    <small>
        -- Harald Sondergaard. "Course on Program Analysis and Transformation". Retrieved 18 September 2014.
    </small><br />
    <small>
        -- Czarnecki, Krzysztof; Eisenecker, Ulrich W. (2000). Generative Programming. ISBN 0-201-30977-7.
    </small>
  </p>
  
</div>
<br />
In this article, I will use metaprogramming to change how properties are defined in a class, how they can be documented, initialized, how to set a default value, how to make them read-only and observable, and as a bonus, I will improve memory usage of the objects created by the class. And as a second bonus, I will seal the object against attribute injections. I call this "Properties on Steroids".

## Requirements

1. A property must be defined in a succinct and pythonic way.
2. A property definition can setup a default value.
3. A property must support docstring (`__doc__`).
4. A Property can have a type hint.
5. A Property can be read-only or read-and-write.
6. A Property can be observable.
7. A Property must not use more memory than traditional python property (@property)
8. Bonus: Field storage can be optimized
9. Bonus2: Objects created from the class must be protected from field injection.
10. **DO NOT USE A METACLASS**

## Important concepts used in the solution

### Classes are created at runtime

In python, classes are created at runtime, so the code inside the class scope can modify the resulting class by adding/removing/replacing things in the class scope (local).

For more info about classes in Python: [https://docs.python.org/3/tutorial/classes.html](https://docs.python.org/3/tutorial/classes.html)

### Scopes

There are two scopes in python: **local** and **global**. Scopes are symbol tables of what is reachable from the current point in code. you can access the symbol table using the builtin functions `locals()` and `globals()`. The important part here is that you can modify the scope just adding/replacing/removing things in the symbol table.

For example, if you want to define a variable in the local scope dynamically:

```python
def myfunc():
  
  var1 = "var1 is defined by programming"
  
  locals()['var2'] = "var2 is defined by metaprogramming"
  
  print(f" var1={var1}, var2={var2}")

```

For more info about scopes in python: [https://realpython.com/python-scope-legb-rule/](https://realpython.com/python-scope-legb-rule/)

### Decorators

In python, functions are objects and can be passed to other functions like any other object. The idea of a decorator is a function that receives another function and returns a new function based on the original. There is a special syntax in python to call a decorator function just on function definition and effectively replacing the original function with the one returned by the decorator.

Example:

```python
def my_decorator(fun):
    def new_func():
        print("Hello from modified fun")
        fun()
        print("By from modified fun")
    return new_func

@my_decorator
def sample_fun():
    print("This is the original function")


sample_fun()

```

The above code will print:

```
> Hello from modified fun
> This is the original function
> By from modified fun
```

For more info about decorators: [https://realpython.com/primer-on-python-decorators/](https://realpython.com/primer-on-python-decorators/)

### Properties

Python has a special class called properties, it allows us to create getter/setter/deleter for a field. In combination with decorators you can define functional properties.

```python
class MyClass:

    def __init__(self, myprop):
        self._myprop = myprop or 'My default value'

    @property
    def myprop(self):
        return self._myprop

    @myprop.setter
    def myprop(self, value):
        self._myprop = value

```

We will change this pattern to add more features like observability, and automatic usage of slots.

For more info about properties: [https://docs.python.org/3/howto/descriptor.html#properties](https://docs.python.org/3/howto/descriptor.html#properties)


### Slots

Objects in Python do store attributes in an internal dictionary called `__dict__`, it allows dynamic creation of attributes in any object but uses additional memory for the dict object itself. If you want that your object do not support dynamic attribute creation, you can remove the `__dict__` mechanism and use object slots with the additional benefit of memory savings.

I will not explain slots here, but you can find detailed info in the following resources: 

- [https://book.pythontips.com/en/latest/__slots__magic.html](https://book.pythontips.com/en/latest/__slots__magic.html)
- [https://docs.python.org/3/reference/datamodel.html#slots](https://docs.python.org/3/reference/datamodel.html#slots)


### Context Managers

Context managers are objects that can execute code at the beginning and at the end of a code block. they are used with the `with` statement.

For more info about context managers: [https://docs.python.org/3/library/contextlib.html](https://docs.python.org/3/library/contextlib.html)

## Proposed Solution

Ok, now with all the tools in the bag, we can create our own monster.

### The final goal:

```python
from objects import properties, self_properties


class Car:
    with properties(locals(), 'meta') as meta:

        @meta.prop(read_only=True)
        def brand(self) -> str:
            """Brand"""

        @meta.prop(read_only=True)
        def max_speed(self) -> float:
            """Maximum car speed"""

        @meta.prop(listener='_on_acceleration')
        def speed(self) -> float:
            """Speed of the car"""
            return 0  # Default stopped

        @meta.prop(listener='_on_off_listener')
        def on(self) -> bool:
            """Engine state"""
            return False

    def __init__(self, brand: str, max_speed: float = 200):
        self_properties(self, locals())

    def _on_off_listener(self, prop, old, on):
        if on:
            print(f"{self.brand} Turned on, Runnnnnn")
        else:
            self._speed = 0
            print(f"{self.brand} Turned off.")

    def _on_acceleration(self, prop, old, speed):
        if self.on:
            if speed > self.max_speed:
                print(f"{self.brand} {speed}km/h Bang! Engine exploded!")
                self.on = False
            else:
                print(f"{self.brand} New speed: {speed}km/h")
        else:
            print(f"{self.brand} Car is off, no speed change")


mycar = Car('Ford')

# Car is turned off
for speed in range(0, 300, 50):
    mycar.speed = speed

# Car is turned on
mycar.on = True
for speed in range(0, 350, 50):
    mycar.speed = speed

    
```

Result:

```
Ford Car is off, no speed change
Ford Car is off, no speed change
Ford Car is off, no speed change
Ford Car is off, no speed change
Ford Car is off, no speed change
Ford Car is off, no speed change
Ford Turned on, Runnnnnn
Ford New speed: 0km/h
Ford New speed: 50km/h
Ford New speed: 100km/h
Ford New speed: 150km/h
Ford New speed: 200km/h
Ford 250km/h Bang! Engine exploded!
Ford Turned off.
Ford Car is off, no speed change
```


### The traditional equivalent code:

```python
class CarTaditional:

    @property
    def brand(self) -> str:
        """Brand"""
        return self._brand

    @property
    def max_speed(self) -> float:
        """Maximum car speed"""
        return self._max_speed

    @property
    def speed(self) -> float:
        """Speed of the car"""
        return self._speed

    @speed.setter
    def speed(self, speed):
        if self._speed != speed:
            self._speed = speed
            if self.on:
                if speed > self.max_speed:
                    print(f"{self.brand} {speed}km/h Bang! Engine exploded!")
                    self.on = False
                else:
                    print(f"{self.brand} New speed: {speed}km/h")
            else:
                print(f"{self.brand} Car is off, no speed change")

    @property
    def on(self) -> bool:
        """Engine state"""
        return self._on

    @on.setter
    def on(self, on):
        if self._on != on:
            self._on = on
            if on:
                print(f"{self.brand} Turned on, Runnnnnn")
            else:
                self._speed = 0
                print(f"{self.brand} Turned off.")

    def __init__(self, brand: str, max_speed: float = 200):
        self._brand = brand
        self._max_speed = max_speed
        self._speed = 0
        self._on = False


mycar2 = CarTaditional('Ford')

# Car is turned off
for speed in range(0, 300, 50):
    mycar2.speed = speed

# Car is turned on
mycar2.on = True
for speed in range(0, 350, 50):
    mycar2.speed = speed

```

### The differences

initially it appears to have no major advantajes, but the classes Car and CarTraditional and the objects mycar and mycar2 are very different.

#### Field injection

With Metaprog:

```python
mycar.model = 2020
# AttributeError: 
#   'Car' object has no attribute 'model'
```

With traditional code:
```python
mycar2.model = 2020
# Will set the model attribute silently
```

#### Default values

With Metaprog:

Default values are specified at property definition.
```python
@meta.prop(listener='_on_acceleration')
def speed(self) -> float:
    """Speed of the car"""
    return 0  # Default stopped
```

With traditional code:

Default values are defined by assignment in constructor
```python
def __init__(self, brand: str, max_speed: float = 200):
    self._brand = brand
    self._max_speed = max_speed
    self._speed = 0
    self._on = False
```

#### Observability

With Metaprog:

Multiple attributes can be observed with the same listener, listener specified on property definition, listener is called only if new value is different from current.
```python
@meta.prop(listener='_on_off_listener')
def on(self) -> bool:
    """Engine state"""
    return False  # Default value
```

With traditional code:
You must implement observability by your own.
```python
@on.setter
def on(self, on):
    if self._on != on:
        self._on = on
        if on:
            print(f"{self.brand} Turned on, Runnnnnn")
        else:
            self._speed = 0
            print(f"{self.brand} Turned off.")
```

#### Read-Only / Read-Write

With Metaprog:

One single definition will create readonly or read-write property.
```python

# Read-Only property
@meta.prop(read_only=True)
def brand(self) -> str:
    """Brand"""

# Read-Write property
@meta.prop()
def on(self) -> bool:
    """Engine state"""

```

With traditional code:

You must define a getter and a setter if the property is writable.
```python

@property
def on(self) -> bool:
    """Engine state"""
    return self._on

@on.setter
def on(self, on):
    self._on = on

```

#### Constructor arguments to field assigment

With Metaprog:

Constructor arguments can set defined properties automatically.
```python
def __init__(self, brand: str, max_speed: float = 200):
    self_properties(self, locals())
```

With traditional code:

You must set properties by hand.
```python
def __init__(self, brand: str, max_speed: float = 200):
    self._brand = brand
    self._max_speed = max_speed
    self._speed = 0
    self._on = False
```

## Implementation

Ok, in a small sample class like Car there is no much advantage but in a large system with many classes and classes with many mutable and "immutable" attributes and complex state changes it will add lot of productivity in a very pythonic way.

Lets see the magic:


```python
# file: objects.py
# Copyright 2020 Frank David Martínez Muñoz (mnesarco)
# License: MIT

from typing import Union

__all__ = ('self_properties', 'properties')


def self_properties(self, scope: dict, exclude=(), save_args: bool = False):
    """Copies all items from `scope` to self as attributes with single underscore prefix.

    :param self: instance ref.
    :param scope: dictionary with attributes.
    :param exclude: tuple with names to exclude from `scope`.
    :param save_args: if True, sets self._args with a tuple with `(scope - exclude).values`
    """
    if save_args:
        args = []
        for (k, v) in scope.items():
            if k != 'self' and k not in exclude:
                setattr(self, '_' + k, v)
                args.append(v)
        self._args = tuple(args)
    else:
        for (k, v) in scope.items():
            if k != 'self' and k not in exclude:
                setattr(self, '_' + k, v)


class properties:
    """
    Utilities for building properties with extended features.
    """

    __slots__ = ['_slots', '_scope', '_var', '_auto_dirty']

    def __init__(self, scope: dict, var_name: str, auto_dirty: bool = False):
        self._slots = []
        self._scope = scope
        self._var = var_name
        self._auto_dirty = auto_dirty

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):

        # Export slots to scope
        slots = self._scope.get('__slots__', None)
        if slots is None:
            self._scope['__slots__'] = tuple(self._slots)
        elif isinstance(slots, list):
            self._scope['__slots__'] += self._slots
        elif isinstance(slots, tuple):
            self._scope['__slots__'] = (*self._scope['__slots__'], *self._slots)

        # Clean scope
        if self._var in self._scope:
            del self._scope[self._var]

        # Clean references
        self._scope = None
        self._slots = None
        self._var = None

    def prop(self, read_only: bool = False, listener: Union[bool, str] = None, auto_dirty=False):
        """Decorator: Generates a property with additional features.

        :param read_only: if True, only a getter is generated.
        :param listener: if str, changes will fire `self.[listener]`, if bool, changes will fire `self._changed`
        :param auto_dirty: if True, changes will set `self._is_dirty`
        :return: property.
        """

        auto_dirty = self._auto_dirty or auto_dirty

        if auto_dirty and '_is_dirty' not in self._slots:
            self._slots.append('_is_dirty')

        def decorator(f):

            field = '_' + f.__name__

            if read_only and listener:
                raise ValueError(f"property {field} cannot be read_only and observable at the same time.")

            self._slots.append(field)

            if read_only:
                setter = None
            else:
                if listener:
                    listener_name = listener if isinstance(listener, str) else '_changed'

                    def setter(inst, new):
                        old = getattr(inst, field, None)
                        if old != new:
                            setattr(inst, field, new)
                            if auto_dirty:
                                inst._is_dirty = True
                            (getattr(inst, listener_name))(field, old, new)
                else:
                    def setter(inst, new):
                        if getattr(inst, field, None) != new:
                            setattr(inst, field, new)
                            if auto_dirty:
                                inst._is_dirty = True

            return property(
                lambda inst: getattr(inst, field, f(inst)),
                setter,
                None,
                f.__doc__
            )

        return decorator

```

## API

### `self_properties(self, scope, exclude=(), save_args=False)`

This utility function copy all the symbols in `scope` to self as properties. If you call it at the beginning of the constructor and pass the local scope, it will just copy the function arguments.

```python
def __init__(self, brand: str, max_speed: float = 200):
    self_properties(self, locals())

# Is equivalent to:

def __init__(self, brand: str, max_speed: float = 200):
    self._brand = brand
    self._max_speed = max_speed
```

if you want to exlude something from the copy, just do this:
```python
def __init__(self, brand: str, max_speed: float = 200, other_param):
    self_properties(self, locals(), exclude('other_param',))
```

if you want to save all arguments as a tuple (additionally):
```python
def __init__(self, brand: str, max_speed: float = 200):
    self_properties(self, locals(), save_args=True)

# is equivalent to:

def __init__(self, brand: str, max_speed: float = 200):
    self._brand = brand
    self._max_speed = max_speed
    self._args = (brand, max_speed)

```

### `properties`

This is where the magic happens. properties is a context manager that do the following:

1. Define a prop decorator to create properties.
2. Manage `__slots__` automatically for the class.
3. Clean itself from the class scope.

```python
class Car:

    # locals() is the class scope.
    # 'meta' is the alias of the context manager, 
    #        specified to be autoremoved at the end of the block.
    with properties(locals(), 'meta') as meta:

        @meta.prop(read_only=True)
        def brand(self) -> str:
            """Brand"""

    # Context manager ends

```

### `@meta.prop`

prop is the decorator, it transforms functions into properties.

Arguments:
- `read_only: bool`: Will create a read only property
- `listener: Union[str,bool]`: specify the method to call on change if str. if bool, it will default to '_changed'
- `auto_dirty: bool`: Will set a field `_is_dirty` to true if the property change.


Thanks for reading.

Gist on Github: [https://gist.github.com/mnesarco/e9440a196824af4bae439e4aeb4b6dcc](https://gist.github.com/mnesarco/e9440a196824af4bae439e4aeb4b6dcc)