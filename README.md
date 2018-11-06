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
hudson.model.Hudson.READ,
hudson.model.Item.READ
com.cloudbees.plugins.credentials.CredentialsProvider.VIEW



For the edit role, in addition to the permissions available to view:
hudson.model.Item.BUILD
hudson.model.Item.CONFIGURE
hudson.model.Item.CREATE
hudson.model.Item.DELETE
hudson.model.Item.CANCEL
hudson.model.Item.WORKSPACE
hudson.scm.SCM.TAG
jenkins.model.Jenkins.RUN_SCRIPTS


> When this plugin manages authentication, the predefined admin user in the default Jenkins user database for the OpenShift Jenkins image is now ignored

> Permissions for users in Jenkins can be changed in OpenShift after those users are initially established in Jenkins. The OpenShift Login plugin polls the OpenShift API server for permissions and will update the permissions stored in Jenkins for each Jenkins user with the permissions retrieved from OpenShift. Technically speaking, you can change the permissions for a Jenkins user from the Jenkins UI as well, but those changes will be overwritten the next time the poll occurs.

## Best practises

### Shared Libraries
To learn more about Shared Libraries, refer to this [Repo](https://github.com/rhappdev/shared-jenkins-pipelines/blob/master/sections/setup.md)

### Integration tests

To guarantee the maximum security over the deployments that we make over the platform, postman has been chosen as the “designer” of the tests collections and ```newman``` as executor of those test created in the collection. 

The idea is that following the API First approach that we use in the App development center of excellence, define tests in postman based on the API definition or Swagger definition.

Setup a configuration folder as defined in one of our templates:
* [Node.js](https://github.com/rhappdev/nodejs-template/tree/master/configuration/postman)
* [Springboot](https://github.com/rhappdev/springboot-template/tree/master/configuration/postman)

### Blue/Green Deployments
Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.
[Jenkins Pipeline](https://github.com/rhappdev/shared-jenkins-pipelines/blob/master/sections/setup.md) it's already prepared to allow a blue-green deployment.

### Publish the images to an external registry (Nexus)
If you want to copy the images from the internal registry to the external registry like nexus, skopeo allows you to do this operation without too much effort:
Requirements:
* fromRegistryImgUrl: The url of our openshift registry and the image with tag that we want to copy
* fromRegistryCredentials: Openshift account with image-puller permissions
* toRegistryImgUrl: The url of the destination docker repository where the image is going to be copied.
* toRegistryCredentials: Username:password with permissions to pushing images.

```skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:<fromRegistryCredentials> --dest-creds <toRegistryCredentials> docker://<fromRegistryImgUrl> docker://<toRegistryImgUrl>```

Example:
```skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/<devNamespace>/<app>:${devTag} docker://nexus-registry.nexus.svc.cluster.local:5000/<app>:${devTag}```

### Store Credentials in Jenkins

To follow the best practises, don’t store the credentials in the Jenkinsfile, always try to retrieve the credentials from Jenkins.

1. Create a username with password credential
2. Username: ```openshift```
3. Password: ```<TOKEN>```
4. Id: ```<desiredCredentialId>```
5. Description: ```<desiredDescription>```

### Copy the image

withCredentials([usernamePassword(credentialsId: 'prod-sa', passwordVariable: 'password', usernameVariable: 'username')]) {
  skopeo copy --remove-signatures
 --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds $username:$password docker://docker-registry.default.svc.cluster.local:5000/<namespaceSrc>/<app>:${desiredTag} docker://<toRegistryUrl>/<app>:${desiredTag}
}

### Don’t: Use input within a node block
The input element pauses pipeline execution to wait for an approval - either automated or manual. Naturally these approvals could take some time. The node element, on the other hand, acquires and holds a lock on a workspace and heavy weight Jenkins executor - an expensive resource to hold onto while pausing for input.

So, create your inputs outside your nodes.
Example:

stage ('deployment') {
input 'Do you approve deployment?'
	node(‘maven’){
   		// code
	}
}


### Do: Prefer stashing files to archiving

If you just need to share files between stages and nodes of your pipeline, you should use stash/unstash instead of archive.
Stash and unstash are designed for sharing files, for example your application’s source code, between stages and nodes. Archives, on the other hand, are designed for longer term file storage (e.g., intermediate binaries from your builds).

```stash excludes: "target/", name: "source"
unstash 'source'```






