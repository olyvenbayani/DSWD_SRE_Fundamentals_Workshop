# Critical DTR App Migration Plan

## Overview
This plan migrates a monolithic DTR application to a highly available, decoupled architecture. 
The goal is 99.9% uptime and zero data loss during the high-traffic first 3 days of the month.

## Step 1: Database Extraction (The "Point of No Return")
Since DTR data is sensitive, we must ensure no data is lost during the move:
1. Stop the application on the current `c6i.xlarge` to freeze the database state.
2. Perform a full backup: `pg_dump -h localhost -U [user] [db_name] > dtr_prod_backup.sql`.
3. Import this into the new RDS instance created by the CloudFormation template below.

## Step 2: Health Check Configuration
DTR apps must be responsive. Ensure your Docker container has a `/health` route that:
- Returns a `200 OK` status.
- Validates the connection to the database.
- The ALB will use this to determine if an instance is "healthy" before sending traffic.

## Step 3: Deployment Strategy
1. Deploy the CloudFormation stack.
2. Update your `UserData` in the template to include your DB connection strings.
3. Point your DNS (e.g., `dtr.clientdomain.com`) to the ALB DNS name.

Cloudformation template > Save this as <filename>.yaml

```
AWSTemplateFormatVersion: '2010-09-09'
Description: 'High-Availability Stack for Critical DTR Application'

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Select at least two subnets in DIFFERENT Availability Zones.
  DBPassword:
    NoEcho: true
    Type: String
    Description: Master password for RDS.

Resources:
  # --- SECURITY GROUPS ---
  ALBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Public HTTP access
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  AppSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App access from ALB
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !GetAtt ALBSG.GroupId

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: DB access from App
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !GetAtt AppSG.GroupId

  # --- DATABASE (HIGH AVAILABILITY) ---
  DTRDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: db.t3.medium
      Engine: postgres
      AllocatedStorage: '50' # Increased for DTR logs
      MasterUsername: dtr_admin
      MasterUserPassword: !Ref DBPassword
      MultiAZ: true  # CRITICAL: Ensures DB survives an AZ failure
      VPCSecurityGroups:
        - !GetAtt DBSG.GroupId
      BackupRetentionPeriod: 7
      StorageType: gp3

  # --- COMPUTE TIER (ALB + ASG) ---
  DTRLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: !Ref Subnets
      SecurityGroups:
        - !GetAtt ALBSG.GroupId

  DTRTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VpcId
      Port: 80
      Protocol: HTTP
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30

  DTRListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref DTRLoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref DTRTargetGroup

  # Launch Template - NO SPOT INSTANCES
  DTRLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        InstanceType: c6i.large
        ImageId: ami-0c55b159cbfafe1f0 # Update to your specific AMI
        SecurityGroupIds:
          - !GetAtt AppSG.GroupId
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            # Commands to pull and start your Docker DTR app
            # docker run -e DB_URL=${DTRDatabase.Endpoint.Address} -p 80:80 dtr-app:latest

  DTRASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier: !Ref Subnets
      LaunchTemplate:
        LaunchTemplateId: !Ref DTRLaunchTemplate
        Version: !GetAtt DTRLaunchTemplate.LatestVersionNumber
      MinSize: '2'      # Always have at least 2 for redundancy
      MaxSize: '6'      # Enough overhead for the 3-day peak
      DesiredCapacity: '2'
      TargetGroupARNs:
        - !Ref DTRTargetGroup

  # Conservative Scaling Policy for DTR Stability
  DTRScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref DTRASG
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 50.0 # Scale earlier (at 50%) to ensure DTR remains snappy

Outputs:
  AppURL:
    Value: !GetAtt DTRLoadBalancer.DNSName
  DBHost:
    Value: !GetAtt DTRDatabase.Endpoint.Address
```
