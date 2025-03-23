# Resources

Resources are the core concept of the reactive framework that expose reactive data to clients. In the metaprogramming API, resources are defined using the `@resource` decorator, which simplifies their creation compared to the class-based approach in the classic API.

## Defining Resources

Use the `@resource` decorator to define a function that sets up a resource:

```python
from reactive.meta import resource, map_collection

# Define a resource with decorator
@resource
def processor(multiplier: float):
    multiplied_data = map_collection(raw_data, multiply_value, multiplier)
    return multiplied_data
```

## Resource Parameters

Unlike the classic API, there is no need to define a parameter model. The parameters of the resource function become the parameters of the resource. The framework automatically creates a Pydantic model for validating these parameters:

```python
# The resource parameters are derived from the function parameters
@resource
def temperature_monitor(min_threshold: float = 0.0, max_threshold: float = 100.0):
    # Resource implementation
    return monitored_data
```

When a client creates a stream for this resource, they'll need to provide these parameters:

```json
POST /v1/streams/temperature_monitor
{
    "min_threshold": 15.0,
    "max_threshold": 30.0
}
```

### Custom Parameter Models

You can also provide a custom Pydantic model for resource parameters:

```python
from pydantic import BaseModel

class AlertConfig(BaseModel):
    threshold: float
    notify_admin: bool = False
    alert_level: str = "warning"

@resource(param_model=AlertConfig)
def alert_monitor(threshold: float, notify_admin: bool, alert_level: str):
    # Resource implementation using these parameters
    return alerts_collection
```

## Resource Registration

Resources are automatically registered when they're defined with the `@resource` decorator. Unlike the classic API, you don't need to explicitly add them to the service - the framework takes care of this when the service starts.

The metaprogramming API maintains the same resource access pattern as the classic API but simplifies the resource definition process.

In the next section, we'll explore how to configure and run services that host these resources.