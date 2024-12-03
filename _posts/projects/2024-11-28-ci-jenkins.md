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

Source Code to perform the below integration is in this [Link](https://github.com/dhanvb/awsdevops/tree/ci-jenkins).

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
    Name - Nexus Server
    Project - Javaprofile
8.	Select the respective security groups and security key pairs created in the beginning of this tutorial.

## Post installation of Jenkins ##

1. To complete post installation of Jenkins, access the Jenkins portal with <public IP>:8080 

![Jenkins1](/assets/images/ci-jenkins/image004.png)

2.	To get the initial password for administrator account, we need to access the server through SSH with default user name “ubuntu” and with the key file to login into the AWS CLI
3.	Switch to root user – “sudo -i”
4.	To get the password – open the file “cat /var/lib/jenkins/secrets/initialAdminPassword
5.	Paste the password into the portal and continue the setup

![Jenkins2](/assets/images/ci-jenkins/image005.png)

6.	Select “Install suggested plugins”

![Jenkins3](/assets/images/ci-jenkins/image006.png)

7.	Create the Admin User

    ![Jenkins4](/assets/images/ci-jenkins/image007.png)

8.	Confirm Instance Configuration and complete the installation. 
Before we configure a job, we need to perform Nexus repository.

## Post installation of Nexus ##

1.	Access the VM with the public IP address with port 8081. <public ip>:8081
2.	To sign in, we need to SSH into the Nexus VM with the key file and the default user is “admin”
3.	The password is in the file “/opt/nexus/sonatype-work/nexus3/admin.password”.
4.	Post installation wizard opens. Follow the instructions to finish the post installation steps.
5.	We need to create three maven repositories. Administration --> Repository --> Repositories --> Create Repository
    - Select maven2(hosted) javaprofile-release (This repository will have artifacts which are well tested and deployed to servers)
    - Select maven2(proxy) javaprofile-maven-central. We have provide the remote repository address to download all dependencies (This repository will have all the dependencies downloaded from the maven public repository)

    ![Jenkins5](/assets/images/ci-jenkins/image008.png)

    - Select maven2(group) javaprofile-maven-group. Make the above two repositories as a member repositories under this group repository.

    ![Jenkins6](/assets/images/ci-jenkins/image009.png)

6.	We can create a snapshot repository (optional). Select maven2(hosted) and select “Snapshot” under Version Policy, Add this snapshot repository also to the “javaprofile-maven-group” as a member.

     ![Jenkins7](/assets/images/ci-jenkins/image010.png)

7.	To configure maven to download the dependencies and place those under “javaprofile-maven-central”, we need to configure the settings.xml. We will pass these variables from maven build job that we are going to configure as a Jenkins job.

    ![Jenkins8](/assets/images/ci-jenkins/image011.png)

## Nexus Integration Job ##

1.	Login into Jenkins - <publicip>:8080
2.	Create a job 

    ![Nexus1](/assets/images/ci-jenkins/image012.png)

3.	For this project, we are using “Freestyle Project”

    ![Nexus2](/assets/images/ci-jenkins/image013.png)

4.	Give unique name for the project, description, provide the git url and branch name to fetch source code

    ![Nexus3](/assets/images/ci-jenkins/image014.png)

5.	In Build section, select “Invoke top-level Maven targets” and for goals mention “install -DskipTests”. This is will use the pom.xml file, which is already in the source code. For Settings, use the filesystem option and mention the path of the settings.xml file.

    ![Nexus4](/assets/images/ci-jenkins/image015.png)

6.	Assign variables for the settings.xml

    ![Nexus5](/assets/images/ci-jenkins/image016.png)

7.	After configuring the variables, paste it under “properties”

    ![Nexus6](/assets/images/ci-jenkins/image017.png)

8.	Save this job and build the job. 

    ![Nexus7](/assets/images/ci-jenkins/image018.png)

9.	Build job is successful.

    ![Nexus8](/assets/images/ci-jenkins/image019.png)

## Slack Integration ##

Build job is successful and the developers should be notified regarding this. So we are going to integrate with Slack to notify the developers and managers.

1.	Create a slack account.
2.	Create a workspace
3.	Create a channel – provide a name and description as per requirement

Create an API for slack

1.	Go to “api.slack.com” and click on “Start building”

 ![Slack1](/assets/images/ci-jenkins/image020.png)

2.	Click on “Create an app”, Give a name and select the workspace, and then “Create app”
3.	Go to settings of the app and click on “Oauth & Permissions”

  ![Slack2](/assets/images/ci-jenkins/image021.png)

4.	Scroll down to see “Scopes” and under “Bot Token Scopes” add the permission for “calls:write”

  ![Slack3](/assets/images/ci-jenkins/image022.png)

5.	Scroll up and click on “Install App to Workspace”

  ![Slack4](/assets/images/ci-jenkins/image023.png)

6.	You will be provided with a token. Copy the token and save it in the notepad. 
7.	We need to add the app into the channel and invite the app to the channel.

    ![Slack5](/assets/images/ci-jenkins/image024.png)

8.	To make use of the slack bot, we need to install a plugin in Jenkins. Head to Jenkins portal and click on “Manage Jenkins”

    ![Slack6](/assets/images/ci-jenkins/image025.png)

9.	Manage Jenkins --> Manage Plugins --> Available --> Search for “Slack” --> “Install without Restart”

    ![Slack7](/assets/images/ci-jenkins/image026.png)

10.	To configure the token, Navigate to Manage Jenkins --> Manage Credentials --> Store ‘Jenkins’ --> ‘Global Credentials’ --> Add Credentials --> Change the kind to “secret text”

    ![Slack8](/assets/images/ci-jenkins/image027.png)

11.	Now we will integrate with Slack with Jenkins. Navigate to Manage Jenkins --> Configure System --> Scroll to the end for Slack settings --> Put checkmark on “Custom Slack app bot user” and save the settings.

    ![Slack9](/assets/images/ci-jenkins/image028.png)

12.	Configure the job to send notifications.

    ![Slack10](/assets/images/ci-jenkins/image029.png)

13.	Navigate to “Post build Actions”

    ![Slack11](/assets/images/ci-jenkins/image030.png)

14.	Click on Advanced, Enter the workspace name, credential (slack-token), channel/member id - #jenkins --> Test Connection
15.	Go back to slack channel and you would have got a notification.

    ![Slack12](/assets/images/ci-jenkins/image031.png)

## Unit Tests, Code Analysis with Sonarqube ##

1.	We are going to create a job in Jenkins called as “Test” and as a Freestyle Project and copy from the Build job that we created earlier.
2.	In the Build phase, provide the goals as “test”, rest of the settings copied from “build” job.

    ![Test1](/assets/images/ci-jenkins/image032.png)

3.	Run the test job.
4.	This test job should automatically run after the build job, so to configure this, Navigate to “Build” job --> Configure --> “Build” Tab --> Add post-build action --> Select “Build Other projects” --> Projects to build as “Test”

    ![Test2](/assets/images/ci-jenkins/image033.png)

5.	Configure Integration job. Freestyle Job --> “Build Environment” tab --> Goals --> “verify -DskipUnitTests --> Rest of the settings comes from “Build” job.

    ![Test3](/assets/images/ci-jenkins/image034.png)

6. This integration job should automatically run after the test job, so to configure this, Navigate to “Build” job --> Configure --> “Build” Tab --> Add post-build action --> Select “Build Other projects” --> Projects to build as “Integration Test”
7.	So upstream job is “Build” and downstream job is “Integration Test”

    ![Test4](/assets/images/ci-jenkins/image035.png)

8.	To perform simple code analysis, we have to install a plugin called “checkstyle”
9.	Manage Jenkins --> Manage Plugins --> Available --> Search for “checks” --> Select “Checkstyle” and “Violations” --> “Install without Restart”

    ![Test5](/assets/images/ci-jenkins/image036.png)

10.	Checkstyle will analyze the code and give the report. If the failures goes beyond threshold defined by quality gates, Violations will make the job unstable, so it won’t be promoted to next step.
11.	Create a job in Jenkins for code analysis. Create new job and copy the settings of build job. For goals, enter “checkstyle:checkstyle”.
12.	Configure “Post Build Actions” --> “Publish Checkstyle analysis results

    ![Test6](/assets/images/ci-jenkins/image037.png)

13.	Run the job and results will be saved in the workspace

    ![Test7](/assets/images/ci-jenkins/image038.png)

14.	Create a new job as “Code_Analysis” and copy the settings of “Build” job
15.	In “Build Environment” tab --> Invoke top-level Maven targets --> Goals “checkstyle:checkstyle”

    ![Test8](/assets/images/ci-jenkins/image039.png)

16.	If you want to perform the Quality Gates checks for the source code in a different branch Under “Source Code Management” --> Select “Git” --> Enter Repository Url --> Then “Branch Specifier”

    ![Test9](/assets/images/ci-jenkins/image040.png)

17.	If you want to perform the quality checks for code in the branch, either create a settings.xml file in the branch or use the default maven settings. 

18.	Under build tab, we configure the violations and stop the code to get promoted to next level if the quality checks are not passed.

    ![Test10](/assets/images/ci-jenkins/image041.png)

19.	After configuring and testing the job, we can integrate the job. Jenkins Home --> Configure --> Integration Test -->Make the “Code_Analysis” job as the downstream job of “Integration Test” job using Post Build Actions.

    ![Test11](/assets/images/ci-jenkins/image042.png)

20.	Now we are going to analyse the code with checkstyle.

21.	Run the “Code Analysis” job to the publish the results.

    ![Test12](/assets/images/ci-jenkins/image043.png)
    

22.	Click “Console Output” and there we can find the location of the published results in .xml format.

    ![Test13](/assets/images/ci-jenkins/image044.png)

23.	We can find this checkstyle-result.xml in our Code Analysis job’s workspace.

    ![Test14](/assets/images/ci-jenkins/image045.png)

24.	Under Workspace, we can see the result.xml.

    ![Test15](/assets/images/ci-jenkins/image046.png)

25.	We get a graph with the data from violations in checkstyle-result.xml, the graph shows the violations between first job and second job. If we have fixed some violations, we will see a decrease in this graph.

    ![Test16](/assets/images/ci-jenkins/image047.png)

26.	We are going to give the checkstyle-result.xml file for scanning. In Post Build Action --> Add post-build section --> Report violations

    ![Test17](/assets/images/ci-jenkins/image048.png)

27.	We can use other code analysis tools as you see in the below screenshot like “codenarc”, “cpd” etc. But for our use case we are using “checkstyle”

    ![Test18](/assets/images/ci-jenkins/image049.png)

28.	According to the organization’s policies, we can set the toleration. In our new case, we have set the maximum toleration to 100.
29.	We have tested the “Code Analysis” Job and now we can configure this job as a downstream job for “Integration Tests” job.

    ![Test19](/assets/images/ci-jenkins/image050.png)

30.	We now have four jobs connected with each other. First is Build, Second is Test, Third is Integration Test and the last one will be Code_Analysis

    ![Test20](/assets/images/ci-jenkins/image051.png)

31.	We have one more code analysis with Sonarqube and we are going to publish the results from checkstyle and sonarqube to Sonarqube dashboard.
32.	We should have Sonarqube server up and running. We will access this server using its public IP address through port 80 with a browser.

    ![Test21](/assets/images/ci-jenkins/image052.png)

33.	Default username is admin and password is admin. Before putting it into production, change to a strong password. After logging in, you can see the projects page.

    ![Test22](/assets/images/ci-jenkins/image053.png)

34.	To integrate Jenkins with Sonarqube, Jenkins needs to authenticate to Sonarqube Server, for that we need to generate a token. 
35.	Top right corner --> Click the profile --> My Account --> Security --> Enter a token name and click on Generate. Copy the token and keep it safe to configure Jenkins.

    ![Test23](/assets/images/ci-jenkins/image054.png)

36.	Login into Jenkins --> Manage Plugins.

    ![Test24](/assets/images/ci-jenkins/image055.png)

37.	Click on “Available” Tab --> Search for “sonar” --> Select “SonarQube Server” and “Sonar Quality Gates” --> Install without Restart

    ![Test25](/assets/images/ci-jenkins/image056.png)

38.	Manage Jenkins --> Global Tool Configuration --> Scroll down to “SonarQube Scanner” --> Click on “Add SonarQube Scanner”.

    ![Test26](/assets/images/ci-jenkins/image057.png)

39.	Select the version and provide the name for the scanner.

    ![Test27](/assets/images/ci-jenkins/image058.png)

40.	Manage Jenkins --> Configure System --> Look for “SonarQube Servers” --> Put checkmark on “Enable Injection” then Click on “Add SonarQube”.

    ![Test28](/assets/images/ci-jenkins/image059.png)

41.	Provide name and server URL. We can get the sonar server’s IP address from the EC2 Dashboard --> Instances --> Private IP Address --> Since we are running nginx, the default port is 80. Add the URL and add the token through credential manager.

    ![Test29](/assets/images/ci-jenkins/image060.png)

42.	Add the credentials through credentials provider. We have to copy the token that we created in Step 35. 

    ![Test30](/assets/images/ci-jenkins/image061.png)

43.	The above setting configured to push Sonarqube results to server. But make the code go through Quality Gates, we have configure the Quality Gates settings in the same job. Scroll further down to see the setting. Quality Gates – SonarQube and click on “Add Sonar Instance”

    ![Test31](/assets/images/ci-jenkins/image062.png)

44.	Fill the details in the below form.

    ![Test32](/assets/images/ci-jenkins/image063.png)

45.	Now we have to create SonarQube Analysis job. Provide any name, Name: SonarScanner-CodeAnalysis --> Freestyle Project --> Copy the settings from Build Job --> Set the “Build” Goals as “Install” --> Add one more step for build --> Execute SonarQube Scanner.
    
    ![Test33](/assets/images/ci-jenkins/image064.png)

46.	We have to configure sonar analysis properties, with the code repo, name etc.

    ![Test34](/assets/images/ci-jenkins/image065.png)

47.	Save and Run the job. You will get slack notification after the build is over. 

    ![Test35](/assets/images/ci-jenkins/image066.png)

48.	Login into Sonarqube server --> Projects --> You will see the Code analysis status below the repo name with Bugs, Vulnerabilities etc. 

    ![Test36](/assets/images/ci-jenkins/image067.png)

49.	We can decide the quality gates to pass the code to next level. For that, we have to configure Quality Gates.

    ![Test37](/assets/images/ci-jenkins/image068.png)

50.	Click Create --> Provide a Name --> Add Condition --> Select Bugs under Reliability

    ![Test38](/assets/images/ci-jenkins/image069.png)

51.	We have to set a value, in order to stop it from passing the quality gates. Here we are going to set it to 50.

    ![Test39](/assets/images/ci-jenkins/image070.png)

52.	Configure your repo with the newly created Quality gates. Repo --> Project Settings --> Quality Gates --> Select the quality gate we created.
53.	After this, we have to configure the Post build job with Project key and Job Status when analysis fails to FAILED – The job will fail and if you configure it as UNSTABLE – The job will become unstable. Save the job.
54.	Check the console output for the status of the job.

    ![Test40](/assets/images/ci-jenkins/image071.png)

## Nexus Repository Integration ##

1.	After passing the Quality Gates, the .war file which is called as an artifact will be generated. This has to be uploaded to the Nexus Artifact Repository.

    ![Nexus1](/assets/images/ci-jenkins/image072.png)

2.	To integrate, we need the plugins to be installed. We need three plugins.
    - Nexus Artifact Uploader
    - Copy Artifact  (Since we already have the artifact, we just need to copy it)
    - Zentimestamp – For versioning
Manage Jenkins --> Available tab --> Search for the above plugins and install it without restart
3.	We have to create a job to copy the artifact to Nexus. Head to Jenkins --> New Project --> Provide a Name --> Choose Freestyle Project --> We don’t need source code hence don’t copy settings from any job.
4.	In the Build section --> Add Build Step --> Copy artifacts from another project --> Provide a name --> Select “Latest Successful Build --> Copy only “**/*.war”

    ![Nexus2](/assets/images/ci-jenkins/image073.png)

5.	Add build step --> Nexus Artifact Uploader --> Fill the details for Nexus Server --> For GroupId, provide QA or Testing, For Version provide “V$BUILD_ID”, For repository, we created a release repository, mention the name of the repository in the last field.

    ![Nexus3](/assets/images/ci-jenkins/image074.png)

6.	Scroll up and change the date pattern under General Tab

    ![Nexus4](/assets/images/ci-jenkins/image075.png)

7.	Add Artifact --> ArtifactID as “$BUILD_TIMESTAMP”, Type as “war” and File name (war file name)
8.	Add Slack Notification Settings as Post Build Job. 

## Create a Different View of the Pipeline ##

1.	To design a different view of our pipeline execution, we need to install a plugin. Manage Jenkins --> Manage Plugins --> Available Tab --> Search for “Build Pipeline” --> Install without restart

    ![View1](/assets/images/ci-jenkins/image076.png)

2.	Home page --> My views --> Provide a “View Name” --> Select “Build Pipeline view” 

    ![View2](/assets/images/ci-jenkins/image077.png)

3.	Select the first job in the Pipeline Flow --> Build (This job as all downstream jobs)

    ![View3](/assets/images/ci-jenkins/image078.png)

4.	Display Options --> No of Displayed Builds – 5 --> Save the job

    ![View4](/assets/images/ci-jenkins/image079.png)

5.	You can see the below “Build Pipeline” view

    ![View5](/assets/images/ci-jenkins/image080.png)

6.	Click Run and you can see all the pipelines to be executed.

    ![View6](/assets/images/ci-jenkins/image081.png)

7.	We need to configure Jenkins to fetch the code automatically from SCM (Source Code Management. For this, we need to configure the Build job. 
8.	Build Triggers --> Poll SCM --> This will be in cron job format --> This is configured to poll every minute and this is for testing purposes. Save the changes.

    ![View7](/assets/images/ci-jenkins/image082.png)

9.	To test this perform a empty Git commit and find the pipeline running. 
10.	If you are doing this as a test, don’t forget to delete the EC2 instances. 

## Complete Flow ##

![Flow1](/assets/images/ci-jenkins/image083.png)

Happy DevOpsing !!!



