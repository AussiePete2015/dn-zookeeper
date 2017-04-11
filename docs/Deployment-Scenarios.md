# Example deployment scenarios

There are a three basic deployment scenarios that are supported by this playbook. In the first two (shown below) we'll walk through the deployment of Zookeeper to a single node and the deployment of a multi-node Zookeeper ensemble using a static inventory file. Finally, in the third scenario we will show how the same multi-node Zookeeper cluster deployment shown in the second scenario could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: deploying Zookeeper to a single node
While this is the simplest of the deployment scenarios that are supported by this playbook, it is more than likely that deployment of Zookeeper to a single node is really only only useful for very simple test environments. Nevertheless, we will start our discussion with this deployment scenario since it is the simplest.

If we want to deploy Zookeeper to a single node with the IP address "192.168.34.12", we could simply create a very simple inventory file that looks something like the following:

```bash
$ cat single-node-inventory

192.168.34.12 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/test_node_private_key'

$ 
```

Note that in this example inventory file the `ansible_ssh_host` and `ansible_ssh_port` will take their default values since they aren't specified for our host in this very simple static inventory file. Once we've built our static inventory file, we can then deploy Zookeeper to our single node by running an `ansible-playbook` command that looks something like this:

```bash
$ ansible-playbook -i single-node-inventory -e "{ host_inventory: ['192.168.34.12'] }" site.yml
```

This will download the Apache Zookeeper distribution file from the default download server defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file, unpack that gzipped tarfile into the `/opt/apache-zookeeper` directory on that host, and configure that node as a single-node Zookeeper deployment, using the default configuration parameters that are defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file.

## Scenario #2: deploying a multi-node Zookeeper ensemble
If you are using this playbook to deploy a multi-node Zookeeper ensemble, then the configuration only slightly more complicated. The Zookeeper ensemble consists of a set of nodes that are configured to work together so that if any one node is lost the ensemble can continue to function.

Let's assume that we are deploying Zookeeper to an ensemble made up of three nodes and, furthermore, let's assume that we're going to be using a static inventory file to control this deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.18 ansible_ssh_host= 192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host= 192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host= 192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

$
```

To deploy Zookeeper to the three nodes in our static inventory file and configure those nodes to work together as an ensemble, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.18', '192.168.34.19', '192.168.34.20'], \
      cloud: vagrant, zookeeper_iface: eth0, \
      zookeeper_url: 'apache-zookeeper/zookeeper-3.4.9.tar.gz', \
      yum_repo_url: 'http://192.168.34.254/centos', zookeeper_data_dir: '/data' \
    }" site.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
cloud: vagrant
zookeeper_iface: eth0
zookeeper_url: 'apache-zookeeper/zookeeper-3.4.9.tar.gz'
yum_repo_url: 'http://192.168.34.254/centos'
zookeeper_data_dir: '/data'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-cluster-deployment-params.yml`, the resulting command would look somethin like this:

```bash
$ ansible-playbook -i test-cluster-inventory -e "{ \
      host_inventory: ['192.168.34.18', '192.168.34.19', '192.168.34.20'], \
      local_vars_file: 'test-cluster-deployment-params.yml' \
    }" site.yml
```

Once that playbook run is complete, we can run the `zkServer.sh status` command on each of the nodes in our ensemble. This will show whether Zookeeper is running or not on each node and whether it is acting as a leader or follower:

```bash
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.18 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.19 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
$ ssh -i keys/zk_cluster_private_key cloud-user@192.168.34.20 "/opt/zookeeper/bin/zkServer.sh status"
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: leader
$ 
```

This `ansible-playbook` command deployed Zookeeper to all three of our nodes and configured them as a single ensemble using a static inventory file to control the deployment.

## Scenario #3: deploying a Zookeeper ensemble via dynamic inventory
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Zookeeper ensemble to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be configuring as an ensemble with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will assign an `Application` tag of `zookeeper`
* Once all of the nodes that will make up our cluster have been tagged appropriately, we can run `ansible-playbook` command similar to the first `ansible-playbook` command shown in the previous scenario; this will deploy Zookeeper to the nodes that make up our ensemble.

In terms of what the commands look like, lets assume for this example that we've tagged the nodes that we are going to use to build our Zookeeper ensemble with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: zookeeper

The `ansible-playbook` command used to deploy Zookeeper to target nodes in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_zookeeper:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        application: zookeeper, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', ansible_user: cloud-user, \
        zookeeper_iface: eth0, zookeeper_data_dir: '/data' \
    }" site.yml
```

In an AWS environment, the `ansible-playbook` command looks quite similar:

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_zookeeper:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        application: zookeeper, cloud: osp, tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', ansible_user: cloud-user, \
        zookeeper_iface: eth0, zookeeper_data_dir: '/data' \
    }" site.yml
```

As you can see, it's basically the same commands that was shown for the OpenStack use case; they only differ in terms of the name of the inventory script passed in using the `-i, --inventory-file` command-line argument, the value passed in for `Cloud` tag (and the value for the associated `cloud` variable), and the prefix used when specifying the tags that should be matched in the `host_inventory` value (`tag_` instead of `meta-`). In both cases the result would be a set of nodes deployed as a Zookeeper ensemble, with the number of nodes in the ensemble determined (completely) by the tags that were assigned to them.

