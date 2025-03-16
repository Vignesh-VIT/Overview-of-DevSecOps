# Overview-of-DevSecOps
This project provides a comprehensive guide to implementing a DevSecOps pipeline in a local environment, without requiring any cloud infrastructure or resources (Only for learning purpose)


## DevSecOps Pipeline Workflow
[![Dev-Sec-Ops.png](https://i.postimg.cc/26LVpPMf/Dev-Sec-Ops.png)](https://postimg.cc/2bmzhXCH)


## Tools involved in DevSecOps pipeline
- Jenkins for continuous integration and continuous deployment (CI/CD)
- Maven (Code compilation, unit tests and package)
- SonarQube (Code quality, static code analysis, Code smell, security and vulnerability)
- Trivy (File system scan, container images scan for vulnerabilities)
- Nexus Repository (Store software like libraries, binaries and docker images)
- Docker (Containerize the application and its dependencies)
- Kubernetes (Orchestrating and managing container applications)
- Prometheus (Scraping the metrics from applications or system / Alerting)
- Grafana (Visualize the metrics collected from various sources in form of graph)
- Slack (Notification on completion for builds status etc...)

## Prerequisite
Since this pipeline is implemented in a local environment solely for learning purposes, it may impact your laptop's performance. We are not provisioning any EC2 instances or cloud-native Kubernetes services for hosting the application. Instead, I have adopted a hybrid approach, running some tools as containers while installing others as binaries on my local machine. However, you are free to set it up in a way that best suits your needs.

1. Install [Docker Desktop](https://www.youtube.com/watch?v=-j6xjn5hSHI) and enable kubernetes
2. Install [Jenkins as service](https://www.youtube.com/watch?v=Zdxko2bPAAw)
   - Required Plugins: [Install via Manage Jenkins -> Plugins -> Available plugins]
     - Eclipse temurin installer
     - Config file provider
     - Pipeline maven integration
     - Maven integration plugin
     - SonarQube scanner for jenkins
     - Docker plugin
     - Docker pipeline
     - Kubernetes plugin
     - Kubernetes CLI plugin
     - Kubernetes credentials plugin
     - Kubernetes API plugin
     - Promethues metrics plugin
     - Slack notification plugin
3. Install [SonarQube as service](https://www.youtube.com/watch?v=xs9fKiJ1Ts0)
4. Run [Nexus repository as docker container](https://www.youtube.com/watch?v=gK4GIW130GA)
5. Install [Trivy as service](https://community.chocolatey.org/packages/trivy) using chocolatey package
6. Create [Docker Hub Registry](https://hub.docker.com/)
7. Install [Prometheus as service](https://prometheus.io/download/)
8. Run [Node Exporter as docker container](https://prometheus.io/download/)
9. Install [Blackbox Exporter as service](https://prometheus.io/download/)
10. Install [Grafana as service](https://grafana.com/grafana/download)
11. Install [Ngrok](https://community.chocolatey.org/packages/ngrok) using chocolatey package - you’ll soon discover why it's needed!
12. [Integrating Slack with Jenkins](https://www.youtube.com/watch?v=9ZUy3oHNgh8)

For each of the steps above, I have either provided a YouTube link for guidance or an official download link for you to follow. 

Once the above setup is complete, we will move on to the next step: setting up the pipeline script and integrating all the tools to work seamlessly within the pipeline, making it configurable and ready for execution.

## Reverse proxy for Jenkins
After installing Ngrok on your local system, ensure that Jenkins is up and running. Next, route the traffic from your locally running Jenkins (on port 8080) through Ngrok. This setup allows SonarQube to send POST requests to your Jenkins environment, enabling code quality results to be displayed on the Jenkins build page.
  - Run the following command: `ngrok http http://localhost:8080`
[![ngrok-jenkins.png](https://i.postimg.cc/1Rp3pcZM/ngrok-jenkins.png)](https://postimg.cc/kDgCCSGt)


## Configure RBAC for Kubernetes to Deploy Our Application
To ensure a secure deployment, we can create a user/service account to deploy the application within a separate namespace. To achieve this, follow these steps:

### Creating Service Account


```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

### Create Role 


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Bind the role to service account


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```

### Generate token using service account in the namespace
Now, the Jenkins user we created has the necessary permissions to perform deployments and other required actions. Since Jenkins needs to connect to the Kubernetes cluster, we will generate a token that will be used for authentication.

Check documentation for more info: [Create Token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)

```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: myserviceaccount
```
Now, apply the configurations to the desired namespace for deployment: [Ensure Kubernetes is up and running]
- `kubectl apply -f serviceaccount.yaml`
- `kubectl apply -f role.yaml`
- `kubectl apply -f bind.yaml`
- `kubectl apply -f secret.yaml -n webapps`

Retrieve the authentication token for deployment using: 
`kubectl describe secret mysecretname -n webapps`

Then, store this token in Jenkins credentials to reference it within the pipeline for deployment.

- Go to Manage Jenkins -> Credentials -> Global credentials -> Add Credentials
- Select kind as **Secret text** and paste the authentication token
- Enter the ID and Description
- Click **Create**

## Pipeline Setup
Refer to the **Jenkinsfile** available in the repository to create a new pipeline job in Jenkins. Copy and paste the content into the script section, then apply the changes. 

As part of this setup, I have selected an open-source full-stack application built on the Spring Boot framework, using Maven as the build tool. However, you can choose any Maven-based project to follow the same process. If you opt for a different technology stack, such as Node.js or Python, you will need to modify the pipeline script accordingly in the build stage to accommodate the specific build process.

Once the setup is complete, watch the video below to see the DevSecOps pipeline running successfully on your local environment.

## Watch out the video!!

[![Watch the video](https://i.postimg.cc/sX4b22tP/play-button.png)](https://drive.google.com/file/d/1uCL1Sj_RffVBk5RdukrARSn4CmxSuCSl/view?usp=sharing)


## Improvement
- Automatically trigger the pipeline when a developer commits code to GitHub.
- Retrieve the Jenkinsfile directly from SCM, instead of hardcoding it in the pipeline script.
- Configure a custom Quality Gate in SonarQube and fail the pipeline if the quality criteria are not met.
- Publish scan results or reports from Trivy and send them as an attachment to Slack as part of the post-deployment process.
- Use Helm to manage Kubernetes deployments and maintain it in a separate repository instead of including it in the same repository.
- Set up a Node Exporter to collect and scrape custom application metrics from Kubernetes.
- And more enhancements to optimize security, automation, and performance.