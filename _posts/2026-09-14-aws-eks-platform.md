---
title: "aws-eks-platform: a platform in Kubernetes"
date: 2026-09-14 06:00:00 +0000
mermaid: true
---


Nowadays LLMs are very popular, so I wanted to experiment with deploying one on brand-new infrastructure, starting everything from zero. A popular choice for serving LLMs is vLLM, an inference engine that delivers high throughput and efficient memory usage during inference. You can test vLLM on a single machine with a GPU, and I ran my first experiments using the GPU sandbox I introduced in the previous post. But to reach the LLM from anywhere in the world, we need to host it behind an architecture that allows public access — in this case, a Kubernetes cluster.

The goal of this post is to explain how to build an LLM platform on Kubernetes, so I'll focus on the architecture itself: starting with the VPC setup and going all the way to running the vLLM server in a pod and exposing it through a load balancer using the Traefik ingress controller.

I'll build this infrastructure with Terraform, one module for the static infra and another for the EKS cluster. The Terraform states will be persisted in an S3 bucket.

Inside the Kubernetes cluster I'll deploy Prometheus for monitoring and Grafana for data visualization. Argo CD will also be installed in the cluster to control the status and sync of all the Kubernetes apps.

Code is up at [github.com/quezadasubiabre/aws-eks-platform](https://github.com/quezadasubiabre/aws-eks-platform).

## Overview

### Static infrastructure

The static infrastructure contains the definition of all the resources that set up the networking for our platform. Everything starts with the VPC definition, which spans 3 availability zones. This gives us 3 public subnets and 3 private subnets.

The VPC uses the CIDR block `10.0.0.0/16`. Each subnet is carved out of it with `cidrsubnet(vpc_cidr, 4, i)`, which adds 4 bits to the prefix and produces `/20` blocks:

- **Public subnets:** `10.0.0.0/20`, `10.0.16.0/20`, `10.0.32.0/20`
- **Private subnets:** offset by the AZ count, so `10.0.48.0/20`, `10.0.64.0/20`, `10.0.80.0/20`

A `/20` gives roughly 4,091 usable IPs per subnet (AWS reserves 5 addresses in every subnet). That headroom matters for pod density: if you use the AWS VPC CNI, every pod gets a real IP from the subnet, so the subnet size directly caps how many pods can run in each AZ.

To allow communication between the resources created inside each subnet, we need an internet gateway and the right route tables, starting with the public side. A public route table is defined, where all traffic from the public subnets flows through the internet gateway. In the first public subnet we also create a NAT gateway, which receives traffic from the private subnets. We use only one NAT gateway to avoid extra costs — each one has a fixed monthly cost of around $32, plus data processing charges. In each private subnet, a route table is created that points to the NAT gateway in the first subnet.

Finally, the static layer saves the key network identifiers — the VPC ID and the lists of public and private subnet IDs — as SSM parameters under `/k8s-cloud-project/network/`. This way they are stored for future use: the later layers (the Kubernetes cluster, the ingress setup) read these values from SSM instead of depending on Terraform remote state, which keeps each layer decoupled and independently applyable.

### Kubernetes cluster

The Kubernetes cluster has two parts: the infrastructure itself, managed by Terraform, and the applications installed on top of it, managed by Argo CD — some of which also create their own AWS resources, like the Traefik controller, which provisions a load balancer.

#### Kubernetes static infra

The cluster has two nodes: a system node for control, monitoring, and platform applications, and a GPU node dedicated to running vLLM.

For the system node we chose a `t3.medium` EC2 instance, and added extra configuration (via `cloudinit_pre_nodeadm`) to raise the number of private IPs — and therefore pods — it can host. See [the note on pod density below](#a-note-on-pod-density-why-prefix-delegation) for why this is necessary.

**IRSA** (IAM Roles for Service Accounts) is enabled with `enable_irsa = true`. This registers an OIDC provider for the cluster, letting individual Kubernetes service accounts assume specific IAM roles via short-lived, federated credentials — instead of every pod inheriting whatever IAM permissions the node has. It's what lets the EBS CSI driver's service account (`ebs-csi-controller-sa`) assume `aws_iam_role.ebs_csi_driver` and get only `AmazonEBSCSIDriverPolicy`, nothing more.

**Admin access** is granted through an `access_entries` block instead of the older `aws-auth` ConfigMap. It maps a specific IAM user (`var.admin_iam_user`) to `AmazonEKSClusterAdminPolicy` at cluster scope — necessary because, by default, only the IAM identity that created the cluster gets API access.

**One security-group rule was opened by hand:** `node_security_group_additional_rules` lets the control plane reach port 10251 on the nodes — the port metrics-server's webhook listens on. The EKS API server calls it directly to serve `kubectl top` and feed Horizontal Pod Autoscalers, and the module doesn't open this port by default.

In the cluster, 5 add-ons were installed:

- **kube-proxy** — runs on every node and implements Kubernetes Services: it programs each node's networking rules so traffic sent to a Service's stable ClusterIP gets load-balanced across the actual pod IPs behind it. Installed at its most recent version.
- **vpc-cni** — assigns each pod a real routable IP from the VPC subnet. Configured here with prefix delegation enabled (`ENABLE_PREFIX_DELEGATION=true`, `WARM_PREFIX_TARGET=1`) so each ENI gets a `/28` block of IPs in one allocation instead of one secondary IP at a time, raising max pods per node beyond the t3.medium's default 17-pod limit.
- **coredns** — the cluster's internal DNS server, resolving Service and pod names (e.g. `vllm-service.default.svc.cluster.local`) to their ClusterIPs. Deliberately installed as a standalone `aws_eks_addon` rather than inside `cluster_addons`, because installing it there makes the module wait on pod scheduling before nodes exist, which stalls the apply.
- **metrics-server** — collects CPU/memory usage from kubelets and exposes it through the Kubernetes Metrics API, powering `kubectl top` and Horizontal Pod Autoscalers. Scaled to 1 replica (not the default 2) since this is a single-node cluster and extra replicas would just burn scarce pod-IP slots.
- **aws-ebs-csi-driver** — lets pods claim persistent storage backed by EBS volumes (used for the vLLM model-weights PVC). The controller is also scaled to 1 replica for the same reason, with IRSA (`aws_iam_role.ebs_csi_driver`) granting it the `AmazonEBSCSIDriverPolicy` via the cluster's OIDC provider.

The GPU node is a `g5.xlarge` (falling back to `g4dn.xlarge` to widen the Spot capacity pool), running on Spot to keep costs down, using the `AL2023_x86_64_NVIDIA` AMI so the NVIDIA driver and container runtime come preinstalled. Its root volume is bumped to 100GB (gp3), since the vLLM image alone is around 11GB compressed and expands considerably once unpacked; model weights are kept separately, on their own PVC.

Unlike the system node, the GPU node group is provisioned as a standalone `aws_eks_node_group` with its own IAM role and launch template, rather than through `eks_managed_node_groups`. That's because the EKS module's managed-node-group submodule hardcodes a lifecycle rule that ignores changes to `desired_size` after creation — scaling it through the module again would silently do nothing. Managing it directly keeps `desired_size` mutable.

That mutability is what makes the node controllable through the `gpu_desired_size` Terraform variable, which defaults to `0`. The GPU node doesn't exist — and isn't being billed — until it's scaled up (`terraform apply -var gpu_desired_size=1`) for active testing, then back down to `0` afterward. A simple on/off switch for the most expensive resource in the cluster: a Spot `g5.xlarge` runs roughly $0.30–0.45/hr in `eu-west-1`.

The node also carries a taint, so only pods that explicitly tolerate it — vLLM's own deployment — get scheduled onto it, keeping regular workloads like Argo CD or monitoring off:

```hcl
taint {
  key    = "nvidia.com/gpu"
  value  = "true"
  effect = "NO_SCHEDULE"
}
```

#### A note on pod density: why prefix delegation

By default, the AWS VPC CNI gives every pod its own IP address by attaching **secondary IP addresses** to the node's ENIs (Elastic Network Interfaces), one IP per pod. Each EC2 instance type has a hardware limit on how many ENIs it supports and how many IPv4 addresses fit on each ENI. For a `t3.medium`, that limit is 3 ENIs, each holding up to 6 IPv4 addresses — 18 addresses total. One of those is the node's own primary IP (reserved for the instance itself, not schedulable to a pod), which leaves **17 IPs for pods**:

```
3 ENIs × 6 IPs/ENI = 18 IPs
18 − 1 (node's own primary IP) = 17 pods max
```

That ceiling is fixed by the instance type alone — it has nothing to do with how much CPU or memory the node actually has free. On a small cluster where the system node also needs to run CoreDNS, metrics-server, the EBS CSI driver, Argo CD, Prometheus, and Grafana, 17 pods gets tight fast.

Prefix delegation changes what gets attached to an ENI. Instead of requesting IPs one at a time, the CNI requests an entire `/28` prefix — 16 addresses — in a single allocation:

```hcl
vpc-cni = {
  most_recent = true
  configuration_values = jsonencode({
    env = {
      ENABLE_PREFIX_DELEGATION = "true"
      WARM_PREFIX_TARGET       = "1"
    }
  })
}
```

- `ENABLE_PREFIX_DELEGATION = "true"` switches the CNI from "one secondary IP per allocation" to "one `/28` prefix per allocation," multiplying the IPs available per ENI roughly 16x.
- `WARM_PREFIX_TARGET = "1"` tells the CNI to always keep one spare `/28` prefix warmed up on the node, so a burst of new pods doesn't have to wait on an EC2 API call before they can get an IP and start.

The effect is that the *subnet's* address space (our `/20`s, with ~4,091 usable IPs — see above) becomes the real constraint on pod count, not the instance type's ENI/secondary-IP limits. This is exactly why the `t3.medium` system node can comfortably run the whole monitoring and GitOps stack alongside the platform add-ons, and it's also why the `maxPods` override in the node's `cloudinit_pre_nodeadm` config was necessary — kubelet doesn't know prefix delegation happened, so it has to be told explicitly that it may now schedule more than the old IP-per-secondary-address default of 17.

<figure>
<pre class="mermaid">
flowchart LR
    subgraph before["Without prefix delegation"]
        direction TB
        eni1["ENI"]
        ip1["IP"] --- eni1
        ip2["IP"] --- eni1
        ip3["IP"] --- eni1
        dots1["... up to 17 pods total"]
        eni1 -.-> dots1
    end

    subgraph after["With prefix delegation"]
        direction TB
        eni2["ENI"]
        prefix["/28 prefix\n(16 IPs, one allocation)"] --- eni2
        dots2["far more pods per node,\nlimited by subnet size instead"]
        prefix -.-> dots2
    end

    before ==> |"ENABLE_PREFIX_DELEGATION=true"| after
</pre>
<figcaption>Prefix delegation replaces per-pod IP requests to the ENI with a single /28 block, removing the instance type's secondary-IP count as the pod-density bottleneck.</figcaption>
</figure>

## Set up the cluster

### 1. Clone repo 
```
git clone https://github.com/quezadasubiabre/aws-eks-platform
```

### 2. Create static infra

```
terraform -chdir=infraestructure/static init
terraform -chdir=infraestructure/static apply
```

### 2. Create EKS cluster

```
terraform -chdir=infraestructure/eks init
terraform -chdir=infraestructure/eks apply
```

After about 12 minutes the cluster will be created.

### 4. Log in to the cluster

```
aws eks update-kubeconfig --region eu-west-1 --name k8s-cloud-project
```
