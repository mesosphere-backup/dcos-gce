###DCOS on Google Compute Engine

This repository contains scripts to configure a DC/OS cluster on Google Compute Engine.

A bootstrap node is required to run the scripts and to bootstrap the DC/OS cluster.

##Bootstrap node configuration

You must create a projection using the google cloud console. The author created a project called trek-treckr

You can create the bootstrap node using the google cloud console. The author used a f1-micro instance running centos with a 50 GB persistent disk in 
zone europe-west1-d.

After creating the micro instance start the instance and run the following from the shell
```bash
> gcloud components update
> sudo yum update
> sudo yum install python-pip
> sudo pip install -U pip
> sudo pip install apache-libcloud docker-py ansible
```

You need to create the rsa public/private keypairs to allow passwordless logins via SSH to the nodes of the DC/OS cluster. This is required by ansible to create the cluster nodes and 
install DC/OS on the nodes.

Run the following to generate the keys
```bash
> ssh-keygen -t rsa -f ~/.ssh/id_rsa -C ajazam
```
Please replace ajazam with your username

Add the rsa public key to your project
```bash
> chmod 400 ~/.ssh/id_rsa
> gcloud compute project-info add-metadata --metadata-from-file sshKeys=~/.ssh/id_rsa.pub
```
Install docker
```bash
> curl -fsSL https://get.docker.com/ | sh
> sudo usermod -aG docker ajazam
```
Please replace ajazam with your username

Start docker
```bash
> sudo service docker start
```
To create and configure the master nodes run
```bash
> ansible-playbook -i hosts install.yml
```
To create and configure the private nodes run
```bash
> ansible-playbook -i hosts add_agent --extra-vars "start_id=0001 end_id=0002 agent_type=private"
```
start_id=0001 and end_id=0002 specify the range of id's that are appended to the hostname "agent" to create unique agent names. If start_is is not specifiec then a default of 0001 is used. 
If the end_id is not specified then a default of 0001 is used.
The values for agent_type are either private or public. If an agent_type is not specified then it is assumed agent_type is private.


To create public nodes type
```bash
> ansibe-playbook ii hosts add_agent --extra-vars "start_id=0003 end_id=0004 agent_type=public"
```
#Configurable parameters

File './hosts'
In an ansible inventory file text wrapped by [] represents a group name and individual entries after the group name represent hosts in that group.
The [[masters]] group contains nodes names and IP addresses for the master nodes. In the supplied file the host name is master0 and the ip address 10.132.0.3 is used. 
You will have to change this for your network.

The [[agents]] group has one entry. It specifies the names of all the agents one can have in the DC/OS cluster. The values specifies that agent0000 to agent9999, a 
total of 10,000 agents are allowed. This really is an artificial because it can easily be changed.

The [[bootstrap]] group has the name of the bootstrap node.

File './group_vars/all'
This contains miscellaneous parameters that will change the behaviour of the installation scripts

>project:
You must change thos this to your project name. Default: trek-trackr

>subnet: 
You must change this for your network. default-6f68d4d6fabcb680

>zone:
You may change this to your preferred zone. Default: europe-west1-d

>master_boot_disk_size:
The size of the master node boot disk. Default 10 GB

>image
The disk image used on the master and agent. Default: /centos-cloud/centos-7-v20160606

>disk_type
The disk type used for the master and agent nodes. Default: pd-standard

>boot_disk_size
The boot disk size used on the master and agent nodes. Default: 10 GB

>master_machine_type
The GCE instance type used for the master nodes. Default: n1-standard-1

>bootstrap_public_ip
The bootstrap nodes public IP. Default: 10.132.0.2

>bootstrap_public_port
The port on the bootstrap node which is used to fetch the dcos installer from each of the master and agent nodes. Default: 8080

>cluster_name
The name of the DC/OS cluster. Default: cluster_name

>agent_machine_type
The GCE instance tpe used for the agent nodes. Default: n1-standard-1

>scopes
Don't change this. Required by the google cloud SDK

>dcos_installer_filename
The filename for the DC/OS installer. Default dcos_generate_config.sh

>dcos_installer_download_path
The location of where the dcos installer is available from dcos.io. Default: https://downloads.dcos.io/dcos/EarlyAccess/{{ dcos_installer_filename }} The value of {{ dcos_installer_file }} is described above.

>login_name
The login name used for accessing each GCE instance. YOU MUST CHANGE THIS. Default: ajazam

>home_directory
The home directory for your logins. Default: /home/{{ login_name }} The value of {{ login_name }} is described above.

>downloads_from_bootstrap
The concurrent downloads of the dcos installer to the cluster of master and agent nodes. Required concurrent downloads can be done but the bootstrap node does not get overwhelmed. You may need to experiment with this to get the best value for your bootstrap node.

>start_id
The number appended to the text agent is used to define the hostname of the first agent. e.g. agent0001. Default: 0001

>end_id:
The number appended to the text agent is used to define the hostname of the last agent. e.g. agent0001. Default: 0001

>dcos_bootstrap_container
Holds the name of the dcos bootstrap container running on the bootstrap node. Default: dcosinstaller

>agent_instance_type
Allows agents to be preemptible. If the value is "MIGRATE" then they are not preemptible. If the value is '"TERMINATE" --preemptible' then the instance is preemptible. Default: "MIGRATE"

>agent_type
Can specify whether an agent is "public" or "private". Default: "private"
