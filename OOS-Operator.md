# OOS Operator
An operator that watches Nodes in a K8s cluster, and applies the `node.kubernetes.io/out-of-service` taint once a node has been offline for a specific amount of time or not ready for a specific amount of time.


> [!NOTE]
> Fencing will not be in scope and should be handled via the CSI driver or other OOB management measures.