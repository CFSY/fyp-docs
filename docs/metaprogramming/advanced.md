---
sidebar_position: 2
title: Advanced Metaprogramming
---

# Advanced Python Metaprogramming

Building on the fundamentals covered in the basics guide, this document explores advanced metaprogramming techniques, patterns, and real-world applications in Python.

## Advanced Metaclass Techniques

### Metaclass Method Hooks

The `__new__` method is just one of several hooks that metaclasses provide. Here's a more comprehensive overview:

```python live_py
class AdvancedMeta(type):
    # Called to create the namespace dictionary before class creation
    @classmethod
    def __prepare__(mcs, name, bases):
        print(f"__prepare__ {name}")
        return {'meta_created': True}
    
    # Called to create the class object
    def __new__(mcs, name, bases, namespace):
        print(f"__new__ {name}")
        return super().__new__(mcs, name, bases, namespace)
    
    # Called to initialize the class object after creation
    def __init__(cls, name, bases, namespace):
        print(f"__init__ {name}")
        super().__init__(name, bases, namespace)
    
    # Called when the class is "called" to create an instance
    def __call__(cls, *args, **kwargs):
        print(f"__call__ {cls.__name__}")
        instance = super().__call__(*args, **kwargs)
        print(f"Instance created: {instance}")
        return instance

class MyClass(metaclass=AdvancedMeta):
    def __init__(self, value):
        self.value = value

# Creating the class triggers __prepare__, __new__, and __init__

# Creating an instance triggers __call__
instance = MyClass(42)

print(instance.value)  # Output: 42
print(instance.meta_created)  # Output: True
```

### Combining Metaclasses

When you need functionality from multiple metaclasses, you can create a combined metaclass:

```python live_py
class TraceMeta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Trace: Creating class {name}")
        return super().__new__(mcs, name, bases, namespace)

class ValidationMeta(type):
    def __new__(mcs, name, bases, namespace):
        for key, value in namespace.items():
            if key.startswith('validate_') and not callable(value):
                raise TypeError(f"Validation method {key} must be callable")
        return super().__new__(mcs, name, bases, namespace)

# Combined metaclass
class CombinedMeta(TraceMeta, ValidationMeta):
    pass

class MyValidator(metaclass=CombinedMeta):
    def validate_data(self, data):
        return isinstance(data, dict)

# This would print traces and then raise an error
try:
    class BadValidator(metaclass=CombinedMeta):
        validate_data = "Not a function"
except TypeError as e:
    print(f"Error: {e}")  # Output: Error: Validation method validate_data must be callable
```

## Advanced Magic Methods

### Customizing Attribute Access

These magic methods allow you to fully customize how attributes are accessed, set, and deleted. In this example, `AttributeMonitor` monitors all attribute access, modification, and deletion operations. It keeps a log of these operations for inspection.:

```python live_py
class AttributeMonitor:
    def __init__(self, **kwargs):
        self.__dict__.update(kwargs)
        self._access_log = []
    
    def __getattribute__(self, name):
        # This is called for ALL attribute access
        if name != '_access_log' and not name.startswith('__'):
            # Avoid infinite recursion by using super().__getattribute__
            log = super().__getattribute__('_access_log')
            log.append(f"Accessed {name}")
        return super().__getattribute__(name)
    
    def __getattr__(self, name):
        # This is only called when the attribute doesn't exist
        return f"Attribute {name} doesn't exist"
    
    def __setattr__(self, name, value):
        if name != '_access_log' and not name.startswith('__'):
            # Avoid infinite recursion using object.__setattr__
            access_log = object.__getattribute__(self, '_access_log')
            access_log.append(f"Set {name} to {value}")
        object.__setattr__(self, name, value)
    
    def __delattr__(self, name):
        if name != '_access_log' and not name.startswith('__'):
            access_log = object.__getattribute__(self, '_access_log')
            access_log.append(f"Deleted {name}")
        object.__delattr__(self, name)
    
    def get_log(self):
        return self._access_log

obj = AttributeMonitor(x=1, y=2)
print(obj.x)       # Accessing existing attribute
print(obj.z)       # Accessing non-existent attribute
obj.y = 20         # Setting existing attribute
obj.z = 30         # Setting new attribute
del obj.x          # Deleting attribute
print(obj.get_log())
# Output:
# 1
# Attribute z doesn't exist
# ['Accessed x', 'Accessed z', 'Set y to 20', 'Set z to 30', 'Deleted x', 'Accessed get_log']
```

### Custom Container Classes

You can implement your own container types by implementing the appropriate magic methods:

```python live_py
class BetterDict:
    def __init__(self, data=None):
        self._data = {} if data is None else dict(data)
    
    def __getitem__(self, key):
        # Allow getting values with dictionary[key]
        if key not in self._data:
            return f"No such key: {key}"
        return self._data[key]
    
    def __setitem__(self, key, value):
        # Allow setting values with dictionary[key] = value
        print(f"Setting {key} to {value}")
        if isinstance(key, str):
            key = key.strip()
        self._data[key] = value
    
    def __delitem__(self, key):
        # Allow deleting with del dictionary[key]
        if key in self._data:
            print(f"Deleting {key}")
            del self._data[key]
    
    def __contains__(self, key):
        # Support the 'in' operator
        return key in self._data
    
    def __len__(self):
        # Support len(dictionary)
        return len(self._data)
    
    def __iter__(self):
        # Support iteration
        return iter(self._data)
    
    def __repr__(self):
        # String representation
        return f"BetterDict({self._data})"

bd = BetterDict({"name": "John", "age": 30})
print(bd["name"])          # Output: John
print(bd["address"])       # Output: No such key: address
bd["  email  "] = "john@example.com"  # Whitespace in key gets stripped
print("email" in bd)       # Output: True
print(len(bd))             # Output: 3
print(list(bd))            # Output: ['name', 'age', 'email']
del bd["age"]
print(bd)                  # Output: BetterDict({'name': 'John', 'email': 'john@example.com'})
```

### Callable Objects

You can make objects callable by implementing the `__call__` method:

```python live_py
class Counter:
    def __init__(self, start=0):
        self.count = start
    
    def __call__(self, increment=1):
        self.count += increment
        return self.count

counter = Counter(10)
print(counter())      # Output: 11
print(counter(5))     # Output: 16
print(counter())      # Output: 17
```

### Advanced Descriptors

Here's a more advanced descriptor that implements lazy properties with caching:

```python live_py
class LazyProperty:
    def __init__(self, function):
        self.function = function
        self.name = function.__name__
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        
        # Compute the value and cache it
        value = self.function(instance)
        setattr(instance, self.name, value)  # Replace descriptor with value
        return value

class Circle:
    def __init__(self, radius):
        self.radius = radius
    
    @LazyProperty
    def area(self):
        print("Computing area...")
        import math
        return math.pi * self.radius ** 2
    
    @LazyProperty
    def circumference(self):
        print("Computing circumference...")
        import math
        return 2 * math.pi * self.radius

c = Circle(5)
print(c.area)           # Output: Computing area... 78.53981633974483
print(c.area)           # Output: 78.53981633974483 (no recomputation)
print(c.circumference)  # Output: Computing circumference... 31.41592653589793
print(c.circumference)  # Output: 31.41592653589793 (no recomputation)
```

### Context Managers

Context managers (the `with` statement) are another example of Python's metaprogramming capabilities:

```python live_py
class Timer:
    def __enter__(self):
        import time
        self.start = time.time()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        self.end = time.time()
        print(f"Elapsed time: {self.end - self.start:.6f} seconds")
        # Returning False allows exceptions to propagate
        return False
    
    def elapsed(self):
        if hasattr(self, 'end'):
            return self.end - self.start
        import time
        return time.time() - self.start

# Using the context manager
with Timer() as timer:
    # Some time-consuming operation
    sum(range(10000000))
    current = timer.elapsed()
    print(f"Current elapsed: {current:.6f} seconds")

# Output:
# Current elapsed: 0.123456 seconds
# Elapsed time: 0.234567 seconds
```

## Advanced Metaprogramming Patterns

### Singleton Pattern

```python live_py
class Singleton(type):
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Logger(metaclass=Singleton):
    def __init__(self):
        self.logs = []
    
    def log(self, message):
        self.logs.append(message)
        print(f"Log: {message}")

# Both instances are actually the same object
logger1 = Logger()
logger2 = Logger()

logger1.log("First message")
logger2.log("Second message")

print(logger1 is logger2)  # Output: True
print(logger1.logs)        # Output: ['First message', 'Second message']
print(logger2.logs)        # Output: ['First message', 'Second message']
```

### Registry Pattern

A registry pattern can help us keep track of anything. In this example, we track all subclasses:

```python live_py
class PluginRegistry(type):
    plugins = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        
        # Don't register the base class
        if bases:
            if hasattr(cls, 'plugin_name'):
                name = cls.plugin_name
            else:
                name = cls.__name__
            
            mcs.plugins[name] = cls
            print(f"Registered plugin: {name}")
        
        return cls
    
    @classmethod
    def get_plugin(mcs, name):
        return mcs.plugins.get(name)
    
    @classmethod
    def list_plugins(mcs):
        return list(mcs.plugins.keys())

class Plugin(metaclass=PluginRegistry):
    """Base class for plugins"""
    pass

class TextPlugin(Plugin):
    plugin_name = "text"
    
    def process(self, data):
        return f"Processing text: {data}"

class ImagePlugin(Plugin):
    plugin_name = "image"
    
    def process(self, data):
        return f"Processing image: {data}"

# New plugins automatically register themselves
class VideoPlugin(Plugin):
    plugin_name = "video"
    
    def process(self, data):
        return f"Processing video: {data}"

# List all available plugins
print(PluginRegistry.list_plugins())  # Output: ['text', 'image', 'video']

# Get a plugin by name
plugin = PluginRegistry.get_plugin("image")
print(plugin().process("photo.jpg"))  # Output: Processing image: photo.jpg
```

## Dynamic Code Execution

Python allows you to dynamically execute code at runtime:

### `eval()` and `exec()`

```python live_py
# Dynamically evaluate expressions
x = 10
y = 20
result = eval("x + y")
print(result)  # Output: 30

# Dynamically execute statements
code = """
def greet(name):
    return f"Hello, {name}!"

message = greet("World")
"""
namespace = {}
exec(code, namespace)
print(namespace['message'])  # Output: Hello, World!
```

### Abstract Syntax Trees

Abstract Syntax Tree (AST) analysis and manipulation allows us to parse, analyze, transform, and execute Python code programmatically.
In this example, we use the `ast` module to optimise constant expressions (e.g. x = 10 + 20 * 2), by pre-calculating it and then executing the optimized version.
The walk through the AST will be printed.

```python live_py
import ast

# Parse Python code into an AST
code = "x = 10 + 20"
tree = ast.parse(code)

# Analyze the AST
for node in ast.walk(tree):
    print(type(node).__name__)

# Create a transformer to modify the AST
class ConstantFolder(ast.NodeTransformer):
    def visit_BinOp(self, node):
        # Recursively transform child nodes
        self.generic_visit(node)
        
        # If both operands are constants, compute the result
        if isinstance(node.left, ast.Constant) and isinstance(node.right, ast.Constant):
            left_val = node.left.value
            right_val = node.right.value
            
            if isinstance(node.op, ast.Add):
                result = left_val + right_val
            elif isinstance(node.op, ast.Sub):
                result = left_val - right_val
            elif isinstance(node.op, ast.Mult):
                result = left_val * right_val
            elif isinstance(node.op, ast.Div):
                result = left_val / right_val
            else:
                return node
            
            return ast.Constant(value=result)
        
        return node

# Parse the code
code = "x = 10 + 20 * 2"
tree = ast.parse(code)

# Transform the AST
transformed_tree = ConstantFolder().visit(tree)
ast.fix_missing_locations(transformed_tree)

# Compile and execute the transformed code
compiled_code = compile(transformed_tree, filename="<ast>", mode="exec")
namespace = {}
exec(compiled_code, namespace)

print(namespace['x'])  # Output: 50
```

## Real-World Examples

Let's look at how some popular Python libraries utilize metaprogramming:

### SQLAlchemy's Declarative Base

SQLAlchemy uses metaclasses to create a declarative ORM (Object-Relational Mapping) system. 
In the example below, `ModelMeta` intercepts class creation to automatically extract column definitions and table names from class declarations, 
enabling the `Model` class to transform Python objects into SQL operations without requiring manual mapping between the object attributes and database columns.

```python live_py
# Simplified version of SQLAlchemy's declarative base
class ModelMeta(type):
    def __new__(mcs, name, bases, namespace):
        if name == 'Model':
            return super().__new__(mcs, name, bases, namespace)
        
        # Extract table name
        if 'table_name' not in namespace:
            namespace['table_name'] = name.lower()
        
        # Extract columns
        columns = {}
        for key, value in list(namespace.items()):
            if isinstance(value, Column):
                columns[key] = value
        
        namespace['columns'] = columns
        
        return super().__new__(mcs, name, bases, namespace)

class Column:
    def __init__(self, column_type, primary_key=False):
        self.column_type = column_type
        self.primary_key = primary_key

class Model(metaclass=ModelMeta):
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            if key in self.columns:
                setattr(self, key, value)
            else:
                raise AttributeError(f"Unknown column: {key}")
    
    def save(self):
        # Simplified demonstration of ORM behavior
        column_names = list(self.columns.keys())
        column_values = [getattr(self, col, None) for col in column_names]
        
        query = f"INSERT INTO {self.table_name} "
        query += f"({', '.join(column_names)}) "
        query += f"VALUES ({', '.join(['%s'] * len(column_names))})"
        
        print(f"Executing: {query}")
        print(f"With values: {column_values}")
        return True

# Using the ORM
class User(Model):
    table_name = "users"
    id = Column(int, primary_key=True)
    name = Column(str)
    email = Column(str)

user = User(id=1, name="John", email="john@example.com")
user.save()
# Output:
# Executing: INSERT INTO users (id, name, email) VALUES (%s, %s, %s)
# With values: [1, 'John', 'john@example.com']
```

### Django's Model System

Django's ORM uses metaclasses for model definition. 
In the example below, a custom metaclass `ModelBase` transforms class definitions into ORM models. 
When a class inherits from `Model`, the metaclass automatically collects field attributes, adds a `_fields` dictionary to track them, 
and injects an `objects` manager for database operations—similar to how Django's ORM works behind the scenes to convert Python class definitions into database-aware models.

```python live_py
# Simplified version of Django's model metaclass
class ModelBase(type):
    def __new__(mcs, name, bases, namespace):
        if name == 'Model':
            return super().__new__(mcs, name, bases, namespace)
        
        # Create Fields dictionary
        fields = {}
        for key, value in namespace.items():
            if isinstance(value, Field):
                fields[key] = value
        
        namespace['_fields'] = fields
        
        # Auto-create a manager
        namespace['objects'] = Manager()
        
        return super().__new__(mcs, name, bases, namespace)

class Field:
    def __init__(self, verbose_name=None, primary_key=False):
        self.verbose_name = verbose_name
        self.primary_key = primary_key

class Manager:
    def all(self):
        print("SELECT * FROM table")
        return []
    
    def filter(self, **kwargs):
        conditions = [f"{k} = %s" for k in kwargs.keys()]
        query = f"SELECT * FROM table WHERE {' AND '.join(conditions)}"
        print(f"Executing: {query}")
        print(f"With values: {list(kwargs.values())}")
        return []

class Model(metaclass=ModelBase):
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            setattr(self, key, value)
    
    def save(self):
        fields = self._fields.keys()
        values = [getattr(self, field) for field in fields]
        print(f"Saving {self.__class__.__name__} with values: {values}")
        return True

# Using the ORM
class Product(Model):
    name = Field(verbose_name="Product Name")
    price = Field()
    description = Field()

# Create and save an instance
product = Product(name="Laptop", price=999.99, description="Powerful laptop")
product.save()
# Output: Saving Product with values: ['Laptop', 999.99, 'Powerful laptop']

# Use the manager
Product.objects.all()  # Output: SELECT * FROM table
Product.objects.filter(name="Laptop", price=999.99)
# Output:
# Executing: SELECT * FROM table WHERE name = %s AND price = %s
# With values: ['Laptop', 999.99]
```

### Flask's Decorator-Based Routing

Flask uses decorators for route registration.
In the example below, the `route()` method returns a decorator function that registers handler functions in a dictionary, 
allowing the framework to dynamically map URL paths to their corresponding functions without modifying the original function behavior.

```python live_py
# Simplified version of Flask's routing system
class Flask:
    def __init__(self, name):
        self.name = name
        self.routes = {}
    
    def route(self, path):
        def decorator(func):
            self.routes[path] = func
            return func
        return decorator
    
    def handle_request(self, path):
        if path in self.routes:
            handler = self.routes[path]
            return handler()
        return "404 Not Found"

# Using the framework
app = Flask(__name__)

@app.route("/")
def index():
    return "Hello, World!"

@app.route("/about")
def about():
    return "About Us"

# Simulate requests
print(app.handle_request("/"))       # Output: Hello, World!
print(app.handle_request("/about"))  # Output: About Us
print(app.handle_request("/contact")) # Output: 404 Not Found
```