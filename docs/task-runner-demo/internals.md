---
sidebar_position: 2
---

# Internal Implementation

## Metaclasses and Class Generation

The `@task` decorator and `Task` metaclass form the foundation of the framework's metaprogramming capabilities. When a function is decorated with `@task`, the following code will run:

```python
  def task(func):
      """Decorator to create a task class from a function"""
      @wraps(func)
      def wrapper(*args, **kwargs):
          return func(*args, **kwargs)
      # Create a new task class dynamically
      task_name = f"{func.__name__.title()}Task"
      class DynamicTask(metaclass=Task):
          execute = staticmethod(func)
          __module__ = func.__module__

      DynamicTask.__name__ = task_name
      return DynamicTask
  ```

This creates a new class that inherits the Task metaclass that the framework defined. The original function is also stored in the execute attribute of the class for later analysis and use.

```python
  class Task:
      """Task metaclass for creating task definitions"""
      _registry = {}
      def __new__(cls, name, bases, attrs):
          new_cls = super().__new__(cls, name, bases, attrs)
          if 'execute' in attrs:
              metadata = TaskMetadata(attrs['execute'])
              new_cls._metadata = metadata
              Task._registry[metadata.id] = new_cls
          return new_cls

  class TaskMetadata:
      def __init__(self, func):
          self.func = func
          self.id = str(uuid.uuid4())
          self.name = func.__name__
          self.signature = inspect.signature(func)
          self.doc = func.__doc__ 
              or "No description available"
          
      @property
      def input_schema(self) -> Dict[str, Type]:
        return {
          name: param.annotation
            if param.annotation != inspect.Parameter.empty
            else Any
          for name, param in 
              self.signature.parameters.items()
        }
  ```

The metaclass serves multiple purposes:
    
    - Maintains a global registry of all tasks
    - Provides introspection capabilities through the TaskMetadata class
    - Allows enforcement of task interface requirements with `input_schema`
    - Enables runtime modification of task behavior

This approach demonstrates how metaclasses can create powerful abstractions that feel natural to Python developers while providing sophisticated functionality under the hood.

## Type Annotations and Runtime Reflection

Python's type annotation system is leveraged extensively for both documentation and runtime behavior:

```python
class TaskAnalyzer:
    @staticmethod
    def analyze_task(task_cls: Type) -> Dict:
        if not hasattr(task_cls, '_metadata'):
            raise ValueError(f"Invalid task class: {task_cls}")
        metadata = task_cls._metadata
        source = inspect.getsource(metadata.func)
        
        # Extract type information and validate
        input_schema = metadata.input_schema
        for param_name, param_type in input_schema.items():
            if param_type == inspect.Parameter.empty:
                raise ValueError(f"Missing type annotation for parameter: {param_name}")
        return {...}
```

The framework uses `inspect.signature()` to extract type information at runtime, enabling:

- Automatic input validation
- Dynamic UI generation
- Type-safe serialization for execution
- Documentation generation

This showcases how Python's type system can be used beyond static type checking, becoming a powerful tool for runtime behavior modification.

## Dynamic Code Generation

One of the most powerful metaprogramming techniques demonstrated is dynamic code generation. The framework generates worker code on-the-fly using:

- [Jinja2](https://jinja.palletsprojects.com/en/stable/) templates for code structure
- AST manipulation for code analysis
- Dynamic module loading for execution

This approach allows for:
- Optimal worker code with minimal dependencies
- Environment-specific optimizations
- Runtime code modification based on execution context