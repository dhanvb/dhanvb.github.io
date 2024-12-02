---
title: "Continuous Integration Using Jenkins, Nexus, SonarQube, Slack"
layout: single
categories:
  - projects
tags:
  - Jenkins
  - Nexus
  - Sonarqube
  - Slack
---

<h2>Benefits</h2>
<ul>
  <li><strong>Short MTTR</strong>: Mean Time To Repair</li>
  <li><strong>Fault Isolation</strong></li>
  <li><strong>Agile</strong></li>
  <li><strong>No Human Intervention Required</strong></li>
</ul>

<h2>Tools Used</h2>
<p>Jenkins, GIT, Maven, Checkstyle, Slack, Sonatype Nexus, Sonarqube, AWS EC2</p>

<h2>Goals Achieved</h2>
<ul>
  <li>Fast TAT on Feature Changes</li>
  <li>Less Disruptive</li>
</ul>

<h2>Workflow</h2>
<p>The below flowchart illustrates how the code passes through every stage.</p>

![FlowDiagram](/assets/images/ci-jenkins/image001.jpg)

# Continuous Integration and Deployment Process

<div>
    <h2>1. Code Commit</h2>
    <p>Developers commit the code into GIT (Source Code Management). This application is built on Java, so we are using Maven as the build tool.</p>
</div>

<div>
    <h2>2. Build and Test</h2>
    <p>Jenkins fetches the code and builds (compiles) it, runs Unit Tests on it, and writes the outcome as a notification in Slack/Teams.</p>
</div>

<div>
    <h2>3. Code Analysis</h2>
    <p>Next, it runs the Code Analysis using Sonarqube and puts the code through Quality Gates. If it passes the quality thresholds, it will send the notification to Slack/Teams.</p>
</div>

<div>
    <h2>4. Bug Fixing</h2>
    <p>If it does not pass the quality thresholds, it will be passed to the Developer to fix the bugs, and the process begins from scratch.</p>
</div>

<div>
    <h2>5. Artifact Packaging</h2>
    <p>After passing through Code Analysis, the artifact will be packaged and uploaded to the artifact repository, Nexus, and a notification will be sent through Slack/Teams.</p>
</div>

# Flow of Execution

<ul>
    <li>Login to AWS Account</li>
    <li>Create login key</li>
    <li>Create Security Group
        <ul>
            <li>Jenkins</li>
            <li>Nexus</li>
            <li>Sonar</li>
        </ul>
    </li>
    <li>Create EC2 instances with user data
        <ul>
            <li>Jenkins</li>
            <li>Nexus</li>
            <li>Sonar</li>
        </ul>
    </li>
    <li>Jenkins Post Installation</li>
    <li>Nexus Repository Setup</li>
    <li>SonarQube post installation</li>
    <li>Jenkins Steps
        <ul>
            <li>Build Job</li>
            <li>Setup Slack Notification</li>
            <li>Check style Code Analysis Job</li>
            <li>Setup Sonar integration</li>
            <li>Sonar code analysis job</li>
            <li>Artifact upload Job</li>
        </ul>
    </li>
    <li>Connect all jobs together with BuildPipeline
        <ul>
            <li>Set Automatic Build Trigger</li>
        </ul>
    </li>
    <li>Test with VSCode or Intellij</li>
</ul>


# Steps to Create EC2 Instances in AWS

1. Login into AWS Management Console. Open the EC2 Pane.

2. Before creating the EC2 instances, we need to create Key pairs and Security Groups for the EC2 instances.

3. Navigate to EC2 → Key pairs.
   - Give a name for the key pair.
   - File format - .pem.

4. Navigate to EC2 → Security Groups.
   - Give names for security group (Below are examples):
     - dprofile-jenkins-sg
     - dprofile-nexus-sg
     - dprofile-sonar-sg

### dprofile-jenkins-sg ###

1. Provide the Group Name (dprofile-jenkins-sg) and Short Description. 

2. Create Inbound Rules – 
Port 8080 and 22 from the machine IP address, where you will manage the instances. All Traffic or Port 80 from SonarQube security group (Allow Sonar to access Jenkins for quality gates result)

3. Create Outbound Rules – 
Allow All traffic

![Rule1](/assets/images/ci-jenkins/image002.png)

### dprofile-nexus-sg ###

1. Provide the Group Name (dprofile-nexus-sg) and Short Description. 
2. Create Inbound Rules – 
Port 8081 and 22 from the machine IP address, where you will manage the instances. Source Port (8081) from Jenkins Security Group - To allow the upload of artifacts 
3. Create Outbound Rules – 
Allow All traffic
![Rule2](/assets/images/ci-jenkins/image003.png)

### dprofile-sonar-sg ###

1. Provide the Group Name (dprofile-sonar-sg) and Short Description. 
2. Create Inbound Rules – 
Port 8081 and 22 from the machine IP address, where you will manage the instances. Source Port Range (8081) from Jenkins Security Group - To allow the upload of artifacts 
3. Create Outbound Rules – 
Allow All traffic

We have to create tags in Security Groups page.
- javaprofile-jenkins-sg
- javaprofile-nexus-sg
- javaprofile-sonar-sg

## Launch EC2 instances ##

1.	Provision the instances using userdata. Start with Jenkins instance
Github --> UserData Folder --> jenkins-setup.sh, nexus-setup.sh and sonar-setup.sh
2.	Clone the Github repository – ci-jenkins. After cloning, always run “git pull” to update the latest changes. Switch to the branch – “git checkout ci-jenkins”
3.	Navigate to “userdata” folder. Shell scripts are available to create the EC2 instances for Jenkins, Nexus and Sonarqube
4.	Install all roles using Ubuntu 22.04.
5.	AWS Portal --> EC2 --> Launch an instance --> Select t2.small (which will have some charges)
6.	Fill the details. Under Advanced Details section --> User Data --> fill it with the shell script for nexus from userdata. Repeat the steps 5 and 6 for Jenkins and Sonarqube servers
7.	Provide proper tags to identify the instances
Name – Nexus Server
Project – Javaprofile
8.	Select the respective security groups and security key pairs created in the beginning of this tutorial.

## Post installation of Jenkins ##

1. To complete post installation of Jenkins, access the Jenkins portal with <public IP>:8080 

![Jenkins1](/assets/images/ci-jenkins/image004.png)



