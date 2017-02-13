# dn-zookeeper
Playbooks/Roles used to deploy Zookeeper

# Installation
To install zookeeper using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:
```bash
$ git clone --recursive https://github.com/Datanexus/dn-zookeeper
```
That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Using this role to deploy Zookeeper
The `site.yml` file at the top-level of this repository pulls in a set of default values for the parameters that are needed to deploy an Apache Zookeeper distribution from the `vars/zookeeper.yml` file.  The contents of that file currently look like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
#
# Defaults that are necessary for all deployments of
# zookeeper
---
application: zookeeper
# the directory where the Zookeeper logs will be written
zookeeper_log_dir: /var/lib
# the interface Zookeeper should listen on when running
zookeeper_iface: eth0
# the user that the zookeeper service should run under
zookeeper_user: zookeeper
# the following parameters are only used when provisioning an instance
# of the apache distribution, but are uncommented here (regardless) to
# provide reasonable default values when provisioning via Vagrant (where
# the distribution being provisioned may be different from the default)
zookeeper_version: "3.4.9"
zookeeper_url: "https://www.apache.org/dist/zookeeper/zookeeper-{{zookeeper_version}}/zookeeper-{{zookeeper_version}}.tar.gz"
zookeeper_dir: "/opt/zookeeper"
# these parameters are used for both confluent and apache distributions
zookeeper_package_list: ["java-1.8.0-openjdk", "java-1.8.0-openjdk-devel"]
# used to add extra repositories to the list of repositories in the
# `local.repo` file that is deployed to a node when we're adding a
# local repository to use in the provisioning process
#local_repository_url: http://{{yum_repository}}/local.repo
#local_repository_extra_keys: http://{{yum_repository}}/local-zeys.json
# used to install zookeeper from the RPM files in a local directory (if it exists)
local_zookeeper_package_path: ""
```

This default configuration defines default values for all of the parameters needed to deploy an instance of Apache Zookeeper to a node, including defining reasonable defaults for the network interface the Zookeeper instance should listen ("eth0" by default).  To deploy Apache Zookeeper to a node the IP address "192.168.34.8" using the role in this repository, one could simply run a command that looks like this:

```bash
$ export ZOOKEEPER_ADDR="192.168.34.8"
$ ansible-playbook -i "${ZOOKEEPER_ADDR}," -e "{host_inventory: ['${ZOOKEEPER_ADDR}']}" site.yml
```

It should be noted here that any of the variables shown in the `vars/zookeeper.yml` file can be overridden on the command-line using the `--extra-vars` command-line flag.  As is the case with any other ansible-playbook run, the values defined on the command-line in this manner will override values defined in any of the `vars` files that are included in the playbook.  Keep in mind, however, that while this mechanism provides a simple method for overriding the underlying values defined in those `vars` files at runtime (without requiring that the underlying `vars` file be modified), the resulting configuration may be harder to track or reproduce over time (in a production deployment, for example) since the values that were set are not maintained under revision control.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Zookeeper host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
A Vagrantfile is included in this repository that can be used to deploy zookeeper to a VM using the `vagrant` command.  From the top-level directory of this repostory a command like the following will (by default) deploy Apache Zookeeper to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):

```bash
$ vagrant -z="192.168.34.8" up
```

Note that the `-z` (or the corresponding `--zookeeper-addr`) flag must be used to pass an IP address into the Vagrantfile, and this IP address will be used as the IP address of the zookeeper server that is created by the vagrant command shown above.

If the Vagrantfile in this repository is being used to deploy the Apache Zookeeper distribution to the specified node, then the command-line options shown above are the only variables that need to be defined.  However, there are two additional parameters that *may* be defined:  the `zookeeper_dir`, and `zookeeper_url` parameters.  As was mentioned earlier, default values for these two parameters are defined in the `vars/zookeeper.yml` file, and this file is pulled into the playbook contained in the `site.yml` file used by the Vagrantfile for provisioning.  As such, the Vagrantfile in this repository can be used (out of the box, so to speak) to deploy the Apache Zookeeper distribution to a node, without requiring editing of either the Vagrantfile or the `vars/zookeeper.yml` file.

## Additional vagrant deployment options
While the `vagrant up` command that is shown above can be used to easily deploy Apache Zookeeper to a node, the Vagrantfile included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's `site.yml` file. To create a virtual machine without provisioning it, simply run a command that looks something like this:

```bash
$ vagrant -z="192.168.34.8" up --no-provision
```

This will create a virtual machine with the appropriate IP address ("192.168.34.8"), but will skip the process of provisioning that VM with a Zookeeper distribution using the playbook in the `site.yml` file.  To provision that machine with a Confluent Zookeeper instance, you would simply run the following command:

```bash
$ vagrant -z="192.168.34.8" provision
```

That command will attach to the named instance (the VM at "192.168.34.8") and run the playbook in this repository's `site.yml` file on that node (resulting in the deployment of an instance of the Confluent Zookeeper distribution to that node).

It should also be noted here that while the commands shown above will install Apache Zookeeper with a reasonable default configuration from a standard location, there are two additional command-line parameters that can be used to override the default values that are embedded in the Vagrantfile that is included as part of this distribution when we are deploying an instance of the Apache Zookeeper distribution:  the `-u` (or corresponding `--url`) flag and the `-p` (or corresponding `--path`) flag.  The `-u` flag can be used to override the default URL that is used to download the Apache Zookeeper distribution (the default URL points back to the main Apache Zookeeper distribution site), while the `-p` flag can be used to override the default path (`/opt/zookeeper`) that that the Apache Zookeeper gzipped tarfile is unpacked into during the provisioning process.

As an example of how these options might be used, the following command will download the gzipped tarfile containing the Apache Zookeeper distribution from a local web server, rather than downloading it from the main Apache Zookeeper distribution site, when provisioning the VM with an IP address of `192.168.34.8` with an instance of the Apache Zookeeper distribution:

```bash
$ vagrant -z="192.168.34.8" -u="https://10.0.2.2/dist/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz" provision
```

Obviously, this option could prove to be quite useful in situations were we are deploying the distribution from a datacenter environment (where access to the internet may be restricted, or even unavailable).