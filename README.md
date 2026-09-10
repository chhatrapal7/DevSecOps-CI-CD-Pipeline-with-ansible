# Automated Java CI/CD Pipeline with Jenkins & Ansible

## Project Overview

This project demonstrates an end-to-end **CI/CD automation pipeline** for a Java web application using **Jenkins, Ansible, SonarQube, Amazon S3, Maven, GitHub, and Apache Tomcat**.

The main purpose of this project is to automate the complete process from **code change to application deployment** on multiple Tomcat worker nodes.

Instead of manually copying and deploying the WAR file on each server, Jenkins and Ansible automate the complete deployment process.

---

## Architecture

```text
Developer
    |
    | Push Code
    v
GitHub Repository
    |
    v
Jenkins + Ansible
    |
    +----> Build & Test
    |
    +----> SonarQube
    |
    +----> WAR Artifact
    |
    v
Amazon S3
    |
    v
Ansible Deployment
    |
    +--------------------+
    |                    |
    v                    v
Tomcat Worker 1      Tomcat Worker 2
    |                    |
    +---------+----------+
              |
              v
       Updated Web Application
```

---

## CI/CD Flow

The complete automation flow is:

```text
GitHub
   ↓
Checkout
   ↓
Maven Build
   ↓
Maven Test
   ↓
WAR Package
   ↓
SonarQube Code Analysis
   ↓
Upload WAR to Amazon S3
   ↓
Jenkins triggers Ansible
   ↓
Deploy WAR to Tomcat Worker 1 & Worker 2
   ↓
Application Updated
```

### How it works

1. Developer makes changes to the Java web application.
2. The updated code is pushed to GitHub.
3. Jenkins checks out the latest source code.
4. Maven compiles and builds the application.
5. Maven executes the tests.
6. A new WAR artifact is generated.
7. SonarQube performs code-quality analysis.
8. The WAR artifact is stored in Amazon S3.
9. Jenkins executes the Ansible deployment playbook.
10. Ansible connects to both Tomcat worker nodes.
11. The latest WAR file is deployed on both servers.
12. Tomcat starts the updated application.

---

# Key Features

* **Automated CI/CD Pipeline**

  * Automates build, test, package and deployment.

* **GitHub Integration**

  * GitHub is used as the source-code repository.

* **Maven Build Automation**

  * Automatically compiles, tests and packages the Java application.

* **SonarQube Integration**

  * Performs static code analysis and provides code-quality information.

* **Amazon S3 Artifact Storage**

  * Stores the generated WAR artifact centrally.

* **Ansible Automation**

  * Automates deployment across multiple Tomcat servers.

* **Multi-Node Deployment**

  * The same application version is deployed to two Tomcat worker nodes.

* **Tomcat Application Deployment**

  * WAR files are automatically deployed to Apache Tomcat.

* **Repeatable Deployment**

  * The same deployment process can be executed consistently whenever new code is released.

---

# Technologies Used

| Technology    | Purpose                               |
| ------------- | ------------------------------------- |
| GitHub        | Source Code Management                |
| Jenkins       | CI/CD Automation                      |
| Maven         | Build & Package                       |
| SonarQube     | Code Quality Analysis                 |
| Amazon S3     | Artifact Storage                      |
| Ansible       | Configuration & Deployment Automation |
| Apache Tomcat | Application Server                    |
| AWS EC2       | Infrastructure                        |
| Linux         | Server Operating System               |

---

# Automation Demonstration

To demonstrate the automation, I changed the **web application code/UI** in the GitHub repository.

After the code change:

```text
Code Change
    ↓
GitHub
    ↓
Jenkins Pipeline
    ↓
Build & Test
    ↓
SonarQube Analysis
    ↓
New WAR
    ↓
Amazon S3
    ↓
Ansible Deployment
    ↓
Tomcat Worker 1 + Worker 2
    ↓
Updated Web Application
```

The updated application was successfully deployed to both Tomcat worker nodes without manually copying the WAR file or deploying the application on each server.

---

# Advantages

### 1. Saves Time

Manual deployment steps are replaced with an automated pipeline.

### 2. Reduces Human Errors

The same deployment process is executed automatically every time.

### 3. Faster Releases

Developers can make changes and deploy new versions quickly.

### 4. Centralized Artifact Storage

WAR artifacts are stored in Amazon S3 instead of being maintained manually on individual servers.

### 5. Consistent Deployment

Ansible ensures that the application is deployed consistently across multiple Tomcat nodes.

### 6. Code Quality

SonarQube provides automated code-quality analysis before deployment.

### 7. Scalable Deployment

Additional Tomcat worker nodes can be added to the Ansible inventory and managed through the same automation.

### 8. Easy Maintenance

The complete deployment process is defined as code using Jenkins Pipeline and Ansible.

---

# Detailed Setup Guide

The complete installation, configuration, commands, scripts, Jenkins configuration, credentials, SonarQube setup, S3 configuration, Tomcat setup and Ansible playbooks are documented separately in:

**[`steps.md`](steps.md)**

Refer to `steps.md` for the complete step-by-step implementation.

---

