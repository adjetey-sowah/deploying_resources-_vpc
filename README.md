# Securely deploying resources in a VPC using cfn LAB
Securely deploying resources in a VPC using cfn LAB
# AWS VPC Lab with SSM-Enabled Instances  
**CloudFormation Template for Secure Infrastructure Deployment**  

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [CloudFormation Template](#cloudformation-template)
3. [Features](#features)
4. [Prerequisites](#prerequisites)
5. [Deployment Guide](#deployment-guide)
6. [Usage Instructions](#usage-instructions)
7. [Testing Connectivity](#testing-connectivity)
8. [Customization Options](#customization-options)
9. [Security Best Practices](#security-best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Cleanup](#cleanup)
12. [License](#license)

---

## Architecture Overview  
![VPC Architecture Diagram](https://via.placeholder.com/800x500.png?text=VPC+with+Public/Private+Subnets,+NAT+Gateway,+and+SSM+VPC+Endpoint)  

**Components**:  
- **VPC** (10.0.0.0/16) with public/private subnets  
- **Internet Gateway** for public subnet access  
- **NAT Gateway** for private subnet internet  
- **EC2 Instances** with SSM access  
- **VPC Endpoint** for private SSM connectivity  

---

## CloudFormation Template
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Secure VPC with Public/Private Subnets and SSM-Enabled Instances

Parameters:
  VpcCIDR:
    Description: "CIDR block for the VPC"
    Type: String
    Default: "10.0.0.0/16"

  PublicSubnet1CIDR:
    Description: "Public Subnet AZ1"
    Type: String
    Default: "10.0.1.0/24"

  PrivateSubnet1CIDR:
    Description: "Private Subnet AZ1"
    Type: String
    Default: "10.0.3.0/24"

  YourName:
    Description: "Your name for the web server"
    Type: String
    Default: "John Doe"

  KeyName:
    Description: "Optional EC2 Key Pair"
    Type: String
    Default: ""

  LatestAmiId:
    Type: "AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>"
    Default: "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"

Conditions:
  HasKeyName: !Not [!Equals [!Ref KeyName, ""]]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: "SSM-VPC-Lab"

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: "SSM-VPC-IGW"

  GatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PublicSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: "Public-Subnet-AZ1"

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Ref PrivateSubnet1CIDR
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: "Private-Subnet-AZ1"

  NatEIP:
    Type: AWS::EC2::EIP
    DependsOn: GatewayAttachment

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatEIP.AllocationId
      SubnetId: !Ref PublicSubnet1

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: "Public-RouteTable"

  PublicRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref InternetGateway

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: "Private-RouteTable"

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      NatGatewayId: !Ref NatGateway

  PublicSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow HTTP access"
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  PrivateSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Private instance SG"
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: "-1"
          CidrIp: 0.0.0.0/0

  SSMInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [!Ref SSMInstanceRole]

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref PublicSubnet1
      SecurityGroupIds: [!Ref PublicSG]
      IamInstanceProfile: !Ref InstanceProfile
      KeyName: !If [HasKeyName, !Ref KeyName, !Ref "AWS::NoValue"]
      UserData: 
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Hello from ${YourName}!</h1>" > /var/www/html/index.html

  PrivateInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId
      SubnetId: !Ref PrivateSubnet1
      SecurityGroupIds: [!Ref PrivateSG]
      IamInstanceProfile: !Ref InstanceProfile

  SSMEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ssm"
      VpcEndpointType: Interface
      SubnetIds: [!Ref PrivateSubnet1]
      SecurityGroupIds: [!Ref PrivateSG]

Outputs:
  WebServerPublicIP:
    Value: !GetAtt WebServer.PublicIp
  SSMConnectCommand:
    Value: !Sub "aws ssm start-session --target ${PrivateInstance}"
```

---

## Features  
- ✅ Auto-scaling ready VPC infrastructure  
- ✅ Secure SSH-less access via AWS Systems Manager  
- ✅ NAT Gateway for private subnet internet  
- ✅ Dynamic AMI selection using SSM Parameter Store  

---

## Prerequisites  
1. **AWS Account** with permissions for:
   - VPC/EC2/IAM resource creation  
   - SSM Session Manager access  
2. **AWS CLI** installed and configured

---

## Deployment Guide  
```bash
# Deploy with CLI
aws cloudformation deploy \
  --template-file vpc-lab.yaml \
  --stack-name vpc-lab \
  --parameter-overrides YourName="YOUR_NAME" \
  --capabilities CAPABILITY_IAM

# Expected Outputs:
# WebServerPublicIP: <public-ip>
# SSMConnectCommand: aws ssm start-session --target <instance-id>
```

---

## Usage Instructions  
**Access Web Server**:  
`http://<WebServerPublicIP>`  

**SSM Connection Commands**:  
```bash
# Public Instance
aws ssm start-session --target $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Public-WebServer" --query "Reservations[].Instances[].InstanceId" --output text)

# Private Instance
aws ssm start-session --target $(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=Private-Instance" --query "Reservations[].Instances[].InstanceId" --output text)
```

---

## Testing Connectivity  
**From Web Server**:  
```bash
ping <private-instance-ip>
curl -I https://google.com
```

**From Private Instance**:  
```bash
ping 8.8.8.8
curl -I https://amazon.com
```

---

## Security Best Practices  
1. Use SSM instead of SSH keys  
2. Private instances have no public IPs  
3. Least-privilege IAM roles  
4. VPC endpoints for AWS services  

---

## Troubleshooting  
| Error | Solution |
|-------|----------|
| `Key pair does not exist` | Leave KeyName blank or create key pair |
| `CREATE_FAILED (PrivateInstance)` | Use t3.micro instance type |
| `100% ping loss` | Target may block ICMP - use `curl` |

---

## Cleanup  
```bash
aws cloudformation delete-stack --stack-name vpc-lab
```

---

## License  
MIT License - [Full License Text](LICENSE)
