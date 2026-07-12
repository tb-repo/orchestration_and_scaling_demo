# End-to-End EKS Deployment, Orchestration, and Scaling Walkthrough

This guide provides a comprehensive, step-by-step walkthrough for building, deploying, orchestrating, and scaling the MERN microservices-based **StreamingApp** on **Amazon EKS**.

---

## Architecture Overview

The system runs as a multi-service application with the following components:

```mermaid
graph TD
    subgraph Developer_Workspace ["Developer Workspace"]
        LocalGit["Local Git Repo"]
    end
    
    subgraph GitHub_SaaS ["GitHub (SaaS)"]
        ForkedRepo["Forked GitHub Repo"]
    end

    subgraph AWS_CI ["AWS CI Environment"]
        EC2_Jenkins["Jenkins EC2 Instance"]
        DockerBuild["Docker Engine (Image Build)"]
    end

    subgraph AWS_Registry ["AWS Registry"]
        ECR_FE["ECR: streaming-app-frontend"]
        ECR_Auth["ECR: streaming-app-auth"]
        ECR_Stream["ECR: streaming-app-streaming"]
        ECR_Admin["ECR: streaming-app-admin"]
        ECR_Chat["ECR: streaming-app-chat"]
    end

    subgraph AWS_EKS ["Amazon EKS Cluster (Kubernetes)"]
        subgraph K8s_Namespace ["Namespace: streaming-app"]
            HelmRelease["Helm Release"]
            
            subgraph FE_Group ["Frontend Application"]
                FE_Service["Frontend Service (LoadBalancer)"]
                FE_Pods["Frontend Pods (React + Nginx Proxy)"]
            end
            
            subgraph Auth_Group ["Auth Microservice"]
                Auth_Service["Auth Service (ClusterIP: 3001)"]
                Auth_Pods["Auth Pods (Node.js)"]
            end
            
            subgraph Stream_Group ["Streaming Microservice"]
                Stream_Service["Streaming Service (ClusterIP: 3002)"]
                Stream_Pods["Streaming Pods (Node.js)"]
            end

            subgraph Admin_Group ["Admin Microservice"]
                Admin_Service["Admin Service (ClusterIP: 3003)"]
                Admin_Pods["Admin Pods (Node.js)"]
            end

            subgraph Chat_Group ["Chat Microservice"]
                Chat_Service["Chat Service (ClusterIP: 3004)"]
                Chat_Pods["Chat Pods (Node.js)"]
            end
            
            subgraph DB_Group ["Database Service"]
                DB_Service["MongoDB Service (ClusterIP: 27017)"]
                DB_Pod["MongoDB Pod (StatefulSet / Deployment)"]
            end
        end
    end

    subgraph AWS_Management ["AWS Operations & Observability"]
        CloudWatch["Amazon CloudWatch"]
        CW_Logs["CloudWatch Logs"]
        CW_Alarms["Alarms (CPU/Memory)"]
        SNS_Topic["Amazon SNS Topic"]
    end

    subgraph Messaging ["Messaging Channel"]
        Slack["Slack / Telegram Chat Notification"]
    end

    ForkedRepo -->|"Clone / Push"| LocalGit
    LocalGit -->|"Webhook Trigger"| EC2_Jenkins
    EC2_Jenkins -->|"Pull Code"| DockerBuild
    DockerBuild -->|"Push Auth"| ECR_Auth
    DockerBuild -->|"Push Streaming"| ECR_Stream
    DockerBuild -->|"Push Admin"| ECR_Admin
    DockerBuild -->|"Push Chat"| ECR_Chat
    DockerBuild -->|"Push Frontend"| ECR_FE
    
    HelmRelease -->|"Deploy"| FE_Service
    FE_Service --> FE_Pods
    HelmRelease -->|"Deploy"| Auth_Service
    Auth_Service --> Auth_Pods
    HelmRelease -->|"Deploy"| Stream_Service
    Stream_Service --> Stream_Pods
    HelmRelease -->|"Deploy"| Admin_Service
    Admin_Service --> Admin_Pods
    HelmRelease -->|"Deploy"| Chat_Service
    Chat_Service --> Chat_Pods
    HelmRelease -->|"Deploy"| DB_Service
    DB_Service --> DB_Pod

    FE_Pods -->|"/api/auth"| Auth_Service
    FE_Pods -->|"/api/streaming"| Stream_Service
    FE_Pods -->|"/api/admin"| Admin_Service
    FE_Pods -->|"/api/chat"| Chat_Service

    Auth_Pods -->|"Read/Write"| DB_Service
    Stream_Pods -->|"Read/Write"| DB_Service
    Admin_Pods -->|"Read/Write"| DB_Service
    Chat_Pods -->|"Read/Write"| DB_Service

    FE_Pods -->|"Send Logs & Metrics"| CloudWatch
    Auth_Pods -->|"Send Logs & Metrics"| CloudWatch
    Stream_Pods -->|"Send Logs & Metrics"| CloudWatch
    Admin_Pods -->|"Send Logs & Metrics"| CloudWatch
    Chat_Pods -->|"Send Logs & Metrics"| CloudWatch
    CloudWatch --> CW_Logs
    CloudWatch --> CW_Alarms
    CW_Alarms -->|"Trigger Alert"| SNS_Topic
    SNS_Topic -->|"Alert Notification"| Slack
```

---

## Step-by-Step Implementation Guide

### Step 1: Local Development & Verification (Docker Compose)
Before moving to AWS, the microservices are built and verified locally on the developer workstation. We use **Docker Compose** to spin up a local instance of MongoDB and run all five microservices, verifying API routes, database connections, and microservice communications.

1. **Verify environment and configuration:**
   ```bash
   cd d:\HeroVired\Assignments\Orchestration_Scaling_Demo\StreamingApp
   copy .env.example .env
   docker-compose up --build -d
   ```
2. **Local Verification Steps:**
   * Run `docker ps` to verify that all 6 containers (`mongo`, `auth-service`, `streaming-service`, `admin-service`, `chat-service`, and `frontend`) are up and running.
   * Access `http://localhost:3000` to test user registration, user login, video streaming, and chat messaging flows locally.

#### Local Deployment Visuals:
![Starting Local Containers](images/StreamingApp_LocalDeployment1.png)

![Verifying Running Services](images/StreamingApp_LocalDeployment2.png)

![Running App Browser Verification](images/StreamingApp_LocalDeployment3.png)

![Local Registry Build List](images/StreamingApp_LocalDeployment4.png)

![Local DB Collections Verification](images/StreamingApp_LocalDeployment5.png)

![Local Account Access Verification](images/StreamingApp_LocalDeployment6.png)

![Local Backend logs check](images/StreamingApp_LocalDeployment7.png)

---

### Step 2: AWS ECR Private Registry Setup
To deploy these containers on EKS, we must host the Docker images in a private registry. We create five private repositories in **Amazon Elastic Container Registry (ECR)** corresponding to our five microservices.

* Go to **Amazon ECR -> Repositories** and click **Create repository**.
* Choose **Private** visibility, name each repository matching the services, and enable **Tag immutability** and **Scan on push** configurations.

![ECR Registry Creation Step 1](images/ECR_Creation1.png)

![ECR Registry Creation Step 2](images/ECR_Creation2.png)

![ECR Registry Creation Step 3](images/ECR_Creation3.png)

![ECR Registry Creation Step 4](images/ECR_Creation4.png)

![ECR Registry Creation Step 5](images/ECR_Creation5.png)

![ECR Registry Creation Step 6](images/ECR_Creation6.png)

![ECR Registry Creation Success](images/ECR_Creation7.png)

---

### Step 3: Jenkins CI/CD Setup & Automated Build
We configure a CI/CD pipeline using **Jenkins** on an AWS EC2 instance. When a developer pushes code changes to GitHub, a webhook automatically triggers the Jenkins server to build the Docker images, tag them, and push them to ECR.

#### 1. GitHub Webhooks Setup
Configure your GitHub repository settings under **Settings -> Webhooks**:
* **Payload URL:** Set to `http://<JENKINS_SERVER_IP>:8080/github-webhook/`
* **Content type:** Select `application/json`
* **Events:** Trigger on the `push` event.

![GitHub Webhook Configuration](images/Git_Jenkins_webhook1.png)

![GitHub Webhook Verified](images/Git_Jenkins_webhook2.png)

#### 2. Jenkins Credentials Vault Configuration
To allow the build server to authenticate against ECR and AWS CLI, configure three keys under **Jenkins Dashboard -> Manage Jenkins -> Credentials -> Global**:
1. `HV-B16A-TB-AWS-ACCOUNT-ID`: AWS Account ID (as **Secret Text**).
2. `HV-B16A-TB-AWS-DEFAULT-REGION`: The AWS region, e.g. `us-east-1` (as **Secret Text**).
3. `HV-B16A-TB-AWS-CREDENTIALS`: AWS Credentials group containing **AWS Access Key ID** and **AWS Secret Access Key**.

![Jenkins Secrets Configuration 1](images/Jenkins_secrets1.png)

![Jenkins Secrets Configuration 2](images/Jenkins_secrets2.png)

![Jenkins Secrets Configuration 3](images/Jenkins_secrets3.png)

#### 3. EKS Pipeline Run (Practical Implementation)
Once the credentials and webhook triggers are configured, push a Git commit to trigger the pipeline. 

![Pipeline Configuration Step 1](images/Jenkins_pipeline1.png)

![Pipeline Configuration Step 2](images/Jenkins_pipeline2.png)

![Pipeline Configuration Step 3](images/Jenkins_pipeline3.png)

![Pipeline Configuration Step 4](images/Jenkins_pipeline4.png)

![Pipeline Configuration Step 5](images/Jenkins_pipeline5.png)

![Pipeline Configuration Step 6](images/Jenkins_pipeline6.png)

Jenkins discovers the pipeline instructions defined in [Jenkinsfile](Jenkinsfile). It starts Docker build environments, compiles the microservices, and pushes the images to ECR.

![Git commit triggering build](images/Jenkins_pipeline7_gitcommit_push.png)

![Pipeline Successful Run](images/Jenkins_pipeline14_pipeline_success.png)

For verification of the real build outputs, refer to the actual build execution log: [Jenkins_deployment_success.log](Jenkins_deployment_success.log).

#### ECR Registry Verification
![ECR Image Lists Auth Service](images/aws_ecr_registry_image1.png)

![ECR Image Lists Frontend](images/aws_ecr_registry_image2.png)

![ECR Image Lists Streaming](images/aws_ecr_registry_image3.png)

![ECR Image Lists Admin](images/aws_ecr_registry_image4.png)

![ECR Image Lists Chat](images/aws_ecr_registry_image5.png)

![ECR Registry Overview](images/aws_ecr_registry_image6.png)

![Docker Images List](images/aws_ecr_registry_image7.png)

---

### Step 4: AWS S3 Storage Setup
We create an **S3 Bucket** to host video assets uploaded by the application. EKS backend pods will access this bucket directly to upload new streams and fetch clips.

* Go to **Amazon S3 -> Buckets** and click **Create bucket**.
* Choose a globally unique name, set the region to **`us-east-1`**, enable bucket versioning, and configure **Block Public Access** settings as required by EKS.

![S3 Bucket Creation 1](images/S31.png)

![S3 Properties Configuration](images/s32.png)

![S3 Permissions Check](images/s33.png)

![S3 Block Public Access Setting](images/s34.png)

![S3 Bucket Ready](images/s35.png)

---

### Step 5: AWS EKS Cluster Provisioning & IRSA Mapping
An **Amazon EKS Cluster** (`streaming-app-cluster`) is provisioned using `eksctl`. We map the OIDC identity provider and configure the IAM role for the Service Account (`streaming-app-sa`) to let pods write to S3 securely.

```bash
# Provision the cluster
eksctl create cluster -f eks-cluster.yaml

# Enable OIDC
eksctl utils associate-iam-oidc-provider --cluster streaming-app-cluster --approve --region us-east-1

# Create Service Account mapping to IAM Role (IRSA)
eksctl create iamserviceaccount \
  --name streaming-app-sa \
  --namespace streaming-app \
  --cluster streaming-app-cluster \
  --role-name streaming-app-s3-role \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --approve
```

#### EKS Creation & Nodes Check:
![eksctl command execution](images/k8s_eksctl_1.png)

![Nodes listing](images/k8s_eksctl_2.png)

![AWS console EKS overview](images/k8s_eksctl_3.png)

![OIDC Provider mapping](images/k8s_oidc1.png)

#### Service Account Mapping:
![Service Account creation command](images/k8s_eksctl_serviceaccount1.png)

![IAM Policy attachment](images/k8s_eksctl_serviceaccount2.png)

![Service Account validated on AWS](images/k8s_eksctl_serviceaccount3.png)

---

### Step 6: Helm Orchestration & Deployment
The microservices stack is orchestrated using **Helm**. We perform checks, deploy the chart, and configure the CORS endpoint to point to the EKS LoadBalancer.

#### 1. Helm Configuration & Dry Run:
```bash
# Validate chart
helm lint ./helm

# Test installation
helm install streaming-app ./helm -n streaming-app --create-namespace --dry-run
```
![Helm Lint](images/helm_lint.png)

![Helm Dry Run Part 1](images/helm_dryrun1.png)

![Helm Dry Run Part 2](images/helm_dryrun2.png)

#### 2. Running the Stack & Verifying Resources:
Deploy the chart to EKS:
```bash
helm upgrade --install streaming-app ./helm -n streaming-app --create-namespace
```
*(Refer to the **Issues Faced & Resolved** section at the bottom if you hit namespace ownership validation block errors).*

![Helm Deployment Successful](images/helm_install_success1.png)

![EKS Resources Up and Running](images/kubectl_resource_status1.png)

![EKS Details Verification](images/kubectl_resource_status2.png)

![Kubernetes Dashboard Nodes](images/k8s_aws_console1.png)

![Kubernetes Dashboard Workloads](images/k8s_aws_console2.png)

To verify the active running services, deployments, replicasets, and nodes in the cluster, view the current resources state dump: [kubectl_get_all.txt](kubectl_get_all.txt).

#### 3. User Browser Access on LoadBalancer Address:
Retrieve the LoadBalancer service details:
```powershell
kubectl get service frontend-service -n streaming-app
```

![LoadBalancer service details](images/StreamingApp_frontendservice_url.png)

![App landing page](images/StreamingApp_successful_access_via_EKS1.png)

![Account Sign In page](images/StreamingApp_successful_access_via_EKS2.png)

![Creating a user account](images/StreamingApp_successful_access_via_EKS3.png)

![User Dashboard](images/StreamingApp_successful_access_via_EKS4.png)

![Admin Video Upload page](images/StreamingApp_successful_access_via_EKS5.png)

![Uploading video file](images/StreamingApp_successful_access_via_EKS6.png)

![Admin Video List](images/StreamingApp_successful_access_via_EKS7.png)

![Video Playback working](images/StreamingApp_successful_access_via_EKS8.png)

![Real-time Group Chat working](images/StreamingApp_successful_access_via_EKS9.png)

---

### Step 7: CloudWatch Container Insights & Fluent Bit Logs
We deploy Fluent Bit and CloudWatch agents to stream container logs and system host logs to CloudWatch. We attached the `CloudWatchAgentServerPolicy` policy to EKS nodes to grant authorization.

```bash
# Apply DaemonSet templates
kubectl apply -f quickstart-processed.yaml
```

![Logging agents running](images/CloudWatch_agent_deployed_pod_statis.png)

![CloudWatch log groups created](images/CloudWatch_agent_deployed.png)

![CloudWatch Log Streams Details](images/CloudWatch_agent_deployed_aws_console_loggroups.png)

---

### Step 8: SNS Alerts & CloudWatch Alarms (Slack ChatOps)
We configure an SNS Topic and link it to **Slack (via AWS Chatbot/Amazon Q Developer)** to trigger real-time operational notifications whenever EKS metrics (like Node CPU usage) enter the `ALARM` state.

#### 1. SNS Topic Configuration
Create your SNS topic:
```bash
aws sns create-topic --name streaming-app-alerts --region us-east-1
```

![SNS Topic Setup 1](images/CloudWatch_sns_topic1.png)

![SNS Topic Setup 2](images/CloudWatch_sns_topic2.png)

![SNS Topic Setup 3](images/CloudWatch_sns_topic3.png)

#### 2. ChatOps Slack Channel Mapping (AWS Chatbot)
Configure AWS Chatbot to authorize your Slack Workspace, configure a new channel configuration (`streaming-app-alerts`), assign basic notifications role permissions, and link the channel to the SNS topic `streaming-app-alerts`.

![Slack Workspace Authorized](images/Slack_channel_integration_sns1.png)

![AWS Chatbot Permissions Configuration 1](images/Slack_channel_integration_sns2.png)

![AWS Chatbot Permissions Configuration 2](images/Slack_channel_integration_sns3.png)

![AWS Chatbot Permissions Configuration 3](images/Slack_channel_integration_sns4.png)

![AWS Chatbot Permissions Configuration 4](images/Slack_channel_integration_sns5.png)

![AWS Chatbot Permissions Configuration 5](images/Slack_channel_integration_sns6.png)

![AWS Chatbot Configuration Complete](images/Slack_channel_integration_sns7.png)

![SNS Subscription Active](images/Slack_channel_integration_sns8.png)

![Inviting bot to alerts channel 1](images/Slack_channel_for_alerts1.png)

![Inviting bot to alerts channel 2](images/Slack_channel_for_alerts2.png)

#### 3. Creating CloudWatch Alarms
Under **CloudWatch -> Alarms -> Create Alarm**, select the metric **`ContainerInsights -> ClusterName, NodeName -> node_cpu_utilization`** for EKS. Set the threshold to static, $>80\%$ for 5 minutes, and link the action to send notifications to your SNS topic.

![Alarms Console 1](images/CloudWatch_alarm_creation1.png)

![Alarms Console 2](images/CloudWatch_alarm_creation2.png)

![Alarms Console 3](images/CloudWatch_alarm_creation3.png)

![Alarms Console 4](images/CloudWatch_alarm_creation4.png)

![Alarms Console 5](images/CloudWatch_alarm_creation5.png)

![Alarms Console 6](images/CloudWatch_alarm_creation6.png)

![Alarms Console 7](images/CloudWatch_alarm_creation7.png)

![Alarms Console 8](images/CloudWatch_alarm_creation8.png)

![Alarms Console 9](images/CloudWatch_alarm_creation9.png)

![Alarms Console 10](images/CloudWatch_alarm_creation10.png)

![Alarms Console 11](images/CloudWatch_alarm_creation11.png)

![Alarms Console 12](images/CloudWatch_alarm_creation12.png)

![Active Alarm Status](images/CloudWatch_alarm_status1.png)

#### 4. Real-Time Alert Delivery
Test the alerts pipeline using the CLI to force the alarm state:
```powershell
# Reset to OK
aws cloudwatch set-alarm-state --alarm-name "streaming-app-node-cpu-utilization-alert" --state-value OK --state-reason "Resetting to test" --region us-east-1

# Trigger ALARM
aws cloudwatch set-alarm-state --alarm-name "streaming-app-node-cpu-utilization-alert" --state-value ALARM --state-reason "Testing transition" --region us-east-1
```
* **Invite the Bot to private channels:** If your Slack channel is private, you must invite the bot to the channel, otherwise messages will fail to deliver:
  ```text
  /invite @Amazon Q
  ```

![AWS Chatbot Test Notification](images/Slack_channel_AWS_test_message.png)

![Live CloudWatch Node CPU Alarm Alert](images/Slack_channel_AWS_alarm_alert.png)

---

## Issues Faced & Resolved

During the course of the deployment, several configuration blocks and permissions issues were encountered. Below is a breakdown of these challenges and the steps taken to resolve them.

### Issue 1: Jenkins Pipeline Build Failure (ECR Auth Denied)
* **Symptom:** The Jenkins pipeline failed during the ECR Push phase with authorization errors (`Denied: Permission Denied` or connection timeout).
* **Root Cause:** The IAM User configured in Jenkins credentials (`HV-B16A-TB-AWS-CREDENTIALS`) did not have ECR access permissions attached to it on AWS.
* **Resolution:** 
  1. We inspected the ECR Builder user in the AWS IAM Console.
  2. Attached the managed policy `AmazonEC2ContainerRegistryPowerUser` to this IAM user.
  3. Generated a new Access Key ID and Secret Access Key, and updated them inside Jenkins Global Credentials Vault.
  4. Pushed a new commit, which compiled and pushed images successfully.

#### Troubleshooting Visuals:
![Pipeline console error logs](images/Jenkins_pipeline8_buildfailure.png)

![Build auth exceptions](images/Jenkins_pipeline9_buildfailure.png)

![Checking ECR User in IAM Console](images/Jenkins_pipeline10_aws_useraccess.png)

![Attaching Container Registry Power User Policy](images/Jenkins_pipeline11_aws_useraccess.png)

![Regenerating AWS credentials keys](images/Jenkins_pipeline12_aws_useraccess.png)

![Validating IAM console updates](images/Jenkins_pipeline13_aws_useraccess.png)

---

### Issue 2: Helm Namespace Ownership Validation Conflict
* **Symptom:** Running the Helm installation threw an ownership validation exception:
  `Error: unable to continue with install: Namespace "streaming-app" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata...`
* **Root Cause:** A `namespace.yaml` was created inside the `templates/` directory of the chart. Since EKS already had the `streaming-app` namespace created manually, Helm refused to overwrite it because it lacked management metadata labels.
* **Resolution:** 
  We annotated and labeled the existing namespace to let Helm adopt it:
  ```powershell
  kubectl label namespace streaming-app app.kubernetes.io/managed-by=Helm --overwrite
  kubectl annotate namespace streaming-app meta.helm.sh/release-name=streaming-app --overwrite
  kubectl annotate namespace streaming-app meta.helm.sh/release-namespace=streaming-app --overwrite
  ```
  After executing these commands, the Helm deploy completed successfully.

#### Troubleshooting Visuals:
![Helm import namespace metadata failure](images/helm_install_issue1.png)

---

### Issue 3: CORS Login Mismatch on EKS LoadBalancer
* **Symptom:** The React frontend was successfully accessible on the EKS Classic LoadBalancer address, but attempting to register or log in returned `"Something went wrong on the server"` in the UI and CORS errors in the browser console.
* **Root Cause:** The microservice backend pods were checking incoming `Origin` headers against allowed CORS domains. Because no client URL environment variable was passed, they defaulted to `http://localhost:3000`. The browser was sending requests from the EKS LoadBalancer URL (e.g. `http://abbcd4f549a5540e3afab313e947abab-670547553.us-east-1.elb.amazonaws.com`), which triggered CORS rejections.
* **Resolution:** 
  1. We added the parameter `clientUrl` to the Helm chart's `values.yaml` containing the LoadBalancer IP/DNS.
  2. Exposed `client-url: {{ .Values.clientUrl | quote }}` in the `configmap.yaml` template.
  3. Mapped the `CLIENT_URL` environment variable from the ConfigMap in all microservice deployment manifests.
  4. Upgraded the Helm deployment, which restarted all containers and successfully enabled CORS login.

#### Troubleshooting Visuals:
![Auth container logs showing CORS rejection](images/StreamingApp_EKS_frontend_login_issue1.png)

![Troubleshooting CORS values mapping](images/StreamingApp_EKS_frontend_login_issue_troubleshooting1.png)

![Exposing CLIENT_URL in Helm templates](images/StreamingApp_EKS_frontend_login_issue_troubleshooting2.png)

![Double-checking configmap values](images/StreamingApp_EKS_frontend_login_issue_troubleshooting3.png)

---

### Issue 4: CloudWatch DaemonSet Node Permission Failures
* **Symptom:** CloudWatch agent and Fluent Bit logging pods were running, but no log groups appeared in the AWS Console. Pod logs showed `CreateLogStream API responded with error='AccessDeniedException'`.
* **Root Cause:** The EKS worker nodes (EC2 instances) did not have permissions to write logs and metrics to CloudWatch.
* **Resolution:** 
  1. We described the EKS nodegroup to find the worker nodes' IAM role name: `eksctl-streaming-app-cluster-nodeg-NodeInstanceRole-zFfGTiuQXje8`.
  2. Attached the AWS-managed policy `CloudWatchAgentServerPolicy` to this IAM role.
  3. Restarted the CloudWatch Agent DaemonSet to apply the credentials change:
     ```powershell
     kubectl rollout restart daemonset cloudwatch-agent -n amazon-cloudwatch
     ```
  4. Log groups `/aws/containerinsights/streaming-app-cluster/application` were successfully created.

#### Troubleshooting Visuals:
![Monitoring pods failing to stream logs](images/CloudWatch_agent_issue1.png)

![Restarting agent daemonset to resolve access errors](images/CloudWatch_agent_restart_issuefix_check.png)

---

### Issue 5: SNS CLI Access Policy Blocking CloudWatch Alarms
* **Symptom:** The CloudWatch Alarm entered the `ALARM` state, but no notifications arrived in Slack. CloudWatch Alarm History showed the SNS Publish action failed silently due to Access Denied.
* **Root Cause:** Creating the SNS topic via the CLI (`aws sns create-topic`) restricts the topic's Access Policy to the owner account only. It did not grant permissions to the CloudWatch Service (`cloudwatch.amazonaws.com`).
* **Resolution:** 
  We updated the SNS topic's Access Policy to include an allow statement for `cloudwatch.amazonaws.com` on `sns:Publish`:
  ```powershell
  aws sns set-topic-attributes --topic-arn "arn:aws:sns:us-east-1:376432388605:streaming-app-alerts" --attribute-name Policy --attribute-value file://sns-policy.json --region us-east-1
  ```
  Alerts immediately flowed through AWS Chatbot/Amazon Q to Slack!

---

## Resources Teardown Guide

To safely teardown all AWS resources, node groups, load balancers, and monitoring storage to avoid ongoing billing, please refer to the detailed, step-by-step instructions in **[termination_guide.md](termination_guide.md)**.
