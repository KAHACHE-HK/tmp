*Thinking...*

> **Deconstructing the Request**
> 
> I'm focused on the user's core need: merging two features. First, the drill-down capability; I need a clear path from a high-level service view to internal details. Second, the dynamic chart creation. I'm imagining how these pieces will fit together, aiming for a cohesive and functional script.
> 
> 
> **Synthesizing the Components**
> 
> I'm now diving into the data structures. Specifically, I'm mapping the `external_connections` dictionary from the drill-down script to fit with the parallel execution script's needs. I'm thinking about how to represent concurrent function calls, possibly adding a `parallel` keyword within the LikeC4 syntax, and how to integrate that with the nested view structure.
> 
> 
> **Revising the Structure**
> 
> I'm refining the data structure design. The previous attempt felt overly complex, so I'm reconsidering how to represent connections. Focusing on the parent span ID as the trigger for parallel identification is key. The modified structure should be able to group multiple function calls. I'm contemplating the best balance of simplicity and expressiveness.
> 
> 
> **Consolidating the Logic**
> 
> I'm refining the data structure for a second pass. I think a nested dictionary keyed by `(source_service, source_comp)` and then `parent_span_id` with the target connections as a set is the best choice. I'm focusing on how to generate the `parallel` blocks in the C4 code, iterating on this new structure.
> 
> 
> **Structuring the Connections**
> 
> I'm focusing on how to represent the connections data. I've shifted my attention to the source of the parallel calls, which are the parent spans. The data will be keyed by a tuple, which will be a structure for the parent service, parent component and a dictionary for the different span ids. The target will be a set of the target service.
> 
> 
> **Refining the Structure**
> 
> I'm now focused on the core data structure. I'm moving from a nested defaultdict to a more direct approach. The key will now be a tuple representing the source service and component. Then a dictionary keyed by parent span ID. The value is a set of the target service and component tuples.
> 
> 
> **Finalizing the Implementation**
> 
> I'm now focusing on the core code generation. The strategy is to construct a LikeC4 model string from the gathered data, and then iterate through the new `connections` data structure to create `parallel` blocks. I'm also verifying how the new C4 output will interface with the existing views structure, ensuring the final output is correct.
> 
> 
> **Mapping the Connections**
> 
> I'm now clarifying the crucial data structure. The focus is on the grouping key: the parent span ID. The ultimate goal is a structure representing calls from a service and component, identified by a parent span ID, with the target service and component connections for each call. Then, I'll build the `parallel` blocks.
> 
> 
Of course. This is an excellent next step, as it combines the macro-level view of parallel operations with the micro-level detail of function-specific tracing. We can merge the logic from both previous scripts to create a truly powerful and representative model of your system.

The core idea is to use the component-level granularity from the drill-down script as the source and target for our connections, while using the parallel-detection logic from the second script.

Here is the plan:
1.  **Adopt the Drill-Down Data Model:** We'll continue to model services as `system` elements and their functions/spans as `component` elements.
2.  **Enhance the Connection Data Structure:** We need a structure that tracks which specific parent span (`parentSpanId`) from a source component initiated calls to which target components. This is the key to detecting parallelism at the component level.
3.  **Refine the C4 Generation Logic:**
    *   When defining relationships, we'll check if a single parent span resulted in calls to multiple downstream components.
    *   If so, we'll generate a `parallel` block originating from the specific source component.
    *   If not, we'll generate a standard component-to-component relationship arrow.
4.  **Maintain the Drill-Down Views:** The view logic for a high-level system map and detailed component maps remains unchanged, as it's a perfect fit for this combined model.

### The Combined Python Script: Drill-Down with Parallel Execution

This script integrates both features into a single, cohesive solution.

```python
import json
from collections import defaultdict

# --- Helper Functions (Unchanged) ---

def get_service_name(span):
    resource = span.get('resource', {})
    if 'attributes' in resource:
        for attr in resource['attributes']:
            if attr.get('key') == 'service.name':
                return attr['value'].get('stringValue')
    return None

def get_span_attribute(span, key):
    if 'attributes' in span:
        for attr in span['attributes']:
            if attr.get('key') == key:
                return next(iter(attr['value'].values()))
    return None

def get_connection_details(span):
    http_method = get_span_attribute(span, 'http.method')
    if http_method:
        return f"HTTP {http_method}"
    rpc_system = get_span_attribute(span, 'rpc.system')
    if rpc_system:
        return f"{rpc_system.upper()}"
    return "calls"

# --- Core Logic for Combined Model ---

def generate_combined_service_map(trace_file_path, output_file_path):
    """
    Generates a LikeC4 service map with drill-down and parallel execution handling.
    """
    try:
        with open(trace_file_path, 'r') as f:
            trace_data = json.load(f)
    except (FileNotFoundError, json.JSONDecodeError) as e:
        print(f"Error reading trace file: {e}")
        return

    all_spans = []
    if 'resourceSpans' in trace_data:
        for resource_span in trace_data['resourceSpans']:
            resource = resource_span.get('resource', {})
            for scope_span in resource_span.get('scopeSpans', []):
                for span in scope_span.get('spans', []):
                    span['resource'] = resource
                    all_spans.append(span)

    spans_by_id = {span['spanId']: span for span in all_spans}
    services = set()
    service_components = defaultdict(set)
    # The key data structure:
    # (source_service, source_comp) -> parent_span_id -> set of (target_service, target_comp, desc)
    connections = defaultdict(lambda: defaultdict(set))
    internal_connections = set()

    for span in all_spans:
        service_name = get_service_name(span)
        span_name = span.get('name', 'unknown_op')
        if not service_name:
            continue
            
        services.add(service_name)
        service_components[service_name].add(span_name)

        parent_span_id = span.get('parentSpanId')
        if parent_span_id and parent_span_id in spans_by_id:
            parent_span = spans_by_id[parent_span_id]
            parent_service_name = get_service_name(parent_span)
            parent_span_name = parent_span.get('name', 'unknown_op')

            if not parent_service_name:
                continue

            if service_name == parent_service_name:
                internal_connections.add((service_name, parent_span_name, span_name))
            else:
                source_key = (parent_service_name, parent_span_name)
                target_tuple = (service_name, span_name, get_connection_details(span))
                connections[source_key][parent_span_id].add(target_tuple)

    # --- Generate LikeC4 Content ---
    def sanitize(name):
        return ''.join(c if c.isalnum() else '_' for c in name)

    likec4_content = "specification {\n"
    likec4_content += "  element system\n"
    likec4_content += "  element component\n"
    likec4_content += "}\n\n"
    likec4_content += "model {\n"

    for service in sorted(list(services)):
        likec4_content += f'  system {sanitize(service)} "{service}" {{\n'
        for component in sorted(list(service_components[service])):
            likec4_content += f'    component {sanitize(component)} "{component}"\n'
        likec4_content += '  }\n\n'

    for (service, source_comp, target_comp) in sorted(list(internal_connections)):
        likec4_content += f'  {sanitize(service)}.{sanitize(source_comp)} -> {sanitize(service)}.{sanitize(target_comp)}\n'

    # Generate external connections, checking for parallelism
    for (source_service, source_comp), span_groups in connections.items():
        for targets in span_groups.values():
            source_element = f'{sanitize(source_service)}.{sanitize(source_comp)}'
            if len(targets) == 1:
                # Single call
                target_service, target_comp, desc = list(targets)[0]
                target_element = f'{sanitize(target_service)}.{sanitize(target_comp)}'
                likec4_content += f'  {source_element} -> {target_element} "{desc}"\n'
            else:
                # Parallel calls
                likec4_content += f"  {source_element} -> 'fan-out' {{\n"
                likec4_content += "    parallel {\n"
                for target_service, target_comp, desc in sorted(list(targets)):
                    target_element = f'{sanitize(target_service)}.{sanitize(target_comp)}'
                    likec4_content += f'      -> {target_element} "{desc}"\n'
                likec4_content += "    }\n"
                likec4_content += "  }\n"

    likec4_content += "}\n\n"
    
    # --- Generate Views (Identical to Drill-Down Script) ---
    likec4_content += "views {\n"
    likec4_content += "  view index of system {\n"
    likec4_content += "    title 'System Landscape'\n"
    likec4_content += "    include *\n"
    for service in sorted(list(services)):
        likec4_content += f'    navigateTo view_of_{sanitize(service)} on {sanitize(service)}\n'
    likec4_content += "  }\n\n"

    for service in sorted(list(services)):
        likec4_content += f'  view view_of_{sanitize(service)} of system {{\n'
        likec4_content += f'    title "Components of {service}"\n'
        likec4_content += f'    include {sanitize(service)}.*\n\n'
        connected_services = set()
        for (s_service, _), span_groups in connections.items():
            if s_service == service:
                for targets in span_groups.values():
                    for t_service, _, _ in targets:
                        connected_services.add(t_service)
        for (s_service, _), span_groups in connections.items():
            for targets in span_groups.values():
                for t_service, _, _ in targets:
                    if t_service == service:
                        connected_services.add(s_service)
        for connected_s in sorted(list(connected_services)):
             likec4_content += f'    include {sanitize(connected_s)}\n'
        likec4_content += '  }\n\n'

    likec4_content += "}\n"

    try:
        with open(output_file_path, 'w') as f:
            f.write(likec4_content)
        print(f"Successfully generated combined service map at {output_file_path}")
    except IOError as e:
        print(f"Error writing to output file: {e}")

# --- Main Execution Block ---
if __name__ == '__main__':
    sample_trace_file = 'sample_traces_combined.json'
    output_c4_file = 'service_map_combined.c4'

    # Sample trace data with intra-service calls, single calls, and parallel calls
    sample_otel_traces = {
      "resourceSpans": [
        {"resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "api-gateway"}}]},
         "scopeSpans": [{"spans": [
            {"traceId": "1", "spanId": "A", "name": "POST /composite-endpoint"},
            {"traceId": "1", "spanId": "B", "parentSpanId": "A", "name": "authenticate_and_route"},
            # Parallel calls originating from span B
            {"traceId": "1", "spanId": "C", "parentSpanId": "B", "name": "call_user_service"},
            {"traceId": "1", "spanId": "D", "parentSpanId": "B", "name": "call_product_service"}
        ]}]},
        {"resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "user-service"}}]},
         "scopeSpans": [{"spans": [
            {"traceId": "1", "spanId": "E", "parentSpanId": "C", "name": "get_user_profile", "attributes": [{"key": "rpc.system", "value": {"stringValue": "grpc"}}]}
        ]}]},
        {"resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "product-service"}}]},
         "scopeSpans": [{"spans": [
            {"traceId": "1", "spanId": "F", "parentSpanId": "D", "name": "find_product_by_id", "attributes": [{"key": "http.method", "value": {"stringValue": "GET"}}]},
            # A single, non-parallel call from within the product-service
            {"traceId": "1", "spanId": "G", "parentSpanId": "F", "name": "check_stock_level"}
        ]}]},
        {"resource": {"attributes": [{"key": "service.name", "value": {"stringValue": "inventory-service"}}]},
         "scopeSpans": [{"spans": [
            {"traceId": "1", "spanId": "H", "parentSpanId": "G", "name": "query_stock_db"}
        ]}]}
      ]
    }
    with open(sample_trace_file, 'w') as f:
        json.dump(sample_otel_traces, f, indent=2)

    generate_combined_service_map(sample_trace_file, output_c4_file)
```

### How It Works: The Best of Both Worlds

1.  **The Data Structure**: The `connections` dictionary is the heart of the solution.
    `connections[(source_service, source_comp)][parent_span_id].add(...)`
    It groups all outgoing calls by their precise origin (`source_service` and `source_comp`) and then further groups them by the `parent_span_id` that initiated them. This allows us to know that the `authenticate_and_route` component made two calls *from the same instance of its execution* (span `B`).

2.  **Generation Logic**: When writing the model, the script iterates through these groups.
    *   If a group of targets tied to a `parent_span_id` has only one element, it's a standard call: `source.component -> target.component`.
    *   If it has multiple elements, it's a parallel call, and the script generates the `source.component -> 'fan-out' { parallel { ... } }` structure.

### Example Output (`service_map_combined.c4`)

The generated `.c4` file from the sample data will be:

```c4
specification {
  element system
  element component
}

model {
  system api_gateway "api-gateway" {
    component POST__composite_endpoint "POST /composite-endpoint"
    component authenticate_and_route "authenticate_and_route"
    component call_product_service "call_product_service"
    component call_user_service "call_user_service"
  }

  system inventory_service "inventory-service" {
    component query_stock_db "query_stock_db"
  }

  system product_service "product-service" {
    component check_stock_level "check_stock_level"
    component find_product_by_id "find_product_by_id"
  }

  system user_service "user-service" {
    component get_user_profile "get_user_profile"
  }

  api_gateway.POST__composite_endpoint -> api_gateway.authenticate_and_route
  product_service.find_product_by_id -> inventory_service.query_stock_db "calls"
  api_gateway.authenticate_and_route -> 'fan-out' {
    parallel {
      -> product_service.find_product_by_id "HTTP GET"
      -> user_service.get_user_profile "GRPC"
    }
  }
}

views {
  /* ... view definitions for drill-down ... */
}
```

When you visualize this:
*   **High-Level View (`index`)**: You will see the four services (`api-gateway`, `user-service`, `product-service`, `inventory-service`) and their interactions.
*   **Drill into `api-gateway`**: Clicking on `api-gateway` will take you to its component view. Here, you will see the `authenticate_and_route` component with two arrows emerging from it in a `parallel` block, one going to `user-service` and the other to `product-service`.
*   **Drill into `product-service`**: You will see its internal components, including the standard, non-parallel call from `find_product_by_id` to the external `inventory-service`.

This final script provides a highly detailed and interactive architectural diagram, automatically generated from the real-world operational data of your system.
