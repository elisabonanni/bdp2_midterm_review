# bdp2_midterm_review

Login to VM1 through ```ssh```, create a directory where to store your files and go to that directory:

```
mkdir -p ~/review
cd ~/review
```

Create a bridge, called _bdp2-net_, to be able to access Redis server from the Jupyter notebook. Jupyter and Redis containers will be set in the same network bridge.

```
docker network create bdp2-net
```

Start the Redis container:

```
docker run -d --rm --name my_redis -v ~/review:/data --network bdp2-net --user 1000 redis redis-server
```

Start the Jupyter container:

```
docker run -d --rm --name my_jupyter -v ~/review:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" jupyter/minimal-notebook
```

Connect to Jupyter visiting the URL _http:/<public_ip>:80_ from the browser, be sure to have the 80 port open (change it in the inbound rules of AWS if necessary). It will ask for the token stated in the above command in the variable ```JUPYTER TOKEN```: _bdp2_password_.

On Jupyter in python notebook:

```
import redis
```

If not there:

```
!pip install redis
import redis
```

Use Redis to run some code:

```
r = redis.Redis(host='my_redis')
r.set('temperature', 18.5)
print(r.get('temperature'))
```

<img width="709" alt="Screenshot 2024-09-17 alle 14 37 21" src="https://github.com/user-attachments/assets/88803018-1385-4dc4-aade-34948fb60206">

### 1. Extend the image
Since containers are ephemeral, stopping and running it again, the Redis module needs to be re-installed. There is the need to extend the _jupyter/minimal-notebook_ Docker image and create a custom image that includes Redis. The changes will be tracked using ```git``` and will be pushed on GitHub.

A new image will be created so:

```
stop my_jupyter
```

Set up a git repo to track the development of the files:
• Log in to GitHub and create a new GitHub repo.
• Clone the new repo to VM1 with the command git clone and cd to the repo directory on VM1.

```
git clone  https://github.com/elisabonanni/bdp2_midterm_review
```

The repository will be now visible from VM1. Change into it:

```
cd bdp2_midterm_review
```

Start the git repo:

```
git init

git branch -M master
```

Create a docker and a work directory, in docker store the Dockerfile for the new image, in work store the Notebook files and the Redis DB files.

```
mkdir work
mkdir docker
```

Write, add and commit a Dockerfile in the docker subdirectory, which install redismodule into the _jupyter/minimal-notebookimage_, as:

```
FROM jupyter/minimal-notebook
RUN pip install redis
EXPOSE 80
```

```
cd docker
vim Dockerfile

git add docker/
git commit -m 'Create Dockerfile'
git push origin master
git status
```

Create the _.gitignorein_ file the master directory and write the following line into it to avoid git complaining about “Notebook Checkpoints” not being tracked:
.ipynb_checkpoints.

```
vim .gitignorein
git add .gitignorein
git status
git commit -m 'Create gitignore file'
git push origin master
git status
```

From the DockerHub can be retrieve a Docker Token, from Account Settings > Security > New Access Token. Create the new token and select "Read, Write, Delete" as access permission.

<img width="672" alt="Screenshot 2024-09-17 alle 15 31 31" src="https://github.com/user-attachments/assets/4a1ce446-7ba2-478d-9b0d-8285e44b9c3a">

Then, from GitHub go to the Settings of the repo, select Secrets and Variables and New repository secret. Call the variable "DOCKER_TOKEN" and paste the token just created on DockerHub. Then add it.

Now, define the GitHub Action that will build the image from Dockerfile to push it to DockeHub. From the GitHub repository, select Action > Docker image, select and write the following lines before committing changes:

```
name: Docker Image Jupyter&Redis

on:
  push:
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]

jobs:

  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Build the Docker image
      run: docker build . --file docker/Dockerfile --tag elibonanni/bdp2_midterm_review
    - name: Push the image to DockerHub
      run: docker login -u elibonanni -p${{ secrets.DOCKER_TOKEN }} && docker push elibonanni/bdp2_midterm_review
```

Verify on DockerHub that a new image has been automatically built through the Action defined. Any git push will trigger a rebuild of the image.

On VM1 run the new image using docker run:

```
docker run -d --rm --name my_jupyter -v ~/review/bdp2_midterm_review/work:/home/jovyan -p 80:8888 --network bdp2-net -e JUPYTER_ENABLE_LAB=yes -e JUPYTER_TOKEN="bdp2_password" --user root -e CHOWN_HOME=yes -e CHOWN_HOME_OPTS="-R" elibonanni/bdp2_midterm_review
```

The Jupyter Notebook can be opened as before from the browser thanks to the link _http://<public_ip>:80_. Create a new notebook (_Test.ipynb_). Differently from before, the redis module will be available in the container without the need to install it with ```!pip```.

<img width="1000" alt="Screenshot 2024-09-17 alle 15 40 19" src="https://github.com/user-attachments/assets/14fadc33-a70a-4791-a2df-58d5056e578c">

The test file should be in the _work_ directory, so the changes can be committed to GitHub.

```
git add work/
git commit -m 'Create Work'
git push origin master
git status
```

Then, stop the containers:

```
docker stop my_redis
docker stop my_redis_jupyter
```

Check with:

```
docker ps
```

### 2. Create an application stack
Create an application stack using ```docker-compose``` with 3 services:
1. Jupyter custom image
2. Redis image
3. Portainer image

All the mount points (bind mounts and Docker volumes), that is, the -v flags that you previously specified in the docker run commands, should be listed in this _docker-compose.yml_ file.
The containers can be build under ```services```.

The _docker-compose.yml_ file created will be:

```
version: "3"
services:
  portainer:
    image: portainer/portainer-ce
    ports:
      - 8000:8000
      - 443:9443
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - bdp2-net

  jupyter:
    image: elibonanni/bdp2_midterm_review
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - JUPYTER_TOKEN=${JUPYTER_PW}
      - CHOWN_HOME=yes
      - CHOWN_HOME_OPTS=-R
    user: root
    ports:
      - 80:8888
    volumes:
      - ~/review/bdp2_midterm_review/work:/home/jovyan
    networks:
      - bdp2-net

  redis:
    image: redis
    user: "1000"
    volumes:
      - ~/review:/data
    networks:
      - bdp2-net
networks:
  bdp2-net:

volumes:
  portainer_data:
```

To create the application stack, run ```docker compose```:

```
docker compose up -d
```

<img width="673" alt="Screenshot 2024-09-17 alle 16 19 58" src="https://github.com/user-attachments/assets/4bdbfe65-05cb-42f5-83df-13f0fe73b736">

Check that they are running using ```docker ps```.

<img width="496" alt="Screenshot 2024-09-17 alle 16 20 28" src="https://github.com/user-attachments/assets/35dc0a2c-6408-493a-83ff-a5d547ee20fb">

Connect to Jupyter and verify that notebook files are stored locally in the work directory and that you can directly operate on Redis.

To stop the stack then:

```
docker compose down
```

<img width="660" alt="Screenshot 2024-09-17 alle 16 20 44" src="https://github.com/user-attachments/assets/09e10692-3c13-49b7-9c5f-35991e38d08d">

The network in the docker compose file has been called the same as the network created initially, _bdp2-net_, but running the ```docker compose``` command, it creates a new bridge network called bdp2-review_bdp2-net.
This can be checked with:

```
docker network ls
```

<img width="437" alt="Screenshot 2024-09-17 alle 16 46 03" src="https://github.com/user-attachments/assets/78b70932-bce7-4dfc-920f-6bfc2f56aa33">

Try to remove the bdp2-net with:

```
docker network rm bdp2-net
```

Then docker compose should continue to work correctly, check it:

```
docker compose up -d
```

<img width="657" alt="Screenshot 2024-09-17 alle 16 47 57" src="https://github.com/user-attachments/assets/1b54d214-d2be-4eb0-b3fd-d26095d61686">

So, docker compose has also create a new network.

Check on Portainer that the containers are working correctly. Connect to the site _https://<public_ip>_. Remember to have open the access to open port 443 from the inbound rules on AWS. Log in to Portainer and Get started.

<img width="1014" alt="Screenshot 2024-09-17 alle 16 37 26" src="https://github.com/user-attachments/assets/f77f6161-3e24-4559-a5df-9d24f7c1b111">

<img width="1018" alt="Screenshot 2024-09-17 alle 16 38 45" src="https://github.com/user-attachments/assets/3c9cafcf-a48d-4944-b858-2e906d6820c7">


Check that everything works also on Jupyter, accessing with _http://<public_ip>_. In this case, to test, the host name must be changed (it can be checked the new name of Redis Container using ```docker ps```), try some command like:

```
import redis

r = redis.Redis(host='bdp2. _midterm_review-redis-1')
r. set ('temperature' , 18.5)
print(r-get('temperature'))
```

<img width="987" alt="Screenshot 2024-09-17 alle 16 38 39" src="https://github.com/user-attachments/assets/46420df7-3888-4c7f-86f7-4bf68183a9a0">

Push all the changes to GitHub:

```
git add work
git add docker-compose.yml
git status
git commit -m 'Docker Compose'
git push origin master
git status
```

Lastly, update also the README.md file with the overall the procedure.
