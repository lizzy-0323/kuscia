# Bandwidth Request Support in Kuscia Tasks

## Overview

Kuscia now supports bandwidth resource specification in KusciaTasks, allowing you to request and limit the network bandwidth available to your tasks. This feature enables better resource management and scheduling based on available bandwidth capacity.

## How to Specify Bandwidth Requests

Bandwidth requests are specified in the same way as other Kubernetes resource requests (CPU, memory) using the standard `resources` section in container specifications:

```yaml
resources:
  requests:
    bandwidth: 500Mi  # Request 500 Mbps bandwidth
  limits:
    bandwidth: 1Gi    # Limit bandwidth to 1 Gbps
```

## Units

Bandwidth is specified using Kubernetes resource quantity format:

- `K` or `k` - Kilobits per second (Kbps)
- `M` - Megabits per second (Mbps)
- `G` - Gigabits per second (Gbps)
- `T` - Terabits per second (Tbps)
- `P` - Petabits per second (Pbps)

You can also use the binary units with the `i` suffix:
- `Ki` - 1024 bits per second
- `Mi` - 1048576 bits per second
- `Gi` - 1073741824 bits per second
- And so on...

## Example

Here's a complete example of a KusciaTask with bandwidth requests:

```yaml
apiVersion: kuscia.secretflow/v1alpha1
kind: KusciaTask
metadata:
  name: bandwidth-request-example
spec:
  initiator: alice
  parties:
  - domainID: alice
    role: host
    template:
      spec:
        containers:
        - name: task-container
          image: secretflow/secretflow-anolis8:latest
          command: ["python3", "-c", "print('Hello, world!')"]
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
              bandwidth: 500Mi  # Request 500 Mbps bandwidth
            limits:
              cpu: 200m
              memory: 200Mi
              bandwidth: 1Gi    # Limit bandwidth to 1 Gbps
```

## Considerations

1. **Node Bandwidth Capacity**: Tasks will only be scheduled on nodes that have sufficient available bandwidth capacity to meet the requests.

2. **Default Values**: If no bandwidth request is specified, no specific bandwidth will be reserved for the task.

3. **Limit Enforcement**: Bandwidth limits are enforced to prevent tasks from using more bandwidth than specified.

4. **Validation**: The scheduler will validate that:
   - Requested bandwidth is a valid quantity format
   - Requested bandwidth doesn't exceed the node's total bandwidth capacity
   - Bandwidth limit is not less than bandwidth request

## Relationship with Node Bandwidth Capacity

Node bandwidth capacity is configured in the node's capacity configuration. Bandwidth requests from KusciaTasks are considered during scheduling to ensure that the sum of all task bandwidth requests on a node doesn't exceed the node's available bandwidth capacity.
