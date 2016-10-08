#DCOS on Google Compute Engine

This repository contains scripts to configure a DC/OS cluster on Google Compute Engine.

A bootstrap node is required to run the scripts and to bootstrap the DC/OS cluster.

**PLEASE READ THE ENTIRE DOCUMENT. YOU MUST MAKE CHANGES FOR THE SCRIPTS TO WORK IN YOUR GCE ENVIRONMENT.**

##Bootstrap node configuration

**YOU MUST CREATE A PROJECT** using the google cloud console. The author created a project called trek-treckr

You can create the bootstrap node using the google cloud console. The author used a n1-standard-1 instance running centos 7 with a 10 GB persistent disk in 
zone europe-west1-c. The bootstrap node must have "Allow full access to all Cloud APIs" in the Identity and API access section. Also enable Block project-wide SSH keys in the SSH Keys section. Create the instance.

After creating the boot instance run the following from the shell
```bash
gcloud components update
sudo yum update
sudo yum install epel-release
sudo yum install python-pip
sudo pip install -U pip
sudo pip install apache-libcloud
sudo pip install 'docker-py==1.9.0'
sudo yum install git ansible
```

You need to create the rsa public/private keypairs to allow passwordless logins via SSH to the nodes of the DC/OS cluster. This is required by ansible to create the cluster nodes and install DC/OS on the nodes.

Run the following to generate the keys
```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -C ajazam
```
**PLEASE REPLACE ajazam** with your username. Do not eneter a password when prompted

Make a backup copy of id_rsa.pub.

Open rsa pub key
```bash
sudo vi ~/.ssh/id_rsa.pub
```

shows

```bash
ssh-rsa abcdefghijklmaasnsknsdjfsdfjs;dfj;sdflkjsd ajazam
```
Prefix your username, followed by a colon, to the above line. Also replace ajazam at the end with your username.

```bash
ajazam:ssh-rsa abcdefghijklmaasnsknsdjfsdfjs;dfj;sdflkjsd ajazam
```
save contents of id_rsa.pub. **Please replace the ajazam with your username**. 



Add the rsa public key to your project
```bash
chmod 400 ~/.ssh/id_rsa
gcloud compute project-info add-metadata --metadata-from-file sshKeys=~/.ssh/id_rsa.pub
```

Disable selinux for docker to work

make the following change to /etc/selinux/config

```bash
SELINUX=disabled
```

reboot host

To install docker add the yum repo

```bash
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

install the docker package
```bash
sudo yum install docker-engine
```

Add following changes to /usr/lib/systemd/system/docker.service

```bash
ExecStart=/usr/bin/dockerd --storage-driver=overlay
```

reload systemd

```bash
sudo systemctl daemon-reload
```

Start docker
```bash
sudo systemctl start docker.service
```

Verify if docker works

```bash
sudo docker run hello-world
```


download the dcos-gce scripts
```bash
git clone https://github.com/ajazam/dcos-gce
```

change directory
```bash
cd dcos-gce
```

Please make appropriate changes to dcos_gce/group_vars/all. You need to review project, subnet, login_name, bootstrap_public_ip & zone

insert following into ~/.ansible.cfg to stop host key checking
```bash
[defaults]
host_key_checking = False

[paramiko_connection]
record_host_keys = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null
```

Ensure the IP address for master0 in ./hosts is the next consecutive IP from bootstrap_public_ip.

To create and configure the master nodes run
```bash
ansible-playbook -i hosts install.yml
```
To create and configure the private nodes run
```bash
ansible-playbook -i hosts add_agents.yml --extra-vars "start_id=0001 end_id=0002 agent_type=private"
```
start_id=0001 and end_id=0002 specify the range of id's that are appended to the hostname "agent" to create unique agent names. If start_id is not specified then a default of 0001 is used. 
If the end_id is not specified then a default of 0001 is used.
The values for agent_type are either private or public. If an agent_type is not specified then it is assumed agent_type is private.

To create public nodes type
```bash
ansible-playbook -i hosts add_agents.yml --extra-vars "start_id=0003 end_id=0004 agent_type=public"
```
##Configurable parameters

File './hosts' is an ansible inventory file. Text wrapped by [] represents a group name and individual entries after the group name represent hosts in that group.
The [masters] group contains node names and IP addresses for the master nodes. In the supplied file the host name is master0 and the ip address 10.132.0.3 is assigned to 
master0. **YOU MUST CHANGE** the IP address for master0 for your network. You can create multiple entries e.g. master1, master2 etc. Each node must have a unique IP address.

The [agents] group has one entry. It specifies the names of all the agents one can have in the DC/OS cluster. The value specifies that agent0000 to agent9999, a 
total of 10,000 agents are allowed. This really is an artificial limit because it can easily be changed.

The [bootstrap] group has the name of the bootstrap node.

File './group_vars/all' contains miscellaneous parameters that will change the behaviour of the installation scripts. The parameters are split into two groups. Group 1 parameters must be changed to reflect your environment. Group 2 parameters can optionally be changed to change the behaviour of the scripts.

###Group 1 parameters YOU MUST CHANGE for your environment

```text
project
```
Your project id. Default: trek-trackr

```text
subnet
```
Your network. Default: default-6f68d4d6fabcb680

```text
login_name
```
The login name used for accessing each GCE instance. Default: ajazam

```text
bootstrap_public_ip
```
The bootstrap nodes public IP. Default: 10.132.0.2

```text
zone
```
You may change this to your preferred zone. Default: europe-west1-d


###Group 2 parameters which optionally change the behaviour of the installation scripts

```text
master_boot_disk_size:
```
The size of the master node boot disk. Default 10 GB

```text
master_machine_type
```
The GCE instance type used for the master nodes. Default: n1-standard-1

```text
master_boot_disk_type
```
The master boot disk type. Default: pd-standard

```text
agent_boot_disk_size
```
The size of the agent boot disk. Default 10 GB

```text
agent_machine_type
```
The GCE instance type used for the agent nodes. Default: n1-standard-1

```text
agent_boot_disk_type
```
The agent boot disk type. Default: pd-standard

```text
agent_instance_type
```
Allows agents to be preemptible. If the value is "MIGRATE" then they are not preemptible. If the value is '"TERMINATE" --preemptible' then the instance is preemptible. Default: "MIGRATE"

```text
agent_type
```
Can specify whether an agent is "public" or "private". Default: "private"

```text
start_id
```
The number appended to the text *agent* is used to define the hostname of the first agent. e.g. agent0001. Intermediate agents between start_id and end_id will be created if required. Default: 0001

```text
end_id
```
The number appended to the text *agent* is used to define the hostname of the last agent. e.g. agent0001. Intermediate agents between start_id and end_id will be created if required. Default: 0001


```text
gcloudbin
```
The location of the gcloudbin binary. Default: /usr/local/bin/gcloud

```text
image
```
The disk image used on the master and agent. Default: /centos-cloud/centos-7-v20160606

```text
bootstrap_public_port
```
The port on the bootstrap node which is used to fetch the dcos installer from each of the master and agent nodes. Default: 8080

```text
cluster_name
```
The name of the DC/OS cluster. Default: cluster_name

```text
scopes
```
Don't change this. Required by the google cloud SDK

```text
dcos_installer_filename
```
The filename for the DC/OS installer. Default dcos_generate_config.sh

```text
dcos_installer_download_path
```
The location of where the dcos installer is available from dcos.io. Default: https://downloads.dcos.io/dcos/EarlyAccess/{{ dcos_installer_filename }} The value of {{ dcos_installer_file }} is described above.

```text
home_directory
```
The home directory for your logins. Default: /home/{{ login_name }} The value of {{ login_name }} is described above.

```text
downloads_from_bootstrap
```
The concurrent downloads of the dcos installer to the cluster of master and agent nodes. You may need to experiment with this to get the best performance. The performance will be a function of the machine type used for the bootstrap node. Default: 2

```text
dcos_bootstrap_container
```
Holds the name of the dcos bootstrap container running on the bootstrap node. Default: dcosinstaller




