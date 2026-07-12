# Step-by-Step AWS Resource Termination Guide

To avoid incurring unexpected charges on AWS, it is highly recommended to cleanly terminate and drop all EKS, Helm, ECR, S3, and CloudWatch resources. 

Follow these steps in your PowerShell terminal to tear down the environment.

---

## Step 1: Uninstall Helm Applications
Uninstall the application Helm chart. This automatically deletes your pods, deployments, ConfigMaps, and the AWS Classic LoadBalancer.
```powershell
helm uninstall streaming-app -n streaming-app
```

---

## Step 2: Delete Kubernetes Namespaces
Delete the custom namespaces. This will clean up any remaining service accounts, roles, or PVCs.
```powershell
# Delete the streaming-app namespace
kubectl delete namespace streaming-app

# Delete the CloudWatch namespace (removes the fluent-bit & agent daemonsets)
kubectl delete namespace amazon-cloudwatch
```

---

## Step 3: Detach IAM Role Policy
Detach the CloudWatch server policy we attached to the worker node IAM role:
```powershell
aws iam detach-role-policy --role-name eksctl-streaming-app-cluster-nodeg-NodeInstanceRole-zFfGTiuQXje8 --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

---

## Step 4: Delete the EKS Cluster (Crucial step to save cost!)
Delete the EKS cluster. This command will cleanly delete your EKS control plane, the VPC, subnets, worker node instances, and the CloudFormation stacks.
```powershell
eksctl delete cluster --name streaming-app-cluster --region us-east-1
```
*Note: This command will take 15–20 minutes to completely execute and clean up everything.*

---

## Step 5: Clean up S3 Bucket
Empty and delete the S3 bucket used for video files.
```powershell
# 1. Empty all files in the bucket
aws s3 rm s3://orchestration-scaling-demo-bucket --recursive --region us-east-1

# 2. Delete the bucket itself
aws s3 rb s3://orchestration-scaling-demo-bucket --force --region us-east-1
```

---

## Step 6: Delete the AWS SNS Topic
Delete the SNS Topic created for your alarms and Slack notifications:
```powershell
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:376432388605:streaming-app-alerts --region us-east-1
```

---

## Step 7: Delete CloudWatch Log Groups
Remove the logs gathered during testing to prevent storage accumulation costs:
```powershell
aws logs delete-log-group --log-group-name /aws/containerinsights/streaming-app-cluster/application --region us-east-1
aws logs delete-log-group --log-group-name /aws/containerinsights/streaming-app-cluster/host --region us-east-1
```

---

## Step 8: Delete ECR Repositories (Optional)
If you do not plan to reuse these Docker repositories, delete them to avoid registry storage charges:
```powershell
aws ecr delete-repository --repository-name streaming-app-frontend --force --region us-east-1
aws ecr delete-repository --repository-name streaming-app-auth --force --region us-east-1
aws ecr delete-repository --repository-name streaming-app-streaming --force --region us-east-1
aws ecr delete-repository --repository-name streaming-app-admin --force --region us-east-1
aws ecr delete-repository --repository-name streaming-app-chat --force --region us-east-1
```
