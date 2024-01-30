## System Setup

step1. install java package

```bash

sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
```

step2. install jenkins

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

step3. check jenkins service status

```bash

sudo systemctl status jenkins
```

! important

if configuration done inside ec2 instance then need to open the port 8080 to the open internet using ec2 instance security group

step4. access the jenkins server

* get default password
```sudo cat /var/lib/jenkins/secrets/initialAdminPassword```

* first install the default plugins

* create a admin user to login after initial setup

step5. Install the Docker Pipeline plugin in Jenkins

our cicd configuration is to manage worker nodes using docker , which is more scalable when it comes to large developments , since we don't need to maintain continouse worker nodes like EC2 instances , as well as no need to maintain the worker enviroments. Using docker is more flexible because , for each cicd job developers can use any specific docker image and configure the enviroment.


1. Log in to Jenkins.
2. Go to Manage Jenkins > Manage Plugins.
3. In the Available tab, search for "Docker Pipeline".
4. Select the plugin and click the Install button.
5. Restart Jenkins after the plugin is installed.

step6. Docker Slave Configuration

* install docker

```bash

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

step7. Grant Jenkins user and Ubuntu user permission to docker deamon.

```bash

sudo su - 
usermod -aG docker jenkins
usermod -aG docker ubuntu
systemctl restart docker
```

step8. Once you are done with the above steps, it is better to restart Jenkins.

### jenkins basic setup completed ...


create a webhook


step1. go to the github project -> settings -> Webhooks

step2. click add webhook

```
Payload URL -> http://<Jenkins server public ip>:<jenkins port>/github-webhook/
Content type -> application/json
```
step3. save the configuration

### configure EC2 instance IAM role to push docker images to ECR

aws quote:
```
We designed IAM roles so that your applications can securely make API requests from your instances, without requiring you to manage the security credentials that the applications use. Instead of creating and distributing your AWS credentials, you can delegate permission to make API requests using IAM roles as follows:
```
follow the instructions here : https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html

after creating the IAM Role we can attach the Role to the already created EC2 instance

steps:  go to main ec2 console -> select running instance -> click Actions from top right bar -> click security -> click modify IAM role -> attach the IAM role

### steup AWS EC2 instance for ECR push

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin 630210676530.dkr.ecr.us-east-1.amazonaws.com
sudo docker tag hello-world:latest 630210676530.dkr.ecr.us-east-1.amazonaws.com/ai-cicd
sudo docker push 630210676530.dkr.ecr.us-east-1.amazonaws.com/ai-cicd

```

### setup sonarqube server

```bash


adduser sonarqube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.4.0.54424.zip
unzip *
chmod -R 755 /home/sonarqube/sonarqube-9.4.0.54424
chown -R sonarqube:sonarqube /home/sonarqube/sonarqube-9.4.0.54424
cd sonarqube-9.4.0.54424/bin/linux-x86-64/
./sonar.sh start
```

### setup sonar scaner cli

```bash

wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip sonar-scanner-cli-5.0.1.3006-linux.zip
export PATH=$PATH:/home/ubuntu/sonar-scanner-5.0.1.3006-linux/bin/
cd AI-CICD-Guide/
```

* run sonar scannaer

```bash

sonar-scanner \
  -Dsonar.projectKey=jenkins-cicd \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://34.192.92.49:9000 \
  -Dsonar.login=13a3bd852307a5651d331d749e0cbe7890483203
```

