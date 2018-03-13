# Canonical Kubernetes with F5 Big-IP (cdk-f5-bigip)

This document describes how to integrate Canonical Kubernetes (CDK) with F5 Networks Big-IP load balancer devices.

We will deploy Canonical Kubernetes, the F5 Networks Big-IP Device and the F5 k8s-bigip-ctlr to control the loadbalancer.adi

## Deploying Canonical Kubernetes (CDK)

Deploying Canonical Kubernetes is really easy, I assume you are running on an Ubuntu machine and you have an AWS account.  Grab a new API key from AWS and put that into:

```
~/.aws/credentials
```

In this format:

```
[default]
aws_access_key_id=<access_id>
aws_secret_access_key=<secret_key>
```

Next install juju on your machine, we will use the snap:

```
sudo apt-get install snap
snap install juju
```

Next we bootstrap juju so it's ready to use aws:

```
juju bootstrap
```

We follow the steps here for bootstrapping our cloud for AWS:

```
calvinh@ubuntu-ws:~/.aws$ juju bootstrap
Clouds
aws
aws-china
aws-gov
azure
azure-china
cloudsigma
google
joyent
localhost
oracle
rackspace
salesmaas

Select a cloud [localhost]: aws

Regions in aws:
ap-northeast-1
ap-northeast-2
ap-south-1
ap-southeast-1
ap-southeast-2
ca-central-1
eu-central-1
eu-west-1
eu-west-2
sa-east-1
us-east-1
us-east-2
us-west-1
us-west-2

Select a region in aws [us-east-1]: eu-west-1

Enter a name for the Controller [aws-eu-west-1]: f5-k8s

Creating Juju controller "f5-k8s" on aws/eu-west-1
Looking for packaged Juju agent version 2.3.2 for amd64
Launching controller instance(s) on aws/eu-west-1...
 - i-08ce69142f943b5a4 (arch=amd64 mem=4G cores=2)eu-west-1a"
Installing Juju agent on bootstrap instance
Fetching Juju GUI 2.11.3
Waiting for address
Attempting to connect to 172.31.20.176:22
Attempting to connect to 34.244.155.220:22
Connected to 34.244.155.220
Running machine configuration script...


Bootstrap agent now started
Contacting Juju controller at 34.244.155.220 to verify accessibility...
Bootstrap complete, "f5-k8s" controller now available
Controller machines are in the "controller" model
Initial model "default" added
```

Once the controller has been configured, we now deploy CDK:

```
calvinh@ubuntu-ws:~/.aws$ juju deploy canonical-kubernetes
Located bundle "cs:bundle/canonical-kubernetes-150"
Resolving charm: cs:~containers/easyrsa-27
Resolving charm: cs:~containers/etcd-63
Resolving charm: cs:~containers/flannel-40
Resolving charm: cs:~containers/kubeapi-load-balancer-43
Resolving charm: cs:~containers/kubernetes-master-78
Resolving charm: cs:~containers/kubernetes-worker-81
Executing changes:
- upload charm cs:~containers/easyrsa-27 for series xenial
- deploy application easyrsa on xenial using cs:~containers/easyrsa-27
  added resource easyrsa
- set annotations for easyrsa
- upload charm cs:~containers/etcd-63 for series xenial
- deploy application etcd on xenial using cs:~containers/etcd-63
  added resource etcd
  added resource snapshot
- set annotations for etcd
- upload charm cs:~containers/flannel-40 for series xenial
- deploy application flannel on xenial using cs:~containers/flannel-40
  added resource flannel-amd64
  added resource flannel-s390x
- set annotations for flannel
- upload charm cs:~containers/kubeapi-load-balancer-43 for series xenial
- deploy application kubeapi-load-balancer on xenial using cs:~containers/kubeapi-load-balancer-43
- expose kubeapi-load-balancer
- set annotations for kubeapi-load-balancer
- upload charm cs:~containers/kubernetes-master-78 for series xenial
- deploy application kubernetes-master on xenial using cs:~containers/kubernetes-master-78
  added resource cdk-addons
  added resource kube-apiserver
  added resource kube-controller-manager
  added resource kube-scheduler
  added resource kubectl
- set annotations for kubernetes-master
- upload charm cs:~containers/kubernetes-worker-81 for series xenial
- deploy application kubernetes-worker on xenial using cs:~containers/kubernetes-worker-81
  added resource cni-amd64
  added resource cni-s390x
  added resource kube-proxy
  added resource kubectl
  added resource kubelet
- expose kubernetes-worker
- set annotations for kubernetes-worker
- add relation kubernetes-master:kube-api-endpoint - kubeapi-load-balancer:apiserver
- add relation kubernetes-master:loadbalancer - kubeapi-load-balancer:loadbalancer
- add relation kubernetes-master:kube-control - kubernetes-worker:kube-control
- add relation kubernetes-master:certificates - easyrsa:client
- add relation etcd:certificates - easyrsa:client
- add relation kubernetes-master:etcd - etcd:db
- add relation kubernetes-worker:certificates - easyrsa:client
- add relation kubernetes-worker:kube-api-endpoint - kubeapi-load-balancer:website
- add relation kubeapi-load-balancer:certificates - easyrsa:client
- add relation flannel:etcd - etcd:db
- add relation flannel:cni - kubernetes-master:cni
- add relation flannel:cni - kubernetes-worker:cni
- add unit easyrsa/0 to new machine 0
- add unit etcd/0 to new machine 1
- add unit etcd/1 to new machine 2
- add unit etcd/2 to new machine 3
- add unit kubeapi-load-balancer/0 to new machine 4
- add unit kubernetes-master/0 to new machine 5
- add unit kubernetes-worker/0 to new machine 6
- add unit kubernetes-worker/1 to new machine 7
- add unit kubernetes-worker/2 to new machine 8
Deploy of bundle completed.
```

You can check the deployment status using the following command:

```
 watch --color juju status --color
```

Note that this will give you the default bundle for CDK which is made up of 9 machines, flannel networking and no RBAC. This is based on the default bundle found here: [https://jujucharms.com/canonical-kubernetes/](https://jujucharms.com/canonical-kubernetes/).

For a more tailored build with Canal or Calico, you can use the bundle builder: [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes). This will generate a bundle file, which is just a big piece of yaml which describes the configuration for the entire cluster, similar to an Ansible Playbook or Puppet Manifest.

If you have a custom bundle, you would deploy that using a command like this instead:

```
 juju deploy bundle.yaml
```

Eventually the colours will all turn green and your cluster is good to go. To access the cluster, we need to install the kubectl command line client and copy the kubernetes configuration file over for it to use:

```
 # If this does not work, try adding the --classic option on the end.
 snap install kubectl --classic
```

Next we copy over the configuration file:

```
  juju scp kubernetes-master/0:/home/ubuntu/config ~/.kube/config
```

Finally, using kubectl we can check that kubernetes cluster interaction is possible:

```
Kubernetes master is running at https://34.253.164.197:443

Heapster is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kube-dns/proxy
kubernetes-dashboard is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
Grafana is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```

**__Note: There are two other ways CDK would be deployed, either using [Conjure-up](https://tutorials.ubuntu.com/tutorial/install-kubernetes-with-conjure-up#0) or using the graphical juju-as-a-service tool provided at [https://jujucharms.com/](https://jujucharms.com/).__**

## Deploying the F5 Big-IP

F5 Network's BigIP Product is shipped either as a physical or virtual device. They also provide an [AWS AMI](https://aws.amazon.com/marketplace/seller-profile?id=74d946f0-fa54-4d9f-99e8-ff3bd8eb2745) on the AWS Market Place which can be used for testing.

As we've just deployed Kubernetes onto AWS, we will spin-up the load-balancer on that platform as well. The AMI I chose was the F5 Big-IP Virtual Edition - GOOD - Hourly, 25Mbps, v13 (https://aws.amazon.com/marketplace/pp/B079C44MFH?ref=cns_srchrow)[https://aws.amazon.com/marketplace/pp/B079C44MFH?ref=cns_srchrow]. To Launch an instance on AWS, perform the following steps:

- Go to the AWS Console and login: (https://aws.amazon.com)[aws.amazon.com]
- Select the correct region for where you deployed your cluster with Juju, in my case it was eu-west-1
- Go to EC2, make sure you have a private key setup and Hit 'Launch Instance'
- Go to AWS Marketplace, Type F5 BIG-IP and pick a flavor.
- Use the defaults for the AMI, but make sure you pick one with version latest version (13 or higher) and make sure you pick the right SSH keypair.

Once the load balancer has been spun up, we need to change the admin password:

- SSH to the loadbalancer using the SSH key you setup, you can find the public IP address for the machine by looking at the machine in the EC2 GUI:

```
 ssh admin@34.241.93.33
```

- Next we change the password:

```
admin@(ip-172-31-47-72)(cfg-sync Standalone)(Active)(/Common)(tmos)# modify auth user admin password admin
admin@(ip-172-31-47-72)(cfg-sync Standalone)(Active)(/Common)(tmos)# save sys config
Saving running configuration...
  /config/bigip.conf
  /config/bigip_base.conf
  /config/bigip_user.conf
Saving Ethernet mapping...done
```

- Finally, we can now use the web interface. As our device only has one interface, the port 8443 is used to access the web interface. If you have more than one port, it is exposed through 443:

```
  # This should be resolvable now publically, try it in firefox.
  wget https://<your F5 public ip or vip>:8443/ --no-check-certificate
```

![f5 big-ip login](https://raw.githubusercontent.com/CalvinHartwell/canonical-kubernetes-f5-bigip/master/images/login.png "F5 Big-IP Login Screen")

Enter your credentials, login and you should see this screen:

![f5 big-ip gui](https://raw.githubusercontent.com/CalvinHartwell/canonical-kubernetes-f5-bigip/master/images/f5-gui.png "F5 Big-IP GUI")

There appears to be several different versions of the Big-IP appliance on AWS for different bandwidth requirements and price ranges. This example uses the cheapest option but it is possible to sign-up for a (free trial of the virtual edition)[https://f5.com/products/deployment-methods/virtual-editions] if you're running on your own estate or you can use existing physical F5 hardware running Big-IP.

If you deploy your own load balancer the default credentials may be different. Note that the container work-load will have quite privileged access to your loadbalancer so using a model which is running in production is not recommended until you are more familiar with its operation.

### Configuring the F5 Big-IP
### Deploying the F5 Big-IP Workload on CDK
## Conclusion
### Known issues, bugs, caveats
### Useful Links

- [http://clouddocs.f5.com/containers/v2/kubernetes/]
- [https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/](https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Authorization-Mode-and-RBAC](https://github.com/juju-solutions/bundle-canonical-kubernetes/wiki/Authorization-Mode-and-RBAC)
