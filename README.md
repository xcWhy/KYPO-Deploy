# KYPO-Deploy
Here I will be storing all the files, commands and sources for the KYPO CRP platform deployment
*For a personal project*

The sources I'm getting all my information from:
- https://docs.crp.kypo.muni.cz/
- https://computingforgeeks.com/openstack-deployment-on-ubuntu-with-devstack/
- https://www.spacesecurity.info/install-kypo-cyber-range-platform-on-openstack/
- For deployment: https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment#preparing-the-deployment-environment

All the commands I will be performing:

# Installing openstack cloud using devstack
* I will be using the spacesecurity tutorial, linked at the beggining.
```
sudo apt update
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

sudo su - stack
sudo apt -y install git
git clone https://git.openstack.org/openstack-dev/devstack

git clone <this repo>

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

# Openstack Deployment

Firstly we should download the **ADMIN** openrc file from the dashboard - which means we need to switch to the admin project firstly...

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

