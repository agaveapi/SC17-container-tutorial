# Tutorial infrastructure

The tutorial infrastructure is comprised of the the following components:

* Jupyter server
* Sandbox development environment
* Continuous integration server
* Repository automation service
* Private Docker registry
* Private Singularity registry
* HPC cluster

You can deploy the infrastructure on the Jetstream cloud using the `agaveplatform/container-tutorial:jetstream` Docker image. It is possible to deploy the infrastructure on other cloud providers with minimal changes to the configuration, but such aternatives have not been tested.

## Architecture

The examples in this tutorial assume the following architecture.:

* Swarm master
* Swarm worker (1 per student)
* Build node (virtual)

The "Swarm master" hosts the Docker Swarm master and orchestrates runtime
deployment of each tutorial user environment. The "Swarm workers" are where
the user sandboxes and notebook servers are run. For simplicity, one Swarm worker is
dedicated to each user. The Swarm master will ensure that two containers are
always running on each user VM: `<username>-jupyter` and `<username>-sandbox`.
The `<username>-jupyter` container runs the user's notebook server. The
`<username>-sandbox` container provides an interactive login node which also
doubles as the private compute and storage system they will interact with
through the Agave Platform. When deploying your own tutorial infrastructure,
persistent volumes will be created per user on a host, but will not persist
once a VM is destroyed.

The "Build node" hosts the services required to provide automated builds of each user's codes, both the Docker and Singularity private registries, image synchronization server, and webhook processing needed to power the examples in this tutorial.


### Slurm HPC cluster (optional)
If you would like to follow the extended examples and run the sample application
on the tutorial HPC cluster, the following systems are needed to deploy the
SLURM described in the tutorial.

* Cluster master
* Cluster node (2 or more)

The "cluster master" serves as the login node to the Slurm cluster we will be
using to run parallel jobs in the second half of the tutorial. It is exposed
as a HPC system in the Agave Platform and is shared as an execution system
with all attendees ahead of time.

The "cluster node" serve as compute system for the Slurm cluster. They are
 only accessible via the Slurm head node and leverage a shared file system
 for convenient data access. Both the cluster master and cluster nodes are
 preconfigured for use in this tutorial as representative HPC systems, similar
 in environment to what you would find in tradition HPC data centers.


## Requirements
### Docker
The provisioning and management of the tutorial services is via a Docker Container. You
will need the following installed on your system to complete the installation.

* Docker 1.13 or greater

## OpenStack
Each of the virtual machines should have the following configuration:

* Ubuntu >=16.04
* m1.medium (2 core / 4G memory)
* 30GB disk

The majority of storage requirements are related to Docker and container
registry hosting.

You will need to use the Agave Platform to both register your VMs as systems,
and share them with the relevant users. You may use [Agave ToGo](https://togo.agaveplatform.org)
for this, or the included registration scripts.

All of your VMs should be on the same logical network and be able to access one
another. You may reference the dev and build nodes by IP address or hostname.

## Agave Platform
Successful completion of the tutorial requires interaction with the Agave Platform. The recommended approach is to use the publicly hosted instance of Agave rather than deploying and configuring an entire instance for this tutorial. The infrastructure examples are all configured to work against the public tenant, so no further work is necessary to get them running.

Deploying an instance of the Agave Platform is beyond the scope of this tutorial. If you choose to do so, please consult the [Agave Deployer](https://github.com/agaveplatform/deployer) project for detailed instructions. Once you have validated your Platform installation, you will need to replace all occurrences of `https://pubic.agaveplatform.org` throughout the tutorials with the URL to your installation and update the `client_key` and `client_secreet` in your Jupyter Hub configuration with a valid set of keys generated by your installation.

# Preparing your environment
## DNS
Throughout the tutorial we use `<username>-jupyter` and `<username>-sandbox` to reference the fully qualified domain names `<username>-jupyter.training.public.agaveplatform.org` and `<username>-sandbox.training.public.agaveplatform.org` respectively. These domains are dynamically created and assigned by the Agave Platform. When running the tutorial infrastructure independently, Agave needs a way to connect to each user's sandbox environment to perform builds, run jobs, and manage data. By default, the tutorial sever will assign a floating ip address to each host VM and use that to register each container as a storage an execution system with Agave. That is entirely sufficient to follow along with the tutorial by yourself on a single host.

If you would like to provide vanity URLs for your own convenience, and you will be the only user, then adding the following entry to your `/etc/hosts` file, replacing the ip address of your VM  is sufficient to accomplish the same thing.

```
129.114.104.18  training.agave
```  

However, if you would like to provide vanity URLs that resolve to the outside world, then read on.

The Agave Platform provides an internal DNS server to resolve vanity URL to runtime environments created by the platform. If you wish to provide similar vanity URLs for each tutorial user, the following steps are required:

1. Assign your swarm master the base hostname of your wildcard subdomain entry by adding your hostname to the host /etc/hosts file.
2. Create a new wildcard A record with a relatively low TTL in your DNS zone file pointing at your Swarm master.  
    ```
    *.training.exmple.com. 300 IN  A 129.114.104.18   ```  
3. Restart the swarm master.

Once Swarm has been restarted, its internal network load balancer will handle the routing to each user sandbox and Jupyter container. The sandbox container will be accessible by on ports 22 (*ssh*, *sftp*), 21 (*scp*), and 3000 (tcp) at the `<username>-sandbox.training.exmple.com` subdomain. The jupyter container will be accessible on ports 80 (*http*) and 443 (*https*) at the `<username>-jupyter.training.exmple.com` subdomain.
