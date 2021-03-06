# K2  
Deploy a __Kubernetes__ cluster on top of __CoreOS__ using __Terraform__  and __Ansible__.

[![Docker Repository on Quay](https://quay.io/repository/samsung_cnct/k2/status "Docker Repository on Quay")](https://quay.io/repository/samsung_cnct/k2)

## What is K2
K2 is an orchestration and cluster level management system for [Kubernetes](https://kubernetes.io).  By default,  K2 will create a production scale Kubernetes
cluster on a range of platforms.  It boasts a large configuration file for extensive tweaking for your environment and use case.  Especially if you are getting 
started and don't need a HA production level cluster right away.

We (Samsung CNCT) built this tool to aid in our own research into high performance and reliability for the Kubernetes control plane.  We realised this would be a useful
tool for the public at large and released it as [Kraken](https://github.com/samsung-cnct/kraken).  Kraken is great but it was developed for research and rather quickly.
After using it ourselves for almost a year and running with the pain points we decided it was best to take the best parts and build anew.  It remains an ansible and 
terraform project because we believe those tools provide flexible and powerful abstractions at the right layers.  

K2 provides the same functionality with much cleaner internal abstractions.  This makes it easier for both external contributions and internal ones.  It will also allow
us to continue to quickly improve and evolve with the Kubernetes ecosystem as a whole.

## What is K2 for
K2 is targeted at operations teams that need to support Kubernetes.  This is becoming known as ClusterOps.  K2 provides a single interface to manage your Kubernetes
clusters across all environments.

K2 uses a single file to drive cluster configuration.  This makes it easy to check the file into a vcs of your choice and solve two major problems:
1. use version control for your cluster configuration as you promote changes from dev through to production for either existing cluster configurations or brand new ones
2. enable Continuous Integration for developer applications against sandboxed and transient Kubernetes clusters.  K2 contains a destroy command that will clean up all
traces of the temporary infrastructure

We believe solving these two problems is a baseline for effectively and efficiently nurturing a Kubernetes based infrastructure.

## K2 supported addons
K2 also supports a number of Samsung CNCT supported addons in the form of Kubernetes Charts.  Those charts can be found in the [K2 Charts repository](https://github.com/samsung-cnct/k2-charts).
These charts are tested and maintained by Samsung CNCT.  They should work on any Kubernetes cluster.  

# Getting Started with K2
The easiest and best supported path for using K2 is to use [K2Cli](https://github.com/samsung-cnct/k2cli).  This cli wraps the K2 image in an easy to use
and configure tool.

If you want to use the K2 image directly, please continue with this guide.

## Prerequisites

You will need to have the following:

- A machine that can run Docker
- A text editor
- Amazon credentials with the following privileges:
  - Launch EC2 instances
  - Create VPCs
  - Create ELBs
  - Create EBSs
  - Create Route 53 Records
  - Create IAM roles for EC2 instances

### Running without tools docker image

You will need the following installed on your machine:

- Python 2.x (virtualenv strongly suggested)
 - pip
 - boto
 - netaddr 
- Ansible 2.2.x
- Cloud SDKs
 - aws cli
 - cli53 (https://github.com/barnybug/cli53/releases)
 - gcloud SDK
- Terraform and providers
 - Terraform 0.7.x
 - Terraform execute provider 0.0.3 (https://github.com/samsung-cnct/terraform-provider-execute/releases)  
 - Terraform coreosbox provider 0.0.2 (https://github.com/samsung-cnct/terraform-provider-coreosbox/releases) 
- kubectl 1.3.x 
- helm alpha.5 or later 


## The K2 image

The easiest way to get started with K2 directly is to use a K2 container image

`docker pull quay.io/samsung_cnct/k2:latest`

## Preparing the environment  
  
### Initial K2 Directory
If this is your first time using K2, use K2 docker image to generate a 'sensible defaults' configuration (this assumes AWS is the infrastructure provider):

With docker container:

```bash
docker run -v ~:/root --rm=true -it quay.io/samsung_cnct/k2:latest ./up.sh --generate
```

With cloned repo:

```bash
./up.sh --generate
```

This will generate a config.yaml file at 

```
~/.kraken/config.yaml
```

### Preparing AWS credentials

_If you already have configured your machine to be able to use AWS, you can skip this step_

To setup AWS credentials, please run the following command

With docker container:

```bash
docker run -v ~/:/root -it --rm=true quay.io/samsung_cnct/k2:latest bash -c 'aws configure'
```

with local awscli:

```bash
 aws configure
```

This will allow you to configure the environment with your AWS credentials

### kubectl

To use the kubectl shipped with K2, run a command similar to:

```bash
docker run -v ~/:/root -it --rm=true quay.io/samsung_cnct/k2:latest kubectl --kubeconfig /root/.kraken/YOURCLUSTER/admin.kubeconfig get nodes
```

with locally installed kubectl:

```bash
`kubectl --kubeconfig ~/.kraken/YOURCLUSTER/admin.kubeconfig get nodes`
```

### helm

To use the helm shipped with K2, run a command similar to:

```bash
docker run -v ~/:/root -it --rm=true -e HELM_HOME=/root/.kraken/YOURCLUSTER/.helm -e KUBECONFIG=/root/.kraken/YOURCLUSTER/admin.kubeconfig quay.io/samsung_cnct/k2:latest helm list
```

with locally installed kubectl:

```bash
export KUBECONFIG=~/.kraken/YOURCLUSTER/admin.kubeconfig
`helm list --home ~/.kraken/YOURCLUSTER/.helm`
```

### ssh

After creating a cluster you should be able to ssh to various cluster nodes

```bash
ssh master-3 -F ~/.kraken/YOURCLUSTER/ssh_config
```

Cluster creating process generates an ssh config file at

```bash
 ~/.kraken/YOURCLUSTER/ssh_config
```

Host names are based on node pool names from your config file. I.e. if you had a config file with nodepool section like so:

```
nodepool:
  -
    name: etcd
    count: 5
    ...
  -
    name: etcdEvents
    count: 5
    ...
  -
    name: masterNodes
    count: 3
    ...
  -
    name: clusterNodes
    count: 3
    ...
  -
    name: specialNodes
    count: 2
    ...
```

Then the ssh hostnames available will be:

- etcd-1 through etcd-5
- etcdEvents-1 through etcdEvents-5
- masterNodes-1 through masterNodes-3
- clusterNodes-1 through clusterNodes-3
- specialNodes-1 through specialNodes-2

## Configure your Kubernetes Cluster

Earlier, you copied a sample cluster configuration over into `~/.kraken`  Please take a moment to review the sample configuration and make changes if you desire

It may be preferable to save it to a name that is consistent to the `cluster` variable in the configuration.  In other words, if your `cluster` is `foo`, then perhaps your file should be named `foo.yaml`

### Important configuration variables to adjust

While all configuration options are available for a reason, some are more important than others.  Some key ones include

- `cluster`
- `kubeConfig.version`
- `kubeConfig.hyperkubeLocation`
- `providerConfig.region`
- `nodepool[x].count`
- `nodepool[x].providerConfig.type`
- `clusterServices.services`

For a detailed explanation of all configuration variables, please consult [our configuration documentation](Documentation/kraken-configs/README.md)

## Starting your own Kubernetes Cluster

### Normal Initial Flow

To boot up a cluster per your configuration, please execute the following command:

```bash
docker run --rm=true -it -v ~/:/root quay.io/samsung_cnct/k2:latest ./up.sh --config /root/.kraken/foo.yaml
```

Replace `foo.yaml` with the name of the configuration file you intended to use 

Normally K2 will take a look at your configuration, generate artefacts like cloud-config files, and deploy VMs that will become your cluster.

During this time errors can happen if the configuration file is not as expected.  Please look at the errors and restart the cluster deployment if needed.

The amount of time it will take to deploy a new cluster is variable, but expect about 5 minutes from the time you start the command to when a cluster should be available for use

### Verifying cluster is available

After K2 has run, you should have a working cluster waiting for workloads.  To verify things are good, type in the following commands:

In all cases, assumption is made that `cluster` value in your configuration was `foo`.  Please change `foo` to your value

#### Getting K8s Nodes

```bash
docker run -v ~/:/root -it --rm=true quay.io/samsung_cnct/k2:latest kubectl --kubeconfig ~/.kraken/foo/admin.kubeconfig get nodes
```

This should result in the following:

```bash
NAME                                         STATUS                     AGE
ip-10-0-113-56.us-west-2.compute.internal    Ready,SchedulingDisabled   2m
ip-10-0-164-212.us-west-2.compute.internal   Ready                      2m
ip-10-0-169-86.us-west-2.compute.internal    Ready,SchedulingDisabled   3m
ip-10-0-194-57.us-west-2.compute.internal    Ready                      2m
ip-10-0-23-199.us-west-2.compute.internal    Ready                      3m
ip-10-0-36-28.us-west-2.compute.internal     Ready,SchedulingDisabled   2m
ip-10-0-58-24.us-west-2.compute.internal     Ready                      3m
ip-10-0-65-77.us-west-2.compute.internal     Ready                      2m
```

#### Getting K8s Deployments

```bash
docker run -v ~/:/root -it --rm=true quay.io/samsung_cnct/k2:latest kubectl --kubeconfig ~/.kraken/foo/admin.kubeconfig get deployments --all-namespaces
```

```bash
NAMESPACE     NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-system   kube-dns        1         1         1            1           8m
kube-system   tiller-deploy   1         1         1            1           8m
```

#### Deploy a new service

_Optional step_

In order to feel a bit more confident things are working, try having helm install a new service, like the K8S dashboard

##### Find Kubernetes Dashboard Version

```bash
docker run -v ~/:/root -it --rm=true -e HELM_HOME=/root/.kraken/foo/.helm -e KUBECONFIG=/root/.kraken/foo/admin.kubeconfig quay.io/samsung_cnct/k2:latest helm search kubernetes

$ atlas/kubernetes-dashboard-0.1.0.tgz
```

Now we know that we want `atlas/kubernetes-dashboard-0.1.0`

##### Install Kubernetes Dashboard

```bash
docker run -v ~/:/root -it --rm=true -e HELM_HOME=/root/.kraken/foo/.helm -e KUBECONFIG=/root/.kraken/foo/admin.kubeconfig quay.io/samsung_cnct/k2:latest helm install atlas/kubernetes-dashboard-0.1.0
```

```bash
Fetched atlas/kubernetes-dashboard-0.1.0 to /kraken/kubernetes-dashboard-0.1.0.tgz
foolhardy-scorpion
Last Deployed: Wed Oct  5 16:20:42 2016
Namespace: 
Status: DEPLOYED

Resources:
==> extensions/Deployment
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1         1         1            0           1s

==> v1/Service
NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   10.37.204.191   <pending>     80/TCP    1s
```

The chart has been installed.  It will take a moment for AWS ELB DNS to propagate, but we can get the DNS now

##### Finding DNS name for Kubernetes Dashboard

```bash
docker run -v ~/:/root -it --rm=true quay.io/samsung_cnct/k2:latest kubectl --kubeconfig ~/.kraken/foo/admin.kubeconfig describe service kubernetes-dashboard --namespace kube-system
```

```bash
Name:     kubernetes-dashboard
Namespace:    kube-system
Labels:     app=kubernetes-dashboard
Selector:   app=kubernetes-dashboard
Type:     LoadBalancer
IP:     10.37.204.191
LoadBalancer Ingress: aa95470398b1711e6b3a706da4d1c1f9-1324035908.us-west-2.elb.amazonaws.com
Port:     <unset> 80/TCP
NodePort:   <unset> 31638/TCP
Endpoints:    10.128.36.2:9090
Session Affinity: None
Events:
  FirstSeen LastSeen  Count From      SubobjectPath Type    Reason      Message
  --------- --------  ----- ----      ------------- --------  ------      -------
  6m    6m    1 {service-controller }     Normal    CreatingLoadBalancer  Creating load balancer
  6m    6m    1 {service-controller }     Normal    CreatedLoadBalancer Created load balancer
```

After a few minutes, we should feel comfortable going to http://aa95470398b1711e6b3a706da4d1c1f9-1324035908.us-west-2.elb.amazonaws.com and viewing the kubernetes dashboard

### Debugging

If K2 hangs during deployment, please hit ctrl-c to break out of the application and try again.  Note that some steps are slow and may not indicate things are hung.  Particularly `TASK [/kraken/ansible/roles/kraken.provider/kraken.provider.aws : Run cluster up] ***` and while waiting for a cluster to come up

If you desire to log into the VMs that have been created during this process, please use the AWS console.  There you will see various items

- EC2 Instances that include the `cluster` value in their name
- Auto Scaling Groups that include the `cluster` value in their name
- ELB (for apiserver) that includes the `cluster` value in its name
- VPC that includes the `cluster` value in its name
- Route 53 Zone that includes the `clusterDomain` value in its name

Using the EC2 instance list one can SSH into VMs and do further debugging

## Changing configuration

Some changes to the cluster can be done with K2

### Things that can be changed (sometimes with some manual intervention)

- nodepools
- nodepool counts and instance types
- cluster services desired to be run
- Kubernetes version / location of hypekube container

If you change these settings, a manual step may be needed.  For example you may need to terminate existing nodes to have new nodes spun up with updated configurations

### Things that should not be changed right now

- cluster name (`cluster` value in configuration)
- etcd settings (beyond machine type)

In order to effect these changes, make appropriate adjustments to the configuration file and re-run the command that created the cluster.

In other words:

```bash
docker run --rm=true -it -v ~/:/root quay.io/samsung_cnct/k2:latest ./up.sh --config /root/.kraken/foo.yaml
```

Replace `foo.yaml` with the name of the configuration file you intended to use 

## Destroying a Kubernetes Cluster

How zen of you - everything must come to end, including kubernetes clusters.  To destroy a cluster created with K2, please do the following:

```bash
docker run --rm=true -it -v ~/:/root quay.io/samsung_cnct/k2:latest ./down.sh --config /root/.kraken/foo.yaml
```

Replace `foo.yaml` with the name of the configuration file you intended to use 

# Docs
Further reading can be found here: 

[kraken documentation](Documentation/README.md)
