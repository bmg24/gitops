# BUILD_LOG: brian-eks-playground (no secrets)

## 2026-11-05
- `aws sts get-caller-identity`, `aws configure list`: account 446130280781, role dtRoleAdvancedUser, us-east-1. Confirmed by Brian.
- Read-only audit of brian/goins resources -> AUDIT.md. Decision: everything fresh (new VPC and roles via eksctl).
- `aws iam simulate-principal-policy` for eks/iam/ec2/cloudformation create actions: all allowed. servicequotas:GetServiceQuota denied (cannot read VPC/EIP quotas; account has 30 VPCs and 47 EIPs already).
- K8s version 1.35 chosen: listed in AWS docs as standard support; Dynatrace support table lists up to 1.35 (1.36 not confirmed). eksctl 0.215.0 accepts it.
- VPC CIDR 10.88.0.0/16 chosen (10.20.0.0/16 overlapped an existing VPC).
- `eksctl create cluster -f eksctl-cluster.yaml --dry-run`: valid.
- Sizing revised with Brian: 2 x m6i.xlarge (min 0, max 3), public subnets, no NAT, to cut cost (~$360/mo always-on; scale to 0 when idle). AG memory (10Gi in dynakube.yaml) to be lowered later in Phase 3; not a Dynatrace requirement.
- `eksctl create cluster -f eksctl-cluster.yaml --dry-run`: valid with revised config.
- `eksctl create cluster -f eksctl-cluster.yaml` -> FAILED at VPC creation: "The maximum number of VPCs has been reached" (account is at its 30-VPC limit). CloudFormation stack eksctl-brian-eks-playground-cluster rolled back; no cluster created. Stack cleanup and next step pending Brian's decision.
- Failed stack eksctl-brian-eks-playground-cluster is ROLLBACK_COMPLETE (needs deleting before retry; awaiting Brian's OK).
- Decision: reuse brian-test-vpc public subnets (subnet-04a6e5a3a182ecd83 / us-east-1a, subnet-089e674d4e07c9bde / us-east-1b). VPC is shared with 4 other people's clusters; not deleting or modifying it. Config updated; dry-run re-checked.
- Deleted failed stack (ROLLBACK_COMPLETE) with `aws cloudformation delete-stack`: ok.
- `eksctl create cluster -f eksctl-cluster.yaml` (reuse brian-test-vpc) -> FAILED: eks:CreateCluster explicit deny from an AWS Organizations service control policy (p-5esynk4f, org o-qqwt3ekky6). Not an IAM permission we can fix; stack rolled back. No workaround attempted.
- Cluster and node group created manually in the console by Brian: K8s 1.37 (not 1.35), Auto Mode off, BrianEKSClusterRole, public subnets of brian-test-vpc, public+private endpoint, API_AND_CONFIG_MAP. Node group ng-main: 2 x m6i.xlarge AL2023 on-demand, min 0 / max 3, 50 GiB gp3 root, role Brian-EKS-NodeGroup. EBS CSI via Pod Identity using Brian-AmazonEKSPodIdentityAmazonEBSCSIDriverRole.
- `aws eks update-kubeconfig --name brian-eks-playground --region us-east-1`: ok; 2 nodes Ready.
- PVC test (1Gi, storageClassName gp2) in ns csi-test -> Pending. Decoded denial: SCP p-5esynk4f statement "DenyCreationOfGP2Volumes" denies ec2:CreateVolume/AttachVolume/RunInstances for VolumeType gp2. Driver auth works. Namespace deleted; no volume created.
- Review: cluster has NO default StorageClass (gp2 exists but is not annotated default). Node root volumes are gp3. No repo workload currently enables a PVC (easytrade persistence disabled). Next: gp3 default StorageClass + retest (awaiting Brian's OK).
- `kubectl apply -f storageclass-gp3.yaml`: StorageClass gp3 (ebs.csi.aws.com, type gp3, WaitForFirstConsumer, expansion on) created and marked default. gp2 left in place (not default).
- PVC test v2 in ns csi-test, PVC with NO storageClassName (tests the default path): bound to gp3 in ~8s; EBS vol-09bd10812540b31e6 (gp3, 1 GiB, us-east-1a) created and attached; pod wrote and read /data. PASS.
- `kubectl delete namespace csi-test`: EBS volume deleted (DescribeVolumes -> InvalidVolume.NotFound); no PVs remain.
- ArgoCD pin review: v3.4.3 (2026-05-28) is affected by 6 critical/high advisories published 2026-10-07 (fixed in 3.4.10+). Latest: 3.4.13, 3.5.7, 3.6.1 (2026-11-04). Tested K8s: 3.4 -> up to 1.35; 3.5 -> up to 1.36; 3.6 -> 1.34 to 1.37. Version choice pending Brian.
- `kubectl create namespace argocd` and `kubectl create namespace dynatrace`: created.
- ArgoCD version chosen by Brian: v3.6.1 (only line tested on K8s 1.37; includes Oct 2026 security fixes). Note: 3.5+ bundles Helm 4.2.
- `kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml` (server-side apply is now required by the ArgoCD docs; ApplicationSet CRD exceeds client-side annotation limit): ok.
- `kubectl -n argocd rollout status deploy/argocd-server`: rolled out. All 7 argocd pods Running; CRDs applications/applicationsets/appprojects present; all services ClusterIP (no AWS load balancer created).
- Dynatrace token Secret: deferred by Brian until right before the root app is applied.
- NOT applied: projects/playground.yaml, bootstrap/root-app.yaml (gitops still points at the old tenant and gke-playground paths).

## Teardown (when done)
1. Delete app namespaces/PVCs and any LoadBalancer Services first (they create EBS volumes and ELBs outside the cluster).
2. Console: delete node group ng-main, then cluster brian-eks-playground (it was created in the console, not by eksctl).
3. Check for leftovers: `aws ec2 describe-volumes --filters Name=tag-key,Values=kubernetes.io/created-for/pvc/name` and `aws elbv2 describe-load-balancers` / `aws elb describe-load-balancers` in brian-test-vpc.
4. Do NOT delete brian-test-vpc: other people's clusters use it.
5. Delete the empty failed stack eksctl-brian-eks-playground-cluster (ROLLBACK_COMPLETE) if still present.
