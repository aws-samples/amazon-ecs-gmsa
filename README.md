## Run ECS Windows Tasks with group Managed Service Account (gMSA)

This repository contains cloudformation templates, powershell scripts, task definitions and sample applications required to set up AWS managed Active Directory and gMSA account setup to demonstrate gMSA end-to-end workflow with Amazon Elastic Container Services (ECS).

# Prerequisites
* AWS CLI
* Powershell core
* ECS Cluster where AWS Managed AD is available

**Clone the repo and execute the commands from the respective directories**

# 1. Infrastructure Setup
Follow the instructions from [./amazon-ecs-gmsa/cloud-formation-templates/README.md](https://github.com/aws-samples/amazon-ecs-gmsa/blob/master/cloud-formation-templates/README.md) to setup infrastructure required to demonstrate ECS gMSA. This step will install following resources.
* Customer Master Key
* Customer Master Key IAM Policy
* SSM Parameters
* AWS Managed Active Directory (AD)
* SSM document to join a AWS managed AD
* SSM document to generate a gMSA account and credential spec content

# 2. Launch ECS Windows Workers with AD Domain Join
Follow the [ECS documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Windows_getting_started.html) to launch ECS Windows worker. 
Follow the instructions [./amazon-ecs-gmsa/ecs-deployments/instance-domain-join.md](https://github.com/aws-samples/amazon-ecs-gmsa/blob/master/ecs-deployments/instance-domain-join.md) to join AD Domain
* Attach Customer Master Key IAM Policy to ECS Windows Instances
* Attach Domain Join SSM document to ECS Windows Autoscaling group

# 3. Create Sample Container images
Follow the instructions from [./amazon-ecs-gmsa/sample-applications/README.md](https://github.com/aws-samples/amazon-ecs-gmsa/blob/master/sample-applications/README.md)

# 4. ECS Deployment
Follow the instructions from [./amazon-ecs-gmsa/ecs-deployments/README.md](https://github.com/aws-samples/amazon-ecs-gmsa/blob/master/ecs-deployments/README.md) to deploy the following resources.
* Create a new gMSA Account 
* Store Credspec to S3 / SSM
* Deploy ECS Task definition with credspec

# 5. Troubleshooting
For troubleshooting, please follow the steps [here](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/gmsa-troubleshooting).
 
## License

This project is licensed under the MIT-0 License.
