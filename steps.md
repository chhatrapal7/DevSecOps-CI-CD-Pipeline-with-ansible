# Enterprise Java Application CI/CD & Automated Deployment

This document explains the complete setup of an end-to-end Java CI/CD pipeline using:

* GitHub
* Jenkins
* Maven
* SonarQube
* Amazon S3
* Ansible
* Apache Tomcat
* AWS EC2

The project uses **4 EC2 instances**.
---

# 1. Project Objective

The objective of this project is to automate the complete Java application deployment process.

Whenever a developer changes the Java application code and pushes the changes to GitHub:

```text
Developer
    |
    | Push Code
    v
GitHub
    |
    | Jenkins Checkout
    v
Jenkins + Ansible
    |
    +---- Maven Compile
    |
    +---- Maven Test
    |
    +---- Maven Package
    |
    +---- SonarQube Analysis
    |
    +---- WAR Artifact
    |
    +---- Upload Artifact to S3
    |
    +---- Ansible Deployment
             |
             +------------+
             |            |
             v            v
        Tomcat Node 1  Tomcat Node 2
             |            |
             +------ + ---+
                    |
                    v
             Java Web Application
```

# 2. EC2 Instance Details

Create four EC2 instances.

## EC2-1

```text
Name: Jenkins-Ansible
OS: Amazon Linux 2023
Instance: big to t3.micro
volume: 20GB
Purpose:
    Jenkins
    Ansible
    Maven
    Git
```

## EC2-2

```text
Name: SonarQube
OS: Amazon Linux 2023
Instance: t3.medium
Port: 9000
Purpose:
    SonarQube Code Quality Analysis
```

## EC2-3

```text
Name: Tomcat-1
OS: Amazon Linux 2023
Instance: t3.micro
Port: 8080
Purpose:
    Tomcat Worker Node 1
```

## EC2-4

```text
Name: Tomcat-2
OS: Amazon Linux 2023
Instance: t3.micro
Port: 8080
Purpose:
    Tomcat Worker Node 2
```

---

# 3. Security Group Configuration

Make sure the required ports are allowed.

## Jenkins + Ansible EC2

```text
22   SSH
8080 Jenkins
```

## SonarQube EC2

```text
22   SSH
9000 SonarQube
```

## Tomcat Worker Nodes

```text
22   SSH
8080 Tomcat
```

For a lab environment, you can temporarily allow the required ports from your IP or VPC CIDR.

For production, do not expose all ports to:

```text
0.0.0.0/0
```

Prefer restricting access to trusted IPs/security groups.

---

# 4. Project Source Code

The Java project is stored in GitHub.

Repository:

```text
https://github.com/chhatrapal7/java-project-maven-new.git
```

The application is a Maven-based Java web application.

The generated artifact is a:

```text
myapp.war
```
# 5. Step 1 - Configure Jenkins + Ansible Master Server

We will use the **same EC2 instance** for Jenkins and Ansible.

This machine acts as:

```text
Jenkins Server
      +
Ansible Control Node / Master
```

Login to EC2-1.

```bash
sudo -i
```

Update the system:

```bash
dnf update -y
```

---

# 11. Install Ansible

Install Ansible:

```bash
sudo dnf install ansible -y
```

Install Python and pip:

```bash
sudo dnf install python3 python3-pip -y
```

Verify Ansible:

```bash
ansible --version
```
Master to Node Configuration for 
https://github.com/chhatrapal7/ansible-notes/blob/c547d0d8cc5e7e0d1224ae1a01dc35f867fefe46/Ansible-setup.md

---
Our Ansible architecture is:

```text
                 Ansible Master
              Jenkins + Ansible
                     |
             +-------+-------+
             |               |
             v               v
         Tomcat Node 1   Tomcat Node 2
```


# 12. Install Jenkins

Import Jenkins repository/key and install Jenkins according to the current Jenkins installation instructions for Amazon Linux.

```bash
vi jenkins.sh
```
```bash
#STEP-1: Installing Git and Maven
yum install git maven -y

#STEP-2: Repo Information (jenkins.io --> download -- > redhat)
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

#STEP-3: Download Java 21 and Jenkins
sudo yum install java-21-amazon-corretto -y
yum install jenkins -y
sudo mount -o remount,size=2G /tmp
#STEP-4: Start and check the JENKINS Status
systemctl start jenkins.service
systemctl status jenkins.service

#STEP-5: Auto-Start Jenkins
chkconfig jenkins on
```

Jenkins normally runs on:

```text
http://<JENKINS-PUBLIC-IP>:8080
```

Get the initial administrator password:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open Jenkins in the browser and complete the initial setup.

---

# 14. configure Ansible 

We can configure hostnames for identification.

On Jenkins + Ansible server:

```bash
sudo -i
hostnamectl set-hostname ansible
```

On worker node 1:

```bash
sudo -i
hostnamectl set-hostname tomcat1
```

On worker node 2:

```bash
sudo -i
hostnamectl set-hostname tomcat2
```

Verify:

```bash
hostname
```

---

# Configure Root Password

For the lab setup, configure the root password.

Run on the required servers:

```bash
passwd root
```

Enter your chosen password.

Example:

```text
New password: ********
Retype new password: ********
```

> Do not commit this password to GitHub. The password shown in old lab notes such as `reyaz123` or `root123456` should be treated as example/lab credentials only.

---

# Enable Root SSH Login

On the servers where root SSH access is required:

```bash
vi /etc/ssh/sshd_config
```

Find/uncomment the required SSH settings.

Depending on the Amazon Linux configuration, the lines may look like:

```text
PermitRootLogin yes
PasswordAuthentication yes
```

Do not blindly rely on line numbers such as `40` or `65`, because line numbers can change between OS versions.

After editing:

```bash
systemctl restart sshd
```

Check:

```bash
systemctl status sshd
```

---

# 17. Check Private IP Addresses

Run:

```bash
hostname -i
```

Example:

```text
172.31.x.x
```

We need the **private IP addresses of Tomcat Node 1 and Tomcat Node 2**.

Do not use the public IP in the Ansible inventory when the servers communicate inside the same VPC.

---

# Generate SSH Key on Ansible Master

Login to EC2-1.

Run:

```bash
ssh-keygen
```

Press Enter for the default location.

You can press Enter when asked for the passphrase if this is a lab setup and you want passwordless automation.

The key will normally be created under:

```text
/root/.ssh/
```

---

# Copy SSH Key to Tomcat Nodes

From the Ansible Master:

```bash
ssh-copy-id root@<TOMCAT-NODE-1-PRIVATE-IP>
```

Example:

```bash
ssh-copy-id root@172.31.20.40
```

Then:

```bash
ssh-copy-id root@<TOMCAT-NODE-2-PRIVATE-IP>
```

Example:

```bash
ssh-copy-id root@172.31.21.25
```

Enter the root password when prompted.

---

# Test SSH Connection

From the Ansible Master:

```bash
ssh root@<TOMCAT-NODE-1-PRIVATE-IP>
```

Then:

```bash
exit
```

Test Node 2:

```bash
ssh root@<TOMCAT-NODE-2-PRIVATE-IP>
```

Then:

```bash
exit
```

The important point is:

```text
Ansible Master
      |
      | SSH
      |
      +----> Tomcat Node 1
      |
      +----> Tomcat Node 2
```

---

# Configure Ansible Inventory

Create/open:

```bash
vi /etc/ansible/hosts
```

Add:

```ini
[prod]
172.31.20.40
172.31.21.25
```

Replace the example IP addresses with the actual private IP addresses of your Tomcat servers.

You can also make the inventory more descriptive:

```ini
[prod]
tomcat1 ansible_host=172.31.20.40
tomcat2 ansible_host=172.31.21.25
```

---

# Test Ansible

Run:

```bash
ansible -m ping all
```

Expected result:

```text
tomcat1 | SUCCESS => {
    "ping": "pong"
}

tomcat2 | SUCCESS => {
    "ping": "pong"
}
```

This confirms:

```text
Ansible
   |
   +---- SSH ----> Tomcat 1
   |
   +---- SSH ----> Tomcat 2
```

At this point the Ansible Master and both worker nodes are connected.

---

# 23. Step 2 - Configure Jenkins Basic CI Pipeline

Before integrating SonarQube, S3 and Ansible deployment, first verify that Jenkins can successfully:

```text
Checkout
   ↓
Build
   ↓
Test
   ↓
Package
```

Create a Jenkins Pipeline job.

Use:

```text
Pipeline
```

Create a Pipeline script.

---

# Basic Jenkins Pipeline

Use this pipeline first:

```groovy
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/chhatrapal7/java-project-maven-new.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
    }
}
```

Run:

```text
Build Now
```

---

# Understand the Basic Pipeline

### Checkout

```groovy
git 'https://github.com/chhatrapal7/java-project-maven-new.git'
```

Jenkins downloads the source code from GitHub.

---

### Build

```groovy
sh 'mvn compile'
```

Maven compiles the Java source code.

---

### Test

```groovy
sh 'mvn test'
```

Maven executes the test cases.

---

### Package

```groovy
sh 'mvn package'
```

Maven creates the WAR artifact.

Usually it will be created inside:

```text
target/
```

For example:

```text
target/myapp.war
```

---

# Verify WAR Artifact

On Jenkins:

```bash
cd /var/lib/jenkins/workspace/
```

Go to your project workspace.

For example:

```bash
cd /var/lib/jenkins/workspace/project/target
```

Check:

```bash
ls -lh
```

You should see:

```text
*.war
```

The exact WAR name depends on your `pom.xml`.

---

# 27. Jenkins Plugins Required

Now we will integrate all components.

Go to:

```text
Jenkins
→ Manage Jenkins
→ Plugins
```

Install the plugins required by your pipeline.

Recommended plugins/features:

```text
Pipeline
Git
Maven Integration
SonarQube Scanner for Jenkins
Sonar Scanner Quality Gates
RESTART jenkins
Sonar Scanner Quality Gates
Ansible
AWS/S3 Publisher plugin
Credentials
```

For the final pipeline in this document, the `s3Upload` step requires the S3 Publisher plugin that provides that Pipeline DSL.

Also make sure your Jenkins installation has the required dependencies for the selected plugins.

After installation, restart Jenkins if Jenkins requests it.

---

# 28. Configure Maven in Jenkins

Go to:

```text
Manage Jenkins
→ Tools
```

Find:

```text
Maven installations
```

Add Maven if your Jenkins configuration requires a managed Maven installation.

Example:

```text
Name: Maven
```

If Maven is already available on the system and `mvn` works from Jenkins, you can use the system installation.

Test from Jenkins:

```bash
mvn -version
```

---

# 29. Configure Ansible in Jenkins

Go to:

```text
Manage Jenkins
→ Tools
→ Ansible installations
```

Add:

```text
Name: ansible
Path: /bin
```

The exact path should match the location of the `ansible` executable.

Verify on the Jenkins/Ansible server:

```bash
which ansible
```

Example:

```text
/bin/ansible
```

If your command returns a different path, use that actual path.

---

# 30. Configure Ansible Credentials in Jenkins

Jenkins needs credentials to run the Ansible deployment.

Go to:

```text
Manage Jenkins
→ Credentials
→ System
→ Global credentials
→ Add Credentials
```

One possible configuration for the lab is:

```text
Kind:
Username with password

Username:
root

Password:
<the root password configured for Tomcat nodes>

ID:
linuxcreds
```

The important relationship is:

```text
Jenkins Pipeline
      |
      | credentialsId = linuxcreds
      v
Jenkins Credential
      |
      | username/password or SSH credentials
      v
Ansible
      |
      v
Tomcat Nodes
```

inside the pipeline.

Instead, Jenkins stores it securely under:

```text
Credentials
```

and the pipeline refers to:

```groovy
credentialsId: 'linuxcreds'
```

---

# 31. Alternative - SSH Private Key Credential

A better approach than root password authentication is SSH key-based authentication.

In Jenkins:

```text
Manage Jenkins
→ Credentials
→ Add Credentials
```

Select:

```text
SSH Username with private key
```

Example:

```text
Username:
root

ID:
linuxcreds
```

Then provide the private key.

Alternatively, if the Ansible connection is configured to use `ec2-user`, use:

```text
Username: ec2-user
```

and the appropriate SSH private key.

Use whichever SSH user/key configuration actually exists on your worker nodes.

---

# 32. Step 3 - Setup SonarQube

SonarQube runs on:

```text
EC2-2
```

Port:

```text
9000
```

Architecture:

```text
Jenkins
   |
   | SonarQube Analysis
   v
SonarQube
   |
   | Port 9000
   v
Code Quality Report
```

---

# 33. SonarQube Initial Configuration

Login to SonarQube

```bash
vi sonar.sh
```

```bash
#! /bin/bash
cd /opt/
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.6.50800.zip
unzip sonarqube-8.9.6.50800.zip
sudo dnf install java-17-amazon-corretto -y
useradd sonar
chown sonar:sonar sonarqube-8.9.6.50800 -R
chmod 777 sonarqube-8.9.6.50800 -R
su - sonar
# use the below command manually after installation
#sh /opt/sonarqube-8.9.6.50800/bin/linux-x86-64/sonar.sh start
```
```bash
sh sonar.sh
sh /opt/sonarqube-8.9.6.50800/bin/linux-x86-64/sonar.sh start
```

#echo "user=admin & password=admin"
Verify that port 9000 is listening:

```bash
ss -lntp | grep 9000
```

Open:

```text
http://<SONARQUBE-IP>:9000
```

---

If Jenkins and SonarQube are inside the same VPC, prefer the **private IP/private DNS** for server-to-server communication.

---

# 36. Jenkins SonarQube Plugin

Go to:

```text
Manage Jenkins
→ Plugins
```

Install:

```text
SonarQube Scanner for Jenkins
```

---

# 37. Configure SonarQube in Jenkins

Go to:

```text
Manage Jenkins
→ System
```

Find:

```text
SonarQube servers
```

Click:

```text
Add SonarQube
```

Configure:

```text
Name:
SonarQube
```

This name is important because the Jenkinsfile uses:

```groovy
withSonarQubeEnv('SonarQube')
```

Set:

```text
Server URL:
http://<SONARQUBE-PRIVATE-IP>:9000
```

Do not use the wrong machine's IP.

The URL must point to the EC2 instance where SonarQube is running.

---

# 39. Test SonarQube Connection

From the Jenkins server, test:

```bash
curl http://<SONARQUBE-PRIVATE-IP>:9000
```

If Jenkins can reach SonarQube, the request should return the SonarQube web response.

Also verify security groups:

```text
Jenkins EC2
     |
     | TCP 9000
     v
SonarQube EC2
```

---

# 40. Add SonarQube Stage to Pipeline

After the basic pipeline is working, add:

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar'
        }
    }
}
```

# 41. Step 4 - Setup Amazon S3 for Artifact Storage

Create an S3 bucket.

Example:

```text
artifact-warfile
```

Region:

```text
ap-south-1
```

The bucket name must be globally unique in AWS.

If the name is already taken, use another unique bucket name and update the Jenkins pipeline accordingly.

---

# 42. Why S3 Is Used

After Maven package:

```text
target/myapp.war
```

the WAR file is an artifact.

We store the artifact in S3 so it can be retained independently from the Jenkins workspace.

Flow:

```text
Source Code
    |
    v
Maven
    |
    v
myapp.war
    |
    v
S3
```

---

# 43. S3 Credentials

The Jenkins server needs permission to upload the WAR to S3.

Recommended approach:

## Jenkins EC2 IAM Role

Attach an IAM role to the Jenkins EC2 instance.

Give the role only the permissions required for the bucket.

For example:

```text
s3:PutObject
```

on:

```text
arn:aws:s3:::artifact-warfile/*
```

If Jenkins needs to list the bucket:

```text
s3:ListBucket
```

on:

```text
arn:aws:s3:::artifact-warfile
```

This is preferable to hard-coding AWS Access Key and Secret Key inside Jenkinsfiles.

---

# 44. S3 Jenkins Credential

If your selected S3 Publisher plugin configuration requires an AWS credential profile, create it in Jenkins.

Go to:

```text
Manage Jenkins
→ Credentials
```

Create the credential/profile expected by your S3 plugin.

Example profile name:

```text
s3creds
```

The pipeline uses:

```groovy
profileName: 's3creds'
```

### Important

put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

---

# S3 Integration

Your final pipeline uses the S3 Publisher plugin's:

```text
s3Upload
```

step.

The stage is:

```groovy
stage('Upload to S3') {
    steps {
        s3Upload consoleLogLevel: 'INFO', dontSetBuildResultOnFailure: false, dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: 'artifact-warfile', excludedFile: '', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: false, selectedRegion: 'ap-south-1', showDirectlyInBrowser: false, sourceFile: '**/*.war', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: false]], pluginFailureResultConstraint: 'FAILURE', profileName: 's3creds', userMetadata: []
    }
}
```

The important values are:

```text
Bucket:
artifact-warfile

Region:
ap-south-1

Source:
**/*.war

AWS profile:
s3creds
```

---

# 46. Verify S3 Upload

After the Jenkins pipeline runs successfully, verify the bucket.

You should see something similar to:

```text
artifact-warfile
       |
       +---- myapp.war
```

You can also verify from AWS CLI if configured:

```bash
aws s3 ls s3://artifact-warfile/
```

---

# 47. Important Note About S3 Publisher

The exact `s3Upload` Pipeline syntax depends on the installed S3 Publisher plugin version.

If Jenkins gives errors such as:

```text
Missing required parameter
Not a valid field
No such DSL method s3Upload
```

do not randomly modify the pipeline.

First verify:

```text
Manage Jenkins
→ Plugins
→ Installed
→ S3 Publisher
```

and check the installed version/documentation.

If the plugin version does not support the exact syntax, an AWS CLI based S3 upload is a reliable alternative.

For example:

```bash
aws s3 cp target/myapp.war s3://artifact-warfile/myapp.war
```

---

# 48. Step 5 - Setup Tomcat Worker Nodes

Now we configure:

```text
EC2-3 → Tomcat Node 1
EC2-4 → Tomcat Node 2
```

Ansible will install/configure Tomcat on both nodes.

Architecture:

```text
                Ansible
                   |
          +--------+--------+
          |                 |
          v                 v
      Tomcat 1          Tomcat 2
       :8080              :8080
```

---

# 49. Tomcat Configuration Files

Instead of manually editing:

```text
tomcat-users.xml
```

and:

```text
context.xml
```

on every worker node, we keep these files locally on the Ansible server/GitHub and use Ansible `template` to copy them to the Tomcat servers.

Files:

```text
tomcat.yml
tomcat-users.xml
context.xml
```

Example project structure:

```text
ansible/
├── hosts
├── tomcat.yml
├── tomcat-users.xml
├── context.xml
└── deploy.yml
```

---

# 50. tomcat-users.xml

Create:

```bash
vi tomcat-users.xml
```

Use the required Tomcat roles/users for the project.

Example:

```xml
<tomcat-users>
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <user username="tomcat" password="CHANGE_ME" roles="manager-gui,manager-script"/>
</tomcat-users>
```

Replace:

```text
CHANGE_ME
```

with your lab password.

### Security

Do not commit the real production password to a public GitHub repository.

If the repository is public, use Ansible Vault/Jenkins Credentials/another secret-management mechanism instead.

---

# 51. context.xml

Create:

```bash
vi context.xml
```

Use the `context.xml` content required for your Tomcat Manager configuration.

You can keep your tested `context.xml` in GitHub and let Ansible copy it to:

```text
/root/tomcat/webapps/manager/META-INF/context.xml
```

Do not blindly delete lines by line number unless you have verified the exact Tomcat version.

---

# 52. Tomcat Ansible Playbook

Create:

```bash
vi tomcat.yml
```

Use:

```yaml
---
- name: Setup Tomcat
  hosts: all
  become: yes
  tasks:

    - name: Download tomcat from dlcdn
      get_url:
        url: "https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.25/bin/apache-tomcat-11.0.25.tar.gz"
        dest: "/root/"

    - name: untar the apache file
      command: tar -zxvf apache-tomcat-11.0.25.tar.gz

    - name: Rename the tomcat
      command: mv apache-tomcat-11.0.25 tomcat

    - name: Install the latest available Java (OpenJDK)
      yum:
        name: java-17-amazon-corretto
        state: present

    - name: Setting the roles in tomcat-users.xml file
      template:
        src: tomcat-users.xml
        dest: /root/tomcat/conf/tomcat-users.xml

    - name: Delete two lines in context.xml
      template:
        src: context.xml
        dest: /root/tomcat/webapps/manager/META-INF/context.xml

    - name: Create Tomcat systemd Service File
      copy:
        dest: /etc/systemd/system/tomcat.service
        content: |
          [Unit]
          Description=Apache Tomcat Server
          After=network.target

          [Service]
          User=root
          Group=root
          Type=forking
          Environment="JAVA_HOME=/usr/lib/jvm/jre"
          Environment="CATALINA_HOME=/root/tomcat"
          ExecStart=/root/tomcat/bin/startup.sh
          ExecStop=/root/tomcat/bin/shutdown.sh
          Restart=on-failure

          [Install]
          WantedBy=multi-user.target

    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Start tomcat Service
      service:
        name: tomcat
        state: started
        enabled: yes
```

---

# 53. IMPORTANT - Tomcat Version

The above playbook uses:

```text
Tomcat 11.0.25
```

Your older notes contain different versions such as:

```text
Tomcat 10.1.33
Tomcat 10.1.42
Tomcat 11.0.24
Tomcat 11.0.25
```

Do not mix these versions accidentally.

If you change the Tomcat version in the download URL, also change:

```text
apache-tomcat-<VERSION>.tar.gz
```

and:

```text
apache-tomcat-<VERSION>
```

inside the playbook.

For example, if the downloaded version is:

```text
11.0.25
```

then:

```text
apache-tomcat-11.0.25.tar.gz
```

must match the downloaded file.

---

# 54. About the sed Command in Your Old Notes

Your old notes contain:

```bash
sed -i 's/87/93/g' tomcat.yml
```

This should only be used if your script/playbook specifically contains a version number that needs replacement.

Do not execute this command blindly.

Always check:

```bash
grep -n "tomcat" tomcat.yml
```

and update the actual version/path intentionally.

---

# 55. Do Not Run the Tomcat Playbook Until Files Are Ready

Before running:

```bash
ansible-playbook tomcat.yml
```

make sure these files exist:

```text
tomcat.yml
tomcat-users.xml
context.xml
```

Check:

```bash
ls -l
```

The Ansible template task expects:

```text
src: tomcat-users.xml
```

and:

```text
src: context.xml
```

Therefore, these files must be available from the Ansible playbook's expected location.

---

# 56. Run Tomcat Setup Playbook

Once everything is ready:

```bash
ansible-playbook tomcat.yml
```

Ansible will perform the following:

```text
Download Tomcat
      ↓
Extract Tomcat
      ↓
Rename to /root/tomcat
      ↓
Install Java
      ↓
Copy tomcat-users.xml
      ↓
Copy context.xml
      ↓
Create systemd service
      ↓
Reload systemd
      ↓
Start Tomcat
      ↓
Enable Tomcat at boot
```

---

# 57. Check Tomcat Service

On worker nodes:

```bash
systemctl status tomcat
```

Check port:

```bash
ss -lntp | grep 8080
```

---

# 58. Access Tomcat

Open:

```text
http://<TOMCAT-NODE-1-IP>:8080
```

and:

```text
http://<TOMCAT-NODE-2-IP>:8080
```

You should see the Tomcat page.

---

# 59. Tomcat Manager

If Manager application is configured, open:

```text
http://<TOMCAT-IP>:8080/manager/html
```

Use the configured:

```text
Username:
tomcat

Password:
<your configured Tomcat password>
```

The password in your old lab notes such as:

```text
root123
root123456
```

is only an example. Use the password actually configured in `tomcat-users.xml`.

---

# 60. Where Is the Tomcat Password Used?

This is important.

If you use Tomcat Manager authentication, the username/password is configured in:

```text
tomcat-users.xml
```

Example:

```xml
<user username="tomcat" password="CHANGE_ME" roles="manager-gui,manager-script"/>
```

If Jenkins is directly using Tomcat Manager deployment, those credentials would also be stored in:

```text
Jenkins → Credentials
```

with the appropriate credential ID.

However, in this project the final deployment is performed by **Ansible**, not Jenkins' Deploy to Container plugin.

Therefore:

```text
Jenkins
   |
   | credentialsId = linuxcreds
   v
Ansible
   |
   | SSH
   v
Tomcat
```

The Tomcat Manager username/password is not required by the final Ansible `copy` deployment method.

---

# 61. Manual Ansible Deployment Test

Before integrating deployment with Jenkins, test Ansible manually.

Create:

```bash
vi deploy.yml
```

Use:

```yaml
---
- name: Deploy war to tomcat servers
  hosts: all
  tasks:

    - name: Deploy WAR file
      copy:
        src: /var/lib/jenkins/workspace/project/target/myapp.war
        dest: /root/tomcat/webapps/
```

---

# 62. Important deploy.yml Path

This line:

```yaml
src: /var/lib/jenkins/workspace/project/target/myapp.war
```

must match the actual Jenkins workspace.

Check the actual workspace:

```bash
ls /var/lib/jenkins/workspace/
```

Then:

```bash
find /var/lib/jenkins/workspace -name "*.war"
```

For example:

```text
/var/lib/jenkins/workspace/project/target/myapp.war
```

If your Jenkins job has another name, update the path accordingly.

---

# 63. Run Manual Deployment

Run:

```bash
ansible-playbook deploy.yml
```

Ansible will copy the WAR from the Jenkins machine to both Tomcat nodes.

Flow:

```text
Jenkins Workspace
       |
       | myapp.war
       v
Ansible Master
       |
       +------------+
       |            |
       v            v
 Tomcat Node 1   Tomcat Node 2
       |            |
       +-----+------+
             |
             v
       Updated Application
```

---

# 64. Why Manual Deployment Is Not Enough

The manual process requires:

```text
Build
   ↓
Find WAR
   ↓
Run Ansible
```

every time.

This is not fully automated.

Therefore, we integrate Ansible with Jenkins.

---

# 65. Step 6 - Integrate Ansible with Jenkins

Move the deployment playbook to:

```bash
/etc/ansible/
```

For example:

```bash
mv deploy.yml /etc/ansible/
```

You can keep:

```text
/etc/ansible/
├── hosts
├── deploy.yml
├── tomcat.yml
├── tomcat-users.xml
└── context.xml
```

---

# 66. Jenkins Ansible Configuration

Go to:

```text
Manage Jenkins
→ Tools
→ Ansible installations
```

Configure:

```text
Name:
ansible

Path:
<actual path returned by which ansible>
```

For your environment this may be:

```text
/bin
```

Verify:

```bash
which ansible
```

---

# 67. Jenkins Ansible Credentials

Go to:

```text
Manage Jenkins
→ Credentials
→ System
→ Global credentials
→ Add Credentials
```

Example:

```text
Kind:
Username with password

Username:
root

Password:
<worker node root password>

ID:
linuxcreds
```

The Jenkinsfile will use:

```groovy
credentialsId: 'linuxcreds'
```

Again, never write the actual password inside:

```text
Jenkinsfile
GitHub
README.md
steps.md
```

---

# 68. Configure Ansible Inventory

Jenkins/Ansible will use:

```text
/etc/ansible/hosts
```

Example:

```ini
[prod]
172.31.20.40
172.31.21.25
```

Replace these with your actual private IP addresses.

---

# 69. Configure Ansible Playbook

Jenkins will use:

```text
/etc/ansible/deploy.yml
```

Example:

```yaml
---
- name: Deploy war to tomcat servers
  hosts: all
  tasks:

    - name: Deploy WAR file
      copy:
        src: /var/lib/jenkins/workspace/project/target/myapp.war
        dest: /root/tomcat/webapps/
```

---

# 70. Final Jenkins Pipeline

Once the following components are working independently:

```text
GitHub
Maven
SonarQube
S3
Ansible
Tomcat
```

use the final pipeline.

```groovy
pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/chhatrapal7/java-project-maven-new.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar'
                }
            }
        }

        stage('Upload to S3') {
            steps {
                s3Upload consoleLogLevel: 'INFO', dontSetBuildResultOnFailure: false, dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: 'artifact-warfile', excludedFile: '', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: false, selectedRegion: 'ap-south-1', showDirectlyInBrowser: false, sourceFile: '**/*.war', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: false]], pluginFailureResultConstraint: 'FAILURE', profileName: 's3creds', userMetadata: []
            }
        }

        stage('Deploy') {
            steps {
                ansiblePlaybook(
                    credentialsId: 'linuxcreds',
                    disableHostKeyChecking: true,
                    installation: 'ansible',
                    inventory: '/etc/ansible/hosts',
                    playbook: '/etc/ansible/deploy.yml'
                )
            }
        }
    }
}
```

---

# 71. Final Pipeline Explained Stage by Stage

## Stage 1 - Checkout

```groovy
stage('Checkout')
```

Jenkins downloads the latest source code from GitHub.

```text
GitHub
   ↓
Jenkins Workspace
```

---

## Stage 2 - Build

```groovy
mvn compile
```

Java source code is compiled.

If compilation fails, the pipeline stops.

---

## Stage 3 - Test

```groovy
mvn test
```

Automated tests are executed.

If tests fail, the pipeline stops.

---

## Stage 4 - Package

```groovy
mvn package
```

Maven creates the WAR file.

Example:

```text
target/myapp.war
```

---

## Stage 5 - SonarQube

```groovy
withSonarQubeEnv('SonarQube')
```

Jenkins connects to the SonarQube server and performs code analysis.

Flow:

```text
Jenkins
   |
   | Code Analysis
   v
SonarQube
   |
   v
Quality Report
```

---

## Stage 6 - Upload to S3

```groovy
s3Upload
```

The generated WAR artifact is uploaded to:

```text
S3 Bucket:
artifact-warfile
```

Flow:

```text
target/myapp.war
       |
       v
Amazon S3
       |
       v
artifact-warfile
```

---

## Stage 7 - Deploy

```groovy
ansiblePlaybook(...)
```

Jenkins starts the Ansible deployment.

Ansible reads:

```text
/etc/ansible/hosts
```

and:

```text
/etc/ansible/deploy.yml
```

Then Ansible connects to both Tomcat nodes and copies the WAR.

```text
Jenkins
   |
   v
Ansible
   |
   +----------------+
   |                |
   v                v
Tomcat 1         Tomcat 2
   |                |
   +-------+--------+
           |
           v
    Updated Application
```

---

# 72. Complete End-to-End Execution

Now the entire project works like this.

### Step 1

Developer changes Java code.

```text
Developer
    |
    v
Java Code
```

### Step 2

Developer pushes code:

```bash
git add .
git commit -m "Update application"
git push origin main
```

### Step 3

Jenkins checks out the new code.

```text
GitHub
   ↓
Jenkins
```

### Step 4

Jenkins compiles:

```bash
mvn compile
```

### Step 5

Jenkins runs tests:

```bash
mvn test
```

### Step 6

Jenkins creates the WAR:

```bash
mvn package
```

### Step 7

Jenkins sends the code for SonarQube analysis.

```text
Jenkins
   ↓
SonarQube
```

### Step 8

Jenkins uploads the WAR to S3.

```text
WAR
 ↓
S3
```

### Step 9

Jenkins calls Ansible.

```text
Jenkins
   ↓
Ansible
```

### Step 10

Ansible connects to both Tomcat nodes.

```text
Ansible
   |
   +----> Tomcat Node 1
   |
   +----> Tomcat Node 2
```

### Step 11

WAR is copied to:

```text
/root/tomcat/webapps/
```

### Step 12

Tomcat deploys the WAR.

### Step 13

The updated application becomes available on both worker nodes.

```text
Tomcat Node 1 → Updated Application

Tomcat Node 2 → Updated Application
```

---

# 73. Complete Project Flow in One Line

```text
Developer
   ↓
GitHub
   ↓
Jenkins
   ↓
Checkout
   ↓
Maven Compile
   ↓
Maven Test
   ↓
Maven Package
   ↓
SonarQube
   ↓
WAR Artifact
   ↓
Amazon S3
   ↓
Ansible
   ↓
Tomcat Node 1
   +
Tomcat Node 2
   ↓
Updated Java Application
```

---

# 74. Credentials and Password Reference

This section is important because several different credentials are used in the project.

## 1. EC2 SSH

Used to login to EC2 servers.

```text
SSH Key / EC2 Key Pair
```

Do not upload the private key to GitHub.

---

## 2. Root Password

Used in the lab when configuring root access:

```bash
passwd root
```

This is used for SSH/password authentication if that method is enabled.

Do not commit it to GitHub.

---

## 3. Jenkins Ansible Credential

Jenkins credential:

```text
ID:
linuxcreds
```

Possible configuration:

```text
Username:
root

Password:
<worker-node root password>
```

This is referenced by:

```groovy
credentialsId: 'linuxcreds'
```

---

## 4. Tomcat Manager Credential

Inside:

```text
tomcat-users.xml
```

Example:

```xml
<user username="tomcat" password="CHANGE_ME" roles="manager-gui,manager-script"/>
```

This is required only for Tomcat Manager-based authentication/deployment.

The final Ansible `copy` deployment does not need the Tomcat Manager password.

---

## 5. SonarQube Credential

SonarQube authentication token/credential should be stored in:

```text
Jenkins Credentials
```

Do not put the token in:

```text
Jenkinsfile
GitHub
steps.md
```

---

## 6. AWS/S3 Credential

Use an IAM role attached to the Jenkins EC2 instance whenever possible.

If the S3 Publisher plugin uses an AWS credential/profile, the Jenkins-side credential/profile is referenced as:

```text
s3creds
```

The Jenkinsfile contains:

```groovy
profileName: 's3creds'
```

but not the actual AWS secret.

---

# 75. Recommended Project Structure

A clean GitHub repository can look like this:

```text
enterprise-java-cicd/
│
├── src/
│   ├── main/
│   │   └── java/
│   │
│   └── test/
│
├── ansible/
│   ├── hosts
│   ├── tomcat.yml
│   ├── tomcat-users.xml
│   ├── context.xml
│   └── deploy.yml
│
├── docs/
│   └── architecture.png
│
├── Jenkinsfile
├── pom.xml
├── README.md
├── steps.md
├── .gitignore
└── LICENSE
```

---

# 76. Jenkinsfile

The final Jenkins pipeline should be stored in the root of the GitHub repository as:

```text
Jenkinsfile
```

There is no extension.

Correct:

```text
Jenkinsfile
```

Incorrect:

```text
Jenkinsfile.txt
Jenkinsfile.groovy
```

---

# 77. Configure Jenkins Pipeline From SCM

Instead of copying the pipeline manually every time, configure Jenkins to read the `Jenkinsfile` from GitHub.

In Jenkins:

```text
New Item
   ↓
Pipeline
```

Then:

```text
Pipeline
→ Definition
→ Pipeline script from SCM
```

Select:

```text
SCM:
Git
```

Repository:

```text
https://github.com/chhatrapal7/java-project-maven-new.git
```

Branch:

```text
*/main
```

Script Path:

```text
Jenkinsfile
```

Now Jenkins automatically reads the pipeline from GitHub.

---

# 78. What Happens When Code Changes?

Suppose the developer changes:

```text
index.jsp
```

or:

```text
Java source code
```

and pushes:

```bash
git push origin main
```

Jenkins executes:

```text
Checkout
   ↓
Build
   ↓
Test
   ↓
Package
   ↓
SonarQube
   ↓
S3
   ↓
Ansible
   ↓
Tomcat 1
   +
Tomcat 2
```

Therefore both worker nodes receive the new application version.

---

# 79. Troubleshooting Checklist

## Jenkins is not starting

Check:

```bash
systemctl status jenkins
```

Check logs:

```bash
journalctl -u jenkins -n 100
```

---

## Maven command not found

Check:

```bash
mvn -version
```

Also:

```bash
which mvn
```

---

## Ansible command not found

Check:

```bash
ansible --version
```

and:

```bash
which ansible
```

---

## Ansible ping fails

Run:

```bash
ansible -m ping all
```

Check:

```text
Security Group
SSH
Private IP
Root login
SSH key
Inventory
```

---

## SSH asks for password every time

Check whether the SSH key was copied:

```bash
ssh root@<worker-private-ip>
```

If it asks for a password, verify:

```bash
ssh-copy-id root@<worker-private-ip>
```

---

## SonarQube connection fails

From Jenkins server:

```bash
curl http://<SONARQUBE-PRIVATE-IP>:9000
```

Check:

```bash
ss -lntp | grep 9000
```

Also check the SonarQube security group.

---

## WAR file is not generated

Check:

```bash
mvn package
```

Then:

```bash
find target -name "*.war"
```

---

## S3 upload fails

Check:

```text
S3 bucket name
AWS permissions
AWS region
Jenkins S3 credential/profile
S3 Publisher plugin
```

If using AWS CLI:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws s3 ls s3://artifact-warfile/
```

---

## Tomcat is not running

Check:

```bash
systemctl status tomcat
```

Check:

```bash
ss -lntp | grep 8080
```

Check Tomcat logs:

```bash
tail -f /root/tomcat/logs/catalina.out
```

---

## Application is not updating

Check:

```bash
ls -lh /root/tomcat/webapps/
```

Verify that the new WAR was copied.

Then check:

```bash
systemctl status tomcat
```

Check:

```bash
tail -f /root/tomcat/logs/catalina.out
```

---

# 80. Important Security Rules

Never commit these to GitHub:

```text
AWS Access Key
AWS Secret Key
SSH Private Key
.pem files
Jenkins passwords
Tomcat passwords
SonarQube tokens
Database passwords
.env files
Production credentials
```

Use:

```text
Jenkins Credentials
IAM Roles
Ansible Vault
AWS Secrets Manager
Environment Variables
```

where appropriate.

For a public GitHub repository, never put real passwords inside:

```text
tomcat-users.xml
```

or:

```text
steps.md
```

Use placeholders such as:

```text
CHANGE_ME
```

---

# 81. Final Technologies Used

This project demonstrates:

```text
Git
GitHub
Jenkins
Maven
Java
SonarQube
Amazon S3
AWS EC2
Ansible
Apache Tomcat
Linux
SSH
CI/CD
Artifact Management
Automated Deployment
```

---

# 82. DevOps Concepts Demonstrated

The project demonstrates the following DevOps concepts:

### Continuous Integration

```text
GitHub
   ↓
Jenkins
   ↓
Build
   ↓
Test
```

### Code Quality

```text
Jenkins
   ↓
SonarQube
```

### Artifact Management

```text
Maven
   ↓
WAR
   ↓
Amazon S3
```

### Configuration Management

```text
Ansible
```

### Automated Deployment

```text
Ansible
   ↓
Tomcat 1
   +
Tomcat 2
```

### Infrastructure

```text
AWS EC2
```

---

# 83. Final Architecture Summary

The final architecture contains four EC2 instances:

```text
                    GITHUB
                       |
                       |
                       v
          +-------------------------+
          | EC2-1                   |
          | Jenkins + Ansible       |
          |                         |
          | CI/CD Pipeline          |
          +------------+------------+
                       |
          +------------+------------+
          |                         |
          v                         v
+-------------------+     +-------------------+
| EC2-2             |     | Amazon S3         |
| SonarQube         |     |                  |
| Port 9000         |     | WAR Artifact     |
+-------------------+     +---------+---------+
                                    |
                                    |
                                    v
                             +------+------+
                             |   Ansible   |
                             +------+------+
                                    |
                         +----------+----------+
                         |                     |
                         v                     v
                +----------------+    +----------------+
                | EC2-3          |    | EC2-4          |
                | Tomcat Node 1  |    | Tomcat Node 2  |
                | Port 8080      |    | Port 8080      |
                +----------------+    +----------------+
                         |                     |
                         +----------+----------+
                                    |
                                    v
                          Java Web Application
```

---

# 84. Final Deployment Flow

Remember this sequence:

```text
1. Developer writes/changes code
            ↓
2. Push code to GitHub
            ↓
3. Jenkins checks out code
            ↓
4. Maven compiles code
            ↓
5. Maven runs tests
            ↓
6. Maven packages WAR
            ↓
7. SonarQube analyzes code
            ↓
8. WAR uploaded to Amazon S3
            ↓
9. Jenkins invokes Ansible
            ↓
10. Ansible connects to Tomcat nodes
            ↓
11. WAR deployed to Tomcat Node 1
            ↓
12. WAR deployed to Tomcat Node 2
            ↓
13. Both applications are updated
```

This is the complete:

```text
CODE → BUILD → TEST → QUALITY → ARTIFACT → STORAGE → DEPLOYMENT
```

pipeline.

---

# 85. One-Line Interview Explanation

If an interviewer asks:

**"Explain your CI/CD project."**

You can explain it like this:

> I implemented an end-to-end Java CI/CD pipeline using GitHub, Jenkins, Maven, SonarQube, Amazon S3 and Ansible. Jenkins checks out the Java application from GitHub, compiles and tests it using Maven, packages it as a WAR file, performs code-quality analysis through SonarQube, and stores the artifact in Amazon S3. Jenkins then triggers an Ansible playbook, which connects to two Tomcat worker nodes and deploys the latest WAR file automatically. Therefore, whenever the application code changes and the pipeline runs, both Tomcat nodes are updated automatically.

---

# 86. Project Flow to Remember

```text
                DEVELOPMENT
                     |
                     v
                  GitHub
                     |
                     v
                  Jenkins
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
     Build          Test       SonarQube
       |             |             |
       +-------------+-------------+
                     |
                     v
                  Package
                     |
                     v
                  WAR File
                     |
                     v
                Amazon S3
                     |
                     v
                  Ansible
                     |
             +-------+-------+
             |               |
             v               v
          Tomcat 1        Tomcat 2
             |               |
             +-------+-------+
                     |
                     v
              LIVE APPLICATION
```

**Nexus is not part of this architecture. Amazon S3 is the artifact storage layer.**
