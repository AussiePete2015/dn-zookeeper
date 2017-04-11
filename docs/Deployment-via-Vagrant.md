# Deployment via Vagrant
A [Vagrantfile](../Vagrantfile) is included in this repository that can be used to deploy Zookeeper locally (to one or more VMs hosted under [VirtualBox](https://www.virtualbox.org/)) using [Vagrant](https://www.vagrantup.com/).  From the top-level directory of this repository a command like the following will (by default) deploy Zookeeper to a single CentOS 7 virtual machine running under VirtualBox:

```bash
$ vagrant -z="192.168.34.12" up
```

Note that the `-z, --zookeeper-list` flag must be used to pass an IP address (or a comma-separated list of IP addresses) into the [Vagrantfile](../Vagrantfile). In the example shown above, we are performing a single-node deployment of Zookeeper. When we are performing a multi-node deployment, then we simply pass in multiple addresses using the `-z, --zookeeper-list` command-line argument, as in this example:

```bash
vagrant -z="192.168.34.18,192.168.34.19,192.168.34.20" up
```

This command will create a three-node Zookeeper ensemble. It should be noted here that if any of the IP addresses in the Zookeeper list is not actually an IP address, then an error will be thrown.

In terms of how it all works, the [Vagrantfile](../Vagrantfile) is written in such a way that the following sequence of events occurs when the `vagrant ... up` command shown above is run:

1. All of the virtual machines in the ensemble (the addresses in the `-z, --zookeeper-list`) are created
1. Zookeeper is deployed to each of those nodes and they are configured to talk to each other as members of the ensemble that is being built
1. The `zookeeper` service is started on all of the nodes that were just provisioned

As was the case with the second deployment scenario described [here](Deployment-Scenarios.md), once the nodes have been created and the playbook run is complete, we can run the `zkServer.sh status` command on each of the nodes in our ensemble. This will show whether Zookeeper is running or not on each node and whether it is acting as a leader or follower:

```bash
$ for host in 192.168.34.18 192.168.34.19 192.168.34.20; do
    vagrant -z="$host" ssh -c "/opt/zookeeper/bin/zkServer.sh status"
done
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
Connection to 127.0.0.1 closed.
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: follower
Connection to 127.0.0.1 closed.
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Mode: leader
Connection to 127.0.0.1 closed.
$
```

So, to recap, by using a single `vagrant ... up` command we were able to quickly spin up an ensemble consisting of of three Zookeeper nodes, and a similar `vagrant ... up` command could be used to build a similar Zookeeper ensemble consisting of any number of nodes.

## Separating instance creation from provisioning
While the `vagrant up` commands that are shown above can be used to easily deploy Zookeeper to a single node or to build a Zookeeper ensemble consisting of multiple nodes, the [Vagrantfile](../Vagrantfile) included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's [site.yml](../site.yml) file.

To create a set of virtual machines that we plan on using to build a Zookeeper ensemble without provisioning Zookeeper to those machines, simply run a command similar to the following:

```bash
$ vagrant -z="192.168.34.18,192.168.34.19,192.168.34.20" up --no-provision
```

This will create a set of three virtual machines with the appropriate IP addresses ("192.168.34.18", "192.168.34.19", and "192.168.34.20"), but will skip the process of provisioning those VMs with an instance of Zookeeper.

To provision the machines that were created above and configure those machines as a Zookeeper ensemble, we simply need to run a command like the following:

```bash
$ vagrant "192.168.34.18,192.168.34.19,192.168.34.20" provision
```

That command will attach to the named instances and run the playbook in this repository's [site.yml](../site.yml) file on those node, resulting in a Zookeeper ensemble consisting of the nodes that were created in the `vagrant ... up --no-provision` command that was shown, above.

## Additional vagrant deployment options
While the commands shown above will install Zookeeper with a reasonable, default configuration from a standard location, there are additional command-line parameters that can be used to override the default values that are embedded in the [vars/zookeeper.yml](../vars/zookeeper.yml) file. Here is a complete list of the command-line flags that can be included in any `vagrant ... up` or `vagrant ... provision` command:

* **`-z, --zookeeper-list`**: the Zookeeper address list; this is the list of nodes that will be created and provisioned, either by a single `vagrant ... up` command or by a `vagrant ... up --no-provision` command followed by a `vagrant ... provision` command; this command-line flag **must** be provided for almost every `vagrant` command supported by the [Vagrantfile](../Vagrantfile) in this repository
* **`-u, --url`**: the URL that the Apache Zookeeer distribution should be downloaded from. This flag can be useful in situations where there is limited (or no) internet access; in this situation the user may wish to download the distribution from a local web server rather than from the standard Zookeeper download site
* **`-l, --local-zk-file`**: the path to a local (to the Ansible host) file containing the Apache Zookeeer distribution; this file will be uploaded to the target machines and unpacked into the `-p, --path` directory (see below). This flag can be useful in situations where there is limited (or no) internet access; in this situation the user may wish to upload the distribution from the Ansible host rather than downloading it from the standard Zookeeper download site
* **`-p, --path`**: the path to the directory that the Zookeeper distribution should be unpacked into during the provisioning process; this defaults to the `/opt/zookeeper` directory if not specified
* **`-d, --data`**: the path to the directory where Zookeeper will store it's data; this defaults to `/var/lib` if not specified
* **`-y, --yum-url`**: the local YUM repository URL that should be used when installing packages during the node provisioning process. This can be useful when installing Zookeeper onto CentOS-based VMs in situations where there is limited (or no) internet access; in this situation the user might want to install packages from a local YUM repository instead of the standard CentOS mirrors. It should be noted here that this parameter is not used when installing Zookeeper on RHEL-based VMs; in such VMs this option will be silently ignored if set
* **`-f, --local-vars-file`**: the *local variables file* that should be used when deploying the ensemble. A local variables file can be used to maintain the configuration parameter definitions that should be used for a given Zookeeper ensemble deployment, and values in this file will override any values that are either embedded in the [vars/zookeeper.yml](../vars/zookeeper.yml) file as defaults or passed into the `ansible-playbook` command as extra variables
* **`-c, --clear-proxy-settings`**: if set, this command-line flag will cause the playbook to clear any proxy settings that may have been set on the machines being provisioned in a previous ansible-playbook run. This is useful for situations where an HTTP proxy might have been set incorrectly and, in a subsequent playbook run, we need to clear those settings before attempting to install any packages or download the Zookeeper distribution without an HTTP proxy

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Zookeeper distribution from a local web server, rather than downloading it from the main Apache distribution site, configure Zookeeper to store it's data in the `/data` directory, and install packages from the CentOS mirror on the eb server at `http://192.168.34.254/centos` when provisioning the machines created by the `vagrant ... up --no-provision` command shown above:

```bash
$ vagrant -z="192.168.34.18,192.168.34.19,192.168.34.20" -d="/data" \
    -l=/Users/tjmcs/tmp/local-gzipped-tarfiles/apache-zookeeper/zookeeper-3.4.9.tar.gz \
    -y='http://192.168.34.254/centos' provision
```

While the list of command-line options defined above may not cover all of the customizations that user might want to make when performing production Zookeeper deployments, our hope is that the list shown above is more than sufficient for deploying Zookeeper ensembles using Vagrant for local testing purposes.
