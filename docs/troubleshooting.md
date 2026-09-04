# Troubleshooting

This guide matches the current demo layout:

| Demo side | Namespace | Node label | Template |
| --- | --- | --- | --- |
| No overcommit | `overcommit-not-enabled` | `demo-overcommit=one-to-one` | `fedora-vm-1to1` |
| Overcommit enabled | `overcommit-enabled` | `overcommit-node=true` | `fedora-vm-10to1` |

## Validate node labels

```bash
oc get nodes -l demo-overcommit=one-to-one
oc get nodes -l overcommit-node=true
```

You should see one worker for each label.

If Prometheus is being used, validate that the labels are visible as normalized metric labels:

```promql
kube_node_labels{label_demo_overcommit="one-to-one"} or kube_node_labels{label_overcommit_node="true"}
```

Kubernetes node labels are normalized in Prometheus:

```text
demo-overcommit=one-to-one becomes label_demo_overcommit="one-to-one"
overcommit-node=true becomes label_overcommit_node="true"
```

## Validate templates

```bash
oc get template fedora-vm-1to1 -n overcommit-not-enabled
oc get template fedora-vm-10to1 -n overcommit-enabled
```

If a template is missing, reapply the demo resources:

```bash
oc apply -f deploy/overcommit-demo-templates.yaml
```

Or:

```bash
oc apply -k .
```

## Validate VM resource settings

Check the VM template output before creating a VM:

```bash
oc process fedora-vm-1to1 -n overcommit-not-enabled -p NAME=test-1to1 | grep -A8 'resources:'
```

```bash
oc process fedora-vm-10to1 -n overcommit-enabled -p NAME=test-10to1 | grep -A8 'resources:'
```

Expected 1:1 values:

```yaml
requests:
  cpu: "1"
  memory: "1Gi"
limits:
  cpu: "1"
  memory: "1Gi"
```

Expected 10:1 values:

```yaml
requests:
  cpu: "100m"
  memory: "1Gi"
limits:
  cpu: "1"
  memory: "1Gi"
```

## VMs do not start immediately

The templates use:

```yaml
runStrategy: Always
```

A VM created from the template should start immediately. If it does not, check the VM, VMI, and virt-launcher pod:

```bash
oc get vm,vmi,pods -n overcommit-not-enabled
oc get vm,vmi,pods -n overcommit-enabled
```

Describe the VM or pod to see events:

```bash
oc describe vm <vm-name> -n <namespace>
oc describe pod <virt-launcher-pod> -n <namespace>
```

## 1:1 VMs do not all schedule

This is expected once requested CPU exceeds node allocatable CPU.

Check pending pods:

```bash
oc get pods -n overcommit-not-enabled
oc describe pod <pending-virt-launcher-pod> -n overcommit-not-enabled
```

Expected event:

```text
Insufficient cpu
```

This is the expected demo outcome. Each 1:1 VM requests 1 CPU.

## 10:1 VMs do not all schedule

The overcommit side should schedule more VMs, but it can still fail for reasons other than CPU request pressure.

Check pod events:

```bash
oc get pods -n overcommit-enabled
oc describe pod <pending-virt-launcher-pod> -n overcommit-enabled
```

Common causes:

```text
Node label is missing or incorrect
Node taint exists but the VM toleration does not match
Memory is exhausted
Image pull is failing
OpenShift Virtualization is not healthy
The selected worker is not schedulable
```

Check memory and CPU allocation:

```bash
oc describe node <overcommit-worker> | sed -n '/Allocated resources:/,/Events:/p'
```

## PromQL graph only shows one side

Confirm the namespaces match the current layout:

```text
overcommit-not-enabled
overcommit-enabled
```

Confirm the VM launcher pods exist:

```promql
kube_pod_info{namespace=~"overcommit-not-enabled|overcommit-enabled",pod=~"virt-launcher-.*"}
```

Confirm both node labels exist:

```promql
kube_node_labels{label_demo_overcommit="one-to-one"} or kube_node_labels{label_overcommit_node="true"}
```

Use the query files in the `promql/` directory:

```text
promql/requested-vs-allocatable-1to1.promql
promql/requested-vs-allocatable-10to1.promql
promql/requested-percent-1to1.promql
promql/requested-percent-10to1.promql
```

## CPU request is visible, but actual CPU usage is low

That is normal. CPU request is the reservation used by the scheduler. It does not mean the VM is actively consuming that much CPU.

Use requested CPU for the placement and density demo. Use actual CPU usage only when showing runtime contention or workload behavior.

Requested CPU query files:

```text
promql/requested-vs-allocatable-1to1.promql
promql/requested-vs-allocatable-10to1.promql
```

Actual CPU usage query file:

```text
promql/actual-cpu-usage.promql
```
