# Demo Talk Track

Use this talk track to position OpenShift Virtualization as a platform approach for running VM workloads more efficiently on ROSA, ARO, or self-managed OpenShift. The demo is intentionally simple: it compares a fixed 1:1 CPU reservation model with a 10:1 CPU overcommit model.

| Demo side | Namespace | Node label | Template | CPU model |
| --- | --- | --- | --- | --- |
| No overcommit | `overcommit-not-enabled` | `demo-overcommit=one-to-one` | `fedora-vm-1to1` | 1 vCPU, 1 CPU request, 1 CPU limit |
| Overcommit enabled | `overcommit-enabled` | `overcommit-node=true` | `fedora-vm-10to1` | 1 vCPU, 100m CPU request, 1 CPU limit |

## Core message

OpenShift Virtualization can help customers reduce stranded capacity by letting them run VMs as Kubernetes-managed workloads. The customer still sees a normal VM with guest-facing vCPU, but the platform can schedule that VM based on a more realistic CPU reservation.

Use this simple framing:

> In a fixed VM model, every VM is sized and paid for as if it needs its full allocation all the time. Many enterprise VM estates do not behave that way. A large percentage of VMs are idle or bursty most of the day. OpenShift Virtualization lets us separate the guest-facing VM size from the CPU reservation used for scheduling. That can improve node utilization and defer additional infrastructure spend, while still giving operations teams placement, policy, observability, and automation controls.

## What this demo proves

This demo proves three things:

1. **Scheduler saturation happens before physical CPU saturation.** A node can stop accepting new workloads because requested CPU has reached allocatable CPU, even when actual CPU usage is low.
2. **Overcommit improves density for low-utilization VM workloads.** By lowering CPU requests while keeping a higher CPU limit, more VMs can be scheduled on the same worker capacity.
3. **OpenShift makes the tradeoff visible and manageable.** The platform shows requested CPU, allocatable CPU, actual CPU usage, VM placement, pending events, and resource pressure through native OpenShift and Prometheus views.

## What this demo does not claim

Do not position overcommit as free capacity.

Say this clearly:

> Overcommit does not create CPU. It changes how much CPU we reserve up front. If all VMs demand their full vCPU at the same time, they will contend for physical CPU. The value is that many VM workloads are not simultaneously busy, and OpenShift gives us a controlled way to use that reality instead of paying for stranded capacity everywhere.

## Why this matters for node saturation

There are two kinds of saturation to explain:

| Saturation type | What it means | What the demo shows |
| --- | --- | --- |
| Scheduling saturation | Requested CPU approaches node allocatable CPU | The 1:1 node fills up quickly and new VMs can remain pending |
| Runtime saturation | Actual CPU usage approaches physical CPU capacity | Optional stress test can show contention when many VMs are busy |

Customer-facing explanation:

> The first problem we are showing is scheduling saturation. The 1:1 node runs out of schedulable capacity because each VM reserves a full CPU. That can happen even if the VMs are mostly idle. On the overcommit node, each VM reserves less CPU, so the node can host more VMs before reaching the scheduler limit. This improves density and helps reduce stranded capacity.

## Why this matters for cost

Use this section to connect the technical demo to the business value.

> In a VM-per-instance cloud model, each workload is commonly mapped to a fixed cloud instance shape. If that instance is mostly idle, the unused CPU is stranded inside that instance. With OpenShift Virtualization, multiple VMs share a worker pool, and the platform schedules them based on requests. For low-utilization or bursty workloads, this can improve density on the same worker capacity and may reduce the number or size of worker nodes required for a given VM estate.

Keep the wording careful:

> This does not automatically guarantee lower cost for every workload. It gives the platform team another lever: measure real utilization, right-size requests, place VMs intentionally, and use shared OpenShift worker capacity more efficiently.

Strong customer line:

> The cost benefit is not just that one VM requests less CPU. The cost benefit is that OpenShift lets us convert unused capacity across many VMs into shared capacity that can be consumed by the platform.

## Why ROSA or ARO strengthens the story

When positioning this against running workloads directly on EC2 or cloud VMs, keep the message focused on platform value:

| Customer concern | EC2 or cloud VM-only approach | OpenShift Virtualization on ROSA or ARO |
| --- | --- | --- |
| VM density | Capacity is often stranded inside individual instance shapes | VMs share worker capacity through Kubernetes scheduling |
| Node saturation | Scaling decisions are usually VM-by-VM or instance-by-instance | Requested CPU vs allocatable CPU is visible at node and namespace level |
| Governance | Controls are spread across VM, account, and cloud policy layers | RBAC, namespaces, labels, templates, and policy are managed in one platform |
| Operations | VM operations are separate from container operations | VMs and containers use a common operational plane |
| Observability | Metrics are often gathered per VM or cloud instance | OpenShift shows node, pod, VM, and cluster-level metrics |
| Modernization | VM remains isolated from cloud-native workflows | VM can run next to containers, GitOps, pipelines, and services |

Talk track:

> ROSA and ARO let customers use OpenShift as the operating model while the cloud provider supplies the managed infrastructure foundation. OpenShift Virtualization then brings traditional VMs into that same platform. The customer is not just buying another place to run VMs. They are gaining a shared control plane for VM placement, density, policy, monitoring, and future modernization.

## Opening statement

Use this to start the demo:

> This demo shows why platform-managed virtualization can be more efficient than treating every VM as a fixed cloud instance. Both demo VMs expose the same guest-facing CPU: 1 vCPU. The difference is the reservation model. The 1:1 VM reserves 1 full CPU. The overcommit VM reserves 100 millicores and can burst up to 1 CPU. We will show how that changes node saturation, VM density, and the cost conversation.

## Step 1: Show the two placement zones

Show the node labels:

```bash
oc get nodes -l demo-overcommit=one-to-one
oc get nodes -l overcommit-node=true
```

Explain:

> These labels do not enable overcommit on the node. They only create two placement zones for the demo. The CPU reservation model is defined inside the VM template.

## Step 2: Show the templates

Show the templates:

```bash
oc get template -n overcommit-not-enabled
oc get template -n overcommit-enabled
```

Explain the 1:1 template:

```text
Namespace: overcommit-not-enabled
Template: fedora-vm-1to1
Guest vCPU: 1
CPU request: 1
CPU limit: 1
Node selector: demo-overcommit=one-to-one
```

Talk track:

> This VM exposes 1 vCPU and reserves 1 CPU. If I create ten of these VMs, I am asking OpenShift to reserve ten CPUs on the selected worker. That is predictable, but it can cause scheduler saturation quickly.

Explain the 10:1 template:

```text
Namespace: overcommit-enabled
Template: fedora-vm-10to1
Guest vCPU: 1
CPU request: 100m
CPU limit: 1
Node selector: overcommit-node=true
```

Talk track:

> This VM also exposes 1 vCPU, but it only requests 100 millicores for scheduling. If I create ten of these VMs, I am only asking OpenShift to reserve one CPU total, while each VM can still burst up to 1 CPU if capacity is available.

## Step 3: Create VMs on the 1:1 side

In the OpenShift console:

1. Go to **Virtualization**.
2. Select project `overcommit-not-enabled`.
3. Create VMs from template `fedora-vm-1to1`.
4. Repeat until some VMs remain pending, or until you have created the planned VM count.

Optional CLI equivalent:

```bash
for i in $(seq -w 1 10); do
  oc process fedora-vm-1to1 \
    -n overcommit-not-enabled \
    -p NAME=vm-1to1-$i \
  | oc apply -f -
done
```

Show the result:

```bash
oc get vm,vmi,pods -n overcommit-not-enabled -o wide
```

If a VM is pending, show why:

```bash
oc describe pod <pending-virt-launcher-pod> -n overcommit-not-enabled
```

Expected event:

```text
Insufficient cpu
```

Talk track:

> This is the fixed reservation model. The scheduler sees each VM as needing 1 CPU. Once requested CPU approaches allocatable CPU on the selected node, additional VMs do not schedule. This is node saturation from a scheduling perspective, even if the physical CPU is not currently busy.

## Step 4: Show the 1:1 Prometheus graph

Use `promql/requested-vs-allocatable-1to1.promql` in **Observe → Metrics**.

Talk track:

> This graph shows requested CPU compared with allocatable CPU for the 1:1 node. The requested CPU rises quickly because every VM reserves a full CPU. This model is conservative and predictable, but it can waste capacity when many VMs are idle most of the time.

## Step 5: Create VMs on the 10:1 side

In the OpenShift console:

1. Go to **Virtualization**.
2. Select project `overcommit-enabled`.
3. Create VMs from template `fedora-vm-10to1`.
4. Create the same number of VMs used on the 1:1 side.

Optional CLI equivalent:

```bash
for i in $(seq -w 1 10); do
  oc process fedora-vm-10to1 \
    -n overcommit-enabled \
    -p NAME=vm-10to1-$i \
  | oc apply -f -
done
```

Show the result:

```bash
oc get vm,vmi,pods -n overcommit-enabled -o wide
```

Talk track:

> These VMs expose the same guest-facing CPU, but each VM only requests 100 millicores. Ten VMs require only one CPU of scheduled capacity. This is how overcommit helps with density when the workloads are low-utilization or bursty.

## Step 6: Show the 10:1 Prometheus graph

Use `promql/requested-vs-allocatable-10to1.promql` in **Observe → Metrics**.

Talk track:

> This graph shows the overcommit side. We have the same type of node capacity, but requested CPU stays much lower because each VM reserves less CPU. The difference between the two graphs is the platform value: OpenShift lets us decide how much CPU to reserve based on workload behavior instead of assuming every VM needs full CPU all the time.

## Step 7: Tie it directly to cost optimization

Use this after both graphs are visible:

> This is where the cost discussion starts. The 1:1 model consumes schedulable capacity quickly, which can force the customer to add more worker nodes or larger cloud instances even when actual CPU usage is low. The overcommit model lets the customer place more low-utilization VMs on the same worker footprint. That can defer scale-out, improve worker utilization, and reduce stranded CPU capacity.

Then add the qualification:

> We still need to size responsibly. Overcommit should be based on measured utilization, workload criticality, and performance requirements. But this gives the platform team a way to turn utilization data into lower infrastructure demand.

## Step 8: Explain the operational benefits beyond cost

Use this section to avoid making the conversation only about CPU math.

> Cost is one benefit, but the larger value is operational control. These VMs are now OpenShift workloads. We can use templates, GitOps, RBAC, namespaces, labels, taints, metrics, alerts, and policy to manage them consistently. That means the customer gets VM compatibility for existing workloads while moving toward a unified platform operating model.

Key points to highlight:

| Benefit | How to explain it |
| --- | --- |
| Better density | More low-utilization VMs can fit before requested CPU reaches allocatable CPU |
| Better saturation visibility | Prometheus shows requested CPU, allocatable CPU, and actual usage separately |
| Better placement control | Labels and node selectors put workloads on the right worker capacity |
| Better governance | Namespaces, templates, RBAC, quotas, and policy can standardize consumption |
| Better modernization path | VMs can coexist with containers, pipelines, GitOps, and services |
| Better operational consistency | Teams can manage VMs and containers through the OpenShift control plane |

## Step 9: Explain the tradeoff

Use this statement:

> Overcommit is a tradeoff, not a shortcut. If all VMs demand their full CPU at the same time, they will contend and performance can degrade. That is why the platform matters. We can monitor actual usage, identify noisy neighbors, adjust requests, separate critical workloads, and choose a more conservative ratio where needed.

Optional validation of actual CPU usage:

```bash
oc adm top pods -n overcommit-enabled
oc adm top pods -n overcommit-not-enabled
```

Use `promql/actual-cpu-usage.promql` to show runtime CPU consumption after generating load inside the VMs.

## Customer question: Are you just requesting less CPU?

Expected customer question:

> Are you just requesting less CPU on the overcommit side?

Answer:

> Yes. That is the mechanism. Both VMs expose 1 vCPU to the guest. The 1:1 VM reserves the full CPU. The 10:1 VM reserves 100 millicores and can burst up to 1 CPU. This works when the customer has workloads that are normally idle or bursty. We are trading unused dedicated reservation for shared platform capacity.

## Customer question: Is the node configured differently?

Expected customer question:

> Did you turn on CPU overcommit on the node?

Answer:

> No. CPU overcommit is not a per-node switch in this demo. The node labels only control placement. The CPU reservation model is defined in the VM template through CPU requests and limits.

## Customer question: Why is this better than EC2?

Expected customer question:

> Why not just put these workloads on EC2 or cloud VMs?

Answer:

> EC2 and cloud VMs are excellent for many workloads, but each VM is commonly tied to a fixed instance shape. If many workloads are underutilized, CPU can be stranded across many instances. OpenShift Virtualization lets VMs share worker capacity, so the platform can schedule based on realistic reservations and improve density. The customer also gets OpenShift placement, policy, observability, GitOps integration, and a modernization path for VM and container workloads on the same platform.

## Customer question: Is 10:1 always safe?

Expected customer question:

> Should we run everything at 10:1?

Answer:

> No. The ratio should be based on utilization data and workload criticality. 10:1 is useful for a demo because it makes the behavior obvious. In production, we would classify workloads, monitor actual CPU demand, start with conservative ratios, and adjust over time. Latency-sensitive or consistently busy workloads might need lower overcommit, dedicated nodes, CPU Manager, or a 1:1 model.

## Closing statement

Use this to close:

> The takeaway is not that every VM should be overcommitted at 10:1. The takeaway is that OpenShift Virtualization gives the customer control over the reservation model. A fixed VM model can saturate infrastructure from a scheduling and cost perspective even when workloads are idle. A platform-managed model lets the customer improve density, reduce stranded capacity, monitor the tradeoff, and standardize VM operations alongside containers. That is the value of running VMs on OpenShift on ROSA, ARO, or self-managed OpenShift.

## Supporting references

- Red Hat OpenShift Container Platform documentation explains that requests help OpenShift schedule pods onto nodes with sufficient resources, while limits define the maximum resources a container can consume.
- Red Hat OpenShift Container Platform overcommit documentation explains that scheduling is based on requested resources and that the difference between request and limit determines the level of overcommit.
- Red Hat OpenShift Virtualization monitoring documentation explains that the OpenShift metrics query browser can run PromQL queries and visualize metrics on a plot.
- AWS EC2 On-Demand pricing documentation explains that customers pay for compute capacity by the hour or second, depending on the instance and operating system.
