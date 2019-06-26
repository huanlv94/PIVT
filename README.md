# Hyperledger Fabric meets Kubernetes
![Fabric Meets K8S](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/fabric_meets_k8s.png)

* [What is this?](#what-is-this)
* [Who made this?](#who-made-this)
* [License](#License)
* [Requirements](#requirements)
* [Network Architecture](#network-architecture)
* [Go over the samples](#go-over-samples)
  * [Launching the network](#launching-the-network)
  * [Creating channels](#creating-channels)
  * [Installing chaincodes](#installing-chaincodes)
  * [Scaled-up Kafka network](#scaled-up-kafka-network)
  * [Scaled-up Raft network](#scaled-up-raft-network)
* [Configuration](#configuration)
* [TLS](#tls)
* [Backup-Restore](#backup-restore)
  * [Requirements](#backup-restore-requirements)
  * [Flow](#backup-restore-flow)
  * [Backup](#backup)
  * [Restore](#restore)
* [Limitations](#limitations)
* [FAQ and more](#faq-and-more)
* [Conclusion](#conclusion)

## [What is this?](#what-is-this)
This repository contains a couple of Helm charts to:
* Configure and launch the whole HL Fabric network, either:
  * A simple one, one peer per organization and Solo orderer
  * Or scaled up one, multiple peers per organization and Kafka orderer
* Populate the network:
  * Create the channels, join peers to channels, update channels for Anchor peers
  * Install/Instantiate all chaincodes, or some of them, or upgrade them to newer version
* Backup and restore the state of whole network

## [Who made this?](#who-made-this)
This work is a result of collaborative effort between [APG](https://www.apg.nl/en) and 
[Accenture NL](https://www.accenture.com/nl-en). 

We had implemented these Helm charts for our project's needs, and as the results looks very promising, 
decided to share the source code with the HL Fabric community. Hopefully it will fill a large gap!
Special thanks to APG for allowing opening the source code :)

We strongly encourage the HL Fabric community to take ownership of this repository, extend it for
further use cases, use it as a test bed and adapt it to the Fabric provided samples to get rid of endless 
Docker Compose files and Bash scripts. 

## [License](#License)
This work is licensed under the same license with HL Fabric; [Apache License 2.0](LICENSE).

## [Requirements](#requirements)
* A running Kubernetes cluster, Minikube should also work, but not tested
* [HL Fabric binaries](https://hyperledger-fabric.readthedocs.io/en/release-1.4/install.html)
* [Helm](https://github.com/helm/helm/releases/tag/v2.11.0), developed with 2.11, newer 2.xx versions should also work
* [jq](https://stedolan.github.io/jq/download/) 1.5+ and [yq](https://pypi.org/project/yq/) 2.6+
* [Argo](https://github.com/argoproj/argo/blob/master/demo.md), both CLI and Controller
* [Minio](https://github.com/argoproj/argo/blob/master/ARTIFACT_REPO.md), only required for backup/restore flows
* Run all the commands in *fabric-kube* folder

## [Network Architecture](#network-architecture)

### Simple Network Architecture

![Simple Network](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/HL_in_Kube_simple.png)

### Scaled Up Kafka Network Architecture

![Scaled Up Network](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/HL_in_Kube_scaled.png)

### Scaled Up Raft Network Architecture

![Scaled Up Raft Network](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_in_Kube_raft.png)
**Note:** Due to TLS, transparent load balancing is not possible with Raft orderer as of Fabric 1.4.1.

## [Go Over Samples](#go-over-samples)

### [Launching The Network](#launching-the-network)
First install chart dependencies, you need to do this only once:
```
helm repo add kafka http://storage.googleapis.com/kubernetes-charts-incubator
helm dependency update ./hlf-kube/
```
Then create necessary stuff:
```
./init.sh ./samples/simple/ ./samples/chaincode/
```
This script:
* Creates the `Genesis block` using `genesisProfile` defined in 
[network.yaml](fabric-kube/samples/simple/network.yaml) file in the project folder
* Creates crypto material using `cryptogen` based on 
[crypto-config.yaml](fabric-kube/samples/simple/crypto-config.yaml) file in the project folder
* Creates channel artifacts by iterating over `channels` in network.yaml using `configtxgen` 
* Compresses chaincodes into tar archives by iterating over `chaincodes` in network.yaml
* Copies everything created into main chart folder: `hlf-kube` 

Now, we are ready to launch the network:
```
helm install ./hlf-kube --name hlf-kube -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml
```
This chart creates all the above mentioned secrets, pods, services, etc. cross configures them 
and launches the network in unpopulated state.

Wait for all pods are up and running:
```
kubectl get pod --watch
```
In a few seconds, pods will come up:
![Screenshot_pods](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_pods.png)
Congrulations you have a running HL Fabric network in Kubernetes!

### [Creating channels](#creating-channels)

Next lets create channels, join peers to channels and update channels for Anchor peers:
```
helm template channel-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml | argo submit - --watch
```
Wait for the flow to complete, finally you will see something like this:
![Screenshot_channel_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_channel_flow.png)

### [Installing chaincodes](#installing-chaincodes)

Next lets install/instantiate/invoke chaincodes
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml | argo submit - --watch
```
Wait for the flow to complete, finally you will see something like this:
![Screenshot_chaincode_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_chaincode_flow.png)

Install steps may fail even many times, nevermind about it, it's a known [Fabric bug](https://jira.hyperledger.org/browse/FAB-15026), 
the flow will retry it and eventually succeed.

Lets assume you had updated chaincodes and want to upgrade them in the Fabric network. Firt update chaincode `tar` archives:
```
./prepare_chaincodes.sh ./samples/simple/ ./samples/chaincode/
```
Then make sure chaincode ConfigMaps are updated with new chaincode tar archives:
```
helm upgrade hlf-kube ./hlf-kube -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml  
```
Or alternatively you can update chaincode ConfigMaps directly:
```
helm template -f samples/simple/network.yaml -x templates/chaincode-configmap.yaml ./hlf-kube/ | kubectl apply -f -
```

Next invoke chaincode flow again with a bit different settings:
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml -f chaincode-flow/values.upgrade.yaml --set chaincode.version=2.0 | argo submit - --watch
```
All chaincodes are upgraded to version 2.0!
![Screenshot_chaincode_upgade_all](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_chaincode_upgade_all.png)

Lets upgrade only the chaincode named `very-simple` to version 3.0:
```
helm template chaincode-flow/ -f samples/simple/network.yaml -f samples/simple/crypto-config.yaml -f chaincode-flow/values.upgrade.yaml --set chaincode.version=3.0 --set flow.chaincode.include={very-simple} | argo submit - --watch
```
Chaincode `very-simple` is upgarded to version 3.0!
![Screenshot_chaincode_upgade_single](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_chaincode_upgade_single.png)

### [Scaled-up Kafka network](#scaled-up-kafka-network)
Now, lets launch a scaled up network backed by a Kafka cluster.

First tear down everything:
```
argo delete --all
helm delete hlf-kube --purge
```
Wait a bit until all pods are terminated:
```
kubectl  get pod --watch
```
Then create necessary stuff:
```
./init.sh ./samples/scaled-kafka/ ./samples/chaincode/
```
Lets launch our scaled up Fabric network:
```
helm install ./hlf-kube --name hlf-kube -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml -f samples/scaled-kafka/values.yaml
```
Again lets wait for all pods are up and running:
```
kubectl get pod --watch
```
This time, in particular wait for 4 Kafka pods and 3 ZooKeeper pods are running and `ready` count is 1/1. 
Kafka pods may crash and restart a couple of times, this is normal as ZooKeeper pods are not ready yet, 
but eventually they will all come up.

![Screenshot_pods_kafka](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_pods_kafka.png)

Congrulations you have a running scaled up HL Fabric network in Kubernetes, with 3 Orderer nodes backed by a Kafka cluster 
and 2 peers per organization. Your application can use them without even noticing there are 3 Orderer nodes and 2 peers per organization.

Lets create the channels:
```
helm template channel-flow/ -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml | argo submit - --watch
```
And install chaincodes:
```
helm template chaincode-flow/ -f samples/scaled-kafka/network.yaml -f samples/scaled-kafka/crypto-config.yaml | argo submit - --watch
```
### [Scaled-up Raft network](#scaled-up-raft-network)
Now, lets launch a scaled up network based on three Raft orderer nodes spanning two Orderer organizations. This sample also demonstrates how to enable TLS and use actual domain names for peers and orderers instead of internal Kubernetes service names. Enabling TLS globally is mandatory as of Fabric 1.4.1. Hopefully will be resolved [soon](https://jira.hyperledger.org/browse/FAB-15648). 

_For TLS, we need [hostAliases support](https://github.com/argoproj/argo/issues/1265) in Argo workflows and also in Argo CLI, which is implemented but not released yet. You can install Argo controller from Argo repo with the below command. We have built Argo CLI binary from Argo repo for Linux which can be downloaded from [here](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/argo/argo-linux-amd64)._ **Use at your own risk!**

```
kubectl apply -n argo -f https://github.com/argoproj/argo/blob/master/manifests/install.yaml
```

Compare [scaled-raft-tls/configtx.yaml](fabric-kube/samples/scaled-raft-tls/configtx.yaml) with other samples, in particular it uses actual domain names like _peer0.atlantis.com_ instead of internal Kubernetes service names like _hlf-peer--atlantis--peer0_. This is necessary for enabling TLS since otherwise TLS certificates won't match service names.

Also in [network.yaml](fabric-kube/samples/scaled-raft-tls/network.yaml) file, there are two additional settings. As we pass this file to all Helm charts, it's convenient to put these settings into this file.
```
tlsEnabled: true
useActualDomains: true
```

First tear down everything:
```
argo delete --all
helm delete hlf-kube --purge
```
Wait a bit until all pods are terminated:
```
kubectl  get pod --watch
```
Then create necessary stuff:
```
./init.sh ./samples/scaled-raft-tls/ ./samples/chaincode/
```
Lets launch our Raft based Fabric network in _broken_ state:
```
helm install ./hlf-kube --name hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml
```
The pods will start but they cannot communicate to each other since domain names are unknown.

Run this command to collect the host aliases:
```
kubectl get svc -l addToHostAliases=true -o jsonpath='{"hostAliases:\n"}{range..items[*]}- ip: {.spec.clusterIP}{"\n"}  hostnames: [{.metadata.labels.fqdn}]{"\n"}{end}' > samples/scaled-raft-tls/hostAliases.yaml
```

Or this one, which is much convenient:
```
./collect_host_aliases.sh ./samples/scaled-raft-tls/ 
```

Let's check the created hostAliases.yaml file.
```
cat samples/scaled-raft-tls/hostAliases.yaml
```

The output will be something like:
```
hostAliases:
- ip: 10.0.110.93
  hostnames: [orderer0.groeifabriek.nl]
- ip: 10.0.32.65
  hostnames: [orderer1.groeifabriek.nl]
- ip: 10.0.13.191
  hostnames: [orderer0.pivt.nl]
- ip: 10.0.88.5
  hostnames: [peer0.atlantis.com]
- ip: 10.0.88.151
  hostnames: [peer1.atlantis.com]
- ip: 10.0.217.95
  hostnames: [peer10.aptalkarga.tr]
- ip: 10.0.252.19
  hostnames: [peer9.aptalkarga.tr]
- ip: 10.0.64.145
  hostnames: [peer0.nevergreen.nl]
- ip: 10.0.15.9
  hostnames: [peer1.nevergreen.nl]
```
The IPs are internal ClusterIPs of related services. Important point here is, as opposed to pod ClusterIPs, service ClusterIPs are stable, they won't change if service is not deleted and re-created.


Next, let's update the network with this host aliases information. These entries goes into pods' `/etc/hosts` file via Pod [hostAliases](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/) spec.
```
helm upgrade hlf-kube ./hlf-kube -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml  
```

Again lets wait for all pods are up and running:
```
kubectl get pod --watch
```
Congrulations you have a running scaled up HL Fabric network in Kubernetes, with 3 Raft orderer nodes spanning 2 Orderer organizations and 2 peers per organization. But unfortunately, due to TLS, your application cannot use them with transparent load balancing, you need to connect to relevant peer and orderer services separately.

Lets create the channels:
```
helm template channel-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```
And install chaincodes:
```
helm template chaincode-flow/ -f samples/scaled-raft-tls/network.yaml -f samples/scaled-raft-tls/crypto-config.yaml -f samples/scaled-raft-tls/hostAliases.yaml | argo submit - --watch
```

## [Configuration](#configuration)

There are basically 2 configuration files: [crypto-config.yaml](fabric-kube/samples/simple/crypto-config.yaml) 
and [network.yaml](fabric-kube/samples/simple/network.yaml). 

### crypto-config.yaml 
This is Fabric's native configuration for `cryptogen` tool. We use it to define the network architecture. We honour `OrdererOrgs`, 
`PeerOrgs`, `Template.Count` in PeerOrgs (peer count) and even `Template.Start`.

```yaml
OrdererOrgs:
  - Name: Groeifabriek
    Domain: groeifabriek.nl
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Karga
    Domain: aptalkarga.tr
    EnableNodeOUs: true
    Template:
      Start: 9  # we also honour Start 
      Count: 1
    Users:
      Count: 1
```
### network.yaml 
This file defines how network is populated regarding channels and chaincodes.

```yaml
network:
  # only used by init script to create genesis block 
  genesisProfile: OrdererGenesis

  # defines which organizations will join to which channels
  channels:
    - name: common
      # all peers in these organizations will join the channel
      orgs: [Karga, Nevergreen, Atlantis]
    - name: private-karga-atlantis
      # all peers in these organizations will join the channel
      orgs: [Karga, Atlantis]

  # defines which chaincodes will be installed to which organizations
  chaincodes:
    - name: very-simple
      # chaincode will be installed to all peers in these organizations
      orgs: [Karga, Nevergreen, Atlantis]
      # at which channels are we instantiating/upgrading chaincode?
      channels:
      - name: common
        # chaincode will be instantiated/upgraded using the first peer in the first organization
        # chaincode will be invoked on all peers in these organizations
        orgs: [Karga, Nevergreen, Atlantis]
        policy: OR('KargaMSP.member','NevergreenMSP.member','AtlantisMSP.member')
        
    - name: even-simpler
      orgs: [Karga, Atlantis]
      channels:
      - name: private-karga-atlantis
        orgs: [Karga, Atlantis]
        policy: OR('KargaMSP.member','AtlantisMSP.member')
```

For chart specific configuration, please refer to the comments in the relevant [values.yaml](fabric-kube/hlf-kube/values.yaml) files.

## [TLS](#tls)
![TLS](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_in_Kube_TLS.png)

Using TLS is a two step process. We first launch the network in broken state, then collect ClusterIPs of services and attach them to pods as DNS entries using pod [hostAliases](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/) spec.

Important point here is, as opposed to pod ClusterIPs, service ClusterIPs are stable, they won't change if service is not deleted and re-created.

## [Backup-Restore](#backup-restore)

### [Backup Restore Requirements](#backup-restore-requirements)
* Persistence should be enabled in relevant components (Orderer, Peer, CouchDB)
* Configure Argo for some artifact repository. Easiest way is to install [Minio](https://github.com/argoproj/argo/blob/master/ARTIFACT_REPO.md) 
* An Azure Blob Storage account with a container named `hlf-backup` (configurable). 
ATM, backups can only be stored at Azure Blob Storage but it's quite easy to extend backup/restore 
flows for other mediums, like AWS S3. See bottom of [backup-workflow.yaml](fabric-kube/backup-flow/templates/backup-workflow.yaml)

**IMPORTANT:** Backup flow does not backup contents of Kafka cluster, if you are using Kafka orderer you need to 
manually back it up. In particular, Kafka Orderer with some state cannot handle a fresh Kafka installation, see this 
[Jira ticket](https://jira.hyperledger.org/browse/FAB-15541), hopefully Fabric guys will fix this soon.

### [Backup Restore Flow](#backup-restore-flow)
![HL_backup_restore](https://raft-fabric-kube.s3-eu-west-1.amazonaws.com/images/HL_backup_restore.png)

First lets create a persistent network:
```
./init.sh ./samples/simple-persistent/ ./samples/chaincode/
helm install --name hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
```
Again lets wait for all pods are up and running, this may take a bit longer due to provisioning of disks.
```
kubectl  get pod --watch
```
Then populate the network, you know how to do it :)

### Backup

Start backup procedure and wait for pods to be terminated and re-launched with `Rsync` containers.
```
helm upgrade hlf-kube --set backup.enabled=true -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml  ./hlf-kube
kubectl  get pod --watch
```
Then take backup:
```
helm template -f samples/simple-persistent/crypto-config.yaml --set backup.target.azureBlobStorage.accountName=<your account name> --set backup.target.azureBlobStorage.accessKey=<your access key> backup-flow/ | argo submit  -  --watch
```
![Screenshot_backup_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_backup_flow.png)

This will create a folder with default `backup.key` (html formatted date `yyyy-mm-dd`), 
in Azure Blob Storage and hierarchically store backed up contents there.

Finally go back to normal operation:
```
helm upgrade hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```
### [Restore](#restore)

Start restore procedure and wait for pods to be terminated and re-launched with `Rsync` containers.
```
helm upgrade hlf-kube --set restore.enabled=true -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```

Then restore from backup:
```
helm template --set backup.key='<backup key>' -f samples/simple-persistent/crypto-config.yaml --set backup.target.azureBlobStorage.accountName=<your account name> --set backup.target.azureBlobStorage.accessKey=<your access key> restore-flow/  | argo submit  -  --watch
```
![Screenshot_restore_flow](https://s3-eu-west-1.amazonaws.com/raft-fabric-kube/images/Screenshot_restore_flow.png)

Finally go back to normal operation:
```
helm upgrade hlf-kube -f samples/simple-persistent/network.yaml -f samples/simple-persistent/crypto-config.yaml -f samples/simple-persistent/values.yaml ./hlf-kube
kubectl  get pod --watch
```

## [Limitations](#limitations)

### TLS

Transparent load balancing is not possible with TLS as of Fabric 1.4.1. So, instead of `Peer-Org`, `Orderer-Org` or `Orderer-LB` services, you need to connect to individual `Peer` and `Orderer` services.

### Multiple Fabric networks in the same Kubernetes cluster

This is possible but they should be run in different namespaces. We do not use Helm release name in names of components, 
so if multiple instances of Fabric network is running in the same namespace, names will conflict.

## [FAQ and more](#faq-and-more)

Please see [FAQ](FAQ.md) page for further details. Also this [post](https://accenture.github.io/blog/2019/06/25/hl-fabric-meets-kubernetes.html) at Accenture's open source blog provides some additional information like motivation, how it works, benefits regarding NFR's, etc.

## [Conclusion](#conclusion)

So happy BlockChaining in Kubernetes :)

And don't forget the first rule of BlockChain club:

**"Do not use BlockChain unless absolutely necessary!"**

*Hakan Eryargi (r a f t)*
