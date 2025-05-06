# minecraft-logging
# Minecraft Server Logging

![Project Overview](images/minecraft-logging-card.png)

## Overview

This project automates deployment of a **Minecraft** server on **AWS Lightsail**, centralizes gameplay logs using **Amazon CloudWatch Logs**, and demonstrates forwarding those logs to backend services (EFS, DynamoDB, or custom EC2/Lambda) for real-time monitoring.

## Objectives

* Deploy a Minecraft Java server on AWS Lightsail
* Install and configure the Amazon CloudWatch Agent to collect `latest.log`
* Stream game logs to CloudWatch Logs with custom retention
* Forward log events to downstream services via Lambda or direct HTTP
* Validate end-to-end log ingestion and monitoring

## Architecture Diagram

```text
[Lightsail Minecraft Server]
       │  (CloudWatch Agent)
       ▼
[CloudWatch Logs] → [AWS Lambda] → [EFS / DynamoDB]
                         ↓
                   [EC2 Web Server]
```

## Prerequisites

1. AWS account in **us-east-1 (N. Virginia)**
2. AWS CLI or Console access
3. Basic knowledge of Linux shell and AWS services

## Repository Structure

```bash
├── images/                          # Screenshots for this README
│   ├── create-cloudwatch-user.png       # IAM user creation for CloudWatch
│   ├── attach-policies.png              # Attaching CloudWatch policies
│   ├── create-lightsail-instance.png    # Launch Lightsail Ubuntu instance
│   ├── startup-script.png               # Minecraft & CloudWatch agent setup script
│   ├── cloudwatch-config-json.png       # CloudWatch agent JSON config
│   ├── cloudwatch-config-toml.png       # CloudWatch agent TOML credentials
│   ├── configure-credentials.png        # AWS credentials file for agent
│   ├── agent-status.png                 # Verifying agent status
│   ├── open-port-25565.png              # Opening Minecraft port 25565
│   ├── update-eula.png                  # Accepting the Minecraft EULA
│   ├── vpc-creation.png                 # Creating a VPC for additional services
│   ├── ec2-web-server.png               # Deploying EC2 web server
│   ├── create-lambda-function.png       # Creating Lambda for log forwarding
│   ├── lambda-env-vars.png              # Setting Lambda environment variables
│   ├── lambda-timeout.png               # Adjusting Lambda timeout
│   ├── create-subscription-filter.png   # Creating CloudWatch Logs subscription
│   ├── test-subscription-filter.png     # Testing subscription filter
│   ├── server-py-run.png                # Starting Python web server on EC2
│   └── verify-end-to-end.png            # Final end-to-end log test
└── README.md                        # Project documentation (this file)
```

## Setup & Deployment

### 1. Create IAM User for CloudWatch

1. In the IAM console, create a new user (e.g., `minecraft-cloudwatch`).
2. Attach the **CloudWatchAgentServerPolicy** and **CloudWatchLogsFullAccess** policies.
3. Generate an access key and record the credentials.

![Create CloudWatch User](images/create-cloudwatch-user.png)
![Attach Policies](images/attach-policies.png)

### 2. Launch AWS Lightsail Instance

1. In Lightsail (us-east-1a), launch an **Ubuntu 24.04** instance.
2. Copy the instance’s public IP.

![Create Lightsail Instance](images/create-lightsail-instance.png)

### 3. Install Minecraft Server & CloudWatch Agent

SSH into your instance and run the startup script:

```bash
apt-get update
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
dpkg -i -E ./amazon-cloudwatch-agent.deb
apt install -y tmux default-jre
mkdir ~/minecraft-server && cd ~/minecraft-server
wget https://piston-data.mojang.com/v1/objects/.../server.jar
java -Xmx1G -Xms1G -jar server.jar nogui
```

![Startup Script](images/startup-script.png)

### 4. Configure CloudWatch Agent

* **config.json** (sets log path, group, and stream):

  ![CloudWatch Config JSON](images/cloudwatch-config-json.png)

* **common-config.toml** (credentials profile override):

  ![CloudWatch Config TOML](images/cloudwatch-config-toml.png)

* **Credentials file** (`~/.aws/credentials`):

  ![Configure Credentials](images/configure-credentials.png)

Start and verify the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
sudo systemctl status amazon-cloudwatch-agent
```

![Agent Status](images/agent-status.png)

### 5. Open Minecraft Port & Accept EULA

1. Open TCP port **25565** in the Lightsail firewall.

![Open Port 25565](images/open-port-25565.png)

2. Accept the EULA in `eula.txt` and restart the server.

![Update EULA](images/update-eula.png)

### 6. Verify Logs in CloudWatch

* Navigate to **CloudWatch → Log Groups → minecraft-server-logs**
* Confirm log streams named `{instance-id}-minecraft` are active.

### 7. Forward Logs to Downstream Services

#### a. Create a VPC and EC2 Web Server

Use the AWS console to configure a VPC and launch an EC2 instance to host a Python log receiver.

![VPC Creation](images/vpc-creation.png)
![EC2 Web Server](images/ec2-web-server.png)

#### b. Deploy AWS Lambda Function

1. Create a Lambda function in Node.js.
2. Paste the log-forwarding code and set `TargetIP` to the EC2 internal DNS.
3. Increase the function **timeout** to **20 seconds**.
4. Configure environment variables for the target endpoint.

![Create Lambda Function](images/create-lambda-function.png)
![Lambda Env Vars](images/lambda-env-vars.png)
![Lambda Timeout](images/lambda-timeout.png)

#### c. Configure CloudWatch Subscription Filter

Attach a subscription filter on **minecraft-server-logs** to invoke your Lambda:

```bash
aws logs put-subscription-filter \
  --log-group-name minecraft-server-logs \
  --filter-name ForwardToLambda \
  --filter-pattern "" \
  --destination-arn arn:aws:lambda:...:function:YourLambda
```

![Create Subscription Filter](images/create-subscription-filter.png)
![Test Subscription Filter](images/test-subscription-filter.png)

### 8. Run Python Log Receiver

SSH into your EC2 web server and start the Flask-based receiver:

```bash
python3 server.py
```

![Start Python Server](images/server-py-run.png)

## Validation

1. Trigger a log event by joining the Minecraft server.
2. Observe real-time log entries in your Python web app or DynamoDB/EFS.

![Verify End-to-End](images/verify-end-to-end.png)

## Next Steps

* Automate deployment using Terraform or CloudFormation
* Add dashboards in CloudWatch or Grafana
* Implement AWS WAF and Shield for protection
