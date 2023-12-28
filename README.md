## register-app DevOps CICD Pipeline instruction

================= Install and Configure the Jenkins-Master & Jenkins-Agent =====================
## Create EC2 instance named as "Jenkins-Master". It must have t3.medium as instance type and ROM as 20GB as we will have Jenkins with multiple installation & plugins in it.
$ sudo apt update
$ sudo apt upgrade
$ sudo nano /etc/hostname
$ sudo init 6
$ sudo apt install openjdk-17-jre
$ java -version

$ sudo nano /etc/ssh/sshd_config
Uncomment :
=> PubkeyAuthentication 
=> AuthorizedKeysFile
$ sudo service sshd reload


## Install Jenkins (to copy Jenkins install command, Refer to this link: https://www.jenkins.io/doc/book/installing/linux/)

curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins

$ sudo systemctl enable jenkins       //Enable the Jenkins service to start at boot
$ sudo systemctl start jenkins        //Start Jenkins as a service
$ systemctl status jenkins

## Create EC2 instance named as "Jenkins-Agent". we can take t2.micro instance here.
$ sudo apt update
$ sudo apt upgrade
$ sudo nano /etc/hostname
$ sudo init 6
$ sudo apt install openjdk-17-jre
$ java -version

$ sudo apt-get install docker.io
$ sudo usermod -aG docker $USER
$ sudo init 6

$ sudo nano /etc/ssh/sshd_config
Uncomment :
=> PubkeyAuthentication 
=> AuthorizedKeysFile
$ sudo service sshd reload

## Go to Jenkins-Master ubuntu machine terminal,
$ ssh-keygen -t ed25519
$ cd .ssh
Copy id_ed25519.pub file content. 

## Go to Jenkins-Agent ubuntu machine terminal,
$ cd .ssh
$ vi authorized_keys
Now paste that id_ed25519.pub public ssh key of Jenkins-Master to authorized_keys of Jenkins-Agent.

=> Now copy ip address of Jenkins-Master and open it in terminal with port 8080, to get Jenkins UI.

Now go to Jenkins-Master ubuntu machine terminal,
$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Copy the password and use it in Jenkins UI. Install suggested plugins.
Provide firstname, username/password, email, etc. (say I gave, username/password as clouduser). Now go to,

Manage Jenkins -> Nodes -> Build-In Node-> configure-> Set “Number of executors” = 0 -> Save.

Manage Jenkins -> Nodes -> New Node -> create new node with name “Jenkins-Agent” -> click on radio button of ‘permanent agent’ -> create.

#Provide below details inside Jenkins-Agent configure.
Name = Jenkins-Agent
Description = Jenkins-Agent
Number of executors = 2
Remote root directory = /home/ubuntu
Lable = Jenkis-Agent
Usage = Use this node as much as possible
Launch method = Launch agents via SSH
Host = provide private ip address of Jenkins-Agent EC2-instance.
Credentials : click on ADD.
  kind = SSH username with private key.
  ID = Jenkins-Agent
  Description = Jenkins-Agent
  Username = ubuntu
  Enter directly private key = provide private key of Jenkins-Master here in text form. use cat command at .ssh directory of ubuntu and paste it here. (id_ed25519)
  click on 'save'.
  Now select this credentials.
Host Key Verification Strategy = Non verifying verification strategy
Availability = Keep this agent as much as possible
Click on 'save' button.

==============Integrate Maven, JDK, GITHUB to Jenkins==============

Dashboard -> Manage Jenkins -> Plugins -> search for 'Maven integration plugin', 'Pipeline maven integration plugin', and 'Eclipse Temurin installer Plugin'.

Dashboard -> tools -> Maven Installations -> add maven ->
Name = Maven3
Give tick mark on 'Install automatically' -> apply -> save.

Dashboard -> tools -> JDK installations -> add jdk ->
Name = Java17
Give tick mark on 'Install automatically' -> add installer -> install from adoptium.net -> in version, select jdk-17.0.5+8 -> apply -> save.

Dashboard-> Manage Jenkins -> Credentials -> Stores scoped to Jenkins -> global -> add credentials.
username = provide github username (AadityaUoHyd)
password = provide access token of your github(github settings -> developer settings -> personal access token -> token classic -> generate new token, OR use older one)

id = github
description = github
click on save.

==================Creating Jenkins Pipeline==================

Go to dashboard -> new item -> enter item name (say, register-app-ci), click on pipeline -> ok
give tick mark on "Discard old builds" -> provide value as '2' at "Max# of builds to keep" 
under pipeline, select "pipeline script from SCM"
provide "git" as SCM,
provide your own github repo url of register-app at "Repository URL".
scriptpath = jenkinsfile
apply -> save.

You can now, test the pipeline with -> "Build now".

================= Install and Configure the SonarQube in new ubuntu EC2 SonarQube machine ==================
Create new EC2 instance with name "SonarQube". It must have t3.medium as instance type as we will have sonarqube and postgresql in it.
## Update Package Repository and Upgrade Packages
    $ sudo apt update
    $ sudo apt upgrade
## Add PostgresSQL repository
    $ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    $ wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
## Install PostgreSQL
    $ sudo apt update
    $ sudo apt-get -y install postgresql postgresql-contrib
    $ sudo systemctl enable postgresql
## Create Database for Sonarqube
    $ sudo passwd postgres
    $ su - postgres
	(say password is, postgres)
    $ createuser sonar
    $ psql 
    $ ALTER USER sonar WITH ENCRYPTED password 'sonar';
    $ CREATE DATABASE sonarqube OWNER sonar;
    $ grant all privileges on DATABASE sonarqube to sonar;
    $ \q
    $ exit
## Add Adoptium repository
    $ sudo bash
    $ wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
    $ echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
 ## Install Java 17
    $ apt update
    $ apt install temurin-17-jdk -y
    $ update-alternatives --config java
    $ /usr/bin/java --version
    $ exit 
## Linux Kernel Tuning
   # Increase Limits
    $ sudo vim /etc/security/limits.conf
    //Paste the below values at the bottom of the file
    sonarqube   -   nofile   65536
    sonarqube   -   nproc    4096

    # Increase Mapped Memory Regions
    sudo vim /etc/sysctl.conf
    //Paste the below values at the bottom of the file
    vm.max_map_count = 262144

#### Sonarqube Installation ####
## Download and Extract
    $ sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
    $ sudo apt install unzip
    $ sudo unzip sonarqube-9.9.0.65466.zip -d /opt
    $ sudo mv /opt/sonarqube-9.9.0.65466 /opt/sonarqube
## Create user and set permissions
     $ sudo groupadd sonar
     $ sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
     $ sudo chown sonar:sonar /opt/sonarqube -R
## Update Sonarqube properties with DB credentials
     $ sudo vim /opt/sonarqube/conf/sonar.properties
     //Find and replace the below values, you might need to add the sonar.jdbc.url
     sonar.jdbc.username=sonar
     sonar.jdbc.password=sonar
     sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
## Create service for Sonarqube
$ sudo vim /etc/systemd/system/sonar.service
//Paste the below into the file
     [Unit]
     Description=SonarQube service
     After=syslog.target network.target

     [Service]
     Type=forking

     ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
     ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

     User=sonar
     Group=sonar
     Restart=always

     LimitNOFILE=65536
     LimitNPROC=4096

     [Install]
     WantedBy=multi-user.target

## Start Sonarqube and Enable service
     $ sudo systemctl start sonar
     $ sudo systemctl enable sonar
     $ sudo systemctl status sonar
	 
# Grab public ip of EC2 instance of sonarqube and hit the url with 9000 port. 
say, http://3.110.104.53:9000/
default username/password of sonarqube is admin/admin.
(say rename the password with, 'sonarqube')

Go to myaccount -> security -> provide "jenkins-sonarqube-token" as Generate token name, select "Global Analysis Token", expiration as "no expiration".
Generate the sonarqube-token and copy it for future use.

Now go to Manage Jenkins -> add credentials -> 
kind = secret text
secret = <paste here the newly generated sonarqube-token>
id = jenkins-sonarqube-token
description = jenkins-sonarqube-token
click on create.

Now go to manage jenkins -> plugins.
Install "SonarQube Scanner", "Sonar Quality Gates", "Quality Gates".

Now go to Dashboard -> Manage Jenkins -> System -> SonarQube Servers -> 
Name = sonarqube-server
Server URL = privateIP of EC2 instance of sonarqube machine with 9000 port
Server authentication token = jenkins-sonarqube-token
apply -> save.

Now go to Dashboard -> Manage Jenkis-> Tools -> SonarQube Scanner installations -> Add sonarqube scanner.
SonarQube Scanner = sonarqube-scanner
let remain Install automatically as tick mark.
apply -> save.

Now go to sonarqube portal and click on Administration -> configuration -> webhooks -> click on create.
name = sonarqube-webhook
url = http://172.31.34.149:8080/sonarqube-webhook/             (here we have given, private ip address of jenkins-master as 172.31.34.149)
click on create.

Test your pipeline with build now, but before that ensure you've written appropriate 'jenkinsfile' in your github(register-app repo).

===============Docker=====================
Install docker related 6 plugins in Jenkins. Go to Dashboard -> Manage Jenkins -> Plugins
click on available plugins -> search name with "docker". select these 6 plugins with tick mark.
Docker, Docker Commons, Docker pipeline, Docker API, Docker-build-step, CloudBees Docker Build and Publish.
click on install.

Now you need to add your docker credentials into jenkins.
Dashboard -> Manage jenkins -> Credentials -> Score scoped to jenkins -> Add credentials 
Kind = username and password
username = provide username of your dockerhub account (say, aadiraj48dockerhub)
password = accesstoken of your dockerhub account (if you dont have any, create one at dockerhub account.)
id = give some label of the credentails(have to provide same name in jenkinsfile of github. Say, dockerhub) 


============================================================= Setup Bootstrap Server for eksctl and Setup Kubernetes using eksctl =============================================================
Create one EC2 instance with name - EKS-Bootstrap-Server (with t3.medium instance type). connect to ec2 with mobaxterm.
# sudo apt-get update
# sudo apt-get upgrade -y
# sudo vi /etc/hostname
rename it as EKS-K8s-Cluster.
# sudo su
# cd ~

## Install AWS Cli on the above EC2 (Refer--https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ apt install unzip
$ unzip awscliv2.zip
$ sudo ./aws/install

         OR
		 
$ sudo yum remove -y aws-cli
$ pip3 install --user awscli
$ sudo ln -s $HOME/.local/bin/aws /usr/bin/aws

$ aws --version

## Installing kubectl (Refer--https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
$ curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
$ ll 
$ chmod +x ./kubectl  //Gave executable permisions
$ mv kubectl /bin   //Because all our executable files are in /bin
$ kubectl version --output=yaml

## Installing  eksctl (Refer---https://github.com/eksctl-io/eksctl/blob/main/README.md#installation)
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ cd /tmp
$ ll
$ sudo mv /tmp/eksctl /bin
$ eksctl version

Now create a IAM role. IAM -> Roles -> create role -> AWS service -> 
In use-case, services or usecase = select 'EC2' -> next
tick mark on 'AdministratorAccess' -> next
Role name = eksctl_role
click on 'create role'.

Now go to EKS-K8s-Cluster EC2 machine -> Actions -> security -> modify IAM role.
select eksctl_role -> update IAM role.

## Setup Kubernetes using eksctl (Refer--https://github.com/aws-samples/eks-workshop/issues/734)
$ eksctl create cluster --name aadi-eks-cluster --region ap-south-1 --node-type t2.small --nodes 3

$ kubectl get nodes

============================================================= ArgoCD Installation on EKS Cluster and Add EKS Cluster to ArgoCD =============================================================
1 ) First, create a namespace
    $ kubectl create namespace argocd

2 ) Next, let's apply the yaml configuration files for ArgoCd
    $ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

3 ) Now we can view the pods created in the ArgoCD namespace.
    $ kubectl get pods -n argocd

4 ) To interact with the API Server we need to deploy the CLI:
    $ curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
    $ chmod +x /usr/local/bin/argocd

5 ) Expose argocd-server
    $ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

6 ) Wait about 2 minutes for the LoadBalancer creation
    $ kubectl get svc -n argocd
    Now copy the URL of the external load-balancer from output. (say, a3dab89c077be4475ae70559e0e1a798-1215184296.ap-south-1.elb.amazonaws.com)

7 ) Get pasword and decode it.
    $ kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
	Now copy the password for future use. (say, c3dudDBFVFJvTXpDVUJvRA==)
	$ echo c3dudDBFVFJvTXpDVUJvRA== | base64 --decode
	Copy the decoded passcode. (say, swnt0ETRoMzCUBoD)
	
Now use the load-balancer url and hit the browser. use username as 'admin', and passcode as what we decoded just now.(say, swnt0ETRoMzCUBoD)
After loggin into argoCD, Go to User Info -> update password -> provide old passcode and give new one. (say, admin123).
Now logout and login with new password.

## Add EKS Cluster to ArgoCD
9 ) login to ArgoCD from CLI
    $ argocd login a3dab89c077be4475ae70559e0e1a798-1215184296.ap-south-1.elb.amazonaws.com --username admin

Now it'll ask for password, provide your newly updated argocd password to login via CLI.

10 ) to check available argocd clusters.
     $ argocd cluster list

11 ) Below command will show the EKS cluster
     $ kubectl config get-contexts
	 copy namespace from here. it'll be used while adding argocd to eks-cluster.

12 ) Add above EKS cluster to ArgoCD with below command
     $ argocd cluster add i-0257c91f29e589e2f@aadi-eks-cluster.ap-south-1.eksctl.io --name aadi-eks-cluster
	 $ argocd cluster list
	 
13 ) Now add your github repo (which contain k8s deploment files),
click on argocd settings -> repositories -> connect repo
choose your connection method via HTTPS
provide github repository (say, https://github.com/AadityaUoHyd/gitops-register-app)
provide your github username (AadityaUoHyd) and password as erstwhile created github token (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).
click on 'connect'.

14 ) Go to argocd. Application -> New App 

Application name = register-app
project name = default
sync policy = automatic
put tick mark on -> Prune Resources and Self-Heal.

In source, select your github repo from drop-down menu.
           give path as ./
		  
In destination, select your eks cluster url from drop-down menu. (ignore the default one)
                namespace = default
				
click on 'create'.

Now run these K8s command on eks cluster, which been added to argocd.
$ kubectl get pods
$ kubectl get svc

Now copy the load-balancer external-ip (say, a745211521a3140a0844369c2400389e-873276740.ap-south-1.elb.amazonaws.com).
Go to browser and hit the url with => a745211521a3140a0844369c2400389e-873276740.ap-south-1.elb.amazonaws.com:8080/webapp/
You'll find your register-app is running successfully.

# now automate this deployment process.
Go to jenkins dashboard -> new item ->
enter an item name = gitops-register-app-cd
pipeline -> ok.

Discard old build -> max# of builds to keep = 2

click tick mark on, this project is parameterized. click on add parameter. select, 'string parameter'.
Name = IMAGE_TAG

Tick mark on, Triggers build remotely. "gitops-token" as AuthenticationToken.

Pipeline -> pipeline script from SCM. select GIT as SCM.
"https://github.com/AadityaUoHyd/gitops-register-app" as Repository URL.
select your Github credentials from drop-down menu.
Chose your correct branch (mine is master, which is anyway default). 
Apply -> Save.

Go to Jenkins -> click on username (clouduser) menu -> configure -> API Token -> Add new token -> provide default name (say, JENKINS_API_TOKEN) -> Generate.
copy this jenkins token for future use.
apply -> save.

Dashboard -> Manage Jenkins -> Credentials -> System -> Global credentials (unrestricted)
kind = secret text
secret = provide newly created jenkins token.
provide "JENKINS_API_TOKEN" at ID and Description. click on create.

Dashboard -> register-app-ci -> Configuration -> Build Triggers 
Tick mark on poll SCM. give 5 star sign.(it'll now monitor every minute to trigger build (Continuous Integration) as per any changes in github repo of register-app, and further it will trigger Continuous Delivery from gitops-register-app repo using AgroCD)
Apply -> save.
