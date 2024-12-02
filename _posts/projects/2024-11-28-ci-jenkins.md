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

![FlowDiagram](/assets/images/ci-jenkins/flow-diagram.png)

<p style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'><strong><u>Flow of Execution</u></strong></p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>Login to AWS Account</li>
    </ul>
</div>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;margin-left:.5in;'>&nbsp;</p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>Create login key</li>
    </ul>
</div>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;margin-left:.5in;'>&nbsp;</p>
<ul style="list-style-type: disc;">
    <li>Create Security Group</li>
</ul>
<ul style="list-style-type: square;margin-left: 0.25in;">
    <li>Jenkins</li>
    <li>Nexus</li>
    <li>Sonar</li>
</ul>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;margin-left:.75in;'>&nbsp;</p>
<ul style="list-style-type: disc;">
    <li>Create EC2 instances with user data</li>
</ul>
<ul style="list-style-type: square;margin-left: 0.25in;">
    <li>Jenkins</li>
    <li>Nexus</li>
    <li>Sonar</li>
</ul>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;margin-left:.75in;'>&nbsp;</p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>Jenkins Post Installation</li>
    </ul>
</div>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;margin-left:.5in;'>&nbsp;</p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>Nexus Repository Setup</li>
    </ul>
</div>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;'>&nbsp;</p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>SonarQube post installation</li>
    </ul>
</div>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;'>&nbsp;</p>
<ul style="list-style-type: disc;">
    <li>Jenkins Steps</li>
</ul>
<ul style="list-style-type: square;margin-left: 0.25in;">
    <li>Build Job</li>
    <li>Setup Slack Notification</li>
    <li>Check style Code Analysis Job</li>
    <li>Setup Sonar integration</li>
    <li>Sonar code analysis job</li>
    <li>Artifact upload Job</li>
</ul>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;'>&nbsp;</p>
<ul style="list-style-type: disc;">
    <li>Connect all jobs together with BuildPipeline</li>
</ul>
<ul style="list-style-type: square;margin-left: 0.25in;">
    <li>Set Automatic Build Trigger</li>
</ul>
<p style='margin:0in;font-size:15px;font-family:"Aptos",sans-serif;'>&nbsp;</p>
<div style='margin-top:0in;margin-right:0in;margin-bottom:8.0pt;margin-left:0in;font-size:11.0pt;font-family:"Aptos",sans-serif;'>
    <ul style="list-style-type: disc;">
        <li>Test with VSCode or Intellij</li>
    </ul>
</div>



