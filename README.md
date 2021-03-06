# DCI OpenShift Agent

`dci-openshift-agent` provides Red Hat OpenShift Container Platform (RHOCP) in Red Hat Distributed CI service.

## Table of Contents

- [Requirements](#requirements)
- [Installation of DCI Jumpbox](#installation-of-DCI-Jumpbox)
- [Configuration](#configuration)
- [Create your DCI account on distributed-ci.io](#create-your-dci-account-on-distributed-ciio)
- [License](#license)
- [Contact](#contact)

## Requirements

### Systems requirements

`DCI OpenShift Agent` needs a dedicated system to act as a `controller node`. It is identified as the `DCI Jumpbox` in this document. This system will be added to a standard OCP topology by being connected to the OCP `baremetal network`. The `DCI OpenShift Agent` will drive the RHOCP installation workflow from there.

Therefore, the simplest working setup must be composed of at least **5** systems (1 system for DCI and 4 systems to match OCP minimum requirements).

Please follow the [OpenShift Baremetal Deploy Guide (a.k.a. `openshift-kni`)](https://openshift-kni.github.io/baremetal-deploy/) for how to properly configure the OCP networks and systems.

Choose either `4.3` or `4.4` and follow steps 1 to 4 to configure the networks and install RHEL 8 on the provisioning host.

Steps from 5 on will be handled by the `dci-openshift-agent`.

As mentioned before, the **DCI Jumpbox** is NOT part of the RHOCP cluster. It is only dedicated to download `RHOCP` artifacts from `DCI` public infrastructure and to schedule the RHOCP cluster deployment across all systems under test (1x OpenShift Provisioning node and several OCP nodes).

The **OpenShift Provisioning node** is used by the OpenShift installer to provision the OpenShift cluster nodes.

The 3 remaining systems will run the freshly installed OCP Cluster. “3” is the minimum required number of nodes to run RHOCP but it can be more if you need to.

#### Jumpbox requirements

The `Jumpbox` can be a physical server or a virtual machine.
In any case, it must:

- Be running the latest stable RHEL release (**7.6 or higher**) and registered via RHSM.
- Have at least 160GB of free space available in `/var`
- Have access to Internet
- Be able to connect the following Web urls:
  - DCI API, https://api.distributed-ci.io
  - DCI Packages, https://packages.distributed-ci.io
  - DCI Repository, https://repo.distributed-ci.io
  - EPEL, https://dl.fedoraproject.org/pub/epel/
  - QUAY.IO, https://quay.io
- Have a static internal (network lab) IP
- Be able to reach all systems under test (SUT) using (mandatory, but not limited to):
  - SSH
  - IPMI
  - Serial-Over-LAN or other remote consoles (details & software to be provided by the partner)
- Be reachable by the Systems Under Test by using:
  - DHCP
  - PXE
  - HTTP/HTTPS

#### Systems under test

`Systems under test` will be **installed** through DCI workflow with each job and form the new “fresh” RHOCP cluster.

All files on these systems are NOT persistent between each `dci-openshift-agent` job as the RHOCP cluster is reinstalled at each time. Therefore, every expected customization and tests have to be automated from the DCI Jumpbox (by using hooks) and will therefore be applied after each deployment (More info at #Configuration and #Usage).

#### Optional

- We strongly advise the partners to provide the Red Hat DCI team with access to their jumpbox. This way, Red Hat engineers can help with initial setup and troubleshooting.

- We suggest to run the `full virtualized` provided example first to understand how the `dci-openshift-agent` works before going to production with a real lab.

## Installation of DCI Jumpbox

Before proceeding you should have set up your networks and systems according to the baremetal-deploy doc that was referenced above.

Provision the Jumphost with RHEL7.

The `dci-openshift-agent` is packaged and available as a RPM file.
However,`dci-release` and `epel-release` must be installed first:

```bash
# yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum -y install https://packages.distributed-ci.io/dci-release.el7.noarch.rpm
# subscription-manager repos --enable=rhel-7-server-extras-rpms
# subscription-manager repos --enable=rhel-7-server-optional-rpms
# yum -y install dci-openshift-agent
```

## Configuration

There are two configuration files for `dci-openshift-agent`: `/etc/dci-openshift-agent/dcirc.sh` and `/etc/dci-openshift-agent/hosts`.

- `/etc/dci-openshift-agent/dcirc.sh`

Note: The initial copy of `dcirc.sh` is shipped as `/etc/dci-rhel-agent/dcirc.sh.dist`.

Copy this to `/etc/dci-openshift-agent/dcirc.sh` to get started, then replace inline some values with your own credentials.

From the web the [DCI web dashboard](https://www.distributed-ci.io), the partner team administrator has to create a `Remote CI` in the DCI web dashboard.

Copy the relative credential and paste it locally on the Jumpbox to `/etc/dci-openshift-agent/dcirc.sh`.

This file should be edited once:

```bash
#!/usr/bin/env bash
DCI_CS_URL="https://api.distributed-ci.io/"
DCI_CLIENT_ID=remoteci/<remoteci_id>
DCI_API_SECRET=<remoteci_api_secret>
export DCI_CLIENT_ID
export DCI_API_SECRET
export DCI_CS_URL
```

- `/etc/dci-openshift-agent/hosts`

This file is an Ansible inventory file (format is `.ini`). It includes the configuration for the `dci-openshift-agent` job and the inventory for the masters, workers (if any) and the provisionhost.
The possible values are:

| Variable     | Required | Type          | Description                                          |
| ------------ | -------- | ------------- | ---------------------------------------------------- |
| topic        | True     | String        | Name of the topic. It can be `OCP-4.3` or `OCP-4.4`. |
| cluster_name | True     | String        | RHCP cluster name.                                   |
| base_domain  | True     | String        | Domain                                               |
| ironic_nodes | True     | String (JSON) | tbd                                                  |

Example:

```console
[all:vars]
dci_topic=OCP-4.3
prov_nic=eno1
pub_nic=eno2
domain=example.com
cluster=dciokd
dnsvip=192.168.10.41
cluster_pro_if=eno1
masters_prov_nic=eno1
prov_ip=172.22.0.3
dir="{{ ansible_user_dir }}/clusterconfigs"
ipv6_enabled=false
cache_enabled=True

# Master nodes
[masters]
master-0 name=master-0 role=master ipmi_user=ADMIN ipmi_password=ADMIN ipmi_address=ipmi-master-0.dciokd.example.com provision_mac=ac:1f:6b:7d:dd:44 hardware_profile=default
master-1 name=master-1 role=master ipmi_user=ADMIN ipmi_password=ADMIN ipmi_address=ipmi-master-1.dciokd.example.com provision_mac=ac:1f:6b:6c:ff:ee hardware_profile=default
master-2 name=master-2 role=master ipmi_user=ADMIN ipmi_password=ADMIN ipmi_address=ipmi-master-2.dciokd.example.com provision_mac=ac:1f:6b:29:33:3c hardware_profile=default

# Worker nodes
[workers]

# Provision Host
[provisioner]
provisionhost ansible_user=kni prov_nic=eno1 pub_nic=ens3 ansible_ssh_common_args="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
```

### Copying the ssh key to your provisionhost

```console
# su - dci-openshift-agent
% ssh-keygen
% ssh-copy-id kni@provisionhost
```

### Starting the DCI OCP Agent

Now that you have configured the `DCI OpenShift Agent`, you can start the service.

Please note that the service is a systemd `Type=oneshot`. This means that if you need to run a DCI job periodically, you have to configure a `systemd timer` or a `crontab`.

```
$ systemctl start dci-rhel-agent
```

If you need to run the `dci-openshift-agent` manually in foreground, you can use this command line:

```
# su - dci-openshift-agent
$ cd /usr/share/dci-openshift-agent && source /etc/dci-openshift-agent/dcirc.sh && /usr/bin/ansible-playbook -vv /usr/share/dci-openshift-agent/dci-openshift-agent.yml
```

### dci-openshift-agent workflow

_Step 1 :_ State “New job”

- Prepare the `Jumpbox`: `/plays/configure.yml`
- Download OpenShift from DCI: `/plays/fetch_bits.yml`

_Step 2 :_ State “Pre-run”

- Deploy infrastructure: `/hooks/pre-run.yml`

_Step 3 :_ State “Running”

- Configure OpenShift nodes: `/hooks/configure.yml`
- Start OpenShift installer: `/hooks/running.yml`

_Step 4 :_ State “Post-run”

- Start DCI tests (This is empty for now): `/plays/dci-tests.yml`
- Start user specific tests: `/hooks/user-tests.yml`

_Step 5 :_ State “Success”

- Launch additional tasks when the job is successful: /hooks/success.yml

_Exit playbooks:_
The 2 following playbooks are executed sequentially at any step that fail:

- Teardown: /hooks/teardown.yml
- Failure: /plays/failure.yml

_All playbooks located in directory `/etc/dci-openshift-agent/hooks/` are empty by default and should be customized by the user._

## Create your DCI account on distributed-ci.io

Every user needs to create his personal account by connecting to `https://www.distributed-ci.io` by using a Red Hat SSO account.

The account will be created in the DCI database at the first connection with the SSO account. For now, there is no reliable way to know your team automatically. Please contact the DCI team when this step has been reached, to be assigned in the correct organisation.

## License

Apache License, Version 2.0 (see [LICENSE](LICENSE) file)

## Contact

Email: Distributed-CI Team <distributed-ci@redhat.com>
IRC: #distributed-ci on Freenode
