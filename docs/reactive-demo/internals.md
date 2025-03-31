---
sidebar_position: 5
---

# Metaprogramming Internals of the Reactive Framework

The Metaprogramming API achieves its concise syntax and automatic behaviors by building upon the Classic API and employing several metaprogramming techniques. It acts as a translation layer, converting the user-friendly meta-constructs into their classic counterparts behind the scenes.

## Decorator-Based Components (`@resource`, `@one_to_one`, `@many_to_one`)

Decorators are the primary mechanism for defining components in the Meta API.

### `@resource`: implemented in [resource.py](https://github.com/CFSY/meta-reactive/blob/main/src/reactive/meta/resource.py#L61)
- **Function:** Transforms a user-defined function into a `Resource` instance.
- **Mechanism:**
    1.  Captures the decorated function (`func`).
    2.  Determines the resource name (either provided or from `func.__name__`).
    3.  Analyzes the function's signature and type hints using `inspect.signature` and `typing.get_type_hints` to automatically generate a Pydantic `param_model` if one isn't explicitly provided.
    4.  Creates an instance of the Meta API's `Resource` class.
    5.  Registers the captured function (`func`) as the `_setup_method` within the `Resource` instance. This method will be called later by the underlying classic resource's `setup_resource_collection`.
    6.  Adds the `Resource` instance to the `global_resource_registry`. The `Service` class later iterates over this registry during startup to automatically add resources.
- **Effect:** Allows users to define a resource and its parameter schema within a single function definition, eliminating the need for separate `Resource` and `ResourceParams` classes and explicit registration.

### `@one_to_one` / `@many_to_one`: implemented in [mapper.py](https://github.com/CFSY/meta-reactive/blob/main/src/reactive/meta/mapper.py#L128)
- **Function:** Transforms a user function into a `MapperWrapper` instance.
- **Mechanism:**
    1.  Captures the decorated function (`func`) and the specified `MapperType` (OneToOne or ManyToOne).
    2.  Creates a `MapperWrapper` instance, storing the function and type.
    3.  Crucially, the `MapperWrapper.__init__` calls `_detect_dependencies` which uses the `FrameworkDetector` (see below) to find any `ComputedCollection` instances referenced within the mapper function's code. These detected dependencies are stored.
- **Effect:** Allows users to define mapping logic in simple functions. The wrapper holds the logic and detected dependencies, ready to be used by `map_collection`.

## The `map_collection` Bridge Function

Defined in [`reactive/meta/mapper.py`](https://github.com/CFSY/meta-reactive/blob/main/src/reactive/meta/mapper.py#L185), this function acts as the bridge between the Meta API's mappers and the Classic API's collection mapping mechanism.

**Function:** Applies a Meta API mapper (a `MapperWrapper`) to a `ComputedCollection`.

**Mechanism:**
1.  Takes the `collection`, the `mapper` (which must be a `MapperWrapper` instance created by the decorators), and any additional `*args`, `**kwargs` to be passed to the user's mapping function.
2.  Checks the `mapper_wrapper.mapper_type`.
3.  Selects the appropriate **internal Classic Mapper implementation** (`_OneToOneMapperImpl` or `_ManyToOneMapperImpl`). These internal classes are simple wrappers that take the user's function and execute it within the `map_value` or `map_values` method required by the Classic API.
4.  Calls the underlying `collection.map` method (from `ComputedCollection`), passing the selected **internal Classic Mapper implementation class** and the user's original mapping function (`mapper_wrapper.mapper_func`) along with the `*args` and `**kwargs`.

**Effect:** Translates the user-friendly Meta API `map_collection(collection, mapper_func, ...)` call into the Classic API's `collection.map(ClassicMapperImpl, mapper_func, ...)` structure, ensuring compatibility with the core computation graph logic while hiding the Classic Mapper class definitions from the user.

## The `FrameworkDetector` and Automatic Dependency Detection

:::tip

To see the detector in action, you can run an example locally. It is available under the [`examples/detector`](https://github.com/CFSY/meta-reactive/tree/main/examples/detector) directory.

FrameworkDetector is implemented in [detector.py](https://github.com/CFSY/meta-reactive/blob/main/src/reactive/meta/detector.py)

:::

The `FrameworkDetector` (defined in `reactive/meta/detector.py`) is perhaps the most sophisticated metaprogramming component, enabling automatic dependency detection within user code.

Its primary goal is to analyze the source code of functions (like those decorated with `@one_to_one` or `@resource`) and identify if they reference global or non-local variables that are instances of framework components, specifically `ComputedCollection`. This allows the framework to automatically establish necessary links in the `ComputeGraph` without requiring explicit declarations from the user.

### Marking and Identification:
The detector uses a unique attribute name (e.g., `__reactive_meta_component__`) to "mark" functions or classes that are part of the framework or created by its decorators.
This marking is done **internally** by helper functions/classes provided by the detector:
- `detector.get_function_decorator()`: Returns a decorator used on functions like `@resource`, `@one_to_one`, etc. This decorator wraps the function and sets the marker attribute on the wrapper.
- `detector.get_metaclass()`: Returns a metaclass used on internal framework classes (like `Service`, `Resource`). This metaclass sets the marker attribute on the class itself and analyzes its methods upon class creation.
- The `is_framework_component()` method simply checks for the presence of this marker attribute.

### Detection via AST Analysis:
When `detect_framework_usage` is called (e.g., inside `MapperWrapper.__init__`), it uses the `CodeAnalyzer` sub-component of FrameworkDetector. The following occurs internally:
- **Source Code Retrieval:** `inspect.getsource()` retrieves the source code of the target function/method.
- **Parsing:** `libcst.parse_module()` converts the source code string into an Abstract Syntax Tree (AST). Libcst is chosen for its robustness and ability to preserve formatting information.
- **AST Traversal:** A custom visitor class, `FrameworkReferenceCollector`, walks the AST.
    - It overrides methods like `visit_Call` (for function calls) and `visit_Attribute` (for attribute access).
    - Inside these visitors, it identifies names (e.g., `my_collection` in `my_collection.get()`).
- **Name Resolution:** The crucial step is linking syntax nodes (like `cst.Name`) back to actual Python objects. 
    - The visitor uses the `global_namespace` (obtained via `func.__globals__` and `inspect.getmodule(func).__dict__`) and `local_namespace` (if applicable, like closures) passed to it. 
    - The `_resolve_name` helper attempts to find the object corresponding to the name in these namespaces.
- **Framework Check:** If a name resolves to an object, `detector.is_framework_component(obj)` is called. If it **is** a marked framework component, its name (e.g., `"my_collection"`, `"my_collection.get()"`) is added to the set of references.
- **Indirect Calls:** The analyzer also uses `FunctionCallExtractor` to find functions called **within** the analyzed function. It then recursively analyzes these called functions (using cached results via `get_framework_references` or calling `detect_framework_usage` again) to find indirect dependencies.
- **Result:** Returns a set of strings representing the names of framework components referenced directly or indirectly within the analyzed code.

### Integration
The detector is instantiated once. In our framework, this is done in [`common.py`](https://github.com/CFSY/meta-reactive/blob/main/src/reactive/meta/common.py)


```python
detector = FrameworkDetector("reactive_meta")
framework_function = detector.get_function_decorator()
FrameworkClass = detector.get_metaclass()
```


The marking decorators/metaclasses (`framework_function`, `FrameworkClass`) are exported from `common.py` and used internally within the `reactive.meta` package on components like `@resource`, `@one_to_one`, `Service`, etc. The detector decorators/metaclasses were implemented to be "invisible", making their internal usage extremely easy. Notice how we are decorating the `@one_to_one` decorator.


```python
class Resource(Generic[K, V], metaclass=FrameworkClass):
...
```
```python
@framework_function
def one_to_one(func):
...
```

This seamless integration means dependency detection happens automatically when a user decorates a function, without them needing to interact with the detector directly.