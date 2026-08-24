# Kubernetes Autoscaling Strategies — Interview Guide

Kubernetes autoscaling operates at three different layers: **pod size**, **pod replica count**, and **cluster node capacity**. A strong interview answer should explain how the Vertical Pod Autoscaler, Horizontal Pod Autoscaler, and Cluster Autoscaler work together without creating conflicting feedback loops.

> **Article correction:** The heading “Vertical Pod Autoscaler (HPA)” should say **Vertical Pod Autoscaler (VPA)**.

## 1. The Three Autoscalers

| Autoscaler | Main question | Observes | Changes |
|---|---|---|---|
| **VPA** | Is each pod sized correctly? | Historical CPU and memory usage | Pod resource requests and sometimes limits |
| **HPA** | How many replicas are required? | CPU, memory, custom, or external metrics | Pod replica count |
| **Cluster Autoscaler** | Does the cluster have enough nodes? | Unschedulable pending pods | Worker-node count |

### Interview summary

> VPA adjusts the size of individual pods, HPA adjusts the number of pods, and Cluster Autoscaler adjusts the number of worker nodes.

## 2. Vertical Pod Autoscaler (VPA)

VPA analyzes actual CPU and memory consumption and recommends—or applies—better resource requests.

### Why VPA matters

- Requests that are too low can cause CPU throttling, unstable performance, scheduling problems, or memory-related termination.
- Requests that are too high reserve capacity unnecessarily and increase cost.
- VPA continuously updates its recommendations as workload behavior changes.

### VPA operating modes

- **Off:** Generates recommendations without changing pods. This is the safest starting point in production.
- **Initial:** Applies recommendations only when pods are created.
- **Auto:** Applies updated recommendations by recreating pods when necessary.

Traditional VPA updates generally require pod replacement. Production workloads should therefore have:

- Multiple replicas
- Readiness probes
- Graceful shutdown
- Appropriate PodDisruptionBudgets
- Correct availability and rollout configuration

### Interview answer

> I normally begin with VPA in recommendation-only mode, observe its recommendations over a representative period, validate them against application behavior, and only then consider automatic updates.

## 3. Horizontal Pod Autoscaler (HPA)

HPA changes the replica count based on observed metrics.

### Replica calculation

```text
desiredReplicas = ceil(
  currentReplicas × currentMetricValue / targetMetricValue
)
```

Example:

```text
Current replicas: 2
Current CPU utilization: 150%
Target CPU utilization: 50%

desiredReplicas = ceil(2 × 150 / 50) = 6
```

CPU utilization is normally measured relative to a container's CPU request.

```yaml
resources:
  requests:
    cpu: 500m
```

If the container consumes `250m`, its CPU utilization is approximately 50% of its request.

### Useful HPA metrics

- CPU utilization
- Memory utilization
- Requests per second
- Queue depth
- Kafka consumer lag
- Active sessions
- Pending job count

Application metrics are often better than CPU when CPU usage does not directly represent demand.

### Interview points

- HPA requires accurate resource requests when scaling on CPU utilization.
- Configure scaling policies and stabilization windows to prevent flapping.
- Select a metric that correlates with user or business demand.
- HPA is reactive and does not instantly create usable node capacity.

### Interview answer

> I select a metric that correlates with demand. CPU may work for compute-bound services, but queue depth, request rate, or consumer lag is often better for asynchronous or I/O-heavy workloads.

## 4. Can HPA and VPA Run Together?

Yes, but they should not compete over the same signal.

The risky combination is:

- HPA scales using percentage of CPU request.
- VPA continuously changes that CPU request.

Example:

1. A pod requests `500m` CPU and consumes `400m`.
2. HPA observes 80% utilization.
3. VPA changes the request to `800m`.
4. The same `400m` consumption now appears as 50% utilization.
5. HPA changes its scaling decision even though real demand did not change.

This creates an interaction between two control loops.

### Safer combinations

- VPA manages CPU and memory while HPA uses request rate or queue depth.
- VPA adjusts memory only while HPA scales on CPU.
- VPA runs in recommendation mode while HPA controls replicas.
- Different workloads use different scaling strategies.

### Interview answer

> I avoid using HPA and VPA against the same resource signal. A common design is HPA on a business metric such as queue depth and VPA for resource right-sizing. Another option is memory-only VPA with CPU-based HPA.

## 5. Cluster Autoscaler

Cluster Autoscaler changes the number of worker nodes. It does not normally add nodes merely because average cluster CPU is high. It reacts when pods cannot be scheduled because suitable cluster capacity is unavailable.

### Scale-up flow

1. HPA increases the number of replicas.
2. The scheduler attempts to place the new pods.
3. Some pods remain in `Pending` state.
4. Cluster Autoscaler determines whether adding a node could make them schedulable.
5. The cloud provider provisions a new node.
6. The scheduler places the pending pods.

### Resource fragmentation

A cluster can have sufficient total free capacity but still be unable to schedule a pod.

Suppose three nodes each have 1 GiB free, while a new pod requests 2 GiB. The cluster has 3 GiB free in total, but no single node has the contiguous capacity required for that pod.

This is a scheduling and bin-packing problem—not simply an average-utilization problem.

### Interview answer

> Cluster Autoscaler reacts to unschedulable pods, not merely to average utilization. Resource fragmentation, affinity, node selectors, taints, volume topology, and available instance types can all affect whether adding a node will help.

## 6. Cluster Scale-Down

Cluster Autoscaler can remove an underutilized node when its movable pods can run safely on other nodes.

Typical process:

1. Determine whether the node's pods can fit elsewhere.
2. Cordon the candidate node.
3. Drain or evict its pods.
4. Terminate the empty node.

Scale-down may be blocked by:

- Restrictive PodDisruptionBudgets
- Pods using local storage
- Node affinity or selectors
- Taints and tolerations
- Singleton workloads
- Certain system pods
- Insufficient capacity on other nodes

A PodDisruptionBudget with `maxUnavailable: 0` can prevent voluntary eviction when no additional healthy replica exists.

## 7. Complete Traffic-Spike Sequence

```text
Traffic increases
        ↓
Application metric rises
        ↓
HPA increases replicas
        ↓
Scheduler places pods where capacity exists
        ↓
Remaining pods become Pending
        ↓
Cluster Autoscaler adds nodes
        ↓
New nodes become Ready
        ↓
Pending pods are scheduled
```

HPA can react in seconds, while cloud nodes may take minutes to become ready. During that delay, existing pods must continue handling the traffic.

### Ways to manage the provisioning gap

- Set a realistic HPA `minReplicas` value.
- Maintain spare node capacity.
- Use scheduled or predictive scaling for known peaks.
- Use low-priority placeholder pods for overprovisioning.
- Build faster-starting node images.
- Pre-pull large container images.
- Keep application startup fast.
- Configure correct readiness and startup probes.
- Support backpressure or temporary request queuing.

## 8. Sample Design Question

### Question

How would you design autoscaling for an API with unpredictable traffic?

### Strong answer

> I would first establish realistic resource requests through load testing and use VPA in recommendation mode to refine them. I would configure HPA using requests per second or CPU, depending on which metric correlates best with demand. I would set sensible minimum and maximum replicas, scaling policies, and stabilization windows.
>
> When HPA-created pods cannot fit, Cluster Autoscaler would add nodes. Because node provisioning takes longer than pod scaling, I would maintain enough minimum replicas or spare capacity to absorb short spikes. I would also configure readiness probes, multiple replicas, topology spreading, and a PodDisruptionBudget so node scale-down does not reduce availability. Finally, I would monitor pending pods, HPA limits, scheduling failures, node-provisioning latency, and application SLOs.

## 9. Common Interview Questions

### What is the difference between HPA and Cluster Autoscaler?

> HPA creates or removes pod replicas. Cluster Autoscaler creates or removes nodes. HPA may indirectly trigger Cluster Autoscaler when its new replicas cannot be scheduled.

### Why can pods remain pending when average cluster utilization is low?

> Kubernetes must place the complete resource request of each pod on one suitable node. Capacity may be fragmented, or scheduling may be restricted by affinity, taints, topology, volumes, or node selectors.

### Why must CPU requests be accurate for HPA?

> CPU utilization is calculated relative to CPU requests. Incorrect requests produce misleading utilization percentages and incorrect scaling decisions.

### Why might HPA fail to scale?

- Metrics Server or another metrics provider is unavailable.
- CPU requests are missing.
- `maxReplicas` has already been reached.
- The selected metric is configured incorrectly.
- Stabilization behavior is delaying the change.
- HPA targets the wrong workload.
- Application load is not reflected by the chosen metric.

### Why might Cluster Autoscaler not add a node?

- The pending pod would not fit any allowed node type.
- The node group's maximum size has been reached.
- Node affinity or selectors cannot be satisfied.
- Required tolerations are missing.
- Volume topology prevents placement.
- The pod is pending for a reason other than insufficient capacity.
- Cloud quota or provisioning errors exist.

### Why might Cluster Autoscaler not remove a node?

> Its pods may not be safely movable because of PodDisruptionBudgets, affinity rules, local storage, insufficient capacity elsewhere, or other eviction restrictions.

## 10. Thirty-Second Interview Answer

> Kubernetes provides autoscaling at three layers. VPA right-sizes CPU and memory requests for individual pods. HPA changes the replica count using resource or application metrics. Cluster Autoscaler changes the worker-node count when pods are unschedulable or nodes can be consolidated. HPA and Cluster Autoscaler naturally work in sequence: HPA creates replicas, and if they cannot fit, Cluster Autoscaler adds nodes. I avoid using HPA and VPA on the same resource signal because changing requests changes the utilization percentage seen by HPA. I also account for node-provisioning delay using minimum replicas, spare capacity, appropriate scaling policies, and disruption budgets.

## 11. Final Points to Remember

1. **VPA changes pod size.**
2. **HPA changes pod count.**
3. **Cluster Autoscaler changes node count.**
4. **Do not let HPA and VPA compete over the same resource metric.**
5. **Cluster Autoscaler reacts to unschedulable pods—not simply high average utilization.**
6. **HPA reacts faster than cloud nodes can be provisioned.**
7. **Use minimum replicas or spare capacity to handle the provisioning delay.**
8. **PodDisruptionBudgets protect availability but can also block node consolidation.**
9. **Choose scaling metrics that represent real application demand.**
10. **Monitor the complete chain from metrics to replicas, scheduling, node provisioning, and application SLOs.**
