# BDP2_mid-term_review
This repository contains the mid term exercise for the course Infrastructures for big data processing of the master of Bioinformatics, University of Bologna.
The exercise consists of the following steps:

- Run a **Jupyter Notebook** and a **Redis** container to perform some basic operations.
- Create a **GitHub** repository and make it accessible from a Virtual Machine.
- Create a custom image from an existing Jupyter Notebook Docker image, using Git Action to automatically build the image on **DockerHub**.
- Build an application stack running multiple containers trough docker-compose.

For this exercise, a virtual machine built with **Amazon Web Service (AWS)** was used.

**Table of Contents**
- [1. Starting the Jupyter and Redis containers](#1-starting-the-jupyter-and-redis-containers)

    - [1.1 Access Jupyter and create a Redis database.](#11-access-jupyter-and-create-a-redis-database)

- [2. Clone a git repository in the VM](#2-clone-a-git-repository-in-the-vm)
- [3. Extend the image](#3-extend-the-image)
    - [3.1 Create a github action](#31-create-a-github-action)
    - [3.2 Create the Dockerfile](#32-create-the-dockerfile)
    - [3.3 Trigger the action](#33-trigger-the-action)
    - [3.4 Run the new image](#34-run-the-new-image)
- [4. Create an application stack](#4-create-an-application-stack)
- [5. Conclusion](#5-conclusion)
  
## 1. Starting the Jupyter and Redis containers
We want to run an existing Jupyter Notebook and Redis image. The aim is to access Jupyter Notebook from the web and create a simple Redis database. 
First, log in the virtual machine with SHH and create the directory review:
```
mkdir -p ~/review
cd ~/review
```
Create a bridge called `bdp2-net`. A “*bridge*” is a type of network device making it possible to transfer packets between devices on the same network segment. To see the existing bridges you can issue the command `docker network ls` (bridge is the default bridge built by docker).
```
docker network create bdp2-net
```
Now start the Redis and Jupyter containers:
```
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server
```
```
docker run -d --rm --name my_jupyter -v ~/review:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" jupyter/minimal-notebook
```
You should change the JUPYTER_TOKEN with a password of your choice.
As you can see, the image we are using is called `jupyter/minimal-notebook`

<u>Note that are mapping the port `8888` to port `80` in the docker host, hence remember to open the security group of the virtual machine for port 80 and your IP adress.</u>


### 1.1 Access Jupyter and create a Redis database.
Now you can acess Jupyter Notebook by browsing the web site `http://<VM1_public_IP>:80` (the password to access Jupyter is the JUPYTER_TOKEN of the previous command).

Create a new **Python3 notebook**, call it for example `test0.ipynb`. We want to use the Redis database, but if we issue the command:
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
If we were to stop the `my_jupyter` container however, or close the virtual machine, and issue again the docker run command, we would have to reinstall Redis again, as containers are ephimeral.

The Jupyter file that you create and saved have to be seen in the `review` directory on your VM. If you can't see them, you probably misspelled the path in the -v flag of the Jupyter `docker run` command. 

## 2. Clone a git repository in the VM
Create a **GitHub repository** (the repository I created is this one, `BDP2_mid-term_review` ), and clone it to the virtual machine, in the `review` directory we previously created. First make sure that GitHub is installed in your VM, if not, issue the following commands:
```
sudo apt update && sudo apt install -y git
git config --global user.name "<username>"
git config --global user.email "<user e-mail"
```
To clone the git repository issue this command in your directory:
```
git clone <link to the git hub repo>.git
```
After this, you should see the git hub repo as a directory on your VM. We want now to create the following subdirectory in it:
```
mkdir docker
mkdir work
```
## 3. Extend the image
We want to create a Jupyter costume image which already includes Redis. We want to make a **GitHub action** so that the image gets created on **DockerHub** as we commit from the Virtual machine to git.
### 3.1 Create a github action
GitHub Actions are procedures, configurable directly from a GitHub repository web page, that allow to automate and execute a software development workflow. 
We want to create a git hub action that automatically builds an image on DockerHub.

To create an action from the git hub repository, go to the Actions menu, then on Configure in the “Docker image" box. This opens a file written in the YAML format. To make it simple, you can copy the file I built (`docker-image.yml`). 
You have to have a DockerHub account in order to retrieve the `DOCKER_TOKEN`. On DockerHub, follow thiese instructions:
```
Account settings > Security > New acess token > Give a name to the token and select “Read, Write, Delete” > generate it and copy it somewhere
```

We need then to make this token a **GitHub secret**. On the GitHub repository follow these procedures:
```
settings > secrets and variables > actions > New repository secret > call it DOCKER_TOKEN and copy the string you generated before > add secret
```
Now create the action. You should see a new structure in your repo as `.github/workflow` was created.
At this point if you commit some changes the action won't work, as we haven't created a Dockerfile yet.

### 3.2 Create the Dockerfile
In the `docker` subdirectory, create a **Dockerfile** with a text editor (e.g. *vi*). This Dockerfile should give the instructions to install the Redis module into the Jupyter Notebook image. It should looke like this:
```
FROM jupyter/minimal-notebook 
RUN pip install redis
```
We are basically giving the instructions to build a new image from the image that we ran in step 1 (the `jupyter/minimal-notebook`), adding the option to already install Redis in it by pip-installing it.

### 3.3 Trigger the action
To trigger the action, we have to do a commitment from our VM to the GitHub repository; in particular, we will commit the Dockerfile we just created, as that is neccessary for the action to work.

Before being able to **push** our new files on GitHub, we have to syncronize the directory on our VM with the git hub repo with the command:
```
git pull
```
This will update our directory with the changes made on GitHub. This is a necessary step to do everytime you make a change to the repository from GitHub instead than from the VM. We are now redy to commit. Issue the following commands: 
```
git status
```
To see which files you can push
```
git add <Names of the file (Dockerfile in this case)>

git commit -a -m "<Message to justify the commitment>"

git push -u origin main
```
Verify on GitHub that your Dockerfile has been pushed. Now the actions should be triggered, you can see its progress from the action menu. If the action has failed, you will see a red dot and a failing message; the message should tell you why the action has failed. If you see a green dot the action was successful. You should check on DockerHub the presence of the new image (in my case, `gorsini / bdp2_mid_term_review`). If the image you wanted to create with the action is not there, then the action  wasn't run succesfully.
At this point, any new `git push` will trigger a rebuild of your image.

### 3.4 Run the new image
On the VM, now run the new image you created using the `docker run` command we used earlier to start Jupyter. <u> Only remember that this time you should specify as image name your custom image and should modify the path in the -v flag so that it leads to the `work` directory.</u> 
```
docker run -d --rm --name bdp2_mid_term_review -v ~/review/BDP2_mid-term_review/work:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" gorsini/bdp2_mid_term_review
```

Now, try again to build a small databse as we did in step 1:
```
import redis
r = redis.Redis(host='my_redis')
r.set('temperature', 18.5)
print(r.get('temperature'))
```
The Redis module should be already available in the container, so you shouldn't receive any error messages. Check that the new Jupyter file is saved in the `work` directory (in my case, the saved file was `test1.ipynb`)

Create the file `.gitignore` in the main directory of your local repo (use e.g vi) and write the following line into it to avoid git complaining about “*Notebook Checkpoints*” not being tracked:
```
.ipynb_checkpoints
```

## 4. Create an application stack
We now want to create an **application stack**, multiple containers linked together to provide a multi-container service, via **docker compose**. `Docker-compose` works by parsing a proper text file, written in the YAML language. Our file should have three services, building the Jupyter custom image we just created, the Redis image and a **Portainer** image. 
If docker-compose is not available in the VM, download it with:
```
sudo apt update && sudo apt install -y docker-compose
```
Create a file called `docker-compose.yml`. You may see the file I created (`docker-compose.yml`). This part is crucial: you have to put all the flags of the docker run commands in this file. Go back on the previous docker run commands to see them. 

Portainer is an open source Docker graphical manager tool that allows us to create and manage docker containers from the browser. During the course we used the following command to run the Portainer container: 
```
docker run -d --rm -p 8000:8000 -p 443:9443 --name=portainer -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```
<u>Note: make sure that all the ports are open for your security group in the inbound rules of your VM</u>.

Verify that everything works by running the command:
```
docker-compose up -d
docker ps
```
You should see the 3 containers. Write a Jupyter file like we did in step 1 and 3 (in my case, `test2.ipynb`). The Jupyter file that you write should be visible from the work directory. In the Jupyuter notebook, Redis as to be already installed: <u>pay attention to connect to the new redis image you created.</u>

Once everything is done, close the application stack with the command:
```
docker-compose down
```
And commit all the new changes to the git hub repo.

## 5. Conclusion
If you have done things correctly, you now have a single docker-compose command that starts up a complete development environment based on Redis and Jupyter, with a web interface based on Portainer to manage everything. In fact, if you access Portinaer, you should be able to see all of your containers running. You may manage them directly from there.

Moreover, all is stored in a GitHub repository, which comprises a GitHub action that automatically builds and store a costume image on DockerHub.

All the files generated for my mid term review are stored in the present directory.
