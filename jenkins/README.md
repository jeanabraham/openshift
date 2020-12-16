# Jenkins Setup

# Prereq

* Install [podman](https://podman.io/getting-started/installation.html)


* Login to Openshift cluster

```
oc login --token=$TOKEN --server=$OCP_SERVER
```

# Setting up Jenkins CI/CD:

```
oc new-project poc-jenkins
oc get templates -n openshift | grep jenkins
oc new-app jenkins-ephemeral
oc get pods
oc get route -n poc-jenkins
```

* Go to Jenkins URL (route) on browser

* Configure Openshift Jenkin Sync plugin to recognize namespace 'poc-openshift' (or whatever project name). This needs to be on the Jenkins UI.
*Manage Jenkins -> Configure System -> Openshift Jenkins Sync -> Namespace*



* The service account associated with the Jenkins deployment must have the edit role for each project monitored by Jenkins. Add the edit role to the service account associated with the Jenkins deployment for your project.
```
oc policy add-role-to-user edit system:serviceaccount:poc-jenkins:jenkins -n poc-openshift
```

* Setup secure git repo access from Openshift/Jenkins

1. Generate [SSH private and public keys](https://help.github.com/en/enterprise/2.15/user/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) 

2. Import private key (id_rsa) to create Openshift secret 
```
oc create secret generic openshift-poc-git-ssh-key --from-file=ssh-privatekey=id_rsa
oc label secret openshift-poc-git-ssh-key  credential.sync.jenkins.openshift.io=true
```
Note: Use `sourceSecret` in buildconfig.yaml because of [this bug](https://github.com/openshift/jenkins-sync-plugin/issues/101)

3. Import public key (id_rsa.pub) into Git repo settings


* Create JenkinsFile in root directory of the git repo to define build/deploy stages

## Enable Maven

Go to *Manage Jenkins -> Global Tool Configuration* and Add Maven. Take note of the Maven name since you need to refer to it when creating deployment pipelines. 


## Setup Build Trigger

* Create trigger config
```
oc set triggers bc/<build-config-name> --from-gitlab
```
* Update git repo settings with generated trigger URL in buildconfig

Example:
```
oc describe bc sample-api-pipeline
Name:		sample-api-pipeline
Namespace:	poc-openshift
.....
Triggered by:		<none>
Webhook GitLab:
	URL:	https://<OCP host:port>/apis/build.openshift.io/v1/namespaces/poc-openshift/buildconfigs/sample-api-pipeline/webhooks/<secret>/gitlab
```

Replace *secret* with value from updated buildconfig yaml.