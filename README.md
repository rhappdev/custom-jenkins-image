# Custom Jenkins Image
Create Custom Jenkins image using, Jenkins S2i Image

## Architecture
![Jenkins custom image autobuild](https://github.com/rhappdev/assets/blob/master/custom_jenkins_image/custom_jenkins_autobuild.png)

## Setup

### Configuration files
[Jenkins S2i Image](https://github.com/openshift/jenkins) requires a folder structure:
![Tree structure](https://github.com/rhappdev/assets/blob/master/custom_jenkins_image/tree_structure.png)

### Create custom jenkins image
1. Login into oc CLI: 
``` oc login <host>```
2. Add username
3. Add password
4. Create project if needed:
```oc new-project <projectName>```
5. (Optional) if your repo is private, create a new secret:
```oc create secret generic git-secret --from-literal=username=<username> --from-literal=password=<password> -n <projectName>```
5. Create the new build (if the repository is private add ```--source-secret=git-secret```):
```oc new-build jenkins:2~https://github.com/rhappdev/custom-jenkins-image.git --name=custom-jenkins -e GIT_SSL_NO_VERIFY=true -e OVERRIDE_PV_CONFIG_WITH_IMAGE_CONFIG=true -e OVERRIDE_PV_PLUGINS_WITH_IMAGE_PLUGINS=true -n <projectName>```
6. Wait until the image is created.

### Create new jenkins app, using jenkins-persistent template
1. Use jenkins-persistent template:
```oc new-app jenkins-persistent -p JENKINS_IMAGE_STREAM_TAG=custom-jenkins:latest -p NAMESPACE=<projectName> -p MEMORY_LIMIT=4Gi -p VOLUME_CAPACITY=10Gi -n <namespace>```
2. Wait until the new pod is running.
3. Access the Jenkins URL and check if jenkins is configured with our initial configuration.

### Create web-hook for autobuild.
If we need to change something in the configuration, this will trigger a new build, and automatically redeploy our jenkins app.
1. Make sure that github webhook is enabled:
```oc describe bc/custom-jenkins -n <projectName>```
2. Check if Webhook GitHub it's created:
```Webhook GitHub:URL:	https://master.na39.openshift.opentlc.com:443/apis/build.openshift.io/v1/namespaces/misanche-jenkins/buildconfigs/custom-jenkins/webhooks/<secret>/github```
If not type the following:
  * Github: ```oc set triggers bc/custom-jenkins --from-github```
  * Bitbucket: ```oc set triggers bc/custom-jenkins --from-bitbucket```
  * Gitlab: ```oc set triggers bc/custom-jenkins --from-bitbucket```
3. Get the Secret:
```oc get bc/custom-jenkins -o json -n misanche-jenkins``` pickup the url and add it to the url in ```<secret>````
```https://master.na39.openshift.opentlc.com:443/apis/build.openshift.io/v1/namespaces/misanche-jenkins/buildconfigs/custom-jenkins/webhooks/<secret>/github```
4. Go to Github -> <yourRepo> -> Settings -> Webhooks -> Add Webhook
  * PayloadURL: ```https://master.na39.openshift.opentlc.com:443/apis/build.openshift.io/v1/namespaces/misanche-jenkins/buildconfigs/custom-jenkins/webhooks/<secret>/github```
  * Content type: ```application/json```
  * Secret: ```<secret>```
  * SSL verification: enabled if certificate is ok
  * which events would you like to trigger this webhook?: Just the push event.
5. Now if you push something to the repo a new build will be triggered.

### Jenkins permissions
Openshift login plugin lets you login to Jenkins with your account on an OpenShift installation using the flag OPENSHIFT_ENABLE_OAUTH when creating the app based on jenkins-persistent template (default to true).

#### Role Mappings
For the admin role, the Jenkins permissions are the same permissions as those assigned to an administrative user within Jenkins

For the view role, the Jenkins permissions are:
```hudson.model.Hudson.READ
hudson.model.Item.READ
com.cloudbees.plugins.credentials.CredentialsProvider.VIEW```



For the edit role, in addition to the permissions available to view:
```hudson.model.Item.BUILD
hudson.model.Item.CONFIGURE
hudson.model.Item.CREATE
hudson.model.Item.DELETE
hudson.model.Item.CANCEL
hudson.model.Item.WORKSPACE
hudson.scm.SCM.TAG
jenkins.model.Jenkins.RUN_SCRIPTS```


> When this plugin manages authentication, the predefined admin user in the default Jenkins user database for the OpenShift Jenkins image is now ignored

> Permissions for users in Jenkins can be changed in OpenShift after those users are initially established in Jenkins. The OpenShift Login plugin polls the OpenShift API server for permissions and will update the permissions stored in Jenkins for each Jenkins user with the permissions retrieved from OpenShift. Technically speaking, you can change the permissions for a Jenkins user from the Jenkins UI as well, but those changes will be overwritten the next time the poll occurs.

## Best practises

### Shared Libraries
To learn more about Shared Libraries, refer to this [Repo](https://github.com/rhappdev/shared-jenkins-pipelines/blob/master/sections/setup.md)


