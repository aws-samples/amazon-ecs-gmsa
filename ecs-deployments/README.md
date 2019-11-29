# ECS Windows instnace and generate gMSA
This documemt gives list of steps to domain join, create gMSA and generate credspec file.

*Set Working Directory to ./amazon-ecs-gmsa/ecs-deployments/*

## Create gMSA account
```powershell
# Check SSM document for the default parameter values
aws ssm describe-document --name $credspecSSMdoc 

# Make sure ECS Windows instances domain joined before proceeding to the next step
aws ssm list-command-invocations --filter key=DocumentName,value=$domainjoinSSMdoc

##### ACTION REQUIRED - START #####
# Create ECS Cluster and store the cluster name in $ECSClusterName.
# Also make sure ECS instances joined the ECS cluster.
$ECSClusterName = "xxxxx"
# If you see any failures in domain join, go to System Manager for further investigation.
# https://us-west-2.console.aws.amazon.com/systems-manager/run-command/complete-commands?region=us-west-2 
# You can execute the ssmdomain join document on-demand on that instnace id.
# To retry on failed instance (replace XXXXX with failed instance id)
$commandId = aws ssm send-command --document-name $domainjoinSSMdoc --targets "Key=InstanceIds, Values=XXXXX" --query "Command.CommandId" --output text

aws ssm list-command-invocations --command-id $commandId

# If the failure is in between (Example: joined AD but not part of AD security group), you've to manually fix that based on the error. The easiest way is, terminate the instance and let autoscaling create a new instance.
##### ACTION REQUIRED - END #####

# Pick an instance to run the SSM command.
$firstInstanceId = aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $autoScalingGroup --query "AutoScalingGroups[*].Instances[0].InstanceId" --output text

# Overwride the default values by passing additional parameters.
$commandId = aws ssm send-command --document-name $credspecSSMdoc --targets "Key=InstanceIds,Values=$firstInstanceId" --parameters "gMSAAccount=$gMSAAccountName,AdditionalCredSpecAccounts=" --query "Command.CommandId" --output text

# Wait for the success
aws ssm list-command-invocations --command-id $commandId

# Format the output and save it as a json file (credspec.json)
# If the json file is generated, save the content to credspec.json and format the json.
aws ssm get-command-invocation --command-id "$commandId" --instance-id "$firstInstanceId" --plugin-name "GenerateCredspec" --query "StandardOutputContent" --output json > credspec.json
 
##### ACTION REQUIRED - Format JSON - START #####
# Make sure credspec.json is a valid json content. 
# We still need to perform the following cleanups.
# Remove \r\n. 
# Remove the backslahes preceeding the double quote (Example: \" ==> ")
# Remove the " at the beginning and end of the file, if it's present.
##### ACTION REQUIRED - Format JSON - END #####
```

## Store Credspec
You can save the credspec file either locally in the instance (%programdata%\docker\credentialspecs\credspec.json) or in S3 or in SSM parameter. For the blog purpose, I'll store them in both SSM parameter and in S3.

```powershell
# SSM
aws ssm put-parameter --name $gMSACredSpecParam  --type String --value file://credspec.json
$ssmCredSpecARN = aws ssm get-parameter --name $gMSACredSpecParam --query "Parameter.ARN" --output text

# S3
aws s3 cp ./credspec.json s3://$gMSACredSpecS3bucket
```

## Create SQL password as secret
```powershell
aws ssm put-parameter --name $gMSASQLPasswordParam --type SecureString --value $sqlSAPassword
$sqlpasswordarn = aws ssm get-parameter --name $gMSASQLPasswordParam --query "Parameter.ARN" --output text
```

## Task Executio Role
```
$TaskExecutionPolicyContent = Get-Content -Path ./task-execution-role-policy.json | Foreach-Object {$_ -replace '\${CREDSPECARN}', $ssmCredSpecARN} | Foreach-Object {$_ -replace '\${SQLPASSWORDARN}', $sqlpasswordarn} |  Foreach-Object {$_ -replace '\${S3BUCKETNAME}', $gMSACredSpecS3bucket}
$TaskPolicyArn = aws iam create-policy --policy-name "$gMSATaskExecutionRole-policy"  --policy-document ("$TaskExecutionPolicyContent" | ConvertTo-Json) --query "Policy.Arn"
$TaskExecutionRoleArn = aws iam create-role --role-name $gMSATaskExecutionRole --assume-role-policy-document file://task-trust-policy.json --query "Role.Arn"
aws iam attach-role-policy --role-name $gMSATaskExecutionRole --policy-arn $TaskPolicyArn
```

### Deploy Applications
```powershell
$containerdef = Get-Content -Path ./sql-task-definition.json | Foreach-Object {$_ -replace '\${NETBIOSNAME}', $adDirectoryShortName} | Foreach-Object {$_ -replace '\${GMSAACCOUNT}', $gMSAAccountName} | Foreach-Object {$_ -replace '\${CREDSPECARN}', $ssmCredSpecARN} | Foreach-Object {$_ -replace '\${GMSASQLSASECRETPARAM}', $sqlpasswordarn}

$containerdef | Set-Content "temp.json"

$TaskDef = aws ecs register-task-definition --execution-role-arn $TaskExecutionRoleArn --cli-input-json file://temp.json | ConvertFrom-Json

$task = aws ecs run-task --cluster $ECSClusterName --launch-type EC2  --task-definition "$($TaskDef.taskdefinition.family):$($TaskDef.taskdefinition.revision)" | ConvertFrom-Json

# Keep on running describe task until the task reaches to RUNNING state.
$runningTask =  aws ecs describe-tasks  --cluster $ECSClusterName --tasks $task.tasks.taskarn | ConvertFrom-Json

Write-Output ("SQL Task status: {0}" -f $runningTask.tasks.lastStatus)

# If ths status is running then proceed. Otherwise investigate the failure form ECS console.
$ec2instanceId = aws ecs describe-container-instances --cluster $ECSClusterName --container-instances $runningTask.tasks.containerinstancearn --query "containerInstances[0].ec2InstanceId" --output text

$sqlserverIP = aws ec2 describe-instances --instance-ids $ec2instanceId --query "Reservations[0].Instances[0].PrivateIpAddress" --output text

$containerdef = Get-Content -Path ./mvc-task-definition.json | Foreach-Object {$_ -replace '\${CREDSPECARN}', $ssmCredSpecARN} | Foreach-Object {$_ -replace '\${SQLSERVER}', $sqlserverIP}

$containerdef | Set-Content "temp.json"

$TaskDef = aws ecs register-task-definition --execution-role-arn $TaskExecutionRoleArn --cli-input-json file://temp.json | ConvertFrom-Json

$task = aws ecs run-task --cluster $ECSClusterName --launch-type EC2  --task-definition "$($TaskDef.taskdefinition.family):$($TaskDef.taskdefinition.revision)" | ConvertFrom-Json

# Keep on running describe task until the task reaches to RUNNING state.
$runningTask =  aws ecs describe-tasks  --cluster $ECSClusterName --tasks $task.tasks.taskarn | ConvertFrom-Json

Write-Output ("MVC Task status: {0}" -f $runningTask.tasks.lastStatus)

# If ths status is running then proceed. Otherwise wait for task to reach running / investigate the failure form ECS console.
$ec2instanceId = aws ecs describe-container-instances --cluster $ECSClusterName --container-instances $runningTask.tasks.containerinstancearn --query "containerInstances[0].ec2InstanceId" --output text

$externalIP = aws ec2 describe-instances --instance-ids $ec2instanceId --query "Reservations[0].Instances[0].PublicIpAddress" --output text

Write-Output ("MVC application URL : {0}:443" -f $externalIP)

##### ACTION REQUIRED - START #####
# Access the containermvc externalIP:443 in browser.
##### ACTION REQUIRED - END #####
```