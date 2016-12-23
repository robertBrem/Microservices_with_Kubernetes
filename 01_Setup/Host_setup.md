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