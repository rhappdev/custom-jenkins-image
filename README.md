# Custom Jenkins Image
Create Custom Jenkins image using, Jenkins S2i Image

## Architecture
![Jenkins custom image autobuild](https://github.com/rhappdev/assets/blob/master/custom_jenkins_image/custom_jenkins_autobuild.png)

## Setup

### Configuration files
[Jenkins S2i Image](https://github.com/openshift/jenkins) requires a folder structure:
```configuration
  |- secrets
  |   |- hudson.util.Secret
  |   |- master.key
  |- credentials.xml (Credentials stored in Jenkins)
  |- org.jenkinsci.plugins.pipeline.modeldefinition.config.GlobalConfig |  (Global configuration files plugin setup)
  |- org.jenkinsci.plugins.workflow.libs.GlobalLibraries (Global shared    pipeline configuration)```

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

