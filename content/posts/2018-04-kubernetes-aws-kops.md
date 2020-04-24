+++
title = "Kubernetes on AWS: An overview of KOPS"
date = "2018-04-10"
tags = ["kops", "aws", "kubernetes"]
aliases = [ "/blog/kubernetes-on-aws-an-overview-of-kops/", "/blurbs/kubernetes-on-aws-an-overview-of-kops/"]
+++

### What is KOPS?

Self described as "The easiest way to get a production grade Kubernetes cluster up
and running” (on AWS (and others, see below)). KOPS looks a lot like Terraform. In broad
strokes it takes cluster, context specific arguments and creates cloud resources
that house and facilitate the usage of a Kubernetes cluster.

### KOPS Highlights

- Automates the provisioning of Highly Available (HA) clusters on AWS from the CLI,
similar to helm or kubectl.
- Leverages platform specific state stores to allow for cluster architecture change
dry runs and idempotent operations.
- Officially supports AWS. KOPS also has alpha support for Google Cloud Platform
and VMware vSphere.

### Cluster creation and customization params

KOPS allows for many options during cluster creation, upgrade, and changes. A few of which are:

- Size and type of Master, Worker nodes (ex: t2.large)
- Network specific settings (public vs internal), CIDRs, CNI impl, etc…
- Creation of bastion nodes
- DNS settings
- User security, RBAC settings, etc...

### Cluster creation steps

KOPS steps through a series of phases as it creates validates a new cluster and whenever it applies changes to existing clusters.

1. Queries for existing cluster information in the external state store (S3)
2. Uses the AWS API to create a series of resources (SG, VPC, Subnet, EC2, ASG, etc...)
3. Awaits communication with the cluster
4. Confirms and validates the new cluster or cluster changes

### AWS Cluster Resources

A KOPS-created Kubernetes cluster leverages the following AWS resource types:

- IAM Roles and Policies
- VPC, Subnets, and Routes, Route Tables, NAT and Internet Gateways
- Security Groups
- Route53 A Records (API, bastion)
- Elastic Load Balancers (ELBs), Auto Scaling Groups (ASG), Launch Configurations w/ defined user data
- Elastic Block Store (EBS) volumes for etcd primary data store and for etcd events

![image](/img/2018-04-kubernetes-aws-kops/diagram.png)

### Cluster Creation: required AWS roles

In order to provision the aforementioned resources, the KOPS client requires, at minimum, the following Roles:

- AmazonEC2FullAccess
- AmazonRoute53FullAccess
- AmazonS3FullAccess
- IAMFullAccess
- AmazonVPCFullAccess

### EC2 Instance Types

There are three types of EC2 instances that get created with an Kubernetes cluster on AWS:

1. Bastion nodes
2. Master nodes
3. Worker nodes

Bastion nodes provide an external facing point of entry into a network containing
private network instances. This host can provide a single point of fortification
or audit and can be started and stopped to enable or disable inbound SSH communication
from the Internet, some call bastion as the "jump server".

### EC2 Policies

The three types of EC2 instances leverage attached AWS IAM Roles along with
client [Instance Profiles](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) in order to execute cluster functions that interact
with AWS resources.

Bastion nodes have the most basic policy, which merely allows them to describe
regions for the account.

Master nodes have the following policies:

- full access to EC2
- full access to Elastic Load Balancing
- describe, terminate, and capacity change operations against Auto Scaling Groups
- read and fetch operations for ECR
- describe for EC2
- read, notification, and limited update operations for Route 53
- read and write operations for S3 (limited to the KOPS state store)

Worker nodes have a subset of the master policies:

- read and fetch operations for ECR
- describe for EC2
- read, notification, and limited update operations for Route 53
- read and write operations for S3 (limited to the KOPS state store)

### EC2 Startup

Master and Worker node user data definitions (described in their respective Auto Scaling
Group Launch Configurations) direct each node to execute a script on first startup.
These scripts executes and runs on top of the already-defined AMI (disk image).

Each script is responsible for transforms each Master and Worker EC2 instance into a Kubernetes node
and collectively a cluster.

The script generates a kube environment YAML describing the node’s in the cluster
role via its InstanceGroupName, retrieves nodeup and [protokube](https://github.com/kubernetes/kops/tree/master/protokube), and boots. [Nodeup](https://github.com/kubernetes/kops/tree/master/cmd/nodeup)
is a standalone binary that handles transforms, bootstraps the Kubernetes cluster.

The following is a sample master user data script:

```
#!/bin/bash
# Copyright 2016 The Kubernetes Authors All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -o errexit
set -o nounset
set -o pipefail

NODEUP_URL=https://kubeupv2.s3.amazonaws.com/kops/1.8.1/linux/amd64/nodeup
NODEUP_HASH=






function ensure-install-dir() {
  INSTALL_DIR="/var/cache/kubernetes-install"
  # On ContainerOS, we install to /var/lib/toolbox install (because of noexec)
  if [[ -d /var/lib/toolbox ]]; then
    INSTALL_DIR="/var/lib/toolbox/kubernetes-install"
  fi
  mkdir -p ${INSTALL_DIR}
  cd ${INSTALL_DIR}
}

# Retry a download until we get it. Takes a hash and a set of URLs.
#
# $1 is the sha1 of the URL. Can be "" if the sha1 is unknown.
# $2+ are the URLs to download.
download-or-bust() {
  local -r hash="$1"
  shift 1

  urls=( $* )
  while true; do
    for url in "${urls[@]}"; do
      local file="${url##*/}"
      rm -f "${file}"

      if [[ $(which curl) ]]; then
        if ! curl -f --ipv4 -Lo "${file}" --connect-timeout 20 --retry 6 --retry-delay 10 "${url}"; then
          echo "== Failed to curl ${url}. Retrying. =="
          break
        fi
      elif [[ $(which wget ) ]]; then
        if ! wget --inet4-only -O "${file}" --connect-timeout=20 --tries=6 --wait=10 "${url}"; then
          echo "== Failed to wget ${url}. Retrying. =="
          break
        fi
      else
        echo "== Could not find curl or wget. Retrying. =="
        break
      fi

      if [[ -n "${hash}" ]] && ! validate-hash "${file}" "${hash}"; then
        echo "== Hash validation of ${url} failed. Retrying. =="
      else
        if [[ -n "${hash}" ]]; then
          echo "== Downloaded ${url} (SHA1 = ${hash}) =="
        else
          echo "== Downloaded ${url} =="
        fi
        return
      fi
    done

    echo "All downloads failed; sleeping before retrying"
    sleep 60
  done
}

validate-hash() {
  local -r file="$1"
  local -r expected="$2"
  local actual

  actual=$(sha1sum ${file} | awk '{ print $1 }') || true
  if [[ "${actual}" != "${expected}" ]]; then
    echo "== ${file} corrupted, sha1 ${actual} doesn't match expected ${expected} =="
    return 1
  fi
}

function split-commas() {
  echo $1 | tr "," "\n"
}

function try-download-release() {
  # TODO(zmerlynn): Now we REALLY have no excuse not to do the reboot
  # optimization.

  local -r nodeup_urls=( $(split-commas "${NODEUP_URL}") )
  local -r nodeup_filename="${nodeup_urls[0]##*/}"
  if [[ -n "${NODEUP_HASH:-}" ]]; then
    local -r nodeup_hash="${NODEUP_HASH}"
  else
  # TODO: Remove?
    echo "Downloading sha1 (not found in env)"
    download-or-bust "" "${nodeup_urls[@]/%/.sha1}"
    local -r nodeup_hash=$(cat "${nodeup_filename}.sha1")
  fi

  echo "Downloading nodeup (${nodeup_urls[@]})"
  download-or-bust "${nodeup_hash}" "${nodeup_urls[@]}"

  chmod +x nodeup
}

function download-release() {
  # In case of failure checking integrity of release, retry.
  until try-download-release; do
    sleep 15
    echo "Couldn't download release. Retrying..."
  done

  echo "Running nodeup"
  # We can't run in the foreground because of https://github.com/docker/docker/issues/23793
  ( cd ${INSTALL_DIR}; ./nodeup --install-systemd-unit --conf=${INSTALL_DIR}/kube_env.yaml --v=8  )
}

####################################################################################

/bin/systemd-machine-id-setup || echo "failed to set up ensure machine-id configured"

echo "== nodeup node config starting =="
ensure-install-dir

cat > cluster_spec.yaml << '__EOF_CLUSTER_SPEC'
cloudConfig: null
docker:
  bridge: ""
  ipMasq: false
  ipTables: false
  logDriver: json-file
  logLevel: warn
  logOpt:
  - max-size=10m
  - max-file=5
  storage: overlay,aufs
  version: 1.13.1
encryptionConfig: null
kubeAPIServer:
  address: 127.0.0.1
  admissionControl:
  - Initializers
  - NamespaceLifecycle
  - LimitRanger
  - ServiceAccount
  - PersistentVolumeLabel
  - DefaultStorageClass
  - DefaultTolerationSeconds
  - NodeRestriction
  - Priority
  - ResourceQuota
  allowPrivileged: true
  anonymousAuth: false
  apiServerCount: 1
  authorizationMode: AlwaysAllow
  cloudProvider: aws
  etcdServers:
  - http://127.0.0.1:4001
  etcdServersOverrides:
  - /events#http://127.0.0.1:4002
  image: gcr.io/google_containers/kube-apiserver:v1.8.7
  insecurePort: 8080
  kubeletPreferredAddressTypes:
  - InternalIP
  - Hostname
  - ExternalIP
  logLevel: 2
  requestheaderAllowedNames:
  - aggregator
  requestheaderExtraHeaderPrefixes:
  - X-Remote-Extra-
  requestheaderGroupHeaders:
  - X-Remote-Group
  requestheaderUsernameHeaders:
  - X-Remote-User
  securePort: 443
  serviceClusterIPRange: 100.64.0.0/13
  storageBackend: etcd2
kubeControllerManager:
  allocateNodeCIDRs: true
  attachDetachReconcileSyncPeriod: 1m0s
  cloudProvider: aws
  clusterCIDR: 100.96.0.0/11
  clusterName: abc.joshdurbin.net
  configureCloudRoutes: true
  image: gcr.io/google_containers/kube-controller-manager:v1.8.7
  leaderElection:
    leaderElect: true
  logLevel: 2
  useServiceAccountCredentials: true
kubeProxy:
  clusterCIDR: 100.96.0.0/11
  cpuRequest: 100m
  featureGates: null
  hostnameOverride: '@aws'
  image: gcr.io/google_containers/kube-proxy:v1.8.7
  logLevel: 2
kubeScheduler:
  image: gcr.io/google_containers/kube-scheduler:v1.8.7
  leaderElection:
    leaderElect: true
  logLevel: 2
kubelet:
  allowPrivileged: true
  cgroupRoot: /
  cloudProvider: aws
  clusterDNS: 100.64.0.10
  clusterDomain: cluster.local
  enableDebuggingHandlers: true
  evictionHard: memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%,imagefs.inodesFree<5%
  featureGates:
    ExperimentalCriticalPodAnnotation: "true"
  hostnameOverride: '@aws'
  kubeconfigPath: /var/lib/kubelet/kubeconfig
  logLevel: 2
  networkPluginMTU: 9001
  networkPluginName: kubenet
  nonMasqueradeCIDR: 100.64.0.0/10
  podInfraContainerImage: gcr.io/google_containers/pause-amd64:3.0
  podManifestPath: /etc/kubernetes/manifests
  requireKubeconfig: true
masterKubelet:
  allowPrivileged: true
  cgroupRoot: /
  cloudProvider: aws
  clusterDNS: 100.64.0.10
  clusterDomain: cluster.local
  enableDebuggingHandlers: true
  evictionHard: memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%,imagefs.inodesFree<5%
  featureGates:
    ExperimentalCriticalPodAnnotation: "true"
  hostnameOverride: '@aws'
  kubeconfigPath: /var/lib/kubelet/kubeconfig
  logLevel: 2
  networkPluginMTU: 9001
  networkPluginName: kubenet
  nonMasqueradeCIDR: 100.64.0.0/10
  podInfraContainerImage: gcr.io/google_containers/pause-amd64:3.0
  podManifestPath: /etc/kubernetes/manifests
  registerSchedulable: false
  requireKubeconfig: true

__EOF_CLUSTER_SPEC

cat > ig_spec.yaml << '__EOF_IG_SPEC'
kubelet: null
nodeLabels:
  kops.k8s.io/instancegroup: master-us-west-2a
taints: null

__EOF_IG_SPEC

cat > kube_env.yaml << '__EOF_KUBE_ENV'
Assets:
- 0f3a59e4c0aae8c2b2a0924d8ace010ebf39f48e@https://storage.googleapis.com/kubernetes-release/release/v1.8.7/bin/linux/amd64/kubelet
- 36340bb4bb158357fe36ffd545d8295774f55ed9@https://storage.googleapis.com/kubernetes-release/release/v1.8.7/bin/linux/amd64/kubectl
- 1d9788b0f5420e1a219aad2cb8681823fc515e7c@https://storage.googleapis.com/kubernetes-release/network-plugins/cni-0799f5732f2a11b329d9e3d51b9c8f2e3759f2ff.tar.gz
- 42b15a0a0a56531750bde3c7b08d0cf27c170c48@https://kubeupv2.s3.amazonaws.com/kops/1.8.1/linux/amd64/utils.tar.gz
ClusterName: abc.joshdurbin.net
ConfigBase: s3://jdurbin-kops/abc.joshdurbin.net
InstanceGroupName: master-us-west-2a
Tags:
- _automatic_upgrades
- _aws
- _kubernetes_master
channels:
- s3://jdurbin-kops/abc.joshdurbin.net/addons/bootstrap-channel.yaml
protokubeImage:
  hash: 0b1f26208f8f6cc02468368706d0236670fec8a2
  name: protokube:1.8.1
  source: https://kubeupv2.s3.amazonaws.com/kops/1.8.1/images/protokube.tar.gz

__EOF_KUBE_ENV

download-release
echo "== nodeup node config done =="
```

The userdata blocks for the worker nodes are the same minus the following sections of the `cluster_spec.yaml`:

- encryptionConfig
- kubeAPIServer
- kubeControllerManager
- kubeScheduler
- masterKubelet

... and differences between the tags and more importantly the `InstanceGroupName`.


### Cluster Access

Most clusters are provisioned with private networking, meaning the only cloud
resources that are accessible from the external world are the Bastion and Master
node ELBs. The Master and Bastion nodes themselves are not directly accessible
from the outside world.

The Bastion ELB provides SSH load balancing between the Bastion nodes. The
Master nodes, which run the Kubernetes REST API (*the API), are available
via API ELB, provisioned and made available via the cluster name with an
"api" prefix (ex: "api.k8s.joshs-test-cluster.insitkcorp.com”).

AWS Internet Gateway and NAT are used by the all nodes for outbound traffic.

AWS VPC Peering connections are made on an as-needed basis to existing VPCs
when needed (ex: RDS, Mongo access).

** Durable data stores will not be migrated into Kubernetes.

### Networking, Security Groups

- ELBs have world ingress rules on 443 (https) for API, 22 (SSH) for Bastion
- ELBs do not handle SSL termination, they forward traffic on port 443 to the Master Nodes for certificate evaluation -- which is particularly important when the cluster is RBAC enabled
- Bastions are only allowed port 22 (SSH) access to Master and Worker nodes
- Master to Master, Master to Worker, and Worker to Worker node traffic is completely unrestricted on all ports, UDP/TCP
- Worker nodes have full access to Master nodes via UDP, and all but two ports via TCP, excluding 4001 for etcd main and 4002 for etcd event traffic

### Addons

- Cluster autoscaler
- nginx-based ingress
- Kubernetes dashboard
- Route 53 (DNS) mapper
- Logging and other monitoring

### Final Highlights

- KOPS supports output of / to Terraform and Cloudformation (AWS cloud resource
infrastructure as code offering)
- Supports partial integration with pre-existing resources (ex: Route53 Zones, VPCs)
- Master nodes are typically tainted so they’re not eligible to run Pods

### Supplemental Resources

- [KOPS Getting Started on AWS](https://github.com/kubernetes/kops/blob/master/docs/aws.md)
- [Low level description of how KOPS works](https://github.com/kubernetes/kops/blob/master/docs/development/how_it_works.md)
- [KOPS addons](https://github.com/kubernetes/kops/blob/master/docs/addons.md)
- [KOPS documentation](https://github.com/kubernetes/kops/blob/master/docs/README.md)
- [AWS EC2 Role Assignment Documentation](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)
