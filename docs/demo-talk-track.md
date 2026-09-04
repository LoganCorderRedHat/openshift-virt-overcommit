# Demo Talk Track

Use this talk track to demonstrate CPU overcommit behavior with OpenShift Virtualization. The demo compares two VM placement zones:

| Demo side | Namespace | Node label | Template | CPU model |
| --- | --- | --- | --- | --- |
| No overcommit | `overcommit-not-enabled` | `demo-overcommit=one-to-one` | `fedora-vm-1to1` | 1 vCPU, 1 CPU request, 1 CPU limit |
| Overcommit enabled | `overcommit-enabled` | `overcommit-node=true` | `fedora-vm-10to1` | 1 vCPU, 100m CPU request, 1 CPU limit |

## Demo objective

Show that OpenShift Virtualization can place more VM workloads on the same worker capacity when the VMs are sized with lower CPU requests and higher CPU limits.

The message is not that overcommit creates free capacity. The message is that OpenShift lets platform teams separate the guest-facing VM shape from the CPU reservation used by the scheduler.

## Opening statement

Use this to start the demo:

> This demo compares a fixed CPU reservation model with a CPU overcommit model. Both VMs expose 1 vCPU to the guest. The difference is how much CPU OpenShift reserves when scheduling the VM. In the 1:1 model, each VM reserves the full CPU it exposes. In the 10:1 model, each VM exposes the same 1 vCPU but reserves only 100 millicores. That lets us improve density for workloads that are normally idle or bursty, while still making the tradeoff visible when workloads contend for CPU.

## Step 1: Show the two placement zones

Show the node labels first:

```bash
oc get nodes -l demo-overcommit=one-to-one
oc get nodes -l overcommit-node=true
```

Explain:

> These labels do not set CPU overcommit on the node. They only control where the VMs land. The CPU ratio comes from the VM template itself.

## Step 2: Show the templates

Show the two templates:

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

> This VM exposes 1 vCPU and reserves 1 CPU. If I create ten of these VMs, I am asking OpenShift to reserve ten CPUs on the selected worker.

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

Show the running and pending VMIs:

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

> This is the fixed reservation model. The VM says it needs 1 CPU reserved, so the scheduler only places as many VMs as the node can actually reserve. Once requested CPU reaches allocatable CPU, additional VMs do not schedule.

## Step 4: Show the 1:1 Prometheus graph

Use `promql/requested-vs-allocatable-1to1.promql` in **Observe → Metrics**.

Talk track:

> This graph shows requested CPU compared with allocatable CPU for the 1:1 node. The requested CPU rises quickly because every VM reserves a full CPU. This is predictable, but it can strand capacity when the guest workloads are not actually using their full vCPU allocation.

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

Show the VMIs:

```bash
oc get vm,vmi,pods -n overcommit-enabled -o wide
```

Talk track:

> These VMs expose the same guest-facing CPU, but each VM only requests 100 millicores. Ten VMs only require one CPU of scheduled capacity. This is why more VMs can be placed on the same worker when the workloads are low-utilization or bursty.

## Step 6: Show the 10:1 Prometheus graph

Use `promql/requested-vs-allocatable-10to1.promql` in **Observe → Metrics**.

Talk track:

> This graph shows the overcommit side. The allocatable CPU is the same type of node capacity, but requested CPU stays much lower because each VM only reserves 100 millicores. This is the density benefit.

## Step 7: Explain the tradeoff

Use this statement:

> Overcommit is not free capacity. It is a controlled oversubscription model. If all of these VMs demand their full vCPU at the same time, they will contend for physical CPU and performance can degrade. The value of OpenShift Virtualization is that the platform gives us scheduling, placement, quotas, policy, monitoring, and automation to manage that tradeoff.

Optional validation of actual CPU usage:

```bash
oc adm top pods -n overcommit-enabled
oc adm top pods -n overcommit-not-enabled
```

Use `promql/actual-cpu-usage.promql` to show runtime CPU consumption after generating workload inside the VMs.

## Customer question: Are you just requesting less CPU?

Expected customer question:

> Are you just requesting less CPU on the overcommit side?

Answer:

> Yes. That is exactly how CPU overcommit works in this model. Both VMs expose 1 vCPU to the guest. The 1:1 VM reserves the full CPU, while the 10:1 VM reserves 100 millicores and can burst up to 1 CPU. The difference between request and limit is what creates the overcommit ratio. This is useful when VM workloads are normally idle or bursty, and it needs to be managed with monitoring and policy.

## Customer question: Is the node configured differently?

Expected customer question:

> Did you turn on CPU overcommit on the node?

Answer:

> No. CPU overcommit is not a per-node switch in OpenShift. The node labels only control placement. The CPU reservation model is defined in the VM template through CPU requests and limits.

## Closing statement

Use this to close:

> In a cloud VM-only model, each workload is commonly tied to a fixed instance shape, and unused guest capacity can become stranded. With OpenShift Virtualization, VMs become Kubernetes-managed workloads. We can expose the guest-facing vCPU that the VM needs, while reserving CPU based on expected utilization. That allows better density for low-utilization VM estates, with the operational guardrails needed to manage contention and performance.
