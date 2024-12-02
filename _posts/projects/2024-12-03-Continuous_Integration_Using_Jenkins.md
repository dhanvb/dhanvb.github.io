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

# Continuous Integration Using Jenkins, Nexus, SonarQube, Slack #

 ## [Benefits]{.underline} ## 

Short MTTR - Mean Time To Repair

Fault Isolation

Agile

No Human Intervention Required

 ## [Tools used]{.underline} ## 

Jenkins, GIT, Maven, Checkstyle, Slack, Sonatype Nexus, Sonarqube, AWS
EC2

 ## [Goals Achieved]{.underline} ## 

Fast TAT on Feature changes

Less Disruptive

 ## [Workflow]{.underline} ## 

The below flowchart illustrates, how the code passes through every stage.

![FlowDiagram](/assets/images/ci-jenkins/image001.jpg)

1)  Developers Commits the Code into GIT(Source Code Management). This
    application is built on Java, so we are using Maven as build tool.

2)  Jenkins fetches the code and builds(compiles) it, runs Unit Tests on
    it and write the outcome as a notification in Slack/Teams.

3)  Next it runs the Code Analysis using Sonarqube and puts the code
    through Quality Gates and if it passes the quality thresholds it
    will send the notification to Slack/Teams.

4)  If it is not passing the quality thresholds - It will be passed to
    Developer to fix the bugs and the process begins from scratch.

5)  After passing through Code Analysis, the artifact will be packaged
    and uploaded to the artifact repository that is Nexus and a
    notification will be sent through Slack/Teams

 ## [Flow of Execution]{.underline} ## 

- Login to AWS Account

- Create login key

- Create Security Group
    - Jenkins
    - Nexus
    - Sonar

<!-- -->

- Create EC2 instances with user data

    - Jenkins
    - Nexus
    - Sonar

<!-- -->

- Jenkins Post Installation

- Nexus Repository Setup

- SonarQube post installation

- Jenkins Steps

    - Build Job
    - Setup Slack Notification
    - Check style Code Analysis Job
    - Setup Sonar integration
    - Sonar code analysis job
    - Artifact upload Job


- Connect all jobs together with BuildPipeline

    - Set Automatic Build Trigger

- Test with VSCode or Intellij

 ## [Detailed Description of Steps to follow]{.underline} ## 

- Login into AWS Management Console. Open the EC2 Pane

- Before creating the EC2 instances, we need to create Key pairs and
  Security Groups for the EC2 instances

- EC2 Key pairs

  - Give a name for the key pair

  - File format - .pem

<!-- -->

- EC2 Security Groups

  - Give names for security group (Below are examples)

    - dprofile-jenkins-sg

    - dprofile-nexus-sg

    - dprofile-sonar-sg

 ## dprofile-jenkins-sg ## 

- Provide the Group Name (dprofile-jenkins-sg) and Short Description.

- Create Inbound Rules --

Port 8080 and 22 from the machine IP address, where you will manage the
instances. All Traffic or Port 80 from SonarQube security group (Allow
Sonar to access Jenkins for quality gates result)

- Create Outbound Rules --

Allow All traffic

![Rules1](/assets/images/ci-jenkins/image002.png)

 ## dprofile-nexus-sg ## 

- Provide the Group Name (dprofile-nexus-sg) and Short Description.

- Create Inbound Rules --

Port 8081 and 22 from the machine IP address, where you will manage the
instances. Source Port (8081) from Jenkins Security Group - To allow the
upload of artifacts

- Create Outbound Rules --

Allow All traffic

![Rule2](/assets/images/ci-jenkins/image003.png)

 ## dprofile-sonar-sg ## 

- Provide the Group Name (dprofile-sonar-sg) and Short Description.

- Create Inbound Rules --

Port 8081 and 22 from the machine IP address, where you will manage the
instances. Source Port Range (8081) from Jenkins Security Group - To
allow the upload of artifacts

- Create Outbound Rules --

Allow All traffic

We have to create tags in Security Groups page.

 - javaprofile-jenkins-sg  
 - javaprofile-nexus-sg  
 - javaprofile-sonar-sg

 ## Launch EC2 instances ## 

1.  Provision the instances using userdata. Start with Jenkins instance

Github --> UserData Folder --> jenkins-setup.sh, nexus-setup.sh and sonar-setup.sh

2.  Clone the Github repository -- ci-jenkins. After cloning, always run "git pull" to update the latest changes. Switch to the branch -- "git checkout ci-jenkins"

3.  Navigate to "userdata" folder. Shell scripts are available to create the EC2 instances for Jenkins, Nexus and Sonarqube

4.  Install all roles using Ubuntu 22.04.

5.  AWS Portal EC2 Launch an instance Select t2.small (which will have some charges)

6.  Fill the details. Under Advanced Details section User Data fill it with the shell script for nexus from userdata. Repeat the steps 5 and 6 for Jenkins and Sonarqube servers

7.  Provide proper tags to identify the instances

Name -- Nexus Server
Project -- Javaprofile

8.  Select the respective security groups and security key pairs created in the beginning of this tutorial.

 ## Post installation of Jenkins ## 

1.  To complete post installation of Jenkins, access the Jenkins portal with \<public IP\>:8080

![Jenkins1](/assets/images/ci-jenkins/image004.png)

2.  To get the initial password for administrator account, we need to access the server through SSH with default user name "ubuntu" and with the key file to login into the AWS CLI

3.  Switch to root user -- "sudo -i"

4.  To get the password -- open the file "cat
    /var/lib/jenkins/secrets/initialAdminPassword

5.  Paste the password into the portal and continue the setup  
      
![Jenkins2](/assets/images/ci-jenkins/image005.png)

6.  Select "Install suggested plugins"

![Jenkins3](/assets/images/ci-jenkins/image006.png)

7.  Create the Admin User

![Jenkins4](/assets/images/ci-jenkins/image007.png)

8.  Confirm Instance Configuration and complete the installation.

Before we configure a job, we need to perform Nexus repository.

 ## Post installation of Nexus ## 

1.  Access the VM with the public IP address with port 8081. \<public-ip\>:8081

2.  To sign in, we need to SSH into the Nexus VM with the key file and the default user is "admin"

3.  The password is in the file "/opt/nexus/sonatype-work/nexus3/admin.password".

4.  Post installation wizard opens. Follow the instructions to finish the post installation steps.

5.  We need to create three maven repositories. Administration
    Repository Repositories Create Repository

    a.  Select maven2(hosted)  ## javaprofile-release ##  (This repository will have artifacts which are well tested and deployed to servers)

    b.  Select maven2(proxy)  ## javaprofile-maven-central. ##  We have provide the remote repository address to download all dependencies (This repository will have all the dependencies downloaded from the maven public repository)

![Nexus1](/assets/images/ci-jenkins/image008.png)

c.  Select maven2(group)  ## javaprofile-maven-group. ##  Make the above two repositories as a member repositories under this group repository.

![Nexus2](/assets/images/ci-jenkins/image009.png)

6.  We can create a snapshot repository (optional). Select
    maven2(hosted) and select "Snapshot" under Version Policy, Add this snapshot repository also to the javaprofile-maven-group" as a member.

![Nexus3](/assets/images/ci-jenkins/image010.png)

7.  To configure maven to download the dependencies and place those under " ## javaprofile-maven-central", ##  we need to configure the settings.xml. We will pass these variables from maven build job that we are going to configure as a Jenkins job.

![Nexus4](/assets/images/ci-jenkins/image011.png)

 ## Nexus Integration Job ## 

1.  Login into Jenkins - \<publicip\>:8080

2.  Create a job

    ![NexInt1](/assets/images/ci-jenkins/image012.png)

3.  For this project, we are using "Freestyle Project"

    ![NexInt2](/assets/images/ci-jenkins/image013.png)

4.  Give unique name for the project, description, provide the git url and branch name to fetch source code

    ![NexInt3](/assets/images/ci-jenkins/image014.png)

5.  In Build section, select "Invoke top-level Maven targets" and for goals mention "install -DskipTests". This is will use the pom.xml file, which is already in the source code. For Settings, use the filesystem option and mention the path of the settings.xml file.

    ![NexInt4](/assets/images/ci-jenkins/image015.png)

6.  Assign variables for the settings.xml

    ![NexInt5](/assets/images/ci-jenkins/image016.png)

7.  After configuring the variables, paste it under "properties"

    ![NexInt6](/assets/images/ci-jenkins/image017.png)

8.  Save this job and build the job.

    ![NexInt7](/assets/images/ci-jenkins/image018.png)

9.  Build job is successful.

    ![NexInt8](/assets/images/ci-jenkins/image019.png)

 ## Slack Integration ## 

Build job is successful and the developers should be notified regarding this. So we are going to integrate with Slack to notify the developers and managers.

1.  Create a slack account.

2.  Create a workspace

3.  Create a channel -- provide a name and description as per
    requirement

### Create an API for slack ###

1.  Go to "api.slack.com" and click on "Start building"

    ![SlackInt1](/assets/images/ci-jenkins/image020.png)

2.  Click on "Create an app", Give a name and select the workspace, and
    then "Create app"

3.  Go to settings of the app and click on "Oauth & Permissions"

    ![SlackInt2](/assets/images/ci-jenkins/image021.png)

4.  Scroll down to see "Scopes" and under "Bot Token Scopes" add the permission for "calls:write"

    ![SlackInt3](/assets/images/ci-jenkins/image022.png)

5.  Scroll up and click on "Install App to Workspace"

    ![SlackInt4](/assets/images/ci-jenkins/image023.png)

6.  You will be provided with a token. Copy the token and save it in the notepad.

7.  We need to add the app into the channel and invite the app to the channel.

    ![SlackInt5](/assets/images/ci-jenkins/image024.png)

8.  To make use of the slack bot, we need to install a plugin in Jenkins. Head to Jenkins portal and click on "Manage Jenkins"

    ![SlackInt6](/assets/images/ci-jenkins/image025.png)

9.  Manage Jenkins --> Manage Plugins --> Available Search for "Slack" "Install without Restart"

    ![SlackInt7](/assets/images/ci-jenkins/image026.png)

10. To configure the token, Navigate to Manage Jenkins --> Manage Credentials --> Store 'Jenkins' --> 'Global Credentials' --> Add Credentials --> Change the kind to "secret text"

    ![SlackInt8](/assets/images/ci-jenkins/image027.png)

11. Now we will integrate with Slack with Jenkins. Navigate to Manage
    Jenkins Configure System Scroll to the end for Slack settings Put
    checkmark on "Custom Slack app bot user" and save the settings.

![A screenshot of a computer Description automatically
generated](media/image28.png){width="3.144115266841645in"
height="0.957237532808399in"}

12. Configure the job to send notifications.

![](media/image29.png){width="1.2765113735783027in"
height="1.5165102799650043in"}

13. Navigate to "Post build Actions"

![A screenshot of a computer Description automatically
generated](media/image30.png){width="1.2152712160979877in"
height="1.059491469816273in"}

14. Click on Advanced, Enter the workspace name, credential
    (slack-token), channel/member id - \#jenkins Test Connection

15. Go back to slack channel and you would have got a notification.

![A screenshot of a computer Description automatically
generated](media/image31.png){width="2.5608398950131233in"
height="0.5830413385826771in"}

 ## Unit Tests, Code Analysis with Sonarqube ## 

1.  We are going to create a job in Jenkins called as "Test" and as a
    Freestyle Project and copy from the Build job that we created
    earlier.

2.  In the Build phase, provide the goals as "test", rest of the
    settings copied from "build" job.

![A white line on a white surface Description automatically
generated](media/image32.png){width="2.360485564304462in"
height="0.6726312335958006in"}

3.  Run the test job.

4.  This test job should automatically run after the build job, so to
    configure this, Navigate to "Build" job Configure "Build" Tab Add
    post-build action Select "Build Other projects" Projects to build as
    "Test"

![](media/image33.png){width="3.075265748031496in"
height="0.91208552055993in"}

5.  Configure Integration job. Freestyle Job "Build Environment" tab
    Goals "verify -DskipUnitTests Rest of the settings comes from
    "Build" job.

![A screenshot of a computer Description automatically
generated](media/image34.png){width="1.9648611111111112in"
height="0.9965802712160979in"}

6.  This integration job should automatically run after the test job, so
    to configure this, Navigate to "Build" job Configure "Build" Tab Add
    post-build action Select "Build Other projects" Projects to build as
    "Integration Test"

7.  So upstream job is "Build" and downstream job is "Integration Test"

![](media/image35.png){width="1.3782436570428696in"
height="1.2932020997375329in"}

8.  To perform simple code analysis, we have to install a plugin called
    "checkstyle"

9.  Manage Jenkins Manage Plugins Available Search for "checks" Select
    "Checkstyle" and "Violations" "Install without Restart"

> ![](media/image36.png){width="1.937607174103237in"
> height="1.0430785214348206in"}

10. Checkstyle will analyze the code and give the report. If the
    failures goes beyond threshold defined by quality gates, Violations
    will make the job unstable, so it won't be promoted to next step.

11. Create a job in Jenkins for code analysis. Create new job and copy
    the settings of build job. For goals, enter "checkstyle:checkstyle".

12. Configure "Post Build Actions" "Publish Checkstyle analysis results

![A screenshot of a computer Description automatically
generated](media/image37.png){width="2.3325721784776903in"
height="1.5497933070866141in"}

13. Run the job and results will be saved in the workspace

![](media/image38.png){width="3.499795494313211in"
height="0.3373392388451444in"}

14. Create a new job as "Code_Analysis" and copy the settings of "Build"
    job

15. In "Build Environment" tab Invoke top-level Maven targets Goals
    "checkstyle:checkstyle"

![A white rectangular object with a white line Description automatically
generated](media/image39.png){width="2.5582195975503064in"
height="0.7235903324584427in"}

16. If you want to perform the Quality Gates checks for the source code
    in a different branch Under "Source Code Management" Select "Git"
    Enter Repository Url Then "Branch Specifier"

![](media/image40.png){width="2.4748140857392826in"
height="1.3761461067366578in"}

17. If you want to perform the quality checks for code in the branch,
    either create a settings.xml file in the branch or use the default
    maven settings.

18. Under build tab, we configure the violations and stop the code to
    get promoted to next level if the quality checks are not passed.

![](media/image41.png){width="2.971868985126859in"
height="2.0512674978127734in"}

19. After configuring and testing the job, we can integrate the job.
    Jenkins Home Configure Integration Test Make the "Code_Analysis" job
    as the downstream job of "Integration Test" job using Post Build
    Actions

![A screenshot of a computer Description automatically
generated](media/image42.png){width="2.8520264654418197in"
height="0.6344860017497813in"}

20. Now we are going to analyse the code with checkstyle

21. Run the "Code Analysis" job to the publish the results

![](media/image43.png){width="1.818001968503937in"
height="1.34708552055993in"}

22. Click "Console Output" and there we can find the location of the
    published results in .xml format

![](media/image44.png){width="3.562288932633421in"
height="0.3540190288713911in"}

23. We can find this checkstyle-result.xml in our Code Analysis job's
    workspace

![A screenshot of a computer Description automatically
generated](media/image45.png){width="2.537905730533683in"
height="1.6544477252843395in"}

24. Under Workspace, we can see the result.xml

![](media/image46.png){width="2.4957491251093615in"
height="0.8416863517060368in"}

25. We get a graph with the data from violations in
    checkstyle-result.xml, the graph shows the violations between first
    job and second job. If we have fixed some violations, we will see a
    decrease in this graph.

![A white background with blue text Description automatically
generated](media/image47.png){width="2.837882764654418in"
height="0.7787981189851269in"}

26. We are going to give the checkstyle-result.xml file for scanning. In
    Post Build Action Add post-build section Report violations

> ![](media/image48.png){width="1.9519466316710412in"
> height="1.2437007874015749in"}

27. We can use other code analysis tools as you see in the below
    screenshot like "codenarc", "cpd" etc. But for our use case we are
    using "checkstyle"

![A screenshot of a computer Description automatically
generated](media/image49.png){width="2.5809120734908135in"
height="1.135761154855643in"}

28. According to the organization's policies, we can set the toleration.
    In our new case, we have set the maximum toleration to 100.

29. We have tested the "Code Analysis" Job and now we can configure this
    job as a downstream job for "Integration Tests" job.

![A screenshot of a test Description automatically
generated](media/image50.png){width="1.9022058180227472in"
height="1.1464654418197726in"}

30. We now have four jobs connected with each other. First is Build,
    Second is Test, Third is Integration Test and the last one will be
    Code_Analysis

![A screenshot of a computer Description automatically
generated](media/image51.png){width="3.253721566054243in"
height="0.886428258967629in"}

31. We have one more code analysis with Sonarqube and we are going to
    publish the results from checkstyle and sonarqube to Sonarqube
    dashboard.

32. We should have Sonarqube server up and running. We will access this
    server using its public IP address through port 80 with a browser.

![A screenshot of a computer Description automatically
generated](media/image52.png){width="2.108494094488189in"
height="1.3133114610673666in"}

33. Default username is admin and password is admin. Before putting it
    into production, change to a strong password. After logging in, you
    can see the projects page

![A screenshot of a computer Description automatically
generated](media/image53.png){width="3.884145888013998in"
height="1.0401049868766403in"}

34. To integrate Jenkins with Sonarqube, Jenkins needs to authenticate
    to Sonarqube Server, for that we need to generate a token.

35. Top right corner Click the profile My Account Security Enter a token
    name and click on Generate. Copy the token and keep it safe to
    configure Jenkins.

![A screenshot of a computer Description automatically
generated](media/image54.png){width="1.9643864829396325in"
height="1.7243339895013123in"}

36. Login into Jenkins Manage Plugins

![](media/image55.png){width="2.631923665791776in"
height="0.8715726159230096in"}

37. Click on "Available" Tab Search for "sonar" Select "SonarQube
    Server" and "Sonar Quality Gates" Install without Restart

![](media/image56.png){width="2.590281058617673in"
height="1.7419685039370079in"}

38. Manage Jenkins Global Tool Configuration Scroll down to "SonarQube
    Scanner" Click on "Add SonarQube Scanner"

![](media/image57.png){width="1.8746730096237971in"
height="0.4390715223097113in"}

39. Select the version and provide the name for the scanner

![](media/image58.png){width="3.0101388888888887in"
height="0.8997736220472441in"}

40. Manage Jenkins Configure System Look for "SonarQube Servers" Put
    checkmark on "Enable Injection" then Click on "Add SonarQube"

![](media/image59.png){width="2.8532611548556432in"
height="0.5038877952755906in"}

41. Provide name and server URL. We can get the sonar server's IP
    address from the EC2 Dashboard Instances Private IP Address Since we
    are running nginx, the default port is 80. Add the URL and add the
    token through credential manager.

![](media/image60.png){width="2.6067268153980754in"
height="0.8978860454943132in"}

42. Add the credentials through credentials provider. We have to copy
    the token that we created in Step 35.

![](media/image61.png){width="1.932661854768154in"
height="0.7211614173228347in"}

43. The above setting configured to push Sonarqube results to server.
    But make the code go through Quality Gates, we have configure the
    Quality Gates settings in the same job. Scroll further down to see
    the setting. Quality Gates -- SonarQube and click on "Add Sonar
    Instance"

![A screen shot of a computer Description automatically
generated](media/image62.png){width="2.169233377077865in"
height="0.2821489501312336in"}

44. Fill the details in the below form

![A screenshot of a computer Description automatically
generated](media/image63.png){width="2.037087707786527in"
height="1.259808617672791in"}

45. Now we have to create SonarQube Analysis job. Provide any name,
    Name: SonarScanner-CodeAnalysis Freestyle Project Copy the settings
    from Build Job Set the "Build" Goals as "Install" Add one more step
    for build Execute SonarQube Scanner  
    ![](media/image64.png){width="2.5853871391076115in"
    height="1.3193318022747156in"}

46. We have to configure sonar analysis properties, with the code repo,
    name etc.

![](media/image65.png){width="2.121809930008749in"
height="1.1126224846894137in"}

47. Save and Run the job. You will get slack notification after the
    build is over.

![](media/image66.png){width="2.770119203849519in"
height="0.6104330708661417in"}

48. Login into Sonarqube server Projects You will see the Code analysis
    status below the repo name with Bugs, Vulnerabilities etc.

![](media/image67.png){width="3.6031463254593175in"
height="0.3113735783027122in"}

49. We can decide the quality gates to pass the code to next level. For
    that, we have to configure Quality Gates

![](media/image68.png){width="2.4004877515310588in"
height="0.8640791776027996in"}

50. Click Create Provide a Name Add Condition Select Bugs under
    Reliability

![](media/image69.png){width="1.6928762029746283in"
height="1.317577646544182in"}

51. We have to set a value, in order to stop it from passing the quality
    gates. Here we are going to set it to 50

![A screenshot of a computer Description automatically
generated](media/image70.png){width="2.636768372703412in"
height="0.7180566491688539in"}

52. Configure your repo with the newly created Quality gates. Repo
    Project Settings Quality Gates Select the quality gate we created.

53. After this, we have to configure the Post build job with Project key
    and Job Status when analysis fails to FAILED -- The job will fail
    and if you configure it as UNSTABLE -- The job will become unstable.
    Save the job.

54. Check the console output for the status of the job.

![](media/image71.png){width="3.135291994750656in" height="1.03375in"}

 ## Nexus Repository Integration ## 

1.  After passing the Quality Gates, the .war file which is called as an
    artifact will be generated. This has to be uploaded to the Nexus
    Artifact Repository.

![A screenshot of a computer Description automatically
generated](media/image72.png){width="2.288976377952756in"
height="1.0232688101487315in"}

2.  To integrate, we need the plugins to be installed. We need three
    plugins.

    a.  Nexus Artifact Uploader

    b.  Copy Artifact (Since we already have the artifact, we just need
        to copy it)

    c.  Zentimestamp -- For versioning

> Manage Jenkins Available tab Search for the above plugins and install
> it without restart

3.  We have to create a job to copy the artifact to Nexus. Head to
    Jenkins New Project Provide a Name Choose Freestyle Project We don't
    need source code hence don't copy settings from any job.

4.  In the Build section Add Build Step Copy artifacts from another
    project Provide a name Select "Latest Successful Build Copy only
    "\*\*/\*.war"

![](media/image73.png){width="2.3810553368328957in"
height="1.4632961504811899in"}

5.  Add build step Nexus Artifact Uploader Fill the details for Nexus
    Server For GroupId, provide QA or Testing, For Version provide
    "V\$BUILD_ID", For repository, we created a release repository,
    mention the name of the repository in the last field.

![A screenshot of a computer Description automatically
generated](media/image74.png){width="3.0129965004374455in"
height="1.532201443569554in"}

6.  Scroll up and change the date pattern under General Tab

![A screenshot of a computer Description automatically
generated](media/image75.png){width="1.9578346456692914in"
height="0.9817366579177603in"}

7.  Add Artifact ArtifactID as "\$BUILD_TIMESTAMP", Type as "war" and
    File name (war file name)

8.  Add Slack Notification Settings as Post Build Job.

 ## Create a Different View of the Pipeline ## 

1.  To design a different view of our pipeline execution, we need to
    install a plugin. Manage Jenkins Manage Plugins Available Tab Search
    for "Build Pipeline" Install without restart  
    ![](media/image76.png){width="3.0681200787401575in"
    height="0.7566622922134734in"}

2.  Home page My views Provide a "View Name" Select "Build Pipeline
    view"

![A screenshot of a computer Description automatically
generated](media/image77.png){width="2.5298654855643044in"
height="0.715009842519685in"}

3.  Select the first job in the Pipeline Flow Build (This job as all
    downstream jobs)

![](media/image78.png){width="2.4584755030621173in"
height="0.7760017497812773in"}

4.  Display Options No of Displayed Builds -- 5 Save the job

![A screenshot of a computer Description automatically
generated](media/image79.png){width="2.127944006999125in"
height="1.0495898950131233in"}

5.  You can see the below "Build Pipeline" view

![A screenshot of a computer program Description automatically
generated](media/image80.png){width="2.92748687664042in"
height="0.8341994750656168in"}

6.  Click Run and you can see all the pipelines to be executed.

![](media/image81.png){width="3.048751093613298in"
height="0.3509470691163605in"}

7.  We need to configure Jenkins to fetch the code automatically from
    SCM (Source Code Management. For this, we need to configure the
    Build job.

8.  Build Triggers Poll SCM This will be in cron job format This is
    configured to poll every minute and this is for testing purposes.
    Save the changes.  
    ![](media/image82.png){width="3.4282688101487313in"
    height="0.8056014873140858in"}

9.  To test this perform a empty Git commit and find the pipeline
    running.

10. If you are doing this as a test, don't forget to delete the EC2
    instances.

> Complete Flow  
>   
> ![](media/image83.png){width="2.619265091863517in"
> height="2.4111986001749783in"}
>
> Happy DevOpsing !!!
