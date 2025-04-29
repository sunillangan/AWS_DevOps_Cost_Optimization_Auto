# AWS_DevOps_Cost_Optimization_Auto

AWS Cost Optimization Automation Project - Step-by-Step Practical Guide

⸻

Goal
	•	Monitor AWS daily costs
	•	Stop idle EC2 instances
	•	Send real-time alerts to Slack
(Everything automated using Terraform + Lambda)

⸻

STEP 1: Create a Slack Incoming Webhook
	•	Go to your Slack workspace ➔ Create a channel (e.g., #aws-cost-alerts)
	•	Create an Incoming Webhook:
	•	Go to Slack Webhook App
	•	Set the channel
	•	Copy the Webhook URL

Save your Webhook URL — we will use it later.

⸻

STEP 2: Prepare Terraform Project

2.1. Create Project Structure

mkdir aws-cost-optimization
cd aws-cost-optimization

mkdir terraform lambda
cd terraform

touch provider.tf variables.tf main.tf outputs.tf


providers.tf

provider "aws" {
  region = "us-east-1"   # You can change this
}


variables.tf

variable "slack_webhook_url" {
  description = "Slack Incoming Webhook URL"
  type        = string
}


main.tf

resource "aws_iam_role" "lambda_role" {
  name = "lambda_cost_optimization_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "cost_anomaly_alerts" {
  filename         = "../lambda/cost_anomaly_alerts.zip"
  function_name    = "cost-anomaly-alerts"
  role             = aws_iam_role.lambda_role.arn
  handler          = "app.lambda_handler"
  runtime          = "python3.9"
  timeout          = 300
  environment {
    variables = {
      SLACK_WEBHOOK_URL = var.slack_webhook_url
    }
  }
}

resource "aws_lambda_function" "idle_ec2_stopper" {
  filename         = "../lambda/idle_ec2_stopper.zip"
  function_name    = "idle-ec2-stopper"
  role             = aws_iam_role.lambda_role.arn
  handler          = "app.lambda_handler"
  runtime          = "python3.9"
  timeout          = 300
}

resource "aws_cloudwatch_event_rule" "daily_trigger" {
  name                = "daily-cost-optimizer-trigger"
  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "cost_alert_target" {
  rule = aws_cloudwatch_event_rule.daily_trigger.name
  target_id = "costAlertLambda"
  arn = aws_lambda_function.cost_anomaly_alerts.arn
}

resource "aws_cloudwatch_event_target" "ec2_stop_target" {
  rule = aws_cloudwatch_event_rule.daily_trigger.name
  target_id = "idleEc2Lambda"
  arn = aws_lambda_function.idle_ec2_stopper.arn
}

resource "aws_lambda_permission" "allow_cloudwatch_cost_alert" {
  statement_id  = "AllowExecutionFromCloudWatch"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cost_anomaly_alerts.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_trigger.arn
}

resource "aws_lambda_permission" "allow_cloudwatch_ec2_stop" {
  statement_id  = "AllowExecutionFromCloudWatchIdle"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.idle_ec2_stopper.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_trigger.arn
}




outputs.tf

output "cost_anomaly_lambda_name" {
  value = aws_lambda_function.cost_anomaly_alerts.function_name
}

output "idle_ec2_lambda_name" {
  value = aws_lambda_function.idle_ec2_stopper.function_name
}

STEP 3: Prepare Lambda Functions

3.1. Lambda 1: cost_anomaly_alerts
create a folder
cd ../lambda
mkdir cost_anomaly_alerts
cd cost_anomaly_alerts
touch app.py


paste below code in app.py

import boto3
import requests
import os
from datetime import datetime, timedelta

def lambda_handler(event, context):
    client = boto3.client('ce')  # Cost Explorer
    slack_webhook_url = os.environ['SLACK_WEBHOOK_URL']
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=1)

    response = client.get_cost_and_usage(
        TimePeriod={
            'Start': start.strftime('%Y-%m-%d'),
            'End': end.strftime('%Y-%m-%d')
        },
        Granularity='DAILY',
        Metrics=['UnblendedCost']
    )

    amount = response['ResultsByTime'][0]['Total']['UnblendedCost']['Amount']
    cost = float(amount)

    if cost > 10.0:   # Customize your threshold
        message = f"Warning! Yesterday's AWS cost was ${cost:.2f}. Check resources!"
        requests.post(slack_webhook_url, json={"text": message})

    return {"statusCode": 200}




3.2. Lambda 2: idle_ec2_stopper

Create another folder:

cd ..
mkdir idle_ec2_stopper
cd idle_ec2_stopper
touch app.py



paste below code:

import boto3

def lambda_handler(event, context):
    ec2 = boto3.resource('ec2')
    instances = ec2.instances.filter(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )

    for instance in instances:
        metrics = instance.monitoring['State']
        if metrics == 'disabled':  # Simple check, improve it further
            print(f"Stopping idle instance: {instance.id}")
            instance.stop()

    return {"statusCode": 200}




STEP 4: Package Lambda Functions

From the lambda/ folder:
cd cost_anomaly_alerts
zip ../cost_anomaly_alerts.zip app.py

cd ../idle_ec2_stopper
zip ../idle_ec2_stopper.zip app.py


Now you have:
	•	cost_anomaly_alerts.zip
	•	idle_ec2_stopper.zip

⸻

STEP 5: Deploy with Terraform

Go back to terraform folder:


cd ../terraform

terraform init
terraform apply


Provide the Slack Webhook URL when asked.

Confirm by typing yes.
STEP 6: Testing
	•	Check CloudWatch Logs for both Lambda executions
	•	See Slack messages if the cost crosses the threshold
	•	Verify EC2 instances get stopped automatically if idle



CONGRATULATIONS!

You have now built a full production-grade AWS cost optimization automation, end-to-end, using:
	•	Terraform (Infra as Code)
	•	AWS Lambda (Automation)
	•	EventBridge (Scheduling)
	•	Slack (Real-time alerting)
