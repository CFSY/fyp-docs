# Mappers

Mappers are functions that transform data from one collection to another. In the metaprogramming API, mappers are defined using decorators, which dramatically simplifies their implementation compared to the class-based approach in the classic API.

## Types of Mappers

The framework provides two main types of mappers:

1. **One-to-One Mappers**: Transform a single value into another value with the same key
2. **Many-to-One Mappers**: Transform a list of values with the same key into a single value

### Defining One-to-One Mappers

Use the `@one_to_one` decorator to define a function that transforms individual values:

```python
from reactive.meta import one_to_one

@one_to_one
def temperature_to_fahrenheit(celsius_data):
    """Convert temperature from Celsius to Fahrenheit"""
    if celsius_data is None:
        return None
    
    celsius = celsius_data["temperature"]
    fahrenheit = celsius * 9/5 + 32
    
    return {
        "temperature": fahrenheit,
        "unit": "F",
        "timestamp": celsius_data["timestamp"]
    }
```

### Defining Many-to-One Mappers

Use the `@many_to_one` decorator for functions that aggregate multiple values:

```python
from reactive.meta import many_to_one

@many_to_one
def calculate_average(temperature_readings):
    """Calculate average temperature from a list of readings"""
    if not temperature_readings:
        return None
    
    avg_temp = sum(reading["temperature"] for reading in temperature_readings) / len(temperature_readings)
    
    return {
        "average_temperature": avg_temp,
        "sample_count": len(temperature_readings),
        "timestamp": temperature_readings[-1]["timestamp"]
    }
```

### Passing Parameters to Mappers

You can pass additional parameters to mappers when mapping collections:

```python
@one_to_one
def scale_value(value, factor):
    """Scale a value by a factor"""
    if value is None:
        return None
    return value * factor

# Use the mapper with a specific factor
scaled_collection = map_collection(original_collection, scale_value, 2.5)
```

## Automatic Dependency Detection

One of the most powerful features of the metaprogramming API is its ability to automatically detect dependencies between collections. When a mapper function references a collection, the framework will detect this and establish the appropriate dependency:

```python
reference_collection = ComputedCollection("reference", compute_graph)

@one_to_one
def enrich_data(value):
    # The reference_collection is automatically detected as a dependency
    reference = reference_collection.get("config")
    return {
        "original_value": value,
        "enriched_value": value * reference["factor"]
    }

enriched_collection = map_collection(data_collection, enrich_data)
```

The metaprogramming API's mapper approach significantly reduces boilerplate code compared to the classic API, while maintaining all the functionality and adding automatic dependency detection.

In the next section, we'll explore how to expose these reactive collections as resources that can be accessed by clients.
