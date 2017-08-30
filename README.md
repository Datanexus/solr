# solr
Playbooks/Roles used to deploy [LucidWorks Fusion](https://lucidworks.com/fusion/) (Solr)

# Installation
To install Solr using the [provision-solr.yml](provision-solr.yml) playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone https://github.com/Datanexus/solr
```

That command will pull down the repository, which includes the [provision-solr.yml](provision-solr.yml) playbook and all of its dependencies.

The only other requirements for using the playbook in this repository are a relatively recent (v2.x) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to deploy Fusion
The [provision-solr.yml](provision-solr.yml) file at the top-level of this repository supports the both single-node Fusion deployments and the deployment of multi-node Fusion clusters. The process of deploying Fusion to these nodes will vary, depending on whether you are managing your inventory dynamically or statically (more on this topic [here](docs/Dynamic-vs-Static-Inventory.md)), whether you are performing a single-node deployment or are deploying a Fusion cluster, and where you are downloading the packages and dependencies from that are needed to run Fusion on those nodes.

We discuss the various deployment scenarios supported by this playbook in [this document](docs/Deployment-Scenarios.md). With no changes to the the contents of repository (more on this below), the [provision-solr.yml](provision-solr.yml) playbook can be used (out of the box, so to speak) to deploy a three node Solr cluster to an AWS environment using a command that looks something like this:

```bash
$ AWS_PROFILE=datanexus_west ./provision-solr.yml
```

Note that the `AWS_PROFILE` environment variable value shown above will likely need to change to reflect the profiles configured in your local AWS credentials file.

The command used to deploy the same three node Solr cluster to an OSP environment is similar, but additional configuration parameters must be passed into the playbook (above and beyond those used for an AWS deployment). The easiest way to do this is to pass in a customized configuration file as an extra variable on the command-line of your Ansible playbook run:

```bash
$ ./provision-solr.yml -e '{ config_file: "./config-osp.yml" }'
```

here, the `./config-osp.yml` file is assumed to be a configuration file that you have created in the current working directory that contains all of the additional parameters needed for an OSP deployment. For example, you might create a file that looks something like this:

```yaml
$ cat ./config-osp.yml
---
# cloud type and VM tags
cloud: osp
tenant: datanexus
project: demo
domain: production
# configuration (internal) and accessible (external) networks
internal_uuid: 849196dc-12e9-4d50-ac52-2e753d03d13f
internal_subnet: 10.10.0.0/24
external_uuid: 92321cab-1a9c-49d6-99a1-62eedbbc2171
external_subnet: 10.10.3.0/24
# the name of the float_pool that the floating IPs assigned to
# the external subnet should be pulled from
float_pool: external
# username used to access the system via SSH
user: cloud-user
# a few configuration options related to the OpenStack
# environment we are targeting
region: region1
zone: az1
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 'prod.small'
image: 'a36db534-5f9d-4a9f-bdd4-8726cc613b4e'
# the data volume size and the block pool to use when creating that data volume
data_volume: 100
block_pool: prod
# the HTTP proxy-related parameters
http_proxy: http://proxy.datanexus.corp:8080
proxy_certs_file: eg-certs.tar.gz
no_proxy: datanexus.corp
```

As you can see, there are a number of additional parameters needed for an OSP deployment (the `internal_uuid`, `external_uuid`, `float_pool`, `block_pool`, and `zone` parameters) and some of the default values from the [vars/solr.yml](../vars/solr.yml) file and the default configuration file (the [config.yml](../config.yml) file) are overridden by the values defined in this example configuration file (the `cloud`, `tenant`, `project`, `domain`, `internal_subnet`, `external_subnet`, `user`, `type`, and `region` values). There are also a few variables defined in this configuration file that are not defined in either the [vars/solr.yml](../vars/solr.yml) file or the default configuration file used when deploying to an AWS environment (the `data_volume`, `image`, `http_proxy`, `proxy_certs_file`, and `no_proxy` parameters). All of the values shown in this example file are for demonstration purposes only, and the correct values to use for these parameters will change from environment to environment. Check with the your local OSP administrator to determine the correct values to use for these parameters in your OSP environment.

## Controlling the configuration
As was mentioned briefly in the prevision section, this repository includes default values for a number of parameters in the [vars/solr.yml](vars/solr.yml) file and the default configuration file (the [config.yml](config.yml) file) that make it possible to perform deployments of a multi-node Solr cluster, out of the box, with few (if any) changes necessary. If you are not happy with the default configuration defined in these two files, there are a number of ways that you can customize the configuration used for your deployment, and which method you use is entirely up to you:

* You can edit the [vars/solr.yml](vars/solr.yml) file and/or [config.yml](config.yml) file to modify the default values that are defined in those files or define additional configuration parameters
* You can pass in values as extra variables on the command-line of the Ansible playbook run to override those defined in the [vars/solr.yml](vars/solr.yml) file; values defined in the default configuration file cannot be overridden in this manner since that file is loaded last by the plays in the playbook (so values defined in a configuration file override any values defined elsewhere)
* You can setup a custom *configuration file* on the local filesystem of the Ansible host that contains the values for the parameters you wish to set or customize, then pass the location of that file into your playbook command as an extra variable

We have provided a summary of the configuration parameters that can be set (using any of these three methods) during a playbook run [here](docs/Supported-Config-Params.md). Overall, we have found the last option to be the easiest and most flexible of those three options because:

* It avoids modifying files that are being tracked under version control in the main, [solr](https://github.com/Datanexus/solr) repository (the first option); making such changes will, more than likely, lead to conflicts at a later date when these files are modified in the main [solr](https://github.com/Datanexus/solr) repository in a way that is inconsistent with the values that you have set locally (in your clone of this repository).
* It lets you maintain your preferred configuration for any given Solr deployment in the form of a configuration file, which you can easily maintain (along with the configuration files used for other deployments you have made) under version control in a separate repository
* It provides a record of the configuration of any given deployment, which is in direct contrast to the second option (where the configuration parameters for any given deployment are passed in on the command-line as extra variables)

That being said, the second option may be useful for some deployment scenarios (a one-off deployment of a local test environment, for example), so it remains a viable option for some users. Overall, we would recommend against trying to maintain your preferred ensemble configuration using the values defined in the [vars/solr.yml](vars/solr.yml) file or the default configuration file (the [config.yml](config.yml) file).

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully). Furthermore, it is assumed that you are interested in deploying a relatively recent version of Fusion using this playbook (the current default is v3.0.0 but any v3.x release should do).
