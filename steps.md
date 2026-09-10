# Automated Java CI/CD Pipeline with Jenkins & Ansible

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

# 6. Install Jenkins

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

# 7. configure Ansible

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

# 8. Check Private IP Addresses

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

# 10. Step 2 - Configure Jenkins Basic CI Pipeline

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

# 11. Jenkins Plugins Required

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

# 12. Configure Maven in Jenkins

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

# 13. Configure Ansible in Jenkins

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

# 14. Configure Ansible Credentials in Jenkins

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

# 15. Alternative - SSH Private Key Credential

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

# 16. Jenkins Ansible Configuration

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

# Jenkins Ansible Credentials

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
<worker node root password -> root123456>

ID:
linuxcreds
```

The Jenkinsfile will use:

```groovy
credentialsId: 'linuxcreds'
```



# 17. Step 3 - Setup SonarQube

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

# 18. SonarQube Initial Configuration

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

# 19. Jenkins SonarQube Plugin

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

# 20. Configure SonarQube in Jenkins

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

# 21. Test SonarQube Connection

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

# 22. Add SonarQube Stage to Pipeline

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

# 23. Step 4 - Setup Amazon S3 for Artifact Storage

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

# 24. Why S3 Is Used

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

# 25. S3 Credentials

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

# 26. S3 Jenkins Credential

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

# 27. Verify S3 Upload

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

# 28. Important Note About S3 Publisher

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

# 29. Step 5 - Setup Tomcat Worker Nodes

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

# 30. Tomcat Configuration Files

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

## tomcat-users.xml

Create:

```bash
vi tomcat-users.xml
```

Use the required Tomcat roles/users for the project.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
<!--
  By default, no user is included in the "manager-gui" role required
  to operate the "/manager/html" web application.  If you wish to use this app,
  you must define such a user - the username and password are arbitrary.

  Built-in Tomcat manager roles:
    - manager-gui    - allows access to the HTML GUI and the status pages
    - manager-script - allows access to the HTTP API and the status pages
    - manager-jmx    - allows access to the JMX proxy and the status pages
    - manager-status - allows access to the status pages only

  The users below are wrapped in a comment and are therefore ignored. If you
  wish to configure one or more of these users for use with the manager web
  application, do not forget to remove the <!.. ..> that surrounds them. You
  will also need to set the passwords to something appropriate.
-->
<!--
  <user username="admin" password="<must-be-changed>" roles="manager-gui"/>
  <user username="robot" password="<must-be-changed>" roles="manager-script"/>
-->
<!--
  The sample user and role entries below are intended for use with the
  examples web application. They are wrapped in a comment and thus are ignored
  when reading this file. If you wish to configure these users for use with the
  examples web application, do not forget to remove the <!.. ..> that surrounds
  them. You will also need to set the passwords to something appropriate.
-->
<!--
  <role rolename="tomcat"/>
  <role rolename="role1"/>
  <user username="tomcat" password="<must-be-changed>" roles="tomcat"/>
  <user username="both" password="<must-be-changed>" roles="tomcat,role1"/>
  <user username="role1" password="<must-be-changed>" roles="role1"/>
-->
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <user username="tomcat" password="root123456" roles="manager-gui, manager-script"/>
</tomcat-users>
```

## context.xml

Create:

```bash
vi context.xml
```

Use the `context.xml` content required for your Tomcat Manager configuration.

```bash
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
```

```text
/root/tomcat/webapps/manager/META-INF/context.xml
```
---

# 31. Tomcat Ansible Playbook

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
        url: "https://dlcdn.apache.org/tomcat/tomcat-11/v11.0.25/bin/apache-tomcat-11.0.25.tar"
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

# 32. IMPORTANT - Tomcat Version

The above playbook uses:

```text
Tomcat 11.0.25
```
    
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

# 33. About the sed Command in Your Old Notes

Your old notes contain:

```bash
sed -i 's/87/93/g' tomcat.yml
```

This should only be used if your script/playbook specifically contains a version number that needs replacement.

Always check:

```bash
grep -n "tomcat" tomcat.yml
```

and update the actual version/path intentionally.

---

## Do Not Run the Tomcat Playbook Until Files Are Ready

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

# 34. Run Tomcat Setup Playbook

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

# 35. Check Tomcat Service

On worker nodes:

```bash
systemctl status tomcat
```

Check port:

```bash
ss -lntp | grep 8080
```

---

# 36. Access Tomcat

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

## Tomcat Manager

If Manager application is configured, open:

```text
http://<TOMCAT-IP>:8080/manager
```

Use the configured:

```text
Username:
tomcat

Password:
<your configured Tomcat password>
```
Use the password actually configured in `tomcat-users.xml`.

```text
admin
root123456
```
---

## Where Is the Tomcat Password Used?

This is important.

If you use Tomcat Manager authentication, the username/password is configured in:

```text
tomcat-users.xml
```

Example:

```xml
<user username="tomcat" password="******" roles="manager-gui,manager-script"/>
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

# 37. Manual Ansible Deployment Test

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

# Important deploy.yml Path

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
/var/lib/jenkins/workspace/pipelin1/target/myapp.war
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

# 38. Why Manual Deployment Is Not Enough

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

# 38. Step 6 - Integrate Ansible with Jenkins

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

## Configure Ansible Playbook

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

# 39. Final Pipeline Explained Stage by Stage

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
# 40. Credentials and Password Reference

This section is important because several different credentials are used in the project.

## 1. EC2 SSH

Used to login to EC2 servers.

```text
SSH Key 
```

---

## 2. Root Password

```bash
passwd root
```

This is used for SSH/password authentication if that method is enabled.

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
<worker-node root password -> root123456>
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
<user username="tomcat" password="*****" roles="manager-gui,manager-script"/>
```

This is required only for Tomcat Manager-based authentication/deployment.

The final Ansible `copy` deployment does not need the Tomcat Manager password.

---

## 5. SonarQube Credential

SonarQube authentication token/credential should be stored in:

```text
Jenkins Credentials
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
# 41. Jenkinsfile

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

# 42. Configure Jenkins Pipeline From SCM

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

# 43. What Happens When Code Changes?

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

# 44. Troubleshooting Checklist

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

# 45. Final Architecture Summary

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
