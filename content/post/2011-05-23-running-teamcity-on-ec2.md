---
title: Running TeamCity on EC2
author: Pedro
type: post
date: 2011-05-23T17:19:07+00:00
aliases:
  - /2011/05/23/running-teamcity-on-ec2
dsq_thread_id:
  - 395135192
categories:
  - tutorial
tags:
  - build
  - cloud
  - ec2
  - TeamCity

---
&nbsp;

One of Headspring’s core mantras is to outsource everything that is not part of our core business. We are true believers on running the business in the cloud. [We are even giving a talk about it][1]. We use many cloud service providers on a daily basis. We use [Google Apps][2] for Email, Calendar, Voice and Intranet. We use [Salesforce][3] as our CRM. We use [BitBucket][4] to host our source code.

We use all this services to do things that are necessary in order to run the company but that we don’t want to manage or build ourselves. So, we decided to do the same thing with our build system. As we were using [TeamCity][5] as our CI server and it has [integration with EC2 out of the box][6] we decided to go with EC2.

First, TeamCity is composed of two pieces. The build server, that runs TeamCity’s dashboard and the build agents, that actually run the builds. As TeamCity Server is a Java application, a small Linux instance on EC2 is enough to run it. The server itself won’t leverage the Elastic features on EC2, but having the server running on the same region and availability zone as the agents is important in order to avoid [Data Transfer charges][7].

&nbsp;

## Server

&nbsp;

To install TeamCity server on Linux, follow this steps:

**1)** [Get TeamCity from JetBrains][8].

**2)** Unpack the file Teamcity<version number>.tar.gz as documented in the [docs][9].

**3)** Elevate your privileges to root:

 <span style="font-family: 'Courier New';">> sudo su</span>

This is needed because we want the server to listen to port 80 and ports below 1024 can only be listened by the root user.

**4)** Install Sun/Oracle JDK.

EC2 instance is a RedHat distribution, that uses OpenJDK by default. To the cloud integration on TeamCity to work properly, [it needs Sun/Oracle’s JDK][10].

**5)** With Sun/Oracle’s JDK installed, set the JAVA_HOME environment variable to use it, instead of OpenJDK:

 <span style="font-family: 'Courier New';">> JAVA_HOME=/usr/java/jdk<version></span>

**6)** [Change TeamCity server port to 80][11] (if you need to)

**7)** Start the Server. On TeamCity’s bin directory, run:

<span style="font-family: 'Courier New';">> ./teamcity-server.sh start</span>

 <span style="font-family: 'Courier New';"></span>

That’s it. The server should be up and running. If you hit instance’s public URL on your browser, after the server finishes starting up you should get a screen to accept TeamCity’s EULA.

[<img style="display: block; float: none; margin-left: auto; margin-right: auto;" src="http://farm3.static.flickr.com/2011/5750784189_1158b37dee.jpg" border="0" alt="TeamCIty_EULA" />][12]

After accepting the EULA, you will be prompted to create an Administrator Account. With the Administrator Account created, TeamCity server is installed and running properly.

## Installing the Build Agent

&nbsp;

With the Server up and running it’s time to create the agents that will run the builds.

To do so, first launch a instance ( I’m going to use a Windows one as we are doing .NET here) on EC2 and configure it to work as a TeamCity agent. In our case, we chose to use a Large Windows x64 instance.

As it’s necessary to remote into the instance to configure it to run as a TeamCity agent, wait approximately 15 minutes after the instance is launched to Remote into it, otherwise you won’t be able to get the password from AWS.

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="Wait_for_instance[1]" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/Wait_for_instance1.png" border="0" alt="Wait_for_instance[1]" width="381" height="127" />][13]

With the instance credentials, follow this steps:

**1)** Remote into the instance

**2)** Open the browser and go to TeamCity Server URL. After authenticating, go to the Agents tab and download MS Windows Installer.

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="Agents_Tab" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/Agents_Tab.png" border="0" alt="Agents_Tab" width="576" height="342" />][14]

**3)** Run the installer. After the agent is installed, a form to “Configure Build Agent Properties” will be prompted. Change the **serverUrl** property to point to the correct URI.

**4)** TeamCity agent-server communication uses port 9090 by default, unless you’ve changed it when configuring Build Agent properties on the previous step. So, in order to enable the communication between agent and server, create a inbound rule on Windows firewall to allow communication trough port 9090.

**5)** If everything is configured properly, you should have a Unauthorized agent showing up on TeamCity Server.

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="unauthorized_agent" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/unauthorized_agent.png" border="0" alt="unauthorized_agent" width="513" height="325" />][15]

**6)** Authorize the agent and check that it’s capable of running properly your builds.

&nbsp;

## Making the Build Agent Elastic

&nbsp;

Right now we have a EC2 based TeamCity server and a TeamCity Agent running on a EC2 instance but it’s not leveraging the Elastic features of TeamCity yet. To enable Elastic, Cloud Based, build agents one has to create a AMI image of the instance  running the Agent.

**1)** On AWS, select the running instances and right-click on the instance running the Build Agent. On the context menu, select “Create Image (EBS AMI)”

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="Create_Image_EBS[4]" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/Create_Image_EBS4.png" border="0" alt="Create_Image_EBS[4]" width="244" height="143" />][16]

**2)** When the AMI becomes available on aws, go to Cloud Tab under TeamCity Agents page:

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="cloud" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/cloud.png" border="0" alt="cloud" width="538" height="213" />][17]

**3)** On the configuration page, create a new Cloud Profile. As of now, the only option of Cloud Type is Amazon EC2. Choose your location and availability zone to match the ones the server instance is running in order to avoid extra Data Transfer costs.

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="cloud_profile[5]" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/cloud_profile5.png" border="0" alt="cloud_profile[5]" width="403" height="372" />][18]

&nbsp;

That’s it. TeamCity is configured to use Elastic Build Agents.

When a build is triggered, if there are no agents available to run it, the build will be placed on the Build Queue. A instance will be requested to EC2. It will take approximately 10 minutes for the instance to be running and authorized as a Build Agent on TeamCity.

[<img style="background-image: none; padding-left: 0px; padding-right: 0px; display: block; float: none; margin-left: auto; margin-right: auto; padding-top: 0px; border: 0px;" title="instance_starting" src="https://pedroreys.com/blog/wp-content/uploads/2011/05/instance_starting.png" border="0" alt="instance_starting" width="538" height="138" />][19]

My goal with this post was to document the process of migrating Headspring’s build system to leverage EC2 features. I hope this helps not only me.

One last thing, can you do me a favor and vote for [this TeamCity feature request][20]?

 [1]: http://www.eventbrite.com/event/1707145117
 [2]: http://www.google.com/apps/intl/en/business/#utm_campaign=en&utm_source=en-ha-na-us-bk&utm_medium=ha&utm_term=google%20apps
 [3]: http://www.salesforce.com/
 [4]: http://bitbucket.org
 [5]: http://www.jetbrains.com/teamcity/
 [6]: http://www.jetbrains.com/teamcity/features/amazon_ec2.html
 [7]: http://aws.amazon.com/ec2/pricing/
 [8]: http://www.jetbrains.com/teamcity/download/index.html#linux
 [9]: http://confluence.jetbrains.net/display/TCD6/Installing+and+Configuring+the+TeamCity+Server#InstallingandConfiguringtheTeamCityServer-InstallingTeamCitybundledwithTomcatservletcontainer%28Linux%2CMacOSX%2CWindows%29
 [10]: http://youtrack.jetbrains.net/issue/TW-14961
 [11]: http://confluence.jetbrains.net/display/TCD6/Installing+and+Configuring+the+TeamCity+Server#InstallingandConfiguringtheTeamCityServer-ChangingServerPort
 [12]: http://www.flickr.com/photos/22313104@N04/5750784189/ "TeamCIty_EULA"
 [13]: http://www.flickr.com/photos/22313104@N04/5751337652/
 [14]: http://www.flickr.com/photos/22313104@N04/5750832543/
 [15]: http://www.flickr.com/photos/22313104@N04/5751442728/
 [16]: http://www.flickr.com/photos/22313104@N04/5750940659/
 [17]: http://www.flickr.com/photos/22313104@N04/5750956671/
 [18]: http://www.flickr.com/photos/22313104@N04/5751563606/
 [19]: http://www.flickr.com/photos/22313104@N04/5751144159/
 [20]: http://youtrack.jetbrains.net/issue/TW-16490
