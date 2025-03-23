---
sidebar_position: 2
---

# Collections

The Metaprogramming API uses the same collection classes as the Classic API but provides more convenient ways to work with them. This section covers how to create and manipulate collections using the Metaprogramming API.

## Creating Collections

Creating collections works the same way as in the Classic API:

```python
from reactive.core.compute_graph import ComputedCollection
from reactive.meta import Service

# Create a service
service = Service("my_service")

# Create a computed collection
users = ComputedCollection[str, dict]("users", service.compute_graph)
```

## Basic Operations

Basic operations on collections are also the same:

```python
# Set values
users.set("user1", {"name": "Alice", "email": "alice@example.com"})
users.set("user2", {"name": "Bob", "email": "bob@example.com"})

# Get values
user = users.get("user1")  # Returns the user dict or None

# Delete values
users.delete("user2")

# Iterate over items
for user_id, user_data in users.iter_items():
    print(f"{user_id}: {user_data['name']}")
```

## Transforming Collections

The key difference with the Metaprogramming API is how you define and apply transformations. In the metaprogramming API, mapping collections is done using mapper functions decorated with `@one_to_one` or `@many_to_one`, along with the `map_collection` function:

```python
from reactive.meta import one_to_one, many_to_one, map_collection

# Define a transformation using a decorator
@one_to_one
def extract_name(user_data):
    return user_data.get("name")

# Apply the transformation to create a new collection
names = map_collection(users, extract_name)
```

## Dependency Detection

A powerful feature of the Metaprogramming API is automatic dependency detection. Dependencies between collections are automatically detected when they're used in mapper functions or resource definitions, reducing the need for explicit declarations.

```python
# This collection will automatically be detected as a dependency
reference_data = ComputedCollection[str, dict]("reference", service.compute_graph)

@one_to_one
def enrich_user(user_data):
    # The framework detects this reference to reference_data
    ref = reference_data.get(user_data.get("id"))
    if ref:
        # Create a new dict with enriched data
        result = dict(user_data)
        result["reference"] = ref
        return result
    return user_data

# Apply the transformation - reference_data is automatically included as a dependency
enriched_users = map_collection(users, enrich_user)
```

In this example, the framework automatically detects that `enrich_user` depends on `reference_data` and establishes the appropriate dependency in the compute graph.

## Multiple Collections

You can also work with multiple collections explicitly:

```python
products = ComputedCollection[str, dict]("products", service.compute_graph)

@one_to_one
def add_product_details(order, product_collection):
    product_id = order.get("product_id")
    product = product_collection.get(product_id)
    if product:
        result = dict(order)
        result["product_name"] = product.get("name")
        result["product_price"] = product.get("price")
        return result
    return order

# Explicitly pass the products collection as a parameter
orders_with_details = map_collection(orders, add_product_details, products)
```

## Combining Multiple Transformations

You can chain transformations to create complex data processing pipelines:

```python
# Start with raw data
raw_readings = ComputedCollection[str, float]("raw_readings", service.compute_graph)

# Define transformations
@one_to_one
def convert_celsius_to_fahrenheit(celsius):
    return (celsius * 9/5) + 32

@one_to_one
def classify_temperature(fahrenheit):
    if fahrenheit < 32:
        return {"temp": fahrenheit, "status": "freezing"}
    elif fahrenheit < 65:
        return {"temp": fahrenheit, "status": "cold"}
    elif fahrenheit < 85:
        return {"temp": fahrenheit, "status": "pleasant"}
    else:
        return {"temp": fahrenheit, "status": "hot"}

# Apply transformations in sequence
fahrenheit_readings = map_collection(raw_readings, convert_celsius_to_fahrenheit)
classified_readings = map_collection(fahrenheit_readings, classify_temperature)
```

## Aggregating Data

The `@many_to_one` decorator allows you to aggregate multiple values:

```python
@many_to_one
def calculate_average(values):
    if not values:
        return None
    return sum(values) / len(values)

# Apply the aggregation to create a new collection
averages = map_collection(readings, calculate_average)
```

In the next section, we'll explore mappers in more detail.