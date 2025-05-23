# Containers in the Cloud: AWS Workshop Lab

This workshop guides you through essential container services in AWS with a focus on practical deployments in production environments. The lab is designed to be completed in 45 minutes, with optional sections for additional learning.

## CORE LAB 1: BUILDING AND PUSHING CONTAINERS TO AMAZON ECR

### Exercise 1.1: Create a Simple API Application

First, let's create a simple Flask API that we'll use throughout our exercises:


#### Create project directory
```bash
mkdir -p ~/container-workshop/flask-api
cd ~/container-workshop/flask-api
```

#### Create Python Flask application that will be deployed as a container
```bash
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import datetime
import os
import json

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from AWS Containers Workshop!',
        'container_id': socket.gethostname(),
        'timestamp': datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
EOF
```

#### Create requirements.txt to store the app's dependencies
```bash
cat > requirements.txt << 'EOF'
flask==2.0.1
werkzeug==2.0.3
EOF
```

#### Create Dockerfile -- the template for the container
```bash
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

ENV PORT=8080
EXPOSE 8080

CMD ["python", "app.py"]
EOF
```

### Exercise 1.2: Build and Test Locally

Let's build and test our application locally. Replace `<YOUR-USERNAME>` in this exercise with your actual username.


#### Build the Docker image 

#### If you are on an intel Mac i.e., the old Macs
```bash
docker build -t flask-api:<YOUR-USERNAME> .
```

#### If you are on the M chip Mac i.e., the newer, fancier Macs
```bash
docker build --platform=linux/amd64 -t flask-api:<YOUR-USERNAME> .
```

#### Run the container locally
```bash
docker run -d -p 8080:8080 --name api-test flask-api:<YOUR-USERNAME> .
```

#### Test the container or go to http://localhost:8080 in your browser
```bash
curl http://localhost:8080
```

#### Clean up the local container
```bash
docker stop api-test
docker rm api-test
```

You should see JSON output with the container ID and timestamp.

### Exercise 1.3: Push Image to Amazon ECR

Now we'll push our image to Amazon ECR:

#### Configure AWS credentials. MAKE SURE the region gets set to us-east-1
```bash
aws configure --profile workshop
# Enter your AWS Access Key ID when prompted
# Enter your AWS Secret Access Key when prompted
# Region: us-east-1
# Output format: json

export AWS_PROFILE=workshop

# Get AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --profile workshop --query Account --output text)
aws_username=$(aws sts get-caller-identity --profile workshop --query "Arn" --output text | cut -d/ -f2)
echo "Your AWS Account ID: $ACCOUNT_ID"
```

#### Login to Amazon ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
```

#### Tag the image for ECR
```bash
docker tag flask-api:${aws_username} ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/flask-api:${aws_username}
```

#### Push the image to ECR to store the image in the private registry
```bash
docker push ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/flask-api:${aws_username}
```

#### Verify the image in ECR
```bash
aws ecr describe-images --repository-name workshop/flask-api
```

## CORE LAB 2: DEPLOYING CONTAINERS WITH ECS FARGATE

### Exercise 2.1: Create ECS Task Definition -- defines the container configuration in ECS

Let's create an ECS task definition for our API:

#### Create directory for task definitions
```bash
mkdir -p ~/container-workshop/ecs
cd ~/container-workshop/ecs
```

#### Create ECS task definition.
```bash
cat > api-task-def.json << EOF
{
  "family": "flask-api-task-${aws_username}",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "flask-api",
      "image": "${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/flask-api:${aws_username}",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "256",
  "memory": "512"
}
EOF
```

#### Register the task definition
```bash
aws ecs register-task-definition --cli-input-json file://api-task-def.json
```

### Exercise 2.2: Deploy to ECS Fargate

Now let's deploy our container as an ECS service:

#### Get VPC details for task networking. We need this to make our app available to the internet in a secure way.
```bash
# Note: This assumes you have a default VPC. If not, use your own VPC ID.
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
```

#### Get public subnets from the VPC
```bash
SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=map-public-ip-on-launch,Values=true" --query "Subnets[0:2].SubnetId" --output text | tr '\t' ',')
```

#### Create a security group for the ECS tasks
```bash
SG_ID=$(aws ec2 create-security-group --group-name EcsApiTaskSG-$(date +%s) --description "Security group for ECS API tasks" --vpc-id $VPC_ID --query "GroupId" --output text)
```

#### Add inbound rule to the security group. Enables us to access the app from the public internet
```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 8080 --cidr 0.0.0.0/0

echo "VPC ID: $VPC_ID"
echo "Subnet IDs: $SUBNET_IDS"
echo "Security Group ID: $SG_ID"
```

#### Create the ECS service
```bash
aws ecs create-service \
  --cluster WorkshopCluster \
  --service-name flask-api-task-${aws_username} \
  --task-definition flask-api-task-${aws_username} \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --tags key=Workshop,value=ContainersInCloud
```

### Exercise 2.3: Test ECS Service

Let's verify our service is running and test it:

```bash
# Wait a moment for the service to start
echo "Waiting for the service to start..."
sleep 20

# Get the running task
TASK_ARN=$(aws ecs list-tasks --cluster WorkshopCluster --service-name flask-api-task-${aws_username} --query "taskArns[0]" --output text)
```

#### Get the ENI details - You need this so you can get the Public IP address of the service
```bash
ENI=$(aws ecs describe-tasks --cluster WorkshopCluster --tasks $TASK_ARN --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
```

#### Get the public IP
```bash
PUBLIC_IP=$(aws ec2 describe-network-interfaces --network-interface-ids $ENI --query "NetworkInterfaces[0].Association.PublicIp" --output text)

echo "API Service available at: http://$PUBLIC_IP:8080"
```

#### Test the API. Feel free to use your browser and go to the address using the IP address:8080 
```bash
curl http://$PUBLIC_IP:8080
```

## CORE LAB 3: DEPLOYING CONTAINERS WITH AWS LAMBDA

### Exercise 3.1: Prepare Container for Lambda

Let's modify our API to work with Lambda:

#### Create directory for Lambda container
```bash
mkdir -p ~/container-workshop/lambda
cd ~/container-workshop/lambda
```

#### Create Lambda handler file -- defines our app code
```bash
cat > app.py << 'EOF'
import json
import socket
import datetime
import os

def lambda_handler(event, context):
    return {
        "statusCode": 200,
        "headers": {
            "Content-Type": "application/json"
        },
        "body": json.dumps({
            "message": "Hello from AWS Lambda Container!",
            "container_id": socket.gethostname(),
            "timestamp": datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
            "environment": "lambda",
            "event": event
        })
    }
EOF
```

#### Create requirements.txt
```bash
cat > requirements.txt << 'EOF'
# No external dependencies for this simple example
EOF
```

#### Create Dockerfile for Lambda -- template for our container
```bash
cat > Dockerfile << 'EOF'
FROM amazon/aws-lambda-python:3.9

# Copy requirements.txt
COPY requirements.txt ${LAMBDA_TASK_ROOT:-/var/task}

# Install the dependencies
RUN pip3 install -r requirements.txt

# Copy function code
COPY app.py ${LAMBDA_TASK_ROOT:-/var/task}

# Set the CMD to your handler
CMD [ "app.lambda_handler" ]
EOF
```

### Exercise 3.2: Build and Push Lambda Container

#### Build the Lambda container image
```bash
docker build -t lambda-api:${aws_username} .
```

#### Tag the image for ECR
```bash
docker tag lambda-api:${aws_username} ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/lambda-api:${aws_username}
```

#### Push the image to ECR
```bash
docker push ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/lambda-api:${aws_username}
```

### Exercise 3.3: Create Lambda Function which runs the container we defined earlier

```bash
aws lambda create-function \
  --function-name workshop-container-function-${aws_username} \
  --package-type Image \
  --code ImageUri=${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/workshop/lambda-api:${aws_username} \
  --role arn:aws:iam::${ACCOUNT_ID}:role/lambda-container-role \
  --timeout 30 \
  --memory-size 512 \
  --tags Workshop=ContainersInCloud
```

### Exercise 3.4: Create API Gateway for Lambda

#### Create an API Gateway REST API. Needed in order to connect to our Lambda Function over the internet securely
```bash
API_ID=$(aws apigateway create-rest-api \
  --name workshop-container-api-${aws_username} \
  --query "id" \
  --output text)
```

#### Get the root resource ID
```bash
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query "items[0].id" \
  --output text)
```

#### Create a resource
```bash
RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part "api" \
  --query "id" \
  --output text)
```

#### Create a method
```bash
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method ANY \
  --authorization-type NONE
```

#### Get the Lambda function ARN which will let us connect API Gateway to the Lambda function
```bash
LAMBDA_ARN=$(aws lambda get-function \
  --function-name workshop-container-function-${aws_username} \
  --query "Configuration.FunctionArn" \
  --output text)
```

#### Create an integration to connect API Gateway and the Lambda
```bash
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method ANY \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations
```

#### Add Lambda permission to let API Gateway to start the container
```bash
aws lambda add-permission \
  --function-name workshop-container-function-${aws_username} \
  --statement-id apigateway-test \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:$ACCOUNT_ID:$API_ID/*/*/api"
```

#### Deploy the API -- making it available on the internet
```bash
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod
```

#### Get the API endpoint
```bash
API_ENDPOINT="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod/api"
echo "Lambda API available at: $API_ENDPOINT"
```

#### Test the API
```bash
curl $API_ENDPOINT
```

## OPTIONAL LAB 1: AWS APP RUNNER DEPLOYMENT

If you have additional time, you can deploy the same container to AWS App Runner:

### Optional Exercise 1.1: Deploy to App Runner

```bash
# Create App Runner service
aws apprunner create-service \
  --service-name flask-api-apprunner \
  --source-configuration '{
    "ImageRepository": {
      "ImageIdentifier": "'$ACCOUNT_ID'.dkr.ecr.us-east-1.amazonaws.com/workshop/flask-api:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ENVIRONMENT": "apprunner"
        }
      }
    },
    "AutoDeploymentsEnabled": true,
    "AuthenticationConfiguration": {
      "AccessRoleArn": ""
    }
  }' \
  --instance-configuration '{
    "Cpu": "1 vCPU",
    "Memory": "2 GB"
  }' \
  --auto-scaling-configuration-arn ""

# Wait for the service to be created
echo "Waiting for App Runner service to be created (this may take a few minutes)..."
sleep 60

# Get the App Runner service URL
APP_RUNNER_URL=$(aws apprunner list-services \
  --query "ServiceSummaryList[?ServiceName=='flask-api-apprunner'].ServiceUrl" \
  --output text)

echo "Your App Runner application is available at: https://$APP_RUNNER_URL"

# Test the App Runner service
curl https://$APP_RUNNER_URL
```

## OPTIONAL LAB 2: AUTO-SCALING CONTAINERS IN ECS

If you want to explore auto-scaling containers:

### Optional Exercise 2.1: Configure ECS Service Auto Scaling

```bash
# Register a scalable target for the API service
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/WorkshopCluster/flask-api-service \
  --min-capacity 1 \
  --max-capacity 4

# Create a scaling policy based on CPU utilization
cat > scaling-policy.json << EOF
{
  "TargetValue": 70.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleOutCooldown": 60,
  "ScaleInCooldown": 60
}
EOF

aws application-autoscaling put-scaling-policy \
  --policy-name cpu-target-tracking-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/WorkshopCluster/flask-api-service \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

### Optional Exercise 2.2: Generate Load to Test Auto Scaling

```bash
# Create a simple load generation script
cd ~/container-workshop
cat > load-test.sh << 'EOF'
#!/bin/bash

# Get the API endpoint from the command line
API_ENDPOINT=$1
if [ -z "$API_ENDPOINT" ]; then
  echo "Usage: $0 <api-endpoint>"
  exit 1
fi

echo "Starting load test on $API_ENDPOINT"
echo "Press Ctrl+C to stop"

# Run continuous requests
while true; do
  for i in {1..50}; do
    curl -s "$API_ENDPOINT" > /dev/null &
  done
  echo "Sent 50 requests to $API_ENDPOINT"
  sleep 1
done
EOF

chmod +x load-test.sh

# Run the load test against your ECS service
./load-test.sh http://$PUBLIC_IP:8080
```

In a separate terminal, monitor the service events:

```bash
# Monitor service events
aws ecs describe-services \
  --cluster WorkshopCluster \
  --services flask-api-service \
  --query "services[0].events[0:5]"
```

## CLEANUP LAB: REMOVING WORKSHOP RESOURCES

At the end of your workshop, clean up your resources:

```bash
# Delete Lambda resources
echo "Cleaning up Lambda resources..."
aws apigateway delete-rest-api --rest-api-id $API_ID || true
aws lambda delete-function --function-name workshop-container-function || true

# Delete ECS resources
echo "Cleaning up ECS resources..."
aws ecs update-service --cluster WorkshopCluster --service flask-api-service --desired-count 0
aws ecs delete-service --cluster WorkshopCluster --service flask-api-service --force
aws logs delete-log-group --log-group-name /ecs/flask-api

# Delete App Runner resources (if created)
echo "Cleaning up App Runner resources (if created)..."
APP_RUNNER_SERVICE=$(aws apprunner list-services --query "ServiceSummaryList[?ServiceName=='flask-api-apprunner'].ServiceArn" --output text)
if [ ! -z "$APP_RUNNER_SERVICE" ]; then
  aws apprunner delete-service --service-arn $APP_RUNNER_SERVICE
fi

# Delete ECR resources
echo "Cleaning up ECR resources..."
aws ecr batch-delete-image --repository-name workshop/flask-api --image-ids imageTag=latest || true
aws ecr batch-delete-image --repository-name workshop/lambda-api --image-ids imageTag=latest || true
aws ecr delete-repository --repository-name workshop/flask-api --force || true
aws ecr delete-repository --repository-name workshop/lambda-api --force || true

# Delete security groups
echo "Cleaning up security groups..."
aws ec2 delete-security-group --group-id $SG_ID || true

echo "Cleanup complete!"
```

## WORKSHOP CONCLUSION

Congratulations on completing the "Containers in the Cloud" AWS workshop! You've learned how to:

1. Build and push container images to Amazon ECR
2. Deploy containers using Amazon ECS with Fargate
3. Deploy containers as serverless functions with AWS Lambda
4. (Optional) Deploy containers with AWS App Runner
5. (Optional) Configure auto-scaling for containerized applications

These skills provide a foundation for running containers in AWS cloud environments.

## NEXT STEPS

To continue your container journey on AWS, consider exploring:

1. Amazon EKS (Elastic Kubernetes Service)
2. Amazon ECS with Service Discovery
3. AWS CDK for infrastructure as code deployment of containers
4. CI/CD pipelines for container deployments
5. Container security best practices
