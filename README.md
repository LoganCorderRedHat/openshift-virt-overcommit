# OpenShift Virtualization CPU Overcommit Demo

This repository provides a simple OpenShift Virtualization demo that compares two VM CPU reservation models:

- **1:1 CPU reservation**, each VM exposes 1 vCPU and requests 1 CPU.
- **10:1 CPU overcommit**, each VM exposes 1 vCPU, requests 100m CPU, and can burst up to 1 CPU.

The demo shows how OpenShift Virtualization can improve VM density for low-utilization or bursty VM workloads by separating **guest-facing vCPU** from the **CPU requested for scheduling**.

This is a demo, not a production sizing recommendation. In production, request and limit values should be based on workload behavior, performance requirements, SLOs, and monitoring data.

## What this demonstrates

OpenShift schedules workloads based on **requested CPU**. A VM can expose a vCPU to the guest while requesting less CPU from the scheduler. That difference is the overcommit ratio.

| Model | Guest sees | CPU request | CPU limit | Demo ratio |
| --- | ---: | ---: | ---: | ---: |
| 1:1 | 1 vCPU | 1 CPU | 1 CPU | 1:1 |
| 10:1 | 1 vCPU | 100m | 1 CPU | 10:1 |

With 10 VMs:

```text
1:1 side:
  10 VMs x 1 CPU request = 10 CPU requested

10:1 side:
  10 VMs x 100m CPU request = 1 CPU requested
```

If the 1:1 worker has less than 10 allocatable CPUs, the 1:1 side should fail to schedule all 10 VMs with an `Insufficient cpu` event. The 10:1 side should schedule all 10 VMs, assuming memory and other resources are not the bottleneck.

## Repository layout

```text
ocp-virt-overcommit-demo/
├── README.md
├── deploy/
│   └── overcommit-demo-templates.yaml
├── promql/
│   ├── requested-vs-allocatable-1to1.promql
│   ├── requested-vs-allocatable-10to1.promql
│   ├── requested-percent-1to1.promql
│   ├── requested-percent-10to1.promql
│   ├── actual-cpu-usage.promql
│   └── validate-node-labels.promql
└── docs/
    ├── demo-talk-track.md
    └── troubleshooting.md
```

## What gets created

Applying the manifest creates:

- Namespace: `overcommit-not-enabled`
- Namespace: `overcommit-enabled`
- Template: `overcommit-not-enabled/fedora-vm-1to1`
- Template: `overcommit-enabled/fedora-vm-10to1`

It does **not** automatically create VMs. Presenters create VMs manually from the templates during the demo so the audience can watch the scheduling behavior.

Both templates use a Fedora `containerDisk` image for a lightweight demo experience.

## Prerequisites

- This is intented to run on Experiece OpenShift Virtualiztion Roadshow (2026) on demo.redhat.com
- An OpenShift cluster with OpenShift Virtualization installed.
- Two worker nodes available for the demo.
- Cluster-admin, or equivalent permissions to create namespaces, templates, and VMs.
- Access to the OpenShift web console.
- Optional: `oc` and `virtctl` for CLI-driven validation.

## Step 1: Label the demo nodes

Pick one worker for the 1:1 VM placement and one worker for the 10:1 VM placement.

```bash
oc label node <one-to-one-worker> demo-overcommit=one-to-one
oc label node <overcommit-worker> overcommit-node=true
```

Optional, but recommended for a clean demo, taint the two workers so only demo VMs with matching tolerations land there:

```bash
oc adm taint node <one-to-one-worker> demo-overcommit=one-to-one:NoSchedule
oc adm taint node <overcommit-worker> overcommit-node=true:NoSchedule
```

Validate the labels:

```bash
oc get nodes -l demo-overcommit=one-to-one
oc get nodes -l overcommit-node=true
```

## Step 2: Deploy the namespaces and templates

Use plain `oc apply`:

```bash
oc apply -f deploy/overcommit-demo-templates.yaml
```

Or use Kustomize:

```bash
oc apply -k .
```

Validate that the templates exist:

```bash
oc get template -n overcommit-not-enabled
oc get template -n overcommit-enabled
```

Expected templates:

```text
overcommit-not-enabled/fedora-vm-1to1
overcommit-enabled/fedora-vm-10to1
```

## Step 3: Review the template behavior

### 1:1 template

Namespace: `overcommit-not-enabled`

```text
Template: fedora-vm-1to1
Guest vCPU: 2
CPU request: 2
CPU limit: 2
Node selector: demo-overcommit=one-to-one
Run strategy: Always
```

Meaning:

```text
The guest sees 2 vCPU.
OpenShift reserves 2 CPU.
The VM can use up to 2 CPU.
This is 1:1 CPU reservation.
```

### 10:1 template

Namespace: `overcommit-enabled`

```text
Template: fedora-vm-10to1
Guest vCPU: 2
CPU request: 200m
CPU limit: 2
Node selector: overcommit-node=true
Run strategy: Always
```

Meaning:

```text
The guest sees 2 vCPU.
OpenShift reserves 200m CPU.
The VM can use up to 2 CPU.
This is 10:1 CPU overcommit.
```

Both templates use `runStrategy: Always`, so each VM starts immediately after it is created from the template.

## Step 4: Create VMs from the OpenShift console

Use this path for a customer-facing demo.

### Create the 1:1 VMs

1. In the OpenShift web console, go to **Virtualization**.
2. Open the `overcommit-not-enabled` project.
3. Create a VM from the `fedora-vm-1to1` template.
4. Repeat until some VMs can no longer schedule.
5. Show the pending `virt-launcher` pod and the `Insufficient cpu` event.

### Create the 10:1 VMs

1. Open the `overcommit-enabled` project.
2. Create a VM from the `fedora-vm-10to1` template.
3. Repeat until you have created the same number of VMs as the 1:1 test.
4. Show that more VMs can schedule because each VM requests only 100m CPU.

## Step 5: Create VMs from the CLI, optional

Create 10 VMs from the 1:1 template:

```bash
oc project overcommit-not-enabled

for i in $(seq -w 1 20); do
  oc process fedora-vm-1to1 \
    -n overcommit-not-enabled \
    -p NAME=vm-1to1-$i \
  | oc apply -f -
done
```

Create 10 VMs from the 10:1 template:

```bash
oc project overcommit-enabled

for i in $(seq -w 1 20); do
  oc process fedora-vm-10to1 \
    -n overcommit-enabled \
    -p NAME=vm-10to1-$i \
  | oc apply -f -
done
```

Because the templates use `runStrategy: Always`, the VMs start as soon as they are created.

## Step 6: Validate scheduling results

Check the VMIs:

```bash
oc get vmi -n overcommit-not-enabled -o wide
oc get vmi -n overcommit-enabled -o wide
```

Check the pods:

```bash
oc get pods -n overcommit-not-enabled -o wide
oc get pods -n overcommit-enabled -o wide
```

Describe a pending `virt-launcher` pod on the 1:1 side:

```bash
oc describe pod <pending-virt-launcher-pod> -n overcommit-not-enabled
```

Look for an event similar to:

```text
Insufficient cpu
```

Show node allocation:

```bash
oc describe node <one-to-one-worker>
oc describe node <overcommit-worker>
```

In the `Allocated resources` section, compare CPU requests and limits.

## Step 7: Show the Prometheus graphs

Go to **Observe > Metrics** in the OpenShift console.

### 1:1, requested CPU vs allocatable CPU

```promql
label_replace(
  sum by (node) (
    kube_pod_container_resource_requests{resource="cpu",unit="core",namespace="overcommit-not-enabled"}
    * on (namespace, pod) group_left(node)
    kube_pod_info{namespace="overcommit-not-enabled"}
    * on (node) group_left(label_demo_overcommit)
    kube_node_labels{label_demo_overcommit="one-to-one"}
  ),
  "metric",
  "CPU requested",
  "node",
  ".*"
)
or
label_replace(
  sum by (node) (
    kube_node_status_allocatable{resource="cpu",unit="core"}
    * on (node) group_left(label_demo_overcommit)
    kube_node_labels{label_demo_overcommit="one-to-one"}
  ),
  "metric",
  "CPU allocatable",
  "node",
  ".*"
)
```

Expected result: requested CPU rises quickly toward allocatable CPU as 1:1 VMs are created.

### 10:1, requested CPU vs allocatable CPU

```promql
label_replace(
  sum by (node) (
    kube_pod_container_resource_requests{resource="cpu",unit="core",namespace="overcommit-enabled"}
    * on (namespace, pod) group_left(node)
    kube_pod_info{namespace="overcommit-enabled"}
    * on (node) group_left(label_overcommit_node)
    kube_node_labels{label_overcommit_node="true"}
  ),
  "metric",
  "CPU requested",
  "node",
  ".*"
)
or
label_replace(
  sum by (node) (
    kube_node_status_allocatable{resource="cpu",unit="core"}
    * on (node) group_left(label_overcommit_node)
    kube_node_labels{label_overcommit_node="true"}
  ),
  "metric",
  "CPU allocatable",
  "node",
  ".*"
)
```

Expected result: requested CPU stays much lower, even with the same number of VMs.

## Step 8: Explain what the customer is seeing

Use this explanation:

> Both VM types expose the same guest-facing CPU: 1 vCPU. The difference is what OpenShift reserves for scheduling. In the 1:1 model, each VM reserves 1 full CPU. In the 10:1 model, each VM exposes 1 vCPU but reserves only 100m CPU. That allows more low-utilization VMs to fit on the same worker capacity. This is not free CPU. If every VM demands a full CPU at the same time, they will contend. The value of OpenShift Virtualization is that this tradeoff is managed through placement, resource policy, observability, and automation.

## Optional: Show actual CPU usage

Requested CPU shows what the scheduler reserves. Actual CPU usage shows what the VM workload consumes at runtime.

Run this query after generating CPU load inside the VMs:

```promql
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{namespace=~"overcommit-not-enabled|overcommit-enabled",pod=~"virt-launcher-.*",container="compute"}[1m])
)
```

This is useful for showing that overcommit is safe for idle or bursty workloads, but can cause contention when many VMs demand CPU at the same time.

## Adjusting the demo

If your workers have 10 or more allocatable CPUs, 10 one-vCPU VMs may all schedule on the 1:1 side. In that case, either create more than 10 VMs or increase the CPU request and limit in both templates.

For example, a 3 vCPU version would use:

```text
1:1 VM:
  guest vCPU: 3
  request: 3 CPU
  limit: 3 CPU

10:1 VM:
  guest vCPU: 3
  request: 300m
  limit: 3 CPU
```

## Cleanup

Delete the demo namespaces:

```bash
oc delete namespace overcommit-not-enabled
oc delete namespace overcommit-enabled
```

Remove the demo labels:

```bash
oc label node <one-to-one-worker> demo-overcommit-
oc label node <overcommit-worker> overcommit-node-
```

Remove the optional taints:

```bash
oc adm taint node <one-to-one-worker> demo-overcommit=one-to-one:NoSchedule-
oc adm taint node <overcommit-worker> overcommit-node=true:NoSchedule-
```

## Troubleshooting

### VMs do not start

Check the VMI and launcher pod:

```bash
oc get vmi -n overcommit-not-enabled
oc get pods -n overcommit-not-enabled
oc describe pod <virt-launcher-pod> -n overcommit-not-enabled
```

### VMs are not landing on the expected nodes

Verify labels:

```bash
oc get nodes -l demo-overcommit=one-to-one
oc get nodes -l overcommit-node=true
```

Verify the VM launcher pod node:

```bash
oc get pods -n overcommit-not-enabled -o wide
oc get pods -n overcommit-enabled -o wide
```

### The 1:1 side schedules all 10 VMs

The node likely has enough allocatable CPU. Create more VMs or increase the CPU request and limit values in the templates.

### The 10:1 side does not schedule all 10 VMs

Check for non-CPU constraints:

```bash
oc describe pod <pending-virt-launcher-pod> -n overcommit-enabled
```

Common causes include memory pressure, image pull issues, taint/toleration mismatch, node selector mismatch, or insufficient device resources.

## References

- [Red Hat OpenShift Virtualization documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/about)
- [Creating a virtual machine from a template](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/creating-a-virtual-machine)
- [Accessing metrics in the OpenShift web console](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.18/html/accessing_metrics/accessing-metrics-as-an-administrator)
