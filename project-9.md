## __AUTOMATING TOOLING WEBSITE DEPLOYMENT WITH CONTINOUS INTEGRATION(CI)-JENKINS__ ##

In [Project-8](https://github.com/dybran/Project-8/blob/main/project-8.md), we introduced horizontal scalability concept, which allow us to add new Web Servers to our Tooling Website and we successfully deployed a set up with 2 Web Servers and also a Load Balancer to distribute traffic between them. It is not a big deal to configure two 0r three servers manually. Imagine that you would need to repeat the same task over and over again adding dozens or even hundreds of servers.

DevOps is about Agility, speedy release of software and web solutions. One of the ways to guarantee fast and repeatable deployments is Automation of routine tasks.

In this project we are going to automate part of our routine tasks with a free and open source automation tool – __Jenkins.__

__Continuous integration__ is a DevOps software development practice where developers regularly merge their code changes into a central repository, after which automated builds and tests are run. Continuous integration most often refers to the build or integration stage of the software release process and entails both an automation component (e.g. a CI or build service) and a cultural component (e.g. learning to integrate frequently). The key goals of continuous integration are to find and address bugs quicker, improve software quality, and reduce the time it takes to validate and release new software updates.

According to Circle CI, Continuous integration (CI) is a software development strategy that increases the speed of development while ensuring the quality of the code that teams deploy. Developers continually commit code in small increments (at least daily, or even several times a day), which is then automatically built and tested before it is merged with the shared repository.

In our project we are going to utilize Jenkins CI capabilities to make sure that every change made to the source code in GitHub [my https://github.com/dybran/tooling will be automatically be updated to the Tooling Website.

Our 3 tier achitecture will look like this

![image](./images/pr9.PNG)


__INSTALL AND CONFIGURE JENKINS SERVER__

Create an AWS EC2 server based on Ubuntu Server 20.04 LTS

![image](./images/jenkins-instance.PNG)

Install JDK-Jenkins is a Java-based application

```$ sudo apt update -y```

```$ sudo apt install default-jdk-headless```

![image](./images/update.PNG)
![image](./images/install-jdk.PNG)

Install Jenkins

```$ wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -```

```sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \```
```    /etc/apt/sources.list.d/jenkins.list'```

```$ sudo apt update -y```

``` $ sudo apt-get install jenkins```

![image](./images/install-jenkins.PNG)

Make sure Jenkins is up and running

```$ sudo systemctl status jenkins```

![image](./images/jenkins-status.PNG)

By default Jenkins server uses __TCP port 8080__. We open it in our Inbound Rule in the EC2 Security Group.

![image](./images/inbound-jenkins.PNG)

We access jenkins from the broswer using

```http://<Jenkins-Server-Public-IP-Address>:8080```

You will be prompted to provide a default admin password

![image](./images/jenkins-page.PNG)

We can retrive the admin password from the path indicated on the screen.
To do this we have to connect to the instance through our terminal and run the command

```$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword```

![image](./images/adminpas.PNG)

Copy the password and paste in the space provided.

Then you will be asked which plugings to install – choose __"install suggested plugins".__

![image](./images/jenkins-after-admin-passwd.PNG)

Once plugins installation is done – create an admin user and you will get your Jenkins server address.

![image](./images/instance-jenkins-config.PNG)

Click on __"save and finish"__

The installation is completed!

![image](./images/ready.PNG)

__CONFIGURE JENKINS TO RETRIEVE SOURCE CODE FROM GITHUB USING WEBHOOKS__

We will be configuring a simple Jenkins job/project. This job will will be triggered by GitHub webhooks and will execute a ‘build’ task to retrieve codes from GitHub and store it locally on Jenkins server.

To enable webhooks in GitHub repository, We open the [tooling repository](https://github.com/dybran/tooling).
Then click on __"settings"__ and click on __"webhooks"__.

![image](./images/git-webhook.PNG)
![image](./images/add-webhook.PNG)

Edit the following

![image](./images/add-webhook2.PNG)

Click on __"add webhook"__.

__N/B:__
If we are not using Elastic IP, We should always edit the URL in the webhook settings whenever we restart our jenkins as the IP address of the jenkins will change.

We go to Jenkins web console, click __"New Item"__ and create a __"Freestyle project".__

![image](./images/new-item-jenkins.PNG)
![image](./images/freestyle-jenkins.PNG)

To connect to the github repository, We copy the github repository URL.

![image](./images/github-url.PNG)

In configuration of your Jenkins freestyle project choose Git repository, provide there the link to your Tooling GitHub repository and credentials (user/password) so Jenkins could access files in the repository.

![image](./images/github-username-pass.PNG)

We save the configuration and try to run the build manually.

Click __"Build Now"__ button, if we have configured everything correctly, the build will be successful and you will see it under __#1__ on the console.

![image](./images/click-build.PNG)
![image](./images/build-%231.PNG)

You can open the build (by clicking on the __#1__ )and check in __"Console Output"__ if it has run successfully.

![image](./images/console-output.PNG)

But this build does not produce anything and it runs only when we trigger it manually.
Let us continue with the configuration process.


Click __"Configure"__ on the job/project and add these configurations to trigger the job automatically from github webhook and to archive all the artifacts from the build.

Configure triggering the job from GitHub webhook. Click on __"configure"__

![image](./images/click-configure.PNG)

Then click on __"build triggers"__ and select __"github hook trigger for GITscm polling".__

![image](./images/click-configure2.PNG)

To archive all the artifacts, click on __"add post build action"__ and select __" archive artifacts".__

![image](./images/archive-artifacts.PNG)

Then edit the __"files to archive"__ section  with ** and __"save".

![image](./images/post-build.PNG)

To test this, we make some change in any file in the GitHub repository (e.g. README.MD file) and push the changes to the master branch.

We should see that a new build has been launched automatically (by webhook) and we can see its results – artifacts, saved on Jenkins server.
![image](./images/2nd-build.PNG)
![image](./images/2nd-build2.PNG)

We have now configured an automated Jenkins job that receives files from GitHub by webhook trigger.

By default, the artifacts are stored on Jenkins server locally

```$ ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/```

![image](./images/artifact-store.PNG)

__CONFIGURE JENKINS TO COPY FILES TO NFS SERVER VIA SSH__

Now we have our artifacts saved locally on Jenkins server, the next step is to copy them to our __NFS server__ to __/mnt/apps__ directory as it is not a good practice to save artifacts on the jenkins server. Jenkins keeps artifacts for a build only as long as the build log itself is kept. This means that artifacts that you deployed to a server may be deleted. There is no easy way to roll back if artifacts are deleted.

Jenkins is a highly extendable application and there are 1400+ plugins available. We will need a plugin that is called __"Publish Over SSH".__ to copy the artifacts to the NFS server.

To install __"Publish Over SSH plugin"__, we open __"Manage Jenkins"__ on the main dashboard and choose __"Manage Plugins"__ menu item. Then select __"available plugins"__ and type __"publish over ssh"__- select it and click on __"install without restart".__

![image](./images/publish-over-ssh-1.PNG)
![image](./images/publish-over-ssh-dwnld.PNG)

After download is completed, Configure the job/project to copy artifacts over to NFS server.

On main dashboard, select __"Manage Jenkins"__ and choose __"Configure System"__ menu item.

![image](./images/publish-over-for-nfs.PNG)

Scroll down to __"Publish over SSH plugin configuration"__ section and configure it to be able to connect to your NFS server.

Provide a private key (content of .pem file that you we use to connect to NFS server)

![image](./images/publish-over-for-nfs1.PNG)
![image](./images/publish-over-for-nfs2.PNG)

Test the configuration and make sure the connection returns __Success.__

![image](./images/publish-test-config.PNG)

save the configuration.

 __N/B:__
 
 - Remember, that __TCP port 22__ on NFS server must be open to receive SSH connections.

 - If the __test configuration__ is not returning __"success"__, we go to the __"configure global security"__ and check __"enable proxy compatibility"__
![image](./images/configure-global-security.PNG)
![image](./images/configure-global-security-1.PNG)

open your Jenkins job/project configuration page and open __"Post-build Action"__.
Then select __"send build artifact over ssh"__ from the options.

![image](./images/send-over-ssh.PNG)

Configure it to send all files produced by the build into our previously defined remote directory i.e __/mnt/apps__. In our case we want to copy all files and directories – so we use **.

![image](./images/send-over-ssh1.PNG)

We save this configuration and go ahead, change something in README.MD file in your GitHub Tooling repository.

Webhook will trigger a new job.

![image](./images/build-after-publish-config.PNG)

When we click on the new job and access the console output we should see the following at the end of the build information.

```
SSH: Transferred 25 file(s)
Finished: SUCCESS
```

To make sure that the files in /mnt/apps have been updated, connect to the NFS server through a terminal and check README.MD file.

```$ sudo cat /mnt/apps/README.md```

![image](./images/readme-edit-on-mnt-apps.PNG)

We have just __Automated Tooling Website Deployment with Jenkins (CI) tool.__

__Problem ecountered during this project:__

- The __Rhel 9 version__ is unstable at the time of this documentation. In the __Publish over SSH__ configuration, the __"Test Configuration"__ was not returning the required output i.e __"success"__ when using the __RHEL version 9__.


























