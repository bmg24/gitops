# AWS audit: existing "brian" resources (read-only, 2026-11-05)

Account 446130280781, us-east-1, role dtRoleAdvancedUser. Shared SE account.
All rows confirmed as Brian's by Brian in chat. Nothing was modified.

## VPC `brian-test-vpc` (vpc-056c104d40850a84b, 10.0.0.0/16)
| Check | Result | Note |
|---|---|---|
| CIDR size | PASS | /16 |
| 2+ AZs | PASS | us-east-1a, us-east-1b (2 public /20, 2 private /20) |
| Internet gateway | PASS | igw-01e4710d93c022ea6, public route tables point to it |
| NAT gateway | FAIL | nat-036eeed327b172699 no longer exists; both private route tables have a blackhole 0.0.0.0/0 route. Private subnets have no outbound internet |
| DNS support + hostnames | PASS | both enabled |
| Subnet tags for ELB | FAIL | only Name tags; no kubernetes.io/role/elb or internal-elb |
| Shared with others | WARN | ENIs/instances in the VPC belong to other clusters: apple-pay-easytrade, adorable-classical-rainbow, robert-cluster2, lagonaSandbox, plus ELBs. The VPC is not exclusively yours |
| Security groups | WARN | brian-bastion-host-security-group (sg-0bd13a035ded2d722) allows 22 from 0.0.0.0/0; several other people's k8s-elb SGs open to the world. Only the bastion SG looks like yours |
Verdict: **fix or avoid**. Reusing it means adding NAT (or using public subnets only), adding tags, and sharing it with other people's clusters. Changing it affects them.

## IAM roles
| Role | Verdict | Reason |
|---|---|---|
| BrianEKSClusterRole | reuse | trusted by eks.amazonaws.com; has AmazonEKSClusterPolicy (plus 4 Auto Mode policies; harmless). Used by goins-cat-poc, last used today |
| Brian-EKS-NodeGroup (+ instance profile of same name) | reuse | trusted by ec2.amazonaws.com; WorkerNode, CNI, ECR ReadOnly. Clean. Last used today |
| Brian-AmazonEKSPodIdentityAmazonEBSCSIDriverRole | fix | trust is pods.eks.amazonaws.com (Pod Identity, valid), but it also has AmazonEC2FullAccess, ECR FullAccess and ECS service role. Far broader than the EBS CSI driver needs; only AmazonEBSCSIDriverPolicy is required |
| eksctl-goins-cat-poc-addon-iamserviceaccount--Role1-... | retire | trusts OIDC id BC19346D..., but goins-cat-poc's OIDC is 46F038EC.... Stale, last used 2026-01-13. Stack of same name is its CloudFormation owner; delete only via that stack |
| JPMCPOC_ROLE_BRIANG | keep (ignore) | 2022, unrelated to EKS; not touched |

## EKS cluster `goins-cat-poc`
- Version 1.34, ACTIVE, created 2026-01-13, role BrianEKSClusterRole, public+private endpoint, public CIDR 0.0.0.0/0.
- Control-plane logging off. Add-ons: ebs-csi, coredns, kube-proxy, vpc-cni, pod-identity-agent, metrics-server, external-dns, node-monitoring-agent, network-flow-monitoring-agent.
- No managed node groups listed. 4 EC2 instances are running in the VPC (owner not yet verified, may be Auto Mode nodes or other clusters).
- Costs money (control plane fee). Verdict: **your call: keep or retire**. Not in the playground contract.

## Other
- CloudFormation stack eksctl-goins-cat-poc-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa: retire with the stale role above.
- Key pairs brian-se-fun, BrianJPMC, brian-access-logs-test: keep; not needed (SSM/no SSH).
- Leftover SG eks-cluster-sg-brian-goins-easytrade-1637584940 (no inbound rules): its cluster no longer exists in the cluster list. Candidate to retire.

## Recommendation (0.6)
Option (b) reuse roles, fresh VPC via eksctl: reuse BrianEKSClusterRole and Brian-EKS-NodeGroup, let eksctl create a dedicated VPC (2 AZs, NAT). Avoids breaking other people's clusters in brian-test-vpc. Option (c) everything fresh is the cleanest alternative; option (a) not recommended (no NAT, shared).
Note: eksctl creates its own roles by default; reusing a cluster role is supported via iam.serviceRoleARN and nodegroup iam.instanceRoleARN (to verify in docs at config time).
