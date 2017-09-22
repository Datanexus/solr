# Dynamic vs. static inventory
The first decision to make when using the playbook in this repository to deploy Solr/Fusion is whether you would like to manage the inventory for your deployments dynamically or statically. Since the static inventory use case is the simplest, we'll start our discussion there. Then we'll move on to a discussion of how to control the deployment of Solr/Fusion in both AWS and OpenStack environments, where the list of nodes that should be created and targeted by a given `ansible-playbook` run is constructed dynamically from the input configuration file.

## Managing deployments with static inventory files
Let's assume that we are planning the deployment of a three-node Solr/Fusion cluster using the playbook in this repository and we are planning on using a [static inventory file](https://docs.ansible.com/ansible/intro_inventory.html) to control that deployment. In this example (and others like it in this document) we'll assume that we're going to construct a single static inventory file (an INI-style file containing the list of hosts that are targeted by any given playbook run), and then pass that file into the `ansible-playbook` command using the `-i, --inventory-file` command-line flag. The assumption in this use case is that the target nodes are already up and running, so the playbook run will only deploy and configure Solr/Fusion on those nodes (in the dynamic inventory use cases the nodes are created if necessary prior to running the plays the deploy and configure Solr/Fusion on those nodes).

In our discussion of the various deployment scenarios supported by this playbook, we show that deployments of a multi-node Solr/Fusion cluster actually require an associated (assumed to be external) Zookeeper ensemble. For purposes of this discussion, we will focus only on the inventory information associated with the Solr/Fusion nodes, not on the inventory information associated with the Zookeeper ensemble, but the inventory information for nodes that make up the Zookeeper ensemble must also be provided so that Ansible can connect to those nodes and gather the meta-data that it needs to configure the members of the Solr/Fusion cluster correctly.

So, returning to our example, the inventory file that we use to deploy our three-node Solr/Fusion cluster might look something like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.28 ansible_ssh_host=192.168.34.28 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.29 ansible_ssh_host=192.168.34.29 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.30 ansible_ssh_host=192.168.34.30 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[solr]
192.168.34.28
192.168.34.29
192.168.34.30

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

$
```

As you can see, this static inventory file consists of a list of the hosts that make up the inventory for our deployment and a corresponding pair of host groups (the `solr` and `zookeeper` host groups). For each host in this file, a list of the parameters that Ansible will need to connect to that host is provided (INI-file style) as a set of `name=value` pairs. In this example, we've defined the following values for each of the entries in our static inventory file:

* **`ansible_ssh_host`**: the hostname/address that Ansible should use to connect to that host; if not specified, the same hostname/address listed at the start of the line for that entry for that host will be used (in this example we are using IP addresses). This parameter can be important when there are multiple network interfaces on each host and only one of them (an admin network, for example) allows for SSH access
* **`ansible_ssh_port`**: the port that Ansible should use when connecting to the host via SSH; if not specified, Ansible will attempt to connect using the default SSH port (port 22)
* **`ansible_ssh_user`**: the username that Ansible should use when connecting to the host via SSH; if not specified, then the value of the global `ansible_user` variable will be used (this variable can be set as an extra variable in the `ansible-playbook` command if the same username is used for all hosts targeted by the playbook run)
* **`ansible_ssh_private_key_file`**: the private key file that should be used when connecting to the host via SSH; if not specified, then the value of the global `ansible_ssh_private_key_file` variable will be used instead (this variable can be set as an extra variable in the `ansible-playbook` command if the same private key is used for all hosts targeted by the playbook run)

With this static inventory file built, it's a relatively simple matter to deploy our Solr/Fusion cluster using a single `ansible-playbook` command. Examples of these commands are shown [here](Deployment-Scenarios.md).

## Managing deployments using a dynamic inventory
For both AWS and OpenStack environments, the playbook in this repository supports the use of a [dynamic inventory](https://docs.ansible.com/ansible/intro_dynamic_inventory.html) when deploying a Solr/Fusion cluster. This is accomplished by making use of the [build-app-host-groups](../roles/build-app-host-groups) role, which builds the host group containing the target nodes for a Solr/Fusion deployment (and the host group containing the nodes in the associated Zookeeper ensemble) dynamically, based on a set of tags that are passed into the playbook run. This provides an attractive alternative to building static inventory files to control the deployment process in these environments, since the meta-data needed to determine which nodes should be targeted by a given playbook run is readily available in the framework itself.

The process of constructing a Solr/Fusion cluster in an AWS or OpenStack environment starts out by determining if the target nodes already exist or if those target nodes need to be created in that environment. This task is accomplished using the [aws](../roles/aws) and [osp](../roles/osp) roles, which search for the target nodes in the targeted environment based on the tags that are passed into the playbook run. Specifically, this role looks for machines that match based on the following tags (which are assumed to be passed into the playbook run, either as extra variables or by defining them in the configuration file that is used to control the playbook run):

* **tenant**: the tenant that will be using the Solr/Fusion cluster being deployed. As an example, you might use a value of `labs` for this tag for Solr/Fusion clusters that the Labs department in your organization will be using
* **project**: this tag will be given the value of the project that will be using the Solr/Fusion cluster that is being deployed; for example we might use the value `projectx` for this tag for clusters that will be used for ProjectX
* **dataflow**: this tag is optional, and is used to identify the various clusters/ensembles (Cassandra, Zookeeper, Kafka, Solr, etc.) that make up a given data flow. To link these applications together, it is necessary to define a unique `dataflow` value for each of the ensembles/clusters that make up that data flow. If a value for this tag is not defined, then a default value of `none` will be used when searching for matching VMs in an AWS or OpenStack environment (or when tagging the VMs created during the provisioning process if no matching nodes are found).
* **domain**: this tag is used to separate out the various domains that might be defined by the project team to control their deployments. For example, there might be separate clusters built for `development`, `test`, `preprod`, and `production`. Each of these environments would be tagged appropriately based on their intended use.
* **cluster**: this tag is optional, and is only necessary if the intent is to deploy more than one Solr/Fusion cluster for the same tenant, project, data flow, and domain. In that case, it is necessary to define a unique `cluster` value for each of the clusters that are are being provisioned using the [provision-solr.yml](../provision-solr.yml) playbook. If a value for this tag is not defined, then a default value of `a` will be used when searching for matching VMs in an AWS or OpenStack environment (or when tagging the VMs created during the provisioning process if no matching nodes are found).

In addition to these input tags, the following tags are assigned to the machines being created or provisioned with Solr/Fusion during the playbook run:

* **application**: used to identify the application that is being deployed to a given virtual machine; in this playbook only nodes that are tagged with the application `solr` are targeted by default (and although the application name is something you could customize, we recommend against doing so)
* **role**: the role is used in some of our playbooks to separate out the nodes that have a different role in that cluster from the other roles in the cluster; for example in a Cassandra cluster some of the nodes are seed nodes, so they are tagged with a `Role` tag of `seed`; in a Solr/Fusion cluster there are no such roles, so all of the machines will be tagged with a `Role` tag of `none`; there is no need to specify a `role` value when running the [provision-solr.yml](../provision-solr.yml) playbook, since the default value of `none` is the value that we want to use for all nodes in the cluster we are provisioning

These input values are used to search for a matching set of nodes in the target environment. If a matching set of nodes is found (based on the corresponding `Tenant`, `Project`, `Dataflow`, `Domain`, `Cluster`, `Application`, and `Role` tags), then those machines will be assumed to be the target of the [provision-solr.yml](../provision-solr.yml) playbook run. If no matching nodes are found, then the [aws](../roles/aws) or [osp](../roles/osp) role will create the nodes needed for the playbook run in the target enviornment, and tag each of those instances appropriately based on the roles and node counts that are defined in the input `node_map` parameter (more on this below) before the playbook run continues.

Once the appropriate Solr/Fusion host group has been constructed, the playbook continues the deployment process (following the same path that is followed for the static inventory use case that was described previously).

It should be noted here that this playbook assumes that an associated Zookeeper ensemble has already been deployed that can be associated with the Solr/Fusion cluster that the playbook run is deploying. In the dynamic inventory use case, the nodes that make up this Zookeeper ensemble must be tagged with the same `Tenant`, `Project`, `Dataflow`, `Domain`, and `Cluster` tag values that are used to build the associated Solr/Fusion cluster. Obviously, these Zookeeper instances would have been tagged with an `Application` tag of `zookeeper`, but if the remaining tags do not match then the playbook will not be able to find the associated Zookeeper ensemble and the Solr/Fusion cluster nodes will fail to start.

### Defining a new ensemble in the dynamic inventory use case
As was mentioned previously, there are two scenarios that are supported when controlling the deployment of a new Solr/Fusion cluster to an AWS or OpenStack environment using a dynamic inventory. In the first scenario, the machines that make up the ensemble have already been provisioned and tagged appropriately (either manually or through some other automated mechanism that is not defined here). In that scenario, the [aws](../roles/aws) or [osp](../roles/osp) role will find that a set of nodes that match the input `tenant`, `project`, `domain`, `cluster`, `application`, and `role` tags already exists in the target environment, and a new Solr/Fusion cluster will be built using those matching nodes. However, if a matching set of nodes does not already exist in the target environment, then a mechanism must be provided during the playbook run to define the ensemble that should be created (so that the machines that are needed for the defined cluster can be instantiated and tagged appropriately during the playbook run).

In all of our playbooks, the cluster that the user wants to create can be defined through the use of a `node_map` variable that is assumed to be provided as an input to the playbook run. As is the case with any of our playbooks, a reasonable default value is provided for this `node_map` variable in the [vars/solr.yml](../vars/solr.yml) file:

```yaml
node_map:
  - { application: solr, count: 3 }
```

Since the the [vars/solr.yml](../vars/solr.yml) file is automatically included in the plays that make up the the [provision-solr.yml](../provision-solr.yml) playbook, this is the value that will be used by default if no other value is defined for this variable. As you can quite clearly see, this default value will result in the deployment of a three-node Solr/Fusion cluster by default.

That being said, this value can be overridden at runtime quite easily, either by defining a different value for this variable in an extra variables or by defining a different value for his variable in the configuration file that is used to control the playbook run.
