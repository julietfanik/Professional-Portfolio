# Professional-Portfolio

## 👤 About Me
- I am Juliet Fanik, an aspiring Cloud Engineer currently building hands-on experience with AWS infrastructure, including scalable architectures and Infrastructure-as-Code (IaC).  
- I am currently pursuing the AWS Solutions Architect Associate certification and actively building real-world AWS projects to strengthen cloud engineering skills.

---

## Overview

This portfolio showcases two AWS cloud projects demonstrating:
- Scalable architecture design
- High availability and fault tolerance
- Infrastructure-as-Code (CloudFormation)
- Monitoring and alerting with CloudWatch and SNS

---

# 🚀 Project 1: Highly Available AWS Architecture (Console Deployment)

## Overview
I designed and deployed this architecture to demonstrate a highly available AWS architecture using EC2, Application Load Balancer, and Auto Scaling Groups across multiple Availability Zones.

---

## Architecture Components
- EC2 instances in us-east-1a and us-east-1b
- Application Load Balancer (ALB)
- Auto Scaling Group with CPU-based scaling
- VPC with subnets, route tables, and security groups
- CloudWatch monitoring
- SNS notifications for alerts

---

## Key Features
- Auto Scaling policy: Automatically scales EC2 instances to maintain ~50% average CPU utilization 
- Health checks configured via ALB target groups  
- Simulated failure testing using unhealthy instance state  
- EC2 user data script displays AZ-specific message:
  - “Hello Juliet from AZ-A / AZ-B”
  
---
## Issues Resolved
- Fixed public EC2 instance exposure by updating the Launch Template security configuration to restrict inbound traffic to only allow access from the Application Load Balancer (ALB).
  
---

## Screenshots
### Application Load Balancer (ALB)
![Application Load Balancer](application-load-balancer.png)
#### The ALB is successfully routing traffic to both EC2 instances in us-east-1a and us-east-1b.

---

### ALB Details
![ALB Details](alb-details.png)
#### The ALB overview page displays key configuration details, including the VPC, subnet IDs, internet-facing scheme, and the associated target group used for routing traffic to backend instances.

---

### ALB Target Groups
![Target Groups](alb-target-groups.png)
#### The target group shows two healthy registered EC2 instances.

---

### Auto Scaling Group Details
![ASG Details](asg-details-page.png)
#### The Auto Scaling Group overview displays scaling configuration, launch template details, and associated subnets, which align with the ALB subnet configuration.

---

### Target Tracking Policy
![ASG Target Tracking Policy](asg-target-tracking-policy.png)
#### The scaling policy is configured using Target Tracking to maintain average CPU utilization around 50%.

---

### Auto Scaling Group Activity
![ASG Activity](asg-activity.png)
#### The activity log shows successful EC2 instance launches and automatic scaling events triggered when an instance was manually terminated.

---

### CloudWatch Alarm
![CloudWatch Alarm](cloudwatch-health-check-failure.png)
#### This shows the CloudWatch alarm being triggered when the health check path was temporarily changed to `/fake`, and returning to normal when restored to `/`.

---

## What I Learned
- How Application Load Balancers distribute traffic across multiple Availability Zones to improve availability
- How Auto Scaling Groups respond to CPU usage to automatically scale infrastructure
- How CloudWatch and SNS work together for monitoring and alerting
- How to design fault-tolerant AWS architectures using core cloud services

---

# 🚀 Project 2: Infrastructure as Code (CloudFormation)

## Overview
This project builds upon the console-based deployment in Project 1 by implementing Infrastructure-as-Code to ensure repeatable, version-controlled infrastructure provisioning.

---

## Services Used
- EC2 Launch Templates
- Application Load Balancer
- Auto Scaling Groups
- VPC (subnets, routing, security groups)
- CloudWatch Alarms
- SNS Notifications

---

## Key Features
- Fully automated multi-AZ deployment using AWS CloudFormation  
- Repeatable Infrastructure-as-Code (IaC) for consistent environment provisioning  
- Automated scaling and monitoring using Auto Scaling policies and CloudWatch alarms  
- Implemented end-to-end observability and alerting using SNS notifications  


---

## Issues Resolved During Development

- **Monitoring & Alerting Enhancement:**  
  - Expanded CloudWatch alarms to include `UnHealthyHostCount > 0`, improving detection of instance and application health issues. This was tested using controlled health check failures.

- **User Data Script Debugging:**  
 - Fixed formatting issues in the EC2 user data script that broke the static web page. Cleaned up the HTML and improved EC2 metadata retrieval using IMDS to show correct instance information.

---

### CloudFormation Infrastructure Template (YAML)

This CloudFormation template defines a highly available AWS architecture using EC2, an Application Load Balancer, Auto Scaling Groups, CloudWatch, and SNS. 
It provisions a highly available architecture including the following:
Application Load Balancer
Auto Scaling Group
EC2 instances
CloudWatch monitoring
SNS notifications

---

## CloudFormation Template

```yaml
Resources:

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: vpc-03f993d95e53dfba0
      GroupDescription: Allow HTTP Traffic to ALB
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: vpc-03f993d95e53dfba0
      GroupDescription: Allow traffic only from ALB
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: ami-0eb38b817b93460ac
        InstanceType: t3.micro
        SecurityGroupIds:
          - !Ref EC2SecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd

            TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
            -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

            INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
            -s http://169.254.169.254/latest/meta-data/instance-id)

            AZ=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
            -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

            cat <<EOF > /var/www/html/index.html
            <h1>CloudFormation Deployment Successful</h1>
            <h3>Instance ID: $INSTANCE_ID</h3>
            <h3>Availability Zone: $AZ</h3>
            EOF

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: HTTP
      VpcId: vpc-03f993d95e53dfba0
      TargetType: instance
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Type: application
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Subnets:
        - subnet-07cd6603c3e713a3c
        - subnet-009b8ab0f4e293de0

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: "2"
      DesiredCapacity: "2"
      MaxSize: "4"
      VPCZoneIdentifier:
        - subnet-07cd6603c3e713a3c
        - subnet-009b8ab0f4e293de0
      TargetGroupARNs:
        - !Ref TargetGroup
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      HealthCheckType: ELB
      HealthCheckGracePeriod: 120

  TargetTrackingScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 50

  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: asg-alerts

  CPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: High CPU on ASG instances
      Namespace: AWS/AutoScaling
      MetricName: GroupAverageCPUUtilization
      Statistic: Average
      Period: 60
      EvaluationPeriods: 2
      Threshold: 70
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      AlarmActions:
        - !Ref SNSTopic

  UnhealthyHostAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: ALB target group has unhealthy hosts
      Namespace: AWS/ApplicationELB
      MetricName: UnHealthyHostCount
      Statistic: Average
      Period: 60
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanOrEqualToThreshold
      Dimensions:
        - Name: TargetGroup
          Value: !GetAtt TargetGroup.TargetGroupFullName
        - Name: LoadBalancer
          Value: !GetAtt ApplicationLoadBalancer.LoadBalancerFullName
      AlarmActions:
        - !Ref SNSTopic

  SNSSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: email
      Endpoint: your-email@example.com
      TopicArn: !Ref SNSTopic
```

## 📸 Screenshots  
	  
### CloudFormation Stack Creation Complete
![Stack Creation Complete](cloudformation-stack-creation-complete.png)
#### Shows the successful creation of the CloudFormation stack resources.

---

### Infrastructure Composer
![Infrastructure Composer](cloudformation-infrastructure-composer.png)
#### Displays the architecture of the deployed CloudFormation stack.

---

### Auto Scaling Group Overview
![ASG Overview](cloudformation-asg-overview.png)
#### Shows Auto Scaling Group configuration including launch template, capacity, and network settings.

---

### Auto Scaling Group Instance Management
![ASG Instance Management](cloudformation-asg-instance-management.png)
#### Shows both healthy EC2 instances running in different Availability Zones.

---

### Auto Scaling Instance Termination Activity
![ASG Termination Activity](cloudformation-autoscaling-instance-termination.png)
#### Demonstrates successful Auto Scaling behavior after an instance was manually terminated.

---

### EC2 Security Group Rules
![Security Group Rules](cloudformation-ec2-security-group-rule.png)
#### Security group configured to allow traffic only from the Application Load Balancer.

---

### Application Load Balancer (ALB)
![ALB Working](cloudformation-alb-working.png)
#### Shows the ALB distributing traffic to two healthy EC2 instances with the deployed user script running successfully.

---

### ALB Overview Page
![ALB Overview](cloudformation-alb-overview-page.png)
#### Displays ALB configuration including VPC, subnets, DNS, and associated target group.

---

### Target Group Overview
![Target Group Overview](cloudformation-target-group-overview.png)
#### Shows registered instances and their health status within the target group.

---

### Health Check Failure Trigger
![Health Check Failure Trigger](cloudformation-health-check-failure-trigger.png)
#### Shows the configuration used to simulate failure by changing the health check path to `/fake`.

---

### Target Group Monitoring
![Target Group Monitoring](cloudformation-targetgroup-monitoring.png)
#### Monitoring graphs show unhealthy host count increasing while healthy hosts drop to zero during failure testing.

---

### CloudWatch Unhealthy Host Alarm
![CloudWatch Alarm](cloudformation-cloudwatch-unhealthyhost.png)
#### CloudWatch alarm triggered when the target group reported unhealthy instances.

---

### SNS Email Notification
![SNS Email Notification](cloudformation-sns-email.png)
#### Shows SNS notification triggered by the CloudWatch alarm indicating unhealthy hosts in the ALB target group.
---

## What I Learned
- How AWS resource dependencies work in IaC
- How to debug CloudFormation stack failures
- How to design repeatable infrastructure deployments
- Difference between manual vs automated infrastructure provisioning

---

# Skills Demonstrated

- AWS Cloud Architecture Design
- High Availability & Fault Tolerance
- Auto Scaling & Load Balancing
- Infrastructure as Code (CloudFormation)
- Cloud Monitoring (CloudWatch)
- Alerting Systems (SNS)
- VPC Networking Design
- Debugging AWS Deployment Issues

---

# Certification
AWS Certified Solutions Architect Associate (In Progress)  
Expected Completion: June 2026

---

## 👤 Author

Juliet Fanik  

- GitHub: https://github.com/julietfanik  
- LinkedIn: https://www.linkedin.com/in/juliet-fanik-9a0594140/  
- Resume: [Juliet Fayez Fanik Resume](juliet-fayez-fanik-resume.pdf)
