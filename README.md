# KYPO-Deploy
Here I will be storing all the files, commands and sources for the KYPO CRP platform deployment
*For a personal project*

The sources I'm getting all my information from:
- https://docs.crp.kypo.muni.cz/
- https://computingforgeeks.com/openstack-deployment-on-ubuntu-with-devstack/
- https://www.spacesecurity.info/install-kypo-cyber-range-platform-on-openstack/
- For deployment: https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment#preparing-the-deployment-environment

All the commands I will be performing:

# Installing openstack using devstack
* I will be using the spacesecurity tutorial, linked at the beggining.
```
sudo apt update
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

sudo su - stack
sudo apt -y install git
git clone https://git.openstack.org/openstack-dev/devstack

git clone https://github.com/xcWhy/KYPO-Deploy.git

cd devstack

*Create a local.conf file in this direcotry and copy the contents from my repo file "local_test.conf"*

nano local.conf

sudo ufw disable
./stack.sh
```

please please please stack T-T

if it stacks... YIPPE
now we can access the openstack dashboard through http://<host_ip>/dashboard

We successfully downloaded openstack cloud on our machine! :D
Now starts the hard part... the openstack depoloyment...

# Installing openstack using kolla-ansible (preffered)
I will be using these turotials on how to:
  - set up the virtual machine : https://youtu.be/W8e8F6_og4c?si=aReMMJuwQxMB1VZJ
  - install openstack using kolla-ansible (YT) : https://youtu.be/TEza4k3MUQs?si=V-zB9sYBtLhv6BsK
  - install openstack using kolla-ansible : https://medium.com/@mukulahmed/deploying-openstack-all-in-one-on-virtualbox-with-kolla-ansible-82cdc23145aa

```
sudo su
nano /etc/netplan/50-cloud-init.yaml

network:
    version: 2
    renderer: networkd
    ethernets:
        enp0s3:
            addresses:
              - 192.168.34.102/25 #machine address
            nameservers:
              addresses: [8.8.8.8]
            routes:
              - to: default
                via:  192.168.34.126 #network default gateaway
        enp0s8:
            dhcp4: false

netplan apply
ip a

apt update && sudo apt -y upgrade
apt autoremove

apt install apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install docker-ce docker-ce-cli containerd.io

systemctl start docker
systemctl enable docker
docker --version

apt-get install -y git python3-dev libffi-dev gcc libssl-dev pkg-config libdbus-1-dev build-essential cmake libglib2.0-dev mariadb-server

apt install -y python3-venv

mkdir openstack
cd openstack

python3 -m venv .
source bin/activate

pip install --upgrade pip
pip install setuptools docker dbus-python
pip install 'ansible-core>=2.16,<2.17.99'
pip install git+https://opendev.org/openstack/kolla-ansible@master

mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla

cp share/kolla-ansible/ansible/inventory/all-in-one .
cp -r  share/kolla-ansible/etc_examples/kolla/* /etc/kolla

kolla-ansible install-deps
kolla-genpwd

nano /etc/kolla/globals.yml

kolla_base_distro: "ubuntu"
kolla_internal_vip_address: "< unused IP from network_interface IP range>"
network_interface: "enp0s3"
neutron_external_interface: "enp0s8"
nova_compute_virt_type: "qemu"

kolla-ansible  bootstrap-servers -i ./all-in-one
kolla-ansible  prechecks -i ./all-in-one
kolla-ansible  deploy -i ./all-in-one (40 min download ;-;)
kolla-ansible post-deploy -i ./all-in-one

pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master

cat /etc/kolla/clouds.yml (passwords for the openstack GUI)

```

# Openstack Deployment (installed with devstack)

Firstly we should download the **ADMIN** openrc file from the dashboard - which means we need to switch to the admin project firstly...

`sudo cp /home/eli/Downloads/admin-openrc.sh admin-openrc.sh`

we need to source this file in the directory so in our case we need to copy the contents from the admin rd file to a file that we will create in the directory.
Additionally we can comment the parts where it will prompt us for a password, so it doesn't require it all the time, like this:

#echo "Please enter your OpenStack Password for project $OS_PROJECT_NAME as user $OS_USERNAME: "
#read -sr OS_PASSWORD_INPUT
#export OS_PASSWORD=$OS_PASSWORD_INPUT
export OS_PASSWORD='xxxxxxxxxxx'

then `source admin-openrc.sh`
```
cp admin-openrc.sh keystonerc_admin
source keystonerc_admin
```

then we can check if everything is functional:
```openstack service list```
```openstack orchestration service list```
- if we don't get an error and a box of services shows up, everything is fine.

**Im starting to use this source:** https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment#preparing-the-deployment-environment

then we obtain application-credentials!
- we need them to be unrestricted
- i will give them all of the rules - admin, uesr, etc

we install Terraform:
`snap install terraform --classic`

we clone the deployment remo:
`git clone https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment.git`

then we go onto the "Deployment of OpenStack base resources"
https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment/-/blob/master/BASE.md

since we have sourced the admin rc file before in the currect shell, I will write this command so they dont overlap: `unset "${!OS_@}"`

then we source the application credentials

then we go to:
`cd tf-openstack-base`

in my case the output will be public:
`openstack network list --external --column Name`

Create **deployment.tfvars** from template deployment.tfvars-template:
`cp tfvars/deployment.tfvars-template tfvars/deployment.tfvars`
and setup external_network_name variable:
- external_network_name = "openstack_external_network"

`terraform init`

then I will try this command.. 
`terraform apply -var-file tfvars/deployment.tfvars -var-file tfvars/vars-all.tfvars`

here is where i get stuck.. the instances seem to not get created properly...

# Openstack deployment (installed with kolla-ansible)

*Keypoints*
- We need to create an external network
- flavors - large and medium
- delete kali image // set the timer for longer in order to successfully download it

By following this guide:
https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment#preparing-the-deployment-environment

**Firstly the base deployment of openstack:**

```
snap install terraform --classic
git clone https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment.git

cd tf-openstack-base

*openstack network list --external --column Name*

```

Now we need to create an external network:

```
openstack network create \
  --provider-network-type flat \
  --provider-physical-network physnet1 \ #the physcial external link - we can check it through neutron ml2.ini file  
  --external \
  public

openstack subnet create \
  --network public \
  --subnet-range 192.168.10.0/24 \
  --allocation-pool start=192.168.10.20,end=192.168.10.29 \
  --gateway 192.168.10.254 \
  --dns-nameserver 8.8.8.8 \
  public-subnet
```

Flavors creation

my.large
```
openstack flavor create \
  --id 1 \
  --ram 16384 \
  --disk 80 \
  --vcpus 4 \
  my.large
```

my.medium
```
openstack flavor create \
  --id 2 \
  --ram 4096 \
  --disk 20 \
  --vcpus 2 \
  my.medium
```

delete kali image

/tf-openstack-base/.terraform/modules/images/images.tf



