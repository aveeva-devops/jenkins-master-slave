# jenkins-master-slave setup

It is pretty common when starting with Jenkins to have a single server which runs the master and all builds, however Jenkins architecture is fundamentally "Master+Agent". The master is designed to do co-ordination and provide the GUI and API endpoints, and the Agents are designed to perform the work. The reason being that workloads are often best "farmed out" to distributed servers.  



# Introduction
This document explains jenkins master slave setup on docker containers. 

### Steps implemented

* Launch two EC2 nodes for jenkins master and slave
* Configure required softwares
* Configure Jenkins Master
* Configure two Jenkins Slaves
* Jenkins Jobs setup

# Traditional Vs Modern way of setup
![Alt text](images/Jenkins_Master_Slave.png?raw=true "Title")


## Assumptions

* Install terraform on local - https://www.terraform.io/intro/getting-started/install.html 
* Learn basics of aws and terraform - Watch https://www.youtube.com/watch?v=XuQnaqm725w&t=7s 
* Knows basics of docker commands - Watch https://www.youtube.com/watch?v=hB-oMjTZ77c&t=23s  
* Open appropriate ports - Terraform from this repo can be used for this:
  - 2376 - Between master and slave
  - 22 - Between master and slave
  - 8080 - From everywhere 
* Install Docker 
* Enable Docker API
* Create Jenkins user and add in sudoers and docker group

## EC2 Servers setup 

Execute below steps:
* Create an aws account at https://aws.amazon.com/
* Create IAM user and note down access key and secret key
* Create key-pair and download the keypair
* Clone git repo and execute below steps:

```
git clone git@github.com:aveeva-devops/jenkins-master-slave.git

```

#### Usage

terraform.tfvars holds variables which should be overriden with valid ones. First two values are AWS Access key and secret key. Create IAM user and get these values. Last two values are aws_key_pair. Create AWS key-pair from EC2 -> Network and Security -> key-pair.
Change permissions of .pem file to 400, and mention key_name without .pem extension. Execute next two steps to create resources.

#### Plan

This will display all the command terraform will execute. This is a dry run.

cd checkoutcode/infra
update terraform.tfvars with appropriate details

```
terraform plan -var-file terraform.tfvars
```

#### Apply

This will create all the resources mentioned in tf scripts. Ensure to provide valid AMI ID's and your desired aws-region.

```
terraform apply -var-file terraform.tfvars
```

#### Destroy

To destroy the resources, execute terraform destroy

```
terraform destroy -var-file terraform.tfvars
```

Above step will provision two EC2 servers in public subnet.


## Software and user setup

Login on EC2 master and slave servers using below command 

```
ssh -i name_of_pem_file.pem ec2-user@IP

```

#### Install docker

```
sudo yum install docker
sudo service docker start

```

#### Create Jenkins user

```
sudo useradd -m -d /home/jenkins jenkins

```
#### Add jenkins in docker group to perform docker operations

```
sudo usermod -a -G docker jenkins

```
Note down Jenkins user UID

```
id jenkins

```
#### Create password for Jenkins user 

```
sudo passwd jenkins

```

#### Enable Password login on EC2. Edit sshd configuration file /etc/ssh/sshd_config as root.
Set PasswordAuthentication value to 'yes'

```
sudo vi /etc/ssh/sshd_config

```

Restart sshd service

```
sudo service sshd restart

```

## Jenkins Master Setup

Once softwares and user setup is complete, execute below command to setup Jenkins master docker container.
Change ownership of /var/lib/jenkins to jenkins:jenkins
Switch to jenkins user
and start Jenkins master as a docker container:

```
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo su jenkins
docker run --name <Name_Of_Container> —u <uid-jenkins> -d -v <Host_directory>:<container_dir> -p jenkins_port:jenkins_port -p exposed_port:exposed_port jenkins/jenkins:lts

```
I used below command for jenkins:

```
docker run --name Jenkins_Master -u 501 -d -v /var/lib/jenkins:/var/jenkins_home -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```


Where 
  - docker run - to start docker container
  - --name  - Name of container
  - -u <uid-jenkins> enter uid of jenkins id (get using id jenkins)
  - -v volume - source:destination /var/lib/jenkins:/var/jenkins_home
  - -p host port
  - -p docker port
  - jenkins/jenkins:lts - docker image
  
After this command jenkins master is running docker container, to validate execute below command

```
docker ps | grep jenkins
```

Open "hostip:8080" on browser

Copy jenkins password from (inside the container). To login on container, execute below command

```
docker exec -it Jenkins_Master bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

When promted, install suggested plug-ins and create admin user/password

Install "docker plugin" from available plugin's and restart jenkins.

Go to Manage Jenkins -> Manage Plug-ins -> Search for Docker -> Install

Master Jenkins setup is complete now !!


## Jenkins Slave Setup

Login on second node (slave node) and enable docker API on port 2376.
Docker API is enabled differently on each operating system. We are targetting linux ec2 here.
Update /etc/sysconfig/docker with below contents. 

```
sudo vi /etc/sysconfig/docker
```
Update Options line with below contents:

```
OPTIONS="--default-ulimit nofile=1024:4096 -H tcp://privateip:2376 -H unix:///var/run/docker.sock"
```
Where privateip is private IP of slave node.
Restart docker service

```
sudo service docker restart
```

Check whether docker service is up and running or not at port 2376

```
ps -ef | grep docker
```

Login on Jenkins master on browser (masterIP:8080), and setup slaves via UI

Go to Manage Jenkins -> Configure System -> Add a new Cloud and enter details as per images below:


![Alt text](images/Jenkins_Slave_Setup_Image1.png?raw=true "Docker Cloud")

![Alt text](images/Jenkins_Slave_Setup_Image2.png?raw=true "Docker Template for slave")

![Alt text](images/Jenkins_Slave_Setup_Image3.png?raw=true "Docker Template for slave")

Using above method, you can add as many Jenkins Slaves as you want. Slaves can be configured on same host or different hosts


## Jenkins Job Setup




