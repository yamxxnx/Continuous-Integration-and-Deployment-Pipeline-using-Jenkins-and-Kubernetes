# CI/CD Pipeline Example with Kubernetes and Jenkins

## 1. Prerequisites:

* Basic knowledge of Docker, Kubernetes, and Jenkins
* A Kubernetes cluster set up and running
* Docker installed on your local machine
* Jenkins installed and running on a server

## 2. Install Jenkins

* Add Helm stable charts repository:

```sh
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
```

* Set Jenkins username and password:

```sh
JENKINS_USER={{USERNAME}}
JENKINS_PASS={{PASSWORD}}
```

* Deploy Jenkins to cluster:

```sh
helm install jenkins stable/jenkins \
--namespace jenkins \
--set master.adminUser="$JENKINS_USER" \
--set master.adminPassword="$JENKINS_PASS" \
```

### 2.2 Expose Jenkins

* Expose Jenkins to the open internet as this will be required when setting up webhooks later. Depending on the cluster setup, this may require setting up an ingress and having an ingress controller installed.

### 2.3 Configure Jenkins

* Go to `Jenkins > Manage Jenkins > Configure System`.
  * Update `# of executors` field to a non-zero value, recommended to 2.
  * Update `Jenkins Location` section.

## 3. Set up Docker Host

To deploy Docker-in-Docker to the cluster, execute the following:

```sh
kubectl apply -f ./setup/docker-in-docker.yaml
```

To run Docker system prune, execute the following:

```sh
kubectl exec -it $(kubectl get pod -l app=dind -o jsonpath='{.items[0].metadata.name}') /bin/sh

docker system prune -a
```

## 4.Install Dependencies

* Make sure following are installed locally:

  * uuid
  * cfssl

  On Ubuntu 18.04, this will be:

```sh
sudo apt install uuid
sudo apt install golang-cfssl
```

### 4.2 Get Kubeconfig

* Then to generate the kubeconfig file, run:

```sh
cd ./setup
./get-kubeconfig.sh
```

* The result file will be in `./setup/build/kubeconfig`.

## 5. Install Plugins

* Go to `Jenkins > Manage Jenkins > Manage Plugins` and install `GitHub` and `GitHub Branch Source` plugins.

#### 5.1.2 Add Access Token Credentials

* Create a new GitHub user with access to the repository (this user will be dedicated to only CI/CD related tasks).
* Go to `GitHub > Settings > Developer settings > Personal access tokens > Generate new token`.
  * Select *admin:repo_hook, repo, repo:status* scopes.
  * Generate the personal access token and keep it safe.
* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `Username and password` and fill in.
  * Username will be the GitHub username.
  * Password will be the generated personal access token.
* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `Secret text` and fill in.
  * Secret will be the generated personal access token.

#### 5.1.3 Configure Plugin

* Go to `Jenkins > Manage Jenkins > Configure System > GitHub`, add GitHub server and fill in.
  * Select credential created previously.
  * Enable manage hooks.
  * Test connection to see if it has been successfully set up.

#### 5.1.4 Add Job

* Go to `Jenkins > New Item`.
  * Enter a job name (preferably the name of the Git repository).
  * Select `Multibranch Pipeline` then click ok.
* Go to the `Branch Sources` section, and add a `GitHub` source and fill in.
  * Select credential created previously.
  * Enter the repository url.
  * Click validate to see if the credentials are ok and can connect to repository.
* After you save, it will start scanning the repository.

### 5.2 Using GitLab

#### 5.2.1 Install Plugin

* Go to `Jenkins > Manage Jenkins > Manage Plugins` and install `GitLab` plugin.

#### 5.2.2 Add SSH Credentials

* Create a new GitLab user with access to the repository (this user will be dedicated to only CI/CD related tasks).
* Create an SSH key pair and link it to the user.
* Go to `Jenkins > Manage Jenkins > Configure System > Git plugin` and fill in.
* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `SSH Username with private key` and fill in.

#### 5.2.3 Add Access Token Credentials

* Go to `GitLab > Settings > Access Tokens`.
  * Select all scopes.
  * Generate the personal access token and keep it safe.
* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `Username and password` and fill in.
  * Username will be the GitLab username.
  * Password will be the generated personal access token.

#### 5.2.4 Configure Plugin

* Go to `Jenkins > Manage Jenkins > Configure System > GitLab`, click add and fill in.
  * Select credential created previously.
  * Test connection to see if it has been successfully set up.

#### 5.2.5 Add Job

* Go to `Jenkins > New Item`.
  * Enter a job name (preferably the name of the Git repository).
  * Select `Multibranch Pipeline` then click ok.
* Go to the `Branch Sources` section, add a `Git` source and fill in.
  * Select credential created previously.
  * Enter the repository url.
* After you save, it will start scanning the Git repository.

#### 5.2.6 Manually Add Webhook

* Go to `GitLab > REPOSITORY > Settings > Webhooks`.
* Enter the url in the following format: `https://JENKINS_URL/project/JOB_NAME`.
* Select push event, and then add webhook.
* Test the webhook by sending a push event, if you receive a 200 response, then it has successfully been set up.

## 6. Required Credentials

### 6.1 Add Docker Registry Credentials

Part of this Pipeline process will require storing the Docker image on a remote container registry.

* Generate a personal access token on the container registry host, and ensure that it has permission to read and write to container/package registry. This token will be used as a password. Keep this token safe.
* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `Username and password` and fill in the details.
* Keep a note of the ID.

### 6.2 Add Kubeconfig Credentials

Part of this Pipeline process will require deploying the application (as defined by a Kubernetes manifest) to a cluster.

* Go to `Jenkins > Credentials > System > Global credentials > Add Credentials`, select kind `Secret file` and fill in the details. Use the kubeconfig file that was generated.
* Keep a note of the ID.

## 7. Define Constants

Here we define the credential IDs, the registry location, the Kubernetes manifest file, and the pods we will need to run. The use of these constants will become apparent when looking at the Pipeline.

```groovy
def REGISTRY_URL = 'docker.pkg.github.com'
def OWNER = 'yamxxnx'
def REPO_NAME = 'kubernetes-jenkins-cicd-pipeline-example'
def IMAGE_NAME = 'helloworld'

def IMAGE_REGISTRY = "${REGISTRY_URL}/${OWNER}/${REPO_NAME}/${IMAGE_NAME}"
def IMAGE_BRANCH_TAG = "${IMAGE_REGISTRY}:${env.BRANCH_NAME}"

def REGISTRY_CREDENTIALS = 'github-yamxxnx'
def CLUSTER_CREDENTIALS = 'cluster-1-kubeconfig'

def KUBERNETES_MANIFEST = 'kubernetes-manifest.yaml'
def STAGING_NAMESPACE = 'staging'
def PRODUCTION_NAMESPACE = 'production'
def PULL_SECRET = "registry-${REGISTRY_CREDENTIALS}"

def DOCKER_HOST_VALUE = 'tcp://dind.default:2375'

def DOCKER_POD = """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:19.03.6
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: ${DOCKER_HOST_VALUE}
"""

def KUBECTL_POD = """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kubectl
    image: lachlanevenson/k8s-kubectl:v1.15.9
    command:
    - cat
    tty: true
"""
```

### 7.2 Build and Store Docker Image

When building a Docker image, it is recommended to run tests as part of the process.

This process will store the image tagged with the branch name and commit ID.

```groovy
pipeline {
  agent any
  stages {
    stage('Run Docker') {
      agent { kubernetes label: 'docker', yaml: "${DOCKER_POD}" }
      stages {
        stage('Build Docker Image') {
          steps {
            container('docker') {
              sh "docker build -t ${IMAGE_BRANCH_TAG}.${env.GIT_COMMIT[0..6]} ."
            }
          }
        }
        stage('Push Image to Registry') {
          steps {
            container('docker') {
              withCredentials([
                usernamePassword(
                  credentialsId: "${REGISTRY_CREDENTIALS}",
                  usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS'
                )
              ]) {
                sh """
                echo ${REGISTRY_PASS} | docker login ${REGISTRY_URL} -u ${REGISTRY_USER} --password-stdin
                docker push ${IMAGE_BRANCH_TAG}.${env.GIT_COMMIT[0..6]}
                docker tag ${IMAGE_BRANCH_TAG}.${env.GIT_COMMIT[0..6]} ${IMAGE_BRANCH_TAG}
                docker push ${IMAGE_BRANCH_TAG}
                """
              }
            }
          }
        }
      }
    }
```

### 7.3 Deploy to Cluster


```groovy
    stage('Deploy Master') {
      when { branch 'master' }
      agent { kubernetes label: 'kubectl', yaml: "${KUBECTL_POD}" }
      stages {
        stage('Deploy Image to Staging') {
          steps {
            container('kubectl') {
              withCredentials([
                file(
                  credentialsId: "${CLUSTER_CREDENTIALS}",
                  variable: 'KUBECONFIG'
                ),
                usernamePassword(
                  credentialsId: "${REGISTRY_CREDENTIALS}",
                  usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS'
                )
              ]) {
                sh """
                kubectl \
                -n ${STAGING_NAMESPACE} \
                create secret docker-registry ${PULL_SECRET} \
                --docker-server=${REGISTRY_URL} \
                --docker-username=${REGISTRY_USER} \
                --docker-password=${REGISTRY_PASS} \
                --dry-run \
                -o yaml \
                | kubectl apply -f -

                sed \
                -e "s|{{NAMESPACE}}|${STAGING_NAMESPACE}|g" \
                -e "s|{{PULL_IMAGE}}|${IMAGE_BRANCH_TAG}.${env.GIT_COMMIT[0..6]}|g" \
                -e "s|{{PULL_SECRET}}|${PULL_SECRET}|g" \
                ${KUBERNETES_MANIFEST} \
                | kubectl apply -f -
                """
              }
            }
          }
        }
        stage('Manual Review') {
          agent none
          steps {
            timeout(time:2, unit:'DAYS') {
              input message: 'Deploy image to production?'
            }
          }
        }
        stage('Deploy Image to Production') {
          steps {
            container('kubectl') {
              withCredentials([
                file(
                  credentialsId: "${CLUSTER_CREDENTIALS}",
                  variable: 'KUBECONFIG'
                ),
                usernamePassword(
                  credentialsId: "${REGISTRY_CREDENTIALS}",
                  usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS'
                )
              ]) {
                sh """
                kubectl \
                -n ${PRODUCTION_NAMESPACE} \
                create secret docker-registry ${PULL_SECRET} \
                --docker-server=${REGISTRY_URL} \
                --docker-username=${REGISTRY_USER} \
                --docker-password=${REGISTRY_PASS} \
                --dry-run \
                -o yaml \
                | kubectl apply -f -

                sed \
                -e "s|{{NAMESPACE}}|${PRODUCTION_NAMESPACE}|g" \
                -e "s|{{PULL_IMAGE}}|${IMAGE_BRANCH_TAG}.${env.GIT_COMMIT[0..6]}|g" \
                -e "s|{{PULL_SECRET}}|${PULL_SECRET}|g" \
                ${KUBERNETES_MANIFEST} \
                | kubectl apply -f -
                """
              }
            }
          }
        }
      }
    }
  }
}
```

### 7.4 Build Status

#### 7.4.1 Send to GitHub

The GitHub plugin by default will automatically send build status back to GitHub, however this will be in the very basic form of a green tick for success and red cross for failure.

#### 7.4.2 Send to GitLab

Assuming the GitLab plugin has been configured, the following shows an example of how you would send build status back to GitLab. The *gitLabConnection* method takes the credential ID of the GitLab access token.

```groovy
pipeline {
  agent any
  options {
    gitLabConnection('GitLab - yamxxnx')
    gitlabBuilds(builds: [
      'Build Docker Image',
      'Push Image to Registry'
    ])
  }
  stages {
    stage('Run Docker') {
      agent { kubernetes label: 'docker', yaml: "${DOCKER_POD}" }
      stages {
        stage('Build Docker Image') {
          steps {
            container('docker') {
              sh "docker build -t ${IMAGE_BRANCH_TAG} ."
            }
          }
          post {
            failure { updateGitlabCommitStatus name: 'Build Docker Image', state: 'failed' }
            success { updateGitlabCommitStatus name: 'Build Docker Image', state: 'success' }
          }
        }
...
```

## 8. Useful Links

* [Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/)
* [GitHub Plugin](https://plugins.jenkins.io/github/)
* [GitLab Plugin](https://plugins.jenkins.io/gitlab-plugin/)
* [Jenkins Pipeline](https://jenkins.io/doc/book/pipeline/)
