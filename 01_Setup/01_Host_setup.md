# Setup your local host machine

## Basic setup
I'm using a Ubuntu 16.04 Desktop installation. After the installation I execute the following
[script](https://gist.github.com/robertBrem/2b382911e967692e240f):
```bash
sudo apt-add-repository ppa:ansible/ansible -y
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install software-properties-common -y
sudo apt-get install ansible -y
sudo apt-get install git -y
ansible --version
```

## Installation of Docker, IntelliJ and Payara
For this installation I have created an Ansible script that can be checked out [here](https://github.com/robertBrem/Microservices_Ansible_Setup).  
Start the installation of the tools:
```bash
ansible-playbook playbooks/basicSetUp.yml
```
Test if Docker can be used without root rights:
```bash
docker ps
```
If this is not the case you have to execute the following command:
```bash
newgrp docker
```

## IntelliJ live templates
I'm using a lot of live templates that I've created for IntelliJ. This templates are 
available on [GitHub](https://github.com/robertBrem/IntelliJ_Live_Templates).  
To include them in IntelliJ we have to check out the repository in the IntelliJ 
configuration folder.
```
cd ~/.IntelliJIdea2016.3/config/
git clone https://github.com/robertBrem/IntelliJ_Live_Templates
mv IntelliJ_Live_Templates/ templates
```
Now I can use my live templates everywhere.