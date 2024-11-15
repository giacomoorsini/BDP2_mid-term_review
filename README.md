# BDP2 mid-term review: Building a custom Jupyter Notebook Docker image using GitHub actions to build it automatically on DockerHub and create an application stack
This repository contains the necessary information and files generated to complete the mid-term review for the Big Data Processing course of the Master's Degree in Bioinformatics of the Università di Bologna, course 2022-2023.
The exercise consists of the following steps:

- Run a **Jupyter Notebook** and a **Redis** container to perform some basic operations.
- Create a **GitHub** repository and make it accessible from a Virtual Machine.
- Create a custom image from an existing Jupyter Notebook Docker image, using Git Action to automatically build the image on **DockerHub**.
- Build an application stack running multiple containers trough docker-compose.

For this exercise, a virtual machine built with **Amazon Web Service (AWS)** was used.

Written by Giacomo Orsini

**Table of Contents**
- [0. Requirements](#0.-requirements)
    - [0.1 First steps](#1.-first-steps) 
- [1. Starting the Jupyter and Redis containers](#1-starting-the-jupyter-and-redis-containers)
    - [1.1 Access Jupyter and create a Redis database.](#11-access-jupyter-and-create-a-redis-database)
- [2. Clone a git repository in the VM](#2-clone-a-git-repository-in-the-vm)
- [3. Extend the image](#3-extend-the-image)
    - [3.1 Create a GitHub action](#31-create-a-github-action)
        - [3.1.1 On GitHub](#311-on-github)
        - [3.1.2 On DockerHub](#312-on-dockerhub)
        - [3.1.3 Create the action](#313-create-the-action)
    - [3.2 Create the Dockerfile](#32-create-the-dockerfile)
    - [3.3 Trigger the action](#33-trigger-the-action)
    - [3.4 Run the new image](#34-run-the-new-image)
- [4. Create an application stack](#4-create-an-application-stack)
    -[4.1 Docker compose](#41-docker-compose)
- [5. Conclusion](#5-conclusion)
- [6. Extra](#6-extra)
    - [6.1 Try to implement https to jupyter](#61-try-to-implement-https-to-jupyter)
    - [6.2  Show if and how it works on your laptop in the same way it works on VM1](#62-show-if-and-how-it-works-on-your-laptop-in-the-same-way-it-works-on-vm1)

## 0. Requirements
To be able to replicate this review, ensure you have installed in your virtual machine the following programs:

- docker: to run containers (`run`), track running containers (`ps`) and inspect already created images (`images`).
- `git`: to track changes in your directory and to clone it to/from GitHub (`push`/`pull`).
- `docker-compose`: to create the application stack.

### 0.1 First step
As we are working with AWS, log in to the virtual machine of preference (in this case, VM1) through `ssh`. Remember to first go to the directory where our key is stored.

```
ssh -i "bdp2.pem" ubuntu@ec2-<VM1_public_IP>.compute-1.amazonaws.com
```
Then, create a new directory from which we will be working, and go to it:

```
mkdir -p ~/review
cd ~/review
```

## 1. Starting the Jupyter and Redis containers
We want to run an existing **Jupyter Notebook** and **Redis** image. The aim is to access Jupyter Notebook from the web and create a simple Redis database. 
First, create a **bridge** called `bdp2-net`. A “*bridge*” is a type of network device that allows packets to be transferred between devices on the same network segment. To see the existing bridges, issue the command `docker network ls` (`bridge` is the default bridge built by docker).
```
docker network create bdp2-net
```
Now start the (default) Redis and Jupyter containers:
```
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server
docker run -d --rm --name my_jupyter -v ~/review:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" jupyter/minimal-notebook
```
Where:
- `-d` is the option to run container in background and print container ID
- `--rm`: automatically remove the volume when it exits
- `--name`: name of the container (`my_redis` and `my_jupyter`)
-  `-p` indicates the ports from which the container is accessible form the host
-  `--network`: the network to which we are connecting both containers
-  `-v` is to bind mount the indicated volume
-  `-e` option introduces environmental variables
-  `--rm` option tells to remove the container when we exit it automatically
  
It would be best if you changed the `JUPYTER_TOKEN` with a password of your choice.
As you can see, the image we are using are called `redis` and `jupyter/minimal-notebook`

<u> Note that we are mapping the port `8888` to port `80` in the docker host, hence remember to open the security group of the virtual machine for port 80 and your IP adress.</u>

### 1.1 Access Jupyter and create a Redis database.
Check that both containers are up and running correctly with `docker ps`. Now you can access Jupyter Notebook by browsing the website http://<VM1_public_IP>:80 (the password to access Jupyter is the JUPYTER_TOKEN of the previous command).

Create a new **Python3 notebook**, call it for example `test0.ipynb`. We want to use the Redis database, but if we issue the command:
```
import redis
```
We will get an error. This is because we have to first install redis module (it is not installed by default):
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

The Jupyter file you created and saved must be seen in the `review` directory on your VM. If you can't see it, you probably misspelled the path in the—v flag of the Jupyter `docker run` command. 

## 2. Clone a git repository in the VM
Create a **GitHub repository** (the repository I created is this one, `BDP2_mid-term_review` ), and clone it to the virtual machine, in the `review` directory we previously created. First make sure that **GitHub** is installed in your VM, if it is not, issue the following commands:
```
sudo apt update && sudo apt install -y git
git config --global user.name "<username>"
git config --global user.email "<user e-mail"
```
To clone the git repository, issue this command in your directory:
```
git clone <link to the git hub repo>.git
```
After this, you should see the GitHub repo as a directory on your VM. We want now to create the following subdirectories:
```
mkdir docker
mkdir work
```
where we will write and save the Dockerfile to create the custom image and the Jupyter notebooks.

## 3. Extend the image
We want to create a Jupyter costume image which already includes Redis. We want to make a **GitHub action** so that the image gets builts on **DockerHub** as we commit from the Virtual machine to git.

As we are creating a new Jupyter image, the first thing to do is to stop my_jupyter:
```
docker stop my_jupyter
```

### 3.1 Create a GitHub action
**GitHub Actions** are procedures configurable directly from a GitHub repository web page that allow to automate and execute a software development workflow. 
We want to create a git hub action that automatically builds an image on DockerHub.

#### 3.1.1 On GitHub
To create an action from the git hub repository, go to the `Actions menu`, then on `Configure` in the “Docker image" box. This opens a file written in the `YAML` format. To make it simpler, you can copy the file I built (`docker-image.yml`). 

```
name: Docker Image CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:

  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Build the Docker image
      run: docker build . --file docker/Dockerfile --tag gorsini/bdp2_mid_term_review
    - name: Push the image to Dockerhub
      run: docker login -u gorsini -p${{ secrets.DOCKER_TOKEN }} && docker push gorsini/bdp2_mid_term_review
```
Line by line: 
- 1: the name of the workflow (Docker Image Jupyter+redis)
- 3-7: when will this action be triggered (every time you push or pull the "main" branch)
- 9-20: build the Docker image and push it to DockerHub:
    - `--file`: path and name of the Dockerfile that contains the information to build the image
    - `--tag`: name of the image (in my case, torresmasdeu/ltm_jupyter)
    - `-u`: username of DockerHub
    - `-p`: DockerHub token (NOT the password)
  
#### 3.1.2 On DockerHub
You have to have a DockerHub account in order to retrieve the `DOCKER_TOKEN`. On DockerHub, follow these instructions:
```
Account settings > Security > New acess token > Give a name to the token (i.e. "Token for GitHub") and select “Read, Write, Delete” > generate it and copy it somewhere
```

#### 3.1.3 Create the action
We need then to make this token a **GitHub secret**. On the GitHub repository, follow these procedures:
```
settings > secrets and variables > actions > New repository secret > call it DOCKER_TOKEN and copy the string you generated before > add secret > verify it is listed as a "Repository secret"
```
Now create the action. You should see a new structure in your repo as `.github/workflow` was created.
At this point if you commit some changes the action won't work, as we haven't created a Dockerfile yet.

Before creating the **Dockerfile**, we have to ensure to pull the new addition that we have on the GitHub repo to the local repo (that is, the `docker-image.yml` file):
```
git pull
```
It will ask you for your GitHub username and password. However, for security purposes, the password is not the one you use to connect to GitHub, but a GitHub Personal Access Token. This can be retrieved in the general "Settings" (by clicking on your profile photo), “<> Developer settings” and heading to "Personal access tokens". Then, click on “Generate new token” , “Generate new token (classic)” and give it a name (i.e. "BDP2 GitHub token"), select a desired expiration date, and in "scopes" select repo. Lastly, click on "Generate token". As before, save the token, as you will not be able to acces it once you exit.

Check that the `docker-image.yml` file is now visible in your local repo (under `.git/workflows/` directory).

### 3.2 Create the Dockerfile
In the `docker` subdirectory, create a **Dockerfile** with a text editor (e.g. *vi*). This Dockerfile should give the instructions to install the **Redis module** into the **Jupyter Notebook image**. It should look like this:
```
FROM jupyter/minimal-notebook 
RUN pip install redis
```
We are basically giving the instructions to build a new image from the image that we ran in step 1 (the `jupyter/minimal-notebook`), adding the option to already install Redis in it by pip-installing it.

### 3.3 Trigger the action
To trigger the action, we have to make a commitment from our VM to the GitHub repository; in particular, we will commit the Dockerfile we just created, as that is necessary for the action to work.
Before being able to **push** our new files on GitHub, we have to syncronize the directory on our VM with the git hub repo with the command. Make sure the `git pull` command works, as this is a necessary step to do every time you make a change to the repository from GitHub instead of from the VM. 

We are now ready to commit. Issue the following commands: 
```
git status
```
To see which files you can push
```
git add <Names of the file (Dockerfile in this case)>
git commit -a -m "<Message to justify the commitment>"
git push -u origin main
```
<u> **Note** that the repo is pushed to the `main` branch, and not the `master`, because in the `docker-image.yml` file we wrote that it was to be executed when pushing to the `main` branch. Moreover, the said file is saved in the `main` branch, so if we were to push things into the `master` branch, we would have two branches that could not interact with each other. </u>

Verify on GitHub that your Dockerfile has been pushed. Now the actions should be triggered, you can see its progress from the action menu. If the action has failed, you will see a red dot and a failing message; the message should tell you why the action has failed. If you see a green dot the action was successful. You should check on DockerHub the presence of the new image (in my case, `gorsini / bdp2_mid_term_review`). If the image you wanted to create with the action is not there, then the action  wasn't run successfully.
At this point, any new `git push` will trigger a rebuild of your image.

### 3.4 Run the new image
On the VM, now run the new image you created using the `docker run` command we used earlier to start Jupyter. <u> Only remember that this time you should specify as image name your custom image (`gorsini/bdp2_mid_term_review` in my case) and should modify the path in the `-v` flag so that it leads to the `work` directory.</u> 
```
docker run -d --rm --name bdp2_mid_term_review -v ~/review/BDP2_mid-term_review/work:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" gorsini/bdp2_mid_term_review
```
Once it is running, open Jupyter Notebooks in your browser, http://<VM1_public_IP>:80, and create a new notebook (i.e. `dir_test.ipynb`). Now, try again to build a small database as we did in step 1:
```
import redis
r = redis.Redis(host='my_redis')
r.set('temperature', 18.5)
print(r.get('temperature'))
```
The Redis module should be already available in the container, so you shouldn't receive any error messages. Check that the new Jupyter file is saved in the `work` directory (in my case, the saved file was `test1.ipynb`)

Before committing all, create the file `.gitignore` in the main directory of your local repo (use e.g vi) and write the following line into it to avoid git complaining about “*Notebook Checkpoints*” not being tracked:
```
.ipynb_checkpoints
```
This file includes the names of any files on the local repo that we do not want to commit to git. 

Now:
```
git add * 
git commit -a -m "Commit after docker run successful command"
git push
docker ps
```

## 4. Create an application stack
We now want to create an **application stack**, multiple containers linked together to provide a multi-container service, via **docker compose**. `Docker-compose` works by parsing a proper text file, written in the YAML language. Our file should have three services, building the **Jupyter custom image** we just created, the **Redis image** and a **Portainer** image. he names of the images can be found by running `docker images`. 

If docker-compose is not available in the VM, download it with:
```
sudo apt update && sudo apt install -y docker-compose
```
**Portainer** is an open source Docker graphical manager tool that allows us to create and manage docker containers from the browser. During the course we used the following command to run the Portainer container: 
```
docker run -d --rm -p 8000:8000 -p 443:9443 --name=portainer -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```
<u>Note: make sure that all the ports are open for your security group in the inbound rules of your VM</u>.

### 4.1 Docker compose
Create a file called `docker-compose.yml`. You may see the file I created (`docker-compose.yml`). This part is crucial: you have to put all the flags of the docker run commands in this file.  Let's remember these three commands:
```
#redis
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server

#custom jupyter
docker run -d --rm --name bdp2_mid_term_review -v ~/review/BDP2_mid-term_review/work:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" gorsini/bdp2_mid_term_review

#portainer
docker run -d --rm -p 8000:8000 -p 443:9443 --name=portainer -v
/var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```
The container building is done under `services`.
This is the parallelism between the `docker run` flags and the `docker-compose.yml` commands:

- Under `services`: we have three different levels, one for each container. The names of these levels are the names that will be used to name the container (before under the `--name`v flag)
- `image`: the name of the image (unflagged in `docker run`)
- `ports`: `-p` flag.
- `volumes`: `-v` flags. Important: if one of these flags links a volume to the container, the volume name has to be specified outside `services`, under `volumes` (see the case of the volume `portainer_data`)
- `networks`: as we want the three containers to be able to communicate between each other, we have to put them under the same network. As with the case of volumes, the name of the network(s) in use has to be specified in another section outside services, `networks`
- `environment`: `-e` flag. In the case of the password needed to connect to Jupyter Notebooks, `JUPYTER_TOKEN`, we do not want it to be visible to anyone. This is why I decided to set an environment variable, called `JUPYTER_PW`, that will look for my password in the `.env` file I have created in the same directory as the `docker-compose.yml` file. Note: to keep from adding this file to git, add the file name (`.env`) to the `.gitignore` file
- user: `--user` tag

Taking all of this into consideration, this is the resulting docker-compose.yml file:

```
version: '3'
volumes:
   portainer_data:
services:
   redis:
     image: redis
     volumes:
       - ~/review:/data
     networks: 
       - bdp2-net
   jupyter:
     image: gorsini/bdp2_mid_term_review
     environment:
       - JUPYTER_ENABLE_LAB=yes
       - JUPYTER_TOKEN=bdp2_password
       - CHOWN_HOME=yes
       - CHOWN_HOME_OPTS=-R
     ports:
       - 80:8888
     volumes:
       - ~/review/BDP2_mid-term_review/work:/home/jovyan
     networks:
       - bdp2-net
   portainer:
     image: portainer/portainer-ce
     volumes:
       - /var/run/docker.sock:/var/run/docker.sock
       - portainer_data:/data 
     ports: 
       - 8000:8000
       - 443:9443
     networks:
       - bdp2-net
networks:
   bdp2-net:
```

Verify that everything works by running the command:
```
docker-compose up -d
docker ps
```
You should see the 3 containers running. The command will automatically create a new bridge network.

Before checking with portainer that the containers are working successfully, remember to open port 443 (of the portainer container) for your laptop to be able to access the portainer website, https://<VM1_public_IP>. Log in to portainer (choose whatever admin password you want when connecting for the first time), and click on "Get started", to visualise your local compartment.

Check that the Redis server and Jupyter are working by connecting to Jupyter Notebooks (http://<VM1_public_IP>:80). Write a Jupyter file like we did in step 1 and 3 (in my case, `test2.ipynb`). The Jupyter file that you write should be visible from the work directory.

Once everything is done, close the application stack with the command:
```
docker-compose down
```
And commit all the new changes to the git hub repo.

## 5. Conclusion
If you have done things correctly, you now have a single docker-compose command that starts up a complete development environment based on Redis and Jupyter, with a web interface based on Portainer to manage everything. In fact, if you access Portinaer, you should be able to see all of your containers running. You may manage them directly from there.

Moreover, all is stored in a GitHub repository, which comprises a GitHub action that automatically builds and store a costume image on DockerHub.

All the files generated for my mid-term review are stored in the present directory.

## 6. Extra
### 6.1 Try to implement `https` to Jupyter
Jupyter Notebook can also be accessed with HTTPS. To do that, we need to add the port `433:8888` to our Jupyter container in the `docker-compose` file.
This, however will issue an error as Portainer already occupies port 443. 

To solve the error, I tried to issue the `docker run` command for the Jupyter image but changing the ports in the `-p` flag:
```
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server

docker run -d --rm --name bdp2_mid_term_review -v ~/review/BDP2_mid-term_review/work:/home/jovyan -p 80:8888 -p 443:8888--network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" gorsini/bdp2_mid_term_review
```
The container is created with no issues, but still Jupyter is not accessible through the `https`. Searching online, the solution seems to be that Jupyter can be accessible from `https` just with an `SSL certificate`, that enables the access.

### 6.2  Show if and how it works on your laptop in the same way it works on VM1
You can execute the `docker-compose` from your laptop. To do so, you must have installed the Docker Desktop app.

For simplicity, we may recreate the directory `review` as we did on VM1 so that we may use the same `docker-compose.yml`. We also should clone the GitHub repo in our new review directory.

Then, by running the docker-compose like before, we should activate the three containers. We can check that with `docker ps`.

To check if Jupyter and Portainer are working, you should try to connect by browsing `http://0.0.0.0:80` and `http://0.0.0.0:443`.
