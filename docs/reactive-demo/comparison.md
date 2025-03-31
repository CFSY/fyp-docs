---
sidebar_position: 4
---

# API Comparison: Classic vs Metaprogramming

The reactive framework provides two API styles that offer the same functionality but with different syntax and levels of abstraction. Let's compare them using the temperature monitor example.

:::tip

You can run this example locally. It is available under the [`examples/temp_monitor`](https://github.com/CFSY/meta-reactive/tree/main/examples/temp_monitor) directory.

:::

## Temperature Monitor Example

### Problem Description

The Temperature Monitor system collects readings from multiple temperature sensors placed in different locations (office, server room, warehouse). The system needs to:

1. Compute average temperatures for each sensor
2. Compare temperatures against acceptable ranges for each location
3. Generate alerts when temperatures exceed thresholds
4. Stream alerts to clients in real-time

### Architecture Diagram

Here's a diagram showing the architecture of the Temperature Monitor application:

```mermaid
graph TD
    A[Sensor Readings Collection] --> |average_temperature mapper| B[Averages Collection]
    C[Location References Collection] -.-> |used by enhanced_alert mapper| E
    B --> |enhanced_alert mapper| E[Alerts Collection]
    E --> |resource| F[Temperature Monitor Resource]
    G[Service] --> |exposes| F
    H[Simulation] --> |updates| A
    H --> |updates| C
    I[Client] --> |connects to| G
    G --> |streams to| I
    
    style A fill:#f9f9f9,stroke:#333,stroke-width:1px
    style B fill:#f9f9f9,stroke:#333,stroke-width:1px
    style C fill:#f9f9f9,stroke:#333,stroke-width:1px
    style E fill:#f9f9f9,stroke:#333,stroke-width:1px
    style F fill:#e6f7ff,stroke:#333,stroke-width:1px
    style G fill:#e6f7ff,stroke:#333,stroke-width:1px
```

1. **Sensor Readings Collection**: Contains lists of temperature readings for each sensor.
2. **Average Temperature Mapper**: Computes the average temperature for each sensor from its readings.
3. **Enhanced Alert Mapper**: Generates alerts by comparing temperatures against reference data.
4. **Location References Collection**: Contains acceptable temperature ranges for each location.
5. **Alerts Collection**: Contains generated alerts for each sensor.
6. **Temperature Monitor Resource**: Exposes the alerts to clients via the service.



### Classic VS Metaprogramming API

The primary advantage of the Metaprogramming API lies in its ability to reduce boilerplate and provide a more declarative syntax, achieving the same functionality as the Classic API with significantly less code. Let's compare key aspects side-by-side, using examples derived from the Temperature Monitor.

#### 1. **Mappers**:
    - **Classic:** Requires defining a class inheriting from a base mapper type.
        ```python
        # Define mappers
        class AverageTemperatureMapper(ManyToOneMapper[str, SensorReading, Tuple[float, str]]):
            """Computes average temperature for each sensor from its readings"""

            def map_values(self, readings: list[SensorReading]) -> Tuple[float, str]:
                if not readings:
                    return 0.0, ""
                avg_temp = sum(r.temperature for r in readings) / len(readings)
                # Return both temperature and location
                return avg_temp, readings[0].location


        class EnhancedAlertMapper(OneToOneMapper[str, Tuple[float, str], Dict[str, str]]):
            """
            Generates enhanced alerts by comparing current temperatures
            with reference data for each location
            """

            def __init__(
                self, location_references: ComputedCollection, global_threshold: float
            ):
                self.location_references = location_references
                self.global_threshold = global_threshold

            def map_value(self, value: Tuple[float, str]) -> Dict[str, str]:
                avg_temp, location = value

                # Get reference data for this location
                location_info = self.location_references.get(location)

                # Use common evaluation logic
                return evaluate_temperature_alert(
                    avg_temp, location, location_info, self.global_threshold
                )
        ```

    - **Metaprogramming:** Uses a simple decorator (`@many_to_one`) on a standard function.
        ```python
        # Define mappers using decorators
        @many_to_one
        def average_temperature(readings: List[SensorReading]) -> Tuple[float, str]:
            """Computes average temperature for each sensor from its readings"""

            if not readings:
                return 0.0, ""
            avg_temp = sum(r.temperature for r in readings) / len(readings)
            # Return both temperature and location
            return avg_temp, readings[0].location


        @one_to_one
        def enhanced_alert(value: Tuple[float, str], global_threshold: float) -> Dict[str, str]:
            """
            Generates enhanced alerts by comparing current temperatures
            with reference data for each location
            """
            avg_temp, location = value

            # Get reference data for this location
            location_info = location_ref_collection.get(location)

            # Use common evaluation logic
            return evaluate_temperature_alert(
                avg_temp, location, location_info, global_threshold
            )
        ```

    The Meta API eliminates class definition boilerplate, focusing purely on the transformation logic. Type hints remain for clarity and potential future tooling.

#### 2. **Resource Definition with Parameters**:
    - **Classic:** Requires a `ResourceParams` subclass and a `Resource` subclass, with explicit parameter passing.
        ```python
        # Define resource
        class MonitorParams(ResourceParams):
            threshold: float  # Global alert threshold in Celsius


        # Resource Implementation
        class TemperatureMonitorResource(Resource[str, dict]):
            def __init__(self, readings_collection, location_ref_collection, compute_graph):
                super().__init__(MonitorParams, compute_graph)
                self.readings = readings_collection
                self.location_references = location_ref_collection

            def setup_resource_collection(self, params: MonitorParams):
                # Compute average temperatures
                averages = self.readings.map(AverageTemperatureMapper)

                # Generate enhanced alerts using both averages and location references
                alerts = averages.map(
                    EnhancedAlertMapper,
                    self.location_references,  # Pass the location references collection
                    params.threshold,  # Pass the global threshold
                )

                return alerts
        ```

    - **Metaprogramming:** Uses the `@resource` decorator. Function parameters *become* resource parameters, automatically generating a Pydantic model internally. Dependencies are often detected automatically.
        ```python
        # Define resource using decorator
        @resource
        def temperature_monitor(threshold: float):
            # Compute average temperatures
            averages = map_collection(readings_collection, average_temperature)

            # Generate enhanced alerts using both averages and location references
            alerts = map_collection(averages, enhanced_alert, threshold)

            return alerts
        ```

    The Meta API drastically reduces code. It merges parameter definition and resource logic, infers parameter models, and automatically handles dependencies (like `location_ref_collection` used within `enhanced_alert`), making the code more declarative.

#### 3. **Resource Registration**:
    - **Classic:** Requires explicit instantiation and registration with the service.
        ```python
        # Create and add resource
        temperature_monitor = TemperatureMonitorResource(
            readings_collection, location_ref_collection, service.compute_graph
        )
        service.add_resource("temperature_monitor", temperature_monitor)
        ```
        - **Metaprogramming:** Registration is automatic for functions decorated with `@resource`.
        ```python
        # Nothing needed
        ```

    Eliminates manual registration steps, reducing potential errors and simplifying setup. The framework discovers resources automatically.

### Summary of Differences:

| Feature             | Classic API                         | Metaprogramming API                       | Benefit of Meta API                       |
| :------------------ | :---------------------------------- | :---------------------------------------- | :---------------------------------------- |
| **Mapper Def.**     | Inherited Class                     | Decorated Function                        | Less boilerplate, focus on logic          |
| **Resource Def.**   | Inherited Class + Param Class       | Decorated Function                        | Concise, integrated parameter definition  |
| **Registration**    | Explicit `service.add_resource()`   | Automatic via `@resource`                 | Simpler setup, less error-prone           |
| **Dependencies**    | Explicit (constructor/map args)     | Automatic Detection                       | Reduced boilerplate, more declarative     |
| **Overall**         | Verbose, Explicit, OOP-standard     | Concise, Declarative, Less "ceremony"     | Improved Developer Experience             |

The comparison clearly shows that the Metaprogramming API provides a more streamlined and developer-friendly interface by leveraging metaprogramming to handle boilerplate and automate common tasks like registration and dependency management.