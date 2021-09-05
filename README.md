# e-movies-app
E-Movies App, a Django project in the context of HUA DIT course 'Basic DevOps Concepts and Tools'

## Run project locally
### Clone and initialize project
```bash
git clone https://github.com/panagiotisbellias/e-movies-app 
python3 -m venv myvenv
source myvenv/bin/activate
pip install -r requirements.txt
cd movies_app
cp movies_app/.env.example movies_app/.env
```
Edit movies_app/.env file to define
```vim
SECRET_KEY='test123'
DATABASE_URL=sqlite:///./db.sqlite3
ALLOWED_HOSTS=localhost
```

If you want to run it with PostgreSQL Database visit the [link](https://www.youtube.com/watch?v=RAFZleZYxsc) and change DATABASE_URL above as:
```vim
DATABASE_URL=postgresql://<DB-USERNAME>:<DB-PASSWORD>@localhost/<DB-NAME>
``` 
after you have created a database using [pgAdmin](https://www.youtube.com/watch?v=1wvDVBjNDys)

### Database migration
```bash
python manage.py makemigrations && python manage.py migrate
```

### Run development server
```bash
python manage.py runserver
```

### Alternatively, run gunicorn application server
```bash
gunicorn --bind 0.0.0.0:8000 movies_app.wsgi:application
```

## Deploy django project to a VM (Virtual Machine)

We are going to need 4 VMs. One for the jenkins server and one for each execution environment (ansible, docker and kubernetes)

* [Create VM in Gcloud](https://cloud.google.com/compute/docs/instances/create-start-instance)
* [Create VM in Azure Portal](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal)
* [SSH Access to VMs](https://help.skytap.com/connect-to-a-linux-vm-with-ssh.html)
* [SSH Automation](https://linuxize.com/post/using-the-ssh-config-file/)
* [Reserve Static IP in Gcloud](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address)
* [Reserve Static IP in Azure](https://azure.microsoft.com/en-au/resources/videos/azure-friday-how-to-reserve-a-public-ip-range-in-azure-using-public-ip-prefix/)

### CI/CD tool configuration (Jenkins Server)

* [Install Jenkins](https://www.jenkins.io/doc/book/installing/linux/)

Make sure service is running
```bash
sudo systemctl status jenkins
netstat -anlp | grep 8080 # needs package net-tools
```

#### Step 1: Configure Shell
Go to Dashboard / Manage Jenkins / Configure System / Shell / Shell Executable and type '/bin/bash'

#### Step 2: Add webhooks both to django and ansible repositories
[Dublicate](https://docs.github.com/en/github/creating-cloning-and-archiving-repositories/creating-a-repository-on-github/duplicating-a-repository) repositories for easier configuration.

* [Add Webhooks - see until Step 4](https://www.blazemeter.com/blog/how-to-integrate-your-github-repository-to-your-jenkins-project)

#### Step 3: Add the credentials needed

* [Add SSH keys & SSH Agent plugin](https://plugins.jenkins.io/ssh-agent/) with id 'ssh-ansible-vm' to access ansible-vm, and 'ssh-docker-vm' to access docker-vm
* [Add Secret Texts](https://www.jenkins.io/doc/book/using/using-credentials/) for every environmental variable we need to define in our projects during deployment, like below

```vim
# ID                What is the value?
psql-user           a username you choose for the db user
psql-pass           a password you choose for the db user
psql-db             a name you choose for your database - must be aligned with the db-urls below
django-key          the django secret key - can be a random string
ansible-db-url      'postgresql://<db-user-name>:<db-user-password>@localhost/<db-name>'
ansible-hosts       the domain name for your ansible vm
docker-db-url       'postgresql://<db-user-name>:<db-user-password>@db/<db-name>'
docker-hosts        the domain name for your docker vm
docker-image        the docker image as it is named in Dockerhub (e.g. belpanos/django-movies)
docker-user         your username for Dockerhub
docker-pass         your password for Dockerhub
k8s-db-url          postgresql://<db-user-name>:<db-user-password>@pg-cluster-ip/<db-name> # NO QUOTES TO AVOID PROBLEMS
k8s-hosts           the domain name for your k8s vm
```

#### Create Jobs
* [Create Freestyle project for Ansible code](https://www.guru99.com/create-builds-jenkins-freestyle-project.html)
* [More for Ansible](https://github.com/panagiotisbellias/ansible-movie-code/blob/main/README.md)
* [Create Pipeline project](https://www.jenkins.io/doc/pipeline/tour/hello-world/)
* [Add Webhooks to both jobs - see until Step 9](https://www.blazemeter.com/blog/how-to-integrate-your-github-repository-to-your-jenkins-project)

In the django job the pipeline will be the [Jenkinsfile](Jenkinsfile)


### Deployment with pure Ansible

In order to be able to use Ansible for automation, there is the [ansible-movie-project](https://github.com/panagiotisbellias/ansible-movie-code.git). There is installation and usage guide.

* [More Details](https://github.com/panagiotisbellias/ansible-movie-code/blob/main/README.md)

### Deployment with Docker and docker-compose using Ansible

In order to deploy our project in Docker environment, we use again the [ansible-movie-project](https://github.com/panagiotisbellias/ansible-movie-code.git) where we use a playbook that uses an Ansible role to run the application with docker-compose according to the [docker-compose.yml](docker-compose.yml). In that file, we have defined three services, the postgres container with its volume in order to be able to store data, the django container for our app taking environmental variables from local .env file (it's ready when we run the playbook from jenkins-server where the sensitive values from environmental variables are parametric). The django container is built according to the [nonroot.Dockerfile](nonroot.Dockerfile) as a nonroot process for safety reasons. Also, the nginx container is defined to start so as to have a web server in front of django container and to be able to pass SSL certificates for HTTPS environment. For the HTTPS part we will talk about [later](https://github.com/panagiotisbellias/e-movies-app#in-docker-environment).

* [More Info Here](https://github.com/panagiotisbellias/ansible-movie-code/blob/main/README.md)

### Deployment using Kubernetes and a few things from Ansible

In order to deploy our project in Kubernetes cluster, we first need to connect to that VM so as to configure a better connection between local PC or jenkins server and deployment vm's:

* [installing microk8s](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#2-deploying-microk8s)
* Do this trick to write less in terminal
```bash
echo "alias k='microk8s.kubectl' " >> ~/.profile
```
The permanent alias will be applied only if you reconnect to your VM.

#### Cluster Configuration & Enable Addons
```bash
sudo usermod -a -G microk8s <your-username>
sudo chown -f -R <your-username> ~/.kube
microk8s enable dns dashboard storage ingress
microk8s status
```

#### Connect Kubernetes Cluster with Local PC or/and Jenkins server
```bash
# VM's terminal
k config view --raw > kube-config
cat kube-config

# Local terminal
mkdir ~/.kube
scp <vm-name>:/home/<vm-username>/kube-config ~/.kube/config
```
Edit ~/.kube/config to replace the 127.0.0.1 with the VM's public ip and the certificate line in clusters section with the below line (not used this way in a real production environment)
```bash
insecure-skip-tls-verify: true
```

* Don't forget to add a firewall rule for the port specified in the ~/.kube/config file
With
```bash
kubectl get pods
```
you can ensure that the connection is established.

If you use CI/CD tool and mostly Jenkins do the following (for better deployment fork the repository to be able to change code where needed):
```bash
# Jenkins terminal

```

#### Kubernetes Entities
Either manually or via jenkins server using Jenkinsfile and secret texts the following will do the trick! The code is located in [k8s](k8s) folder. (The .yaml files)

* Don't forget to have a docker image in DockerHub with the project because the deployment entity for django uses it. You can follow the logic located in Jenkinsfile in the 'Preparing k8s Deployment' stage. You must have docker installed in your local machine (or jenkins server)

* [Docker Image](https://hub.docker.com/repository/docker/belpanos/django-movies)

```bash
# Persistent Volume Claim
kubectl apply -f db/postgres-pvc.yaml

# Secret (for the postgresql database)
kubectl create secret generic pg-user \
--from-literal=PGUSER=<put user name here> \
--from-literal=PGPASSWORD=<put password here>

# Config Map (for .env variables)
vim movies_app/movies_app/.env # change to the correct values
kubectl create configmap django-config --from-env-file=movies_app/movies_app/.env

# Deployments
kubectl apply -f db/postgres-deployment.yaml
kubectl apply -f django/django-deployment.yaml

# Services (Cluster IPs)
kubectl apply -f db/postgres-clip.yaml
kubectl apply -f django/django-clip.yaml

# Ingress (For just HTTP - Edit file changing host to your own dns name)
kubectl apply -f django/django-ingress.yaml

# For possible errors with init-containers & migrations do this
kubectl get pods # to see the full name of django pod (e.g. django-r4nd0m-str1n0)
kubectl exec -it <pod name> bash # to apply migrations manually inside pod

# Now inside the container's bash shell
python manage.py makemigrations && python manage.py migrate ## to migrate database
python manage.py createsuperuser # and answer the prompts in case you want to have an admin present
exit # or press ctrl-D to exit container's bash
```

## Creating Domain Names
### DNS Zone

### A and CNAME records

## Installing SSL Certificates
### in pure Ansible environment

### in Docker environment

### in Kubernetes environment

# Extra things for exploration
* [Using Visual Studio Code with WSL](https://code.visualstudio.com/docs/remote/wsl)
* [Install Docker](https://dev.to/semirteskeredzic/docker-docker-compose-on-ubuntu-20-04-server-4h3k)
* [k9s tool - handle kubernetes clusters](https://github.com/derailed/k9s)
* [Static files in Kubernetes - whitenoise](http://whitenoise.evans.io/en/stable/)