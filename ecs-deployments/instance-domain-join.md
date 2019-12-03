# Associate SSM document to ECS Worker instances
This document gives list of steps to attach the domain join SSM document.

## 1. Attach Customer Master Key IAM Policy to ECS Windows NodeInstanceRole
```powershell
# Attach Customer Master key IAM policy to ECS Windows nodeinstancerole.
aws iam attach-role-policy --role-name $nodeInstanceRole --policy-arn $CMKPolicyArn

# Attach SSM Policy to EC2 Instance
aws iam attach-role-policy --role-name $nodeInstanceRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

##### ACTION REQUIRED - START #####
```powershell
$nodeInstanceRole = "xxxxx" # ECS Windows worker node's instance role.
$autoScalingGroup = "xxxxx" # Autoscaling group name of ECS Windows workers. You can get the value from instance tag. Tag name : aws:autoscaling:groupName.
```
##### ACTION REQUIRED - END #####

## 2. Attach Domain Join SSM document to ECS Windows Autoscaling group
```powershell
# Create SSM association between autoscaling group and SSM Document
aws ssm create-association --name $domainjoinSSMdoc --document-version 1 --targets "Key=tag:aws:autoscaling:groupName,Values=$autoScalingGroup"

# Validate the association is created
aws ssm list-associations --association-filter-list "key=Name, value=$domainjoinSSMdoc"
```

## 3. Create and join gMSA AD security group (Optional)
*Before proceeding further, you need to wait for the domain join SSM document execution. You can check the status in SSM Runcommand or under State manager.*

*If the AD security group exists already prior to domain join, the worker instance will be added to that security group during domain join. Otherwise, you need to execute this document to create and join AD. AD security group creation shouldn't be executed concurrently. Concurrent execution will result into duplicate AD group creation. Hence this needs to be run one instance at a time. This SSM document shoudn't be attached to autoscaling group*

```powershell
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $autoScalingGroup --query "AutoScalingGroups[*].Instances[*].InstanceId" --output text

##### ACTION REQUIRED - START #####
# Replace XXXXX with each of the above instance id.
# You need to send the following commands one by one.
$commandId = aws ssm send-command --document-name $adGroupCreateSSMdoc --targets "Key=InstanceIds, Values=XXXXX" --parameters "ADSecurityGroup=$gMSAADSecurityGroup" --query "Command.CommandId" --output text

aws ssm list-command-invocations --command-id $commandId
##### ACTION REQUIRED - END #####
```