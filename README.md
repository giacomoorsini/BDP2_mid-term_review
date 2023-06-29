# BDP2_mid-term_review
This repository contains the mid term exercise for the course Infrastructures for big data processing of the master of Bioinformatics, University of Bologna.
The exercise consists of the following steps:

- Create a custom image from an existing Jupyter Notebook Docker image, using git to automatically build the image on DockerHub
- Build an application stack running multiple containers trough docker-compose

For this exercise, a virtual machine built Amazon Web Service was used.

**Table of Contents**
- [St
  
## Starting the Jupyter and Redis containers
We want to run our existing Jupyter Notebook and Redis image. The aim is to access Jupyter Notebook from the web and create a simple Redis database. 
First, log in the virtual machine with shh and create the directory review:
```
mkdir -p ~/review
cd ~/review
```
Create a bridge called bdp2-net. A “bridge” is a type of network device making it possible to transfer packets between devices on the same network segment. To see the existing bridges you can issue the command docker network ls (bridge is the default bridge built by docker).
```
docker network create bdp2-net
```
Now start the Redis and Jupyter containers:
```
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server
```
```
docker run -d --rm --name my_jupyter -v ~/review:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="password" --user root -e CHOWN_HOME=yes -e 
CHOWN_HOME_OPTS="-R" jupyter/minimal-notebook
```
Remember to change the password.
We are mapping the port 8888 to port 80 in the docker host, hence remember to open the security group of the virtual machine for port 80 and your IP adress.

### Access Jupyter and create a Redis database.
Now you can acess Jupyter Notebook by browsing the web site  http://<VM1_public_IP>:80 (the password to access Jupyter is the JUPYTER_TOKEN of the previous command).

Create a new Python3 notebook. We want to use the Redis database, but if we issue the command:
```
import redis
```
we will get an error. This is because we have to first install redis:
```
! pip install redis
```
We can see that now Redis works:
```
import redis
r = redis.Redis(host='my_redis')
r.set('temperature', 18.5)
print(r.get('temperature'))
```
If we were to stop the my_jupyter container however, or close the virtual machine, and issue again the docker run command, we would have to reinstall Redis again, as containers are ephimeral.

## Extend the image
### Clone a git repository in the VM
We want to create a Jupyer costume image which include Redis. We want to make a git hub action so that the image gets created on DockerHUb as we commit from the Virtual machine to git.
Create a git hub repository (the repository i created is this one) from git. 
Then, we have to clone this repository to the virtual machine, in the directory we previously created. First make sure that git hub is installed, if not, issue the following commands:
```
sudo apt update && sudo apt install -y git
git config --global user.name "user name"
git config --global user.email "user e mail"
```
To clone the git repository:
```
git clone <link to the git hub repo>.git
```
After this, you should see the git hub repo as a directory on your VM. We want now to create a structure comprising the following subdirectory:
```
mkdir docker
mkdir work
```
### Create a github action
GitHub Actions are procedures, configurable directly from a GitHub repository web page, that allow to automate and execute a software development workflow. 
From the git hub repository, go to the Actions menu, then on Configure in the “Docker image" box. This opens a file written in the YAML format. To make it simple, you can copy the file I built
