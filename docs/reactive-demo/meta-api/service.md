# Service

In the metaprogramming API, the Service class is a wrapper around the classic Service implementation with automatic resource registration.

:::info
The API details are identical to the [Classic API](../classic-api/service.md#http-endpoints).
:::

### Creating a Service

Use the `Service` class to create a service:

```python
from reactive.meta import Service

# Create a service with a name and port
service = Service("my_reactive_app", port=8080)
```

### Starting the Service

Starting a service is an asynchronous operation:

```python
import asyncio

async def main():
    # Create service
    service = Service("my_reactive_app", port=8080)
    
    # Start the service
    await service.start()

# Run the main function
asyncio.run(main())
```

### Resource Registration

Unlike the classic API, you don't need to explicitly add resources to the service. Resources defined with the `@resource` decorator are automatically registered when the service starts:

```python
from reactive.meta import resource, Service

@resource
def my_resource(param1: str, param2: int = 0):
    # Resource implementation
    return result_collection

# Create and start the service
service = Service("my_app")
# No need to call service.add_resource - it happens automatically!
```
