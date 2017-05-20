# Supported configuration parameters
The playbook in the [provision-zookeeper.yml](../provision-zookeeper.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Zookeeper from the [vars/zookeeper.yml](../vars/zookeeper.yml) file. The parameters defined in these files define a reasonable set of defaults for a fairly generic Zookeeper deployment, either to a single node or an ensemble, including defaults for the URL that the Zookeeper distribution should be downloaded from, the directory the distribution should be unpacked into, and the packages that must be installed on the node before the `zookeeper` service can be started.

In addition to the defaults defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file, there are a large number of parameters that can be used to either control the deployment of Zookeeper to the nodes that will make up a ensemble during the `ansible-playbook` run or to configure those Zookeeper nodes once the installation is complete. In this section, we summarize all of these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure our Zookeeper nodes once Zookeeper has been installed locally.

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the Zookeeper distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`zookeeper_url`**: the URL that the Zookeeper distribution should be downloaded from
* **`local_zk_file`**: the local file (to the Ansible host) containing the Zookeeper distribution; this file will be uploaded to the target hosts and unpacked into the `zookeeper_dir` (see below) during the playbook run
* **`zookeeper_version`**: the version of Zookeeper that should be downloaded; used to switch versions when the distribution is downloaded using the default `zookeeper_url`, which is defined in the [vars/zookeeper.yml](../vars/zookeeper.yml) file
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`local_vars_file`**: used to define the location of a *local variables file* (see the discussion of this topic, below); this file is a YAML file containing definitions for any of the configuration parameters that are described in this section and is more than likely a file that will be created to manage the process of creating a specific ensemble. Storing the settings for a given ensemble in such a file makes it easy to guarantee that all of the nodes in that ensemble are configured consistently
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying Zookeeper to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process

## Parameters used during the deployment process
These parameters are used to control the deployment process itself, defining things like where to unpack the distribution into and what packages need to be installed as part of the deployment process:

* **`zookeeper_dir`**: the path to the directory that the Apache Zookeeer distribution should be unpacked into; defaults to `/opt/zookeeper` if unspecified
* **`zookeeper_user`**: the name of the user/group that should be used when unpacking the distribution and when running the `zookeeper` service
* **`zookeeper_package_list`**: the list of packages that should be installed on the Zookeeper nodes; typically this parameter is left unchanged from the default (which installs the OpenJDK packages needed to run Zookeeper), but if it is modified the default, OpenJDK packages must be included as part of this list or an error will result when attempting to start the `zookeeper` service

## Parameters used to configure the Zookeeper nodes
These parameters are used configure the Zookeeper nodes themselves during a playbook run, defining things like the interface that Zookeeper should listen on for requests, the directory where Zookeper should store its data and logs, and the JVM flags that should be used when running Zookeeper:

* **`data_iface`**: the name of the interface that the Zookeeper nodes should use when talking with each other and handling user requests. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`iface_description_array`**: this parameter can be used in place of the `data_iface ` parameter described above; provides users with the ability to specify a description for this interface rather than identifying the interface by name (more on this, below)
* **`zookeeper_data_dir`**: the name of the directory where Zookeeper should use to store its data; defaults to `/var/lib` if unspecified. If necessary, this directory will be created as part of the playbook run
* **`zookeeper_data_log_dir`**: the name of the directory where Zookeeper should store it's transaction logs; defaults to `/var/log` if unspecified. If necessary, this directory will be created as part of the playbook run
* **`jvm_flags`**: used to set any JVM flags that should be used as part of the Zookeeper configuration; defaults to `-Xmx1g` if inspecified

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the names of the interface that should be passed into the playbook using the `data_iface` parameter that was described, above. In those situations, the playbook in this repository provides an alternative; specifying those interfaces using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact. This fact can then be used later in the playbook to correctly configure the Zookeeper nodes to talk to each other and listen on the proper interface for user requests.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for the `data_iface` network, otherwise the default value of `eth0` will be used for this fact (and the playbook run may result in nodes that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).
