# Detecting stack drift with CloudFormation

## Overview
This is a demo of how to create a stack using AWS CloudFormation, detect drift in the stack, and perform a stack update. The demo task is creating an environment for a dev team, who asked for an Apache server with HTTP access. The stack consists of the following:
- A dedicated VPC
- Single public subnet
- Amazon EC2 instance

## Pre-requisites
- Have AWS CLI installed
- Have an AWS account
```bash
$ aws --version
$ aws-cli/2.17.18 Python/3.9.20 ...
```
## Details
### Step 1: Create the required parameters
Parameters are reusable inputs that allow flexibility when building IaC templates. They are popular to specify property values of stack resources.
```yaml
InstanceType:
  Description: Webserver EC2 instance type
  Type: String
  Default: t2.nano
  AllowedValues:
    - t2.nano
    - t2.micro
    - t2.small
  ConstraintDescription: must be a valid EC2 instance type.
```
The `ConstraintDescription` is interesting because it provides the user with details when a constraint is violated. In this case, if a user tries to create an invalid EC2 instance type, then they get a useful error message.

### Step 2: Adding resources to the CloudFormation template
Here the AWS resources that CloudFormation will provision are declared. Below is a demonstration of how to specify a route in a route table. The `Type` of this resource is `AWS::EC2::Route`.
```yaml
Route:
  Type: 'AWS::EC2::Route'
  DependsOn:
    - VPC
    - AttachGateway
  Properties:
    RouteTableId: !Ref RouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway
```
Note how the `Route` resource refers to other resources with the `!Ref` function. In this case, both `RouteTable` and `InternetGateway`.

### Step 3: Adding an output
Outputs are mainly used to capture important details about the resources in the stack, as they allow a convenient way to store the information in a separate file, or simply make later reference easier using the `aws cli` utility.
```yaml
Outputs:
  AppURL:
    Description: New created application URL
    Value: !Sub 'http://${WebServerInstance.PublicIp}'
```
In this case, we are using `!Sub` to dynamically insert the public IP of the EC2 instance, and get a full HTTP URL when the stack is provisioned. In other words, using the `!Sub` function will allow us to retrieve the URL to access the web server.

### Step 4: Create the stack
Once the `YAML` file is finished, it's simple to create the stack.
```bash
$ aws cloudformation create-stack --stack-name Demo_Web_Server --parameters ParameterKey=InstanceType,ParameterValue=t2.micro --template-body file://cf_stack.yaml
$ {
    "StackId": "arn:aws:cloudformation:....."
}
```
It might be useful to query the status of the stack process with the following command.
```bash
$ aws cloudformation describe-stacks --stack-name Demo_Web_Server --query "Stacks[0].StackStatus"
$ "CREATE_COMPLETE" # This is the desired output
```
**Stack creation complete**
(https://github.com/bernie-cm/cloudformation_lab/blob/main/assets/20250308_cloudformation_stack_created.png)
**INSERT PHOTO OF THE OUTPUT CREATED IN THE STACK**

### Step 5: Testing drift detection in a CloudFormation stack
CloudFormation is powerful because it can be used to detect stack changes **not initiated within CloudFormation**. In other words, if someone were to make changes to the stack using the AWS Console, CloudFormation can be used to detect and rectify those changes.
First, it's necessary to run the 'Detect Drift' stack action, and once that's run, select 'View drift results'.
**INSERT PHOTO OF STACK ACTIONS**
**INSERT PHOTO OF DRIFT REPORT**
You can also detect drift via the CLI.
```bash
$ aws cloudformation describe-stack-resource-drifts ...
```

### Step 6: Rectify the stack using a change set
Once drift has been detected, the resource can be modified to the expected value in the CloudFormation template. To do this, we can use a Change Set, and then implement the changes to the environment.
```bash
$ aws cloudformation create-change-set --stack-name Demo_WebServer --change-set-name Demo_Change_set --parameters ParameterKey=InstanceyType,ParameterValue=t2.micro, --template-body file://cf_stack-CS.yaml
```
Once the Change Set has been created, it will appear under the original stack. The next step would be to **Execute change set** and if the rollback is successful the Change Set is no longer available.
