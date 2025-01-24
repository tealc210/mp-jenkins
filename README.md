# If you want to use your own Jenkins Instance...

You need to install the following Jenkins plugins :

- docker-build-step
- Docker Commons Plugin
- Sonar Quality Gates Plugin
- SonarQube Scanner for Jenkins
- SSH Agent Plugin
- Eclipse Temurin installer Plugin
- Slack Notification Plugin

NB : eazytraining/jenkins docker image is used

**################################################################################**

In order to use the pipeline, you'll have to configure Jenkins.
First create a "multibranch pipeline" job

![](images/job.png)

Fill the "Branch Sources" section with the information related to your repository. In the "Build Configuration" section, leave the "Script Path" value as it is, unless you place the Jenkinsfile in another location.

![](images/jfile.png)

Then go to "Manage Jenkins > System", and set the "SonarQube servers" section as follow :

![](images/sonar.png)

Following is the "Quality Gates - Sonarqube" section :

![](images/gates.png)

And in the "Slack" section fill the fields with the required informations related to your account.

![](images/slack.png)

After this, go to "Manage Jenkins > Tools" and set "Maven installations as follow" :

![](images/mvn_bin.png)

Then set "JDK installations" the same way :

![](images/ava_bin.png)

When all of this is done, you'll be able to use the pipeline, with a simple push to your repository.

![](images/pipeline.png)
