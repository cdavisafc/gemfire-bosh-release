# Gemfire BOSH release

The BOSH Gemfire deployment establishes Gemfire clusters into a virtualized environment.  BOSH takes care of provisioning virtual machines (via interface to vCenter or equivalent for AWS or OpenStack), installing dependencies (like Java) and the Gemfire binaries, and could, as well, install grid configurations and grid-resident code (more on the latter in a moment).

In the BOSH deployment manifest the characteristics of the Gemfire cluster are specified (i.e. number of locators, number of cache servers, number of CPUs and size of Ram (etc.) for the machines on which those components will be run, etc.) and the BOSH deployment will take care of provisioning the servers, installing any desired infrastructure software and installing the Gemfire cluster. 

This project contains a BOSH release for the Gemfire product from Pivotal.  This should be considered a reference or sample BOSH deployment as it 1) does a very simple deployment of a Gemfire cluster and 2) incudes the Gemfire tutorial social networking application running in the Gemfire cluster.

## Anatomy of a BOSH Release
The bosh-gemfire-release contains the following pieces:

### blobs

Three blobs are referenced from this release: a JDK, the Gemfire server and the tutorial code that will be loaded into the Gemfire cache server.  Only the last of these things is found in this repository as the first two are licensed. There are various ways that blobs can be provided to BOSH, the simplest is by having them included in the blobs directory.  The installation instructions below cover both how the JDK and Gemfire binaries should be added to the directory structure and how they are in turn packaged using bosh commands.

### src

BOSH also allows source code to be included as a part of a release and for the bosh-gemfire-release this includes only a single utilities script.  This script is taken from the base Cloud Foundry BOSH release. No other source code is used in the basic BOSH Gemfire release.

### packages

BOSH packages encapsulate all of the portions of a component that will be installed onto provisioned virtual machines.  More than one package can be installed onto a virtual machine; this is controlled by other portions of the BOSH release and the BOSH deployment manifest.  A package lists the blobs and src code that is included in the package; at packaging time these are found in the respective blob and src portions of the BOSH release. Four packages are included in the basic BOSH Gemfire release:

* common: This package includes a utilities script that is helpful for any type of bosh job and refers to the file from the src portion of the bosh release.
* gemfire_jvm: This package includes the java jdk and refers to a blob from the bosh release.
* gemfireserver: This package includes the Gemfire binaries and refers to the blob in the bosh release. Note that the binaries are the same for nodes that act as Gemfire locators and those that are Gemfire cache servers.  How the binaries are run for each of these types of servers is driven by the release jobs.
* gemfire-config: This package primarily consists of the jar file for the Gemfire tutorial code that will be installed into the Gemfire cacheserver.

### jobs

BOSH jobs define the processes that are started on a BOSH provisioned virtual machine.  Two jobs are defined for this BOSH Gemfire cluster deployment:

* gemfire locator: This job runs the locator of the Gemfire server, taking in a list of server address and port pairs as input parameters.  These input parameters are set in the BOSH deployment manifest (properties are fully described below).  Thejobs/gemfirelocator/templates/gemfirelocator_ctldefines the process that is started for the Gemfire locator.  Both the startup and shutdown are done via the gfsh command-line tool.
* gemfire server: This job runs the cache server Gemfire server, taking in the list of locator address and port pairs, and the cache server port as input parameters.  These input parameters are set in the BOSH deployment manifest (properties are fully described below).Thejobs/gemfireserver/templates/gemfireserver_ctldefines the process that is started for the Gemfire server.  Both the startup and shutdown are done via the gfsh command-line tool.

## Deployment Manifest

Two sample manifests are included in this repository; they are generally equivalent, and are simply targeting two different environments. Either one may be used as a starting point for you to generate your own manifest.

There are two release-specific portions of the deployment manifest:

1. Definition of the jobs: The deployment manifest specifies the jobs that are to be deployed. For the basic Gemfire BOSH deployment there are two jobs, the gemfirelocator and the gemfireserver. Each job declaration defines the number of instances of the job as well as certain other attributes such as the IP addresses for the virtual machines created for a job.  Note that each job is deployed to its own virtual machine (but a job can contain multiple packages and can launch any number of processes).  The following shows the declaration of two jobs, one for two locators and and another for two cache servers:  
```
jobs:
- name: gemfirelocator
  template: gemfirelocator
  instances: 2
  resource_pool: infrastructure1
  persistent_disk: 16384
  networks:
  - name: gf1
    static_ips:
    - 10.244.0.102
	- 10.244.0.106

- name: gemfireserver
  template: gemfireserver
  instances: 2
  resource_pool: infrastructure1
  persistent_disk: 16384
  networks:
  - name: gf1
    static_ips:
    - 10.244.0.110
    - 10.244.0.114
```
   Note that each job entry refers to a resource_pool. Resource pools define the characteristics of the compute resource that the job will be deployed to.  
   
1. Properties: As described above, the jobs defined in the basic Gemfire BOSH deployment launch the Gemfire locators and cache servers with certain parameters such as the location server address.  The following are the supported properties for the basic Gemfire BOSH release:   
```
properties:
  gemfire:
    locator:
      addresses: 10.244.0.102[55221], 10.244.0.106[55221]
      archivewd: true
    cacheserver:
      port: 55001
      archivewd: true
    license: xxxx

  networks:
    cluster: gf1
```

** The "archivewd" property controls whether the working directory for a locator or cache server is preserved when the node is restarted; a value of true saves the working directory, a value of false overwrites it.  If BOSH detects that a server process has failed it will restart that process. To ensure that a locator or cache server with stale or corrupt state is not joined into the cluster, the process is started "fresh", that is, with a blank initial state.  Gemfire is designed for exactly this type of scenario, allowing "blank" nodes to be added when needed, and having those nodes eventually fully incorporated into the cluster by having load moved onto them incrementally.  Archiving the working directories, which include Gemfire logs, can be very helpful in diagnosing problems.
** While the above example shows two locators listening on the same port, each IP address may have associated with it a different port. There must be at least one locator address/port pair specified.
** All cache servers are configured to listen on the same port.


## Installation Instructions

The installation of Gemfire via this BOSH release generally follows the same steps that any BOSH release does; you need to create, upload and deploy the release.  Because some of the portions of this installation are not open source, however, these pieces need to be provided to BOSH out of band of this repository.  These details are described below.

1. Clone the repsoitory

We will refer to the directory into which you cloned this project as "GEMFIRE_RELEASE_HOME"

1. Add licensed blobs

Create the following two directories:

```
GEMFIRE_RELEASE_HOME/blobs/gemfire
GEMFIRE_RELEASE_HOME/blobs/java
```

Place the Gemfire zip file into the ```GEMFIRE_RELEASE_HOME/blobs/gemfire``` directory, and the java jdk tar.gz file into the ```GEMFIRE_RELEASE_HOME/blobs/java``` directory.

The Gemfire and Java packages reference these binaries by filename so depending on your exact releases you may need to update the spec and packaging files to reflect those changes.  Please look at the following files:

1. Create the BOSH release

From the ```GEMFIRE_RELEASE_HOME``` directory type the following command:

```
bosh create release --force
```

1. Upload the release

Note that it is assumed that you have already targeted your BOSH director.  Now type the following command:

```
bosh upload release
```

1. Update the deployment manifest

Your updates should cover:

* configuring to your IaaS specifics
* designating the number of locators and cache servers you would like (in the jobs section)
* setting the list of locator address and port pairs, the cache server port and archivewd options (in the properties section)

1. Deploy the release

First you must point to your deployment manifest with the command

```
bosh deployment <path-to-deployment-manifest>
```

Then have BOSH do the deployment with the following command

```
bosh deploy
```

If at any time you wish to change the number of locators or increase the number of cache servers you may do so by changing those values in the deployment manifest as described above and simply running the "bosh deploy" step again.

## Limitations

* There is no scale down functionality, except that I suppose you could scale down your grid by one node at a time and allow things to normalize in the process.
