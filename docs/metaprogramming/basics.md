---
sidebar_position: 1
title: Basics
---
# Python Metaprogramming Basics

Metaprogramming is the practice of writing code that manipulates or generates other code. It involves treating programs as data that can be inspected, modified, or generated during runtime. Python's dynamic nature makes it particularly well-suited for metaprogramming techniques.

Key benefits of metaprogramming include:

- Reducing repetitive code through automation
- Creating more flexible and adaptable APIs
- Enabling runtime adaptation of code
- Implementing domain-specific languages

## Everything is an Object

```python live_py
# Functions are objects
def hello():
    return "Hello, world!"

# You can assign a function to a variable
greeting = hello
print(greeting())  # Output: Hello, world!

# Classes are objects
class Cat:
    def meow(self):
        return "Meow!"

# You can assign a class to a variable
Animal = Cat
whiskers = Animal()
print(whiskers.meow())  # Output: Meow!

# Even the type of an object is an object
print(type(hello))  # Output: <class 'function'>
print(type(Cat))    # Output: <class 'type'>
print(type(whiskers))   # Output: <class 'Cat'>
```

This object-oriented nature allows us to manipulate code elements at runtime, forming the foundation of metaprogramming.

## The `type()` Function

The built-in `type()` function has two uses:

1. When called with one argument, it returns the type (class) of an object
2. When called with three arguments, it dynamically creates a new class

### Getting the Type of an Object

```python live_py
x = 42
print(type(x))  # Output: <class 'int'>

def some_function():
    pass
print(type(some_function))  # Output: <class 'function'>
```

### Creating Classes Dynamically

The `type()` function can dynamically create classes when called with three arguments:

- The name of the class as a string
- A tuple of base classes for inheritance
- A dictionary defining class attributes and methods

```python live_py
# Creating a simple class
def say_hello(self):
    print(f"Hello, my name is {self.name}")

# Create a class dynamically
Person = type('Person', (), {
    'name': 'Anonymous',
    'say_hello': say_hello
})

# Use the dynamically created class
p = Person()
p.name = "Tan Ah Kow"
p.say_hello()  # Output: Hello, my name is Tan Ah Kow
```

This is equivalent to the traditional class definition:

```python live_py
class Person:
    name = 'Tan Ah Kow'
  
    def say_hello(self):
        print(f"Hello, my name is {self.name}")
    
Person().say_hello()
```

## Python's Attribute System

Python's attribute system is highly dynamic, allowing for runtime manipulation of an object's properties and methods.

### Getting and Setting Attributes

```python live_py
class Person:
    def __init__(self, name):
        self.name = name

p = Person("Alice")

# Get an attribute
print(p.name)  # Output: Alice
print(getattr(p, 'name'))  # Output: Alice

# Set an attribute
p.age = 30
setattr(p, 'job', 'Developer')

# Check what attributes exist
print(dir(p))  # Shows all attributes, including name, age, and job

# Check if an attribute exists
print(hasattr(p, 'name'))  # Output: True
print(hasattr(p, 'address'))  # Output: False

# Delete an attribute
delattr(p, 'age')
print(hasattr(p, 'age'))  # Output: False
```

## Decorators

Decorators provide a powerful way to modify or enhance functions and classes without changing their code directly. They are functions that take another function (or class) as an argument and return a modified version.

### Function Decorators

```python live_py
def simple_decorator(func):
    def wrapper(*args, **kwargs):
        print("Something is happening before the function is called.")
        result = func(*args, **kwargs)
        print("Something is happening after the function is called.")
        return result
    return wrapper

# Apply the decorator
@simple_decorator
def say_hello(name):
    print(f"Hello, {name}!")

# This is equivalent to:
# say_hello = simple_decorator(say_hello)

say_hello("World")
# Output:
# Something is happening before the function is called.
# Hello, World!
# Something is happening after the function is called.
```

### Decorators with Arguments

Decorators can also accept arguments by adding another level of function nesting:

```python live_py
from functools import wraps

def repeat(times=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hello {name}!")

print(greet("World"))
```

### Using `functools.wraps`

As shown above, `@wraps` from the `functools` module preserves the metadata (like name, docstring, etc.) of the original function:

```python live_py
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        """Wrapper function"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def my_function():
    """Original docstring"""
    pass

print(my_function.__name__)  # Output: my_function
print(my_function.__doc__)   # Output: Original docstring
```

Without `@wraps`, the wrapper function would lose this metadata, which can make debugging more difficult.

### Class Decorators

Decorators can also be applied to classes:

```python live_py
def add_greeting(cls):
    cls.greet = lambda self: f"Hello, I'm {self.name}"
    return cls

@add_greeting
class Person:
    def __init__(self, name):
        self.name = name

p = Person("Tan Ah Kow")
print(p.greet())
```

## Metaclasses

Metaclasses are classes that create classes. Just as a class defines how an instance behaves, a metaclass defines how a class behaves.

### The Metaclass Hierarchy

- `type` is the default metaclass in Python
- Classes are instances of metaclasses
- Regular objects are instances of classes

```python live_py
# This demonstrates the relationship
class MyClass:
    pass

obj = MyClass()

print(type(obj))       # Output: <class '__main__.MyClass'>
print(type(MyClass))   # Output: <class 'type'>
print(type(type))      # Output: <class 'type'> - type is its own metaclass
```

### Creating a Custom Metaclass

To create a custom metaclass, subclass `type` and override its methods:

```python live_py
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class {name}")
        print(f"Bases: {bases}")
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=Meta):
    x = 1
  
    def hello(self):
        return "Hello, world!"

# Output:
# Creating class MyClass
# Bases: ()
```

### SnakeCase Metaclass Example

Here's a metaclass that converts all attribute names to snake_case:

[Run this code in Programiz ▶](https://www.programiz.com/online-compiler/7wQoMdrJYTsJi)
```python
import re

def camel_to_snake(name):
    s1 = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
    return re.sub('([a-z0-9])([A-Z])', r'\1_\2', s1).lower()

class SnakeCaseMetaclass(type):
    def __new__(mcs, name, bases, attrs):
        snake_case_attrs = {}
        for attr_name, attr_value in attrs.items():
            if not attr_name.startswith('__'):
                snake_case_attrs[camel_to_snake(attr_name)] = attr_value
            else:
                snake_case_attrs[attr_name] = attr_value
        return super().__new__(mcs, name, bases, snake_case_attrs)

class TestClass(metaclass=SnakeCaseMetaclass):
    someVariable = 42
    anotherCamelCaseAttribute = "hello"
  
    def testMethod(self):
        return "This is a test"

print(TestClass.some_variable)  # Output: 42
print(TestClass.another_camel_case_attribute)  # Output: hello
print(TestClass().test_method())  # Output: This is a test
```

### Metaclass Inheritance

When a class uses a metaclass, all of its subclasses will automatically use that metaclass as well:

```python live_py
class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        return super().__new__(mcs, name, bases, namespace)

class Base(metaclass=MyMeta):
    pass

class Child(Base):  # Automatically uses MyMeta
    pass

# Output:
# Creating class: Base
# Creating class: Child
```

## Introspection Tools

Python provides several built-in functions for introspection (examining objects at runtime):

### `dir()`: List Attributes

```python live_py
class Person:
    name = "Anonymous"
  
    def greet(self):
        return f"Hello, my name is {self.name}"

p = Person()
print(dir(p))  # Shows all attributes, including inherited ones
```

### `vars()`: Get `__dict__`

```python live_py
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(3, 4)
print(vars(p))  # Output: {'x': 3, 'y': 4}
```

### `inspect` Module

The `inspect` module provides higher-level introspection functions:

```python live_py
import inspect

def example_function(a, b=1, *args, **kwargs):
    """This is an example function."""
    pass

# Get function signature
print(inspect.signature(example_function))  # Output: (a, b=1, *args, **kwargs)

# Get function docstring
print(inspect.getdoc(example_function))  # Output: This is an example function.
```
