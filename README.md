# Hello World CI/CD Sample
## CI/CD process developed using Jenkins, SonarQube, and Kubernetes.


I will cover the CI/CD process of a simple Java application built using SpringBoot,Maven.
In this process, we will see that the code is artifacted and built with Maven, and the code is scanned on SonarQube.

#### ✨Prerequisites
- Jenkins Installation
- SonarQube Installation
- Jenkins SonarQube Integration
- Creating a subscription on DockerHub
- Creating the GitHub account
- Kubernetes Cluster

## Jenkins Installation
>Let's start our project by installing the Jenkins application that we will manage our CI/CD process and the plugins to be used.
- I preferred an EC2 instance of Ubuntu 20.04 in a cloud environment.
[If you have problems with the installation, you can get help for the installation here.][df1]

- To use this repository, first add the key to your system:
```sh
  curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```
- Then add a Jenkins apt repository entry:
```sh
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
```
- Update your local package index, then finally install Jenkins:
```sh
  sudo apt-get update
  sudo apt-get install fontconfig openjdk-11-jre
  sudo apt-get install jenkins
```
- The apt packages were signed using this key:
```sh
  pub   rsa4096 2020-03-30 [SC] [expires: 2023-03-30]
      62A9756BFD780C377CF24BA8FCEF32E745F2C3D5
uid                      Jenkins Project 
sub   rsa4096 2020-03-30 [E] [expires: 2023-03-30]
```
- You can enable the Jenkins service to start at boot with the command:
```sh
sudo systemctl enable jenkins
```
- You can start the Jenkins service with the command:
```sh
sudo systemctl start jenkins
```
- Browse to http://<IP_ADDRESS>:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears.

![](images/Unlock%10Jenkins.png)


## SonarQube Installation
>Now that we have completed the Jenkins installation, we can start the SonarQube installation. Later, we will communicate these two applications with each other.

- I preferred the EC2 instance of Ubuntu 20.04 in a cloud environment as with the Jenkins installation.

- You can set them dynamically for the current session by running the following commands as root:
```sh
apt update && apt upgrade -y

sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```
- If the user running SonarQube (sonarqube in this example) does not have the permission to have at least 131072 open descriptors, you must insert this line in /etc/security/limits.d/99-sonarqube.conf (or /etc/security/limits.conf as you wish):
```sh
sudo nano /etc/security/limits.conf

 sonarqube   -   nofile   65536
 sonarqube   -   nproc    4096
```
- Unzip, Openjdk and PostgreSQL (which I prefer) are needed, so they are installed and configured.
```sh
sudo apt-get install wget unzip -y

sudo apt-get install openjdk-11-jdk -y
sudo apt-get install openjdk-11-jre -y

wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O- | sudo apt-key add -
echo "deb [arch=amd64] http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main" | sudo tee /etc/apt/sources.list.d/postgresql.list

sudo apt update
sudo apt-get -y install postgresql postgresql-contrib
```
- You can enable the PostgreSQL service to start at boot with the command:
```sh
sudo systemctl enable postgresql
```
- You can start the PostgreSQL service with the command:
```sh
sudo systemctl start postgresql
```
- To create user and database:
```sh
sudo passwd postgres
su - postgres
createuser sonaruser
psql
ALTER USER sonaruser WITH ENCRYPTED password 'sonaruser';
CREATE DATABASE sonardb OWNER sonaruser;
grant all privileges on DATABASE sonardb to sonaruser;
```
- Once the system requirements are complete, we can begin the SonarQube installation.
```sh
 wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.3.0.51899.zip
 sudo unzip sonarqube-*.zip -d /opt
 sudo mv /opt/sonarqube-* /opt/sonarqube

 useradd -M -d /opt/sonarqube/ -r -s /bin/bash sonarh2s
 chown -R sonarh2s: /opt/sonarqube

 vim /opt/sonarqube/conf/sonar.properties

 sonar.jdbc.username=sonaruser
 sonar.jdbc.password=sonaruser
 sonar.jdbc.url=jdbc:postgresql://localhost/sonardb

 sudo vim /etc/systemd/system/sonar.service
```
- SonarQube is configured as a service:
```sh
 sudo vim /etc/systemd/system/sonar.service
 
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=simple
User=sonarh2s
Group=sonarh2s
PermissionsStartOnly=true
ExecStart=/bin/nohup java -Xms32m -Xmx32m -Djava.net.preferIPv4Stack=true -jar /opt/sonarqube/lib/sonar-application-9.3.0.51899.jar
StandardOutput=syslog
LimitNOFILE=131072
LimitNPROC=8192
TimeoutStartSec=5
Restart=always
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```
- You can restart daemon with the command:
```sh
sudo systemctl daemon-reload
```
- You can enable the SonarQube service to enable at boot with the command:
```sh
sudo systemctl enable sonar
```
- You can start the SonarQube service with the command:
```sh
sudo systemctl start sonar
```
- After the necessary configurations are completed, http://server-ip-address:9000 is used to access SonarQube's UI.
Username: admin     /       Password: admin

##### SonarQube & Jenkins Integration

> We create a token under
> Administration > Security > Users
> tabs on SonarQube UI.

<SONARQUBE TOKEN CREATION RESIM>

Then, on Jenkins, the configurations for SonarQube are defined with the token we created.

<SonarQube Jenkins Integration Resim>


The plugins we need are installed via Jenkins Plugin Manager. Like Docker plugin, Git plugin, SonarQube Scanner for Jenkins, Maven Integration plugin.
In addition to plugins, tokens are created on DockerHub and added under Global Credentials to keep credential information more reliable.

The pipeline structure is configured to run on every change, depending on the relevant branch in the GitHub repository via Jenkinsfile.
Our Pipeline goes to SonarQube code analysis after checking GitHub and the versions of the tools used.
After this stage, our application is turned into a container and sent to [DockerHub].

<PIPILINE STAGES RESIM>

- The infrastructure was created in the form of IaC using Terraform and was not included in the code as it was not suitable for the structure of the project.

#### The containerized application can be run on Kubernetes either using Helm Chart or with deployment. Both options are available in the repository. You can make it meet your needs with a few changes.

- If the application will be run in Kubernetes as a deployment dont forget the expose your service:
```sh
kubectl expose deploy hello-world --type=NodePort --port=80 --target-port=8085
```


| Contact | With me |
| ------ | ------ |
| LinkedIn | [https://www.linkedin.com/in/mtulun/][PlDb] |
| GitHub | [https://github.com/mtulun][PlGh] |
| Email | [taylan.ulun@outlook.com][PlGd] |

 




[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [dill]: <https://github.com/joemccann/dillinger>
   [git-repo-url]: <https://github.com/joemccann/dillinger.git>
   [DockerHub]: <https://hub.docker.com>
   [df1]: https://pkg.jenkins.io/debian-stable/
   [markdown-it]: <https://github.com/markdown-it/markdown-it>
   [Ace Editor]: <http://ace.ajax.org>
   [node.js]: <http://nodejs.org>
   [Twitter Bootstrap]: <http://twitter.github.com/bootstrap/>
   [jQuery]: <http://jquery.com>
   [@tjholowaychuk]: <http://twitter.com/tjholowaychuk>
   [express]: <http://expressjs.com>
   [AngularJS]: <http://angularjs.org>
   [Gulp]: <http://gulpjs.com>

   [PlDb]: <https://www.linkedin.com/in/mtulun/>
   [PlGh]: <https://github.com/mtulun>
   [PlGd]: <https://outlook.live.com/owa/>
