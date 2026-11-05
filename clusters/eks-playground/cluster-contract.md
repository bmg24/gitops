# EKS playground: cluster contract

What any cluster must provide for this repo to deploy cleanly. No creation commands here:
how the current cluster was built is in [`BUILD_LOG.md`](BUILD_LOG.md), and connecting
plus ArgoCD is in [`bootstrap.md`](bootstrap.md).

## Current cluster (2026-11-05)

| Item | Value |
|---|---|
| AWS cluster name | `brian-eks-playground` (Dynatrace-facing name `eks-playground`) |
| Account / region | `446130280781` / `us-east-1` (shared Dynatrace SE account) |
| Kubernetes | **1.37** (standard support) |
| Compute | EKS **managed node group** `ng-main`, on-demand **m6i.xlarge** (4 vCPU / 16 GiB, ~14.4 GiB allocatable), **2 nodes, min 0 / max 3**, 50 GiB gp3 root, Amazon Linux 2023 |
| Network | Public subnets of the shared VPC `brian-test-vpc` (us-east-1a, us-east-1b); nodes have public IPs, no NAT |
| Endpoint | Public and private |
| Auth | EKS API and ConfigMap; the creator has cluster-admin |
| Add-ons | vpc-cni, coredns, kube-proxy, eks-pod-identity-agent, aws-ebs-csi-driver (Pod Identity), plus metrics-server, eks-node-monitoring-agent, external-dns (not needed) |
| Storage | Default StorageClass `gp3` from [`storageclass-gp3.yaml`](storageclass-gp3.yaml) |
| ArgoCD | v3.6.1 |

## Requirements

| Requirement | Why |
|---|---|
| **EC2 nodes; no EKS Auto Mode, no Fargate** | OneAgent full-stack (cloudNativeFullStack) needs real hosts it can instrument. |
| **A node with well over 10 GiB free memory** | The ActiveGate requests 10Gi (`dynatrace/dynakube.yaml`). That value is our choice, not a Dynatrace minimum; Dynatrace samples use 1.5 to 6 Gi. Lower it before shrinking nodes. |
| **Kubernetes version supported by Dynatrace and ArgoCD** | Dynatrace lists K8s 1.37 with Operator 1.6.x minimum, 1.9.0+ recommended (repo pins 1.11.1), OneAgent/ActiveGate 1.335+ (repo pins 1.339). ArgoCD 3.6 is tested on 1.34 to 1.37. |
| **EBS CSI driver with working IAM** | Needed by any PVC. Uses Pod Identity (`eks-pod-identity-agent` add-on plus an association for `kube-system/ebs-csi-controller-sa`). |
| **Default gp3 StorageClass** | The AWS org policy (`DenyCreationOfGP2Volumes`) blocks gp2. EKS sets no default class here. |
| **Outbound internet from nodes** | Images come from public registries (public ECR, ghcr, Docker Hub); the DynaKube and Bindplane gateway call the Dynatrace tenant and Bindplane Cloud. |
| **`LoadBalancer` Services** | easytrade's `frontendreverseproxy` gets an external address from the default AWS cloud provider. Each one is a billable AWS load balancer. If one can't find subnets, tag the public subnets `kubernetes.io/role/elb=1` (shared VPC: coordinate first). |
| **cluster-admin kubeconfig** | ArgoCD installs CRDs and cluster-scoped resources. |

## Which workloads use storage

None of the apps that deploy by default (`apps/`) request a PVC today: easytrade's database
persistence is disabled. Check the Astroshop chart's defaults before enabling it from
`apps-disabled/`.

## Cost (approximate, us-east-1 on-demand)

| Piece | Per month |
|---|---|
| Control plane | ~$73 |
| m6i.xlarge node | ~$140 each |
| Node root volumes (50 GiB gp3) | ~$4 each |
| Each `LoadBalancer` Service | ~$16 to $20 plus data |

Scale the node group to 0 between demos to pay only for the control plane:
`aws eks update-nodegroup-config --cluster-name brian-eks-playground --nodegroup-name ng-main --scaling-config minSize=0,maxSize=3,desiredSize=0 --region us-east-1`.
PVC volumes keep billing while nodes are at 0.
