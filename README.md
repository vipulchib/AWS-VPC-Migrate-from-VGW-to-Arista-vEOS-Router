# Migrate an AWS VPC utilizing VGW to Arista vEOS Router 

# Overview
Purpose of this post is to outline the steps involved in migration of an AWS VPC running VGW for VPN termination / connectivit to Arista vEOS Router.

When utilizing the AWS VGW we are employing IPSec and VTI on the Arista vEOS Router running in the Transit VPC router as shown in the image below.  Our goal is to migrate from this setup:
![AWS-VPC-with-VGW](https://github.com/vipulchib/AWS-VPC-Migrate-from-VGW-to-Arista-vEOS-Router/blob/master/AWS-VPC-with-VGW.png)

To this setup:
![AWS-VPC-with-Arista](https://github.com/vipulchib/AWS-VPC-Migrate-from-VGW-to-Arista-vEOS-Router/blob/master/AWS-VPC-with-Arista.png)

# What is CloudFormation and creation of templates in YAML
I have broken down every section of the template and provided my thought process with regards to what I am doing and how in another post - https://github.com/vipulchib/aws-cloudformation

# Migrating the VPC from VGW to Arista vEOS Router
I will focus on how to migrate an existing VPC with AWS VGW to Arista vEOS Router.  In this exercise it is assumed that the Transit VPC is already provisioned with an Arista vEOS Router.  I will demonstrate the following in order:

1.  Creation of VPC-1 using the AWS CLI and a CloudFormation stack using the YAML file that comprises of the Parameters and the following Resouces: 
     - VPC.
     - Security Groups.
     - Internet Gateway.
     - Subnets that the hosts would reside in the VPC.
     - Route Tables with Subnet Associations.
     - Routing entries for Internet and connectivity to the host in Transit VPC.
     - EC2 instances for hosts in this VPC.
     - Elastic IP for these EC2 host instances.
     - A Customer Gateway, which will be the Transit VPC Arista vEOS Router we will establish an IPSec tunnel with.
     - A VPN Gateway for the VPC with a custom BGP ASN. Since the Arista vEOS Router in the Transit VPC is able to support  
     Border Gateway Protocol (BGP), we will specify dynamic routing wwhile configuring this section.
     - Attach that VPN Gateway for the VPC.
     - Enable Route Propagation from the VPN Gateway to the Subnets in the VPC for reachability to the Transit VPC Subnets.
     - Lastly, a VPN connection to the Transit VPN Arista vEOS Router.
     
     Here is the AWS CLI command referencing the YAML file:
     ```
     aws cloudformation create-stack --stack-name AristaVPCStack --template-body file://Edge1-VPC-with-VGW-CF.yaml
     ```

2.  Transit VPC Arista vEOS Router will be configured with the following EOS config, which includes IPSec and BGP
     configuration: 


## Building the Cloudformation YAML template.

1. **AWSTemplateFormatVersion: "version date"** - The AWS CloudFormation template version that the template conforms to.
     ```
     AWSTemplateFormatVersion: '2010-09-09'
     ```

2. **Description** - A text string that describes the template.
     ```
     Description: VPC with Arista vEOS Router, subnets, route tables, igw, sg, Linux VMs
     ```

3. **Parameters** - Parameters enable you to input custom values to your template each time you create or update a stack.  We will create the following:

   A. A Parameters for the VPC ID, provide a Description (VPC ID) and then specify the Type of Parameter 
   with a 'Value (default)' as **Arista**
     
   B. A Parameters for the VPC CIDR, provide a Description (VPC Supernet) and then specify the Type of 
   Parameter with a 'Value (default)' as **10.1.0.0/16**

   C. A Parameters for the 1st Subnet in the VPC, provide a Description (VPC Subnet A-1) and then specify the 
   Type of Parameter with a 'Value (default)' as **10.1.1.0/24**
     
   D. A Parameters for the 2nd Subnet in the VPC, provide a Description (VPC Subnet A-2) and then specify the 
   Type of Parameter with a 'Value (default)' as **10.1.11.0/24**
     
   ```
   Parameters: 
    ID:
      Description: Arista VPC ID
      Type: String
      Default: Arista
    VPCCidr: 
      Description: VPC Supernet
      Type: String
      Default: 10.1.0.0/16
    SubnetA1Cidr: 
      Description: VPC Subnet A-1
      Type: String
      Default: 10.1.1.0/24
    SubnetA2Cidr: 
      Description: VPC Subnet A-2
      Type: String
      Default: 10.1.11.0/24
   ```

4. **Resources** -  The required Resources section declares the AWS resources that you want to include in the stack, such as 
     an Amazon EC2 instance.  We will create the following:
     
     A. A Resource for VPC creation and name it **AristaVPC**.  For *'CidrBlock'* section of the Properties we 
     will reference the *'VPCCidr'* Parameter we previously defined.  VPC will be named as *'VPC-Arista'*  but *'Arista'* 
     portion of VPC's name will be derived by referencing the VPC ID we defined in the Parameters section.
      ```
      AristaVPC:
       Type: AWS::EC2::VPC
       Properties:
        CidrBlock: !Ref VPCCidr
        Tags:
          - Key: Name
            Value: !Sub
            - VPC-${ID}
            - {ID: !Ref ID}
      ```
     B. A Resource for Security Group and name it **AristaSecurityGroup**.  For *'VpcId'* section of the 
     Properties we will reference the *'AristaVPC'* Parameter we previously defined.  In the following example we are 
     allowing SSH (TCP port 22) and ICMP from 'everywhere' (0.0.0.0/0).  The Security Group will be named as *'VPC-Arista-
     SecurityGroup'* but *'Arista'* portion of VPC's name will be derived by referencing the VPC ID we defined in the 
     Parameter section.
      ```
      AristaSecurityGroup:
       Type: AWS::EC2::SecurityGroup
       Properties:
        GroupName: Arista VPC Security Group
        GroupDescription: Allow SSH and ICMP from Everywhere
        VpcId: !Ref 'AristaVPC'
        SecurityGroupIngress:
         - IpProtocol: tcp
           FromPort: '22'
           ToPort: '22'
           CidrIp: 0.0.0.0/0
         - IpProtocol: icmp
           FromPort: -1
           ToPort: -1
           CidrIp: 0.0.0.0/0
        Tags:
        - Key: Name
          Value: !Sub
          - VPC-${ID}-SecurityGroup
          - {ID: !Ref ID}
      ```
     C. A Resource for the Internet Gateway and name it **AristaVPCIGW**. The Internet Gateway will be named as 
     *'VPC-Arista-IGW'* but *'Arista'* portion of VPC's name will be derived by referencing the VPC ID we defined in the 
     Parameter section.
      ```
      AristaVPCIGW:
       Type: AWS::EC2::InternetGateway
       Properties:
        Tags:
         - Key: Name
           Value: !Sub
           - VPC-${ID}-IGW
           - {ID: !Ref ID}
      ```
     D. A Resource to Attach the Internet Gateway and name it **AttachIGW**.  For *'VpcId'* section of the 
     Properties we will reference the *'AristaVPC'* Parameter we previously defined.  Similarly for *'InternetGatewayId'* 
     section of the Properties we will reference the *'AristaVPCIGW'* Parameter we previously defined.
      ```
      AttachIGW:
       Type: AWS::EC2::VPCGatewayAttachment
       Properties:
        VpcId: !Ref 'AristaVPC'
        InternetGatewayId: !Ref 'AristaVPCIGW'
      ```
     E. A Resource for the 1st Subnet and name it **subnetA1**.
      ```
      subnetA1:
       Type: AWS::EC2::Subnet
       Properties:
        VpcId: !Ref 'AristaVPC'
        CidrBlock: !Ref SubnetA1Cidr
        AvailabilityZone: ca-central-1a
        MapPublicIpOnLaunch: false
        Tags:
         - Key: Name
           Value: !Ref SubnetA1Cidr
      ```
     F. A Resource for the 1st Subnet Routing Table and name it **subnetA1RT**
      ```
      subnetA1RT:
       Type: AWS::EC2::RouteTable
       Properties:
        VpcId: !Ref 'AristaVPC'
        Tags:
         - Key: Name
           Value: !Ref SubnetA1Cidr
      ```
     G. A Resource for Associating the 1st Subnet to Routing Table and name it **subnetA1RTAssoc**.
      ```
      subnetA1RTAssoc:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
        SubnetId: !Ref subnetA1
        RouteTableId: !Ref subnetA1RT  
      ```
     H. A Resource for the 2nd Subnet and name it **subnetA2**.
      ```
      subnetA2:
       Type: AWS::EC2::Subnet
       Properties:
        VpcId: !Ref 'AristaVPC'
        CidrBlock: !Ref SubnetA2Cidr
        AvailabilityZone: ca-central-1a
        MapPublicIpOnLaunch: false
        Tags:
         - Key: Name
           Value: !Ref SubnetA2Cidr
      ```
     I. A Resource for the 2nd Subnet Routing Table and name it **subnetA2RT**
      ```
      subnetA2RT:
       Type: AWS::EC2::RouteTable
       Properties:
        VpcId: !Ref 'AristaVPC'
        Tags:
         - Key: Name
           Value: !Ref SubnetA2Cidr
      ```
     J. A Resource for Associating the 2nd Subnet to Routing Table and name it **subnetA2RTAssoc**.
      ```
      subnetA2RTAssoc:
       Type: AWS::EC2::SubnetRouteTableAssociation
       Properties:
        SubnetId: !Ref subnetA2
        RouteTableId: !Ref subnetA2RT  
      ```
     K. A Resource for Route for 10.0.0.0/8 to point to *'vEOSIntfSubnetA2'* and name it **privateRouteA2**.
      ```
      privateRouteA2:
       Type: AWS::EC2::Route
       DependsOn: vEOSIntfSubnetA2
       Properties:
        RouteTableId: !Ref 'subnetA2RT'
        DestinationCidrBlock: 10.0.0.0/8
        NetworkInterfaceId: !Ref 'vEOSIntfSubnetA2' 
      ```     
     L. A Resource for the default Route for 0.0.0.0/0 to point to *'AristaVPCIGW'* and name it **publicRouteA2**.
      ```
      publicRouteA2:
       Type: AWS::EC2::Route
       DependsOn: AttachIGW
       Properties:
        RouteTableId: !Ref 'subnetA2RT'
        DestinationCidrBlock: 0.0.0.0/0
        GatewayId: !Ref 'AristaVPCIGW'
      ``` 
     M. A Resource for Compute Instance's Interface and name it **VMIntf1a**.
      ```
      VMIntf1a:
      Type: AWS::EC2::NetworkInterface
      Properties:
       SourceDestCheck: false
       SubnetId: !Ref 'subnetA2'
       GroupSet:
       - !Ref AristaSecurityGroup
       PrivateIpAddress: 10.1.11.10
       Tags:
        - Key: Name
          Value: !Sub
          - ${ID}-Host-ETH0
          - {ID: !Ref ID}
      ```  
     N. A Resource to Create and Associate an Elastic IP to the previously created Compute Instance's Interface and name it 
     **VMIntf1aEIP**.
      ```
      VMIntf1aEIP:
       Type: AWS::EC2::EIP
       Properties:
        Domain: vpc
      AssociateVMIntf1aEIP:
       Type: AWS::EC2::EIPAssociation
       Properties:
        AllocationId: !GetAtt VMIntf1aEIP.AllocationId
        NetworkInterfaceId: !Ref VMIntf1a
      ```  
     O. A Resource for the Compute Instance and name it **Ec2Instance1a**.  Note that we are referencing a pre-defined Key 
     named *'Arista-vEOS-Router'* while creating this instance.  Ensure that this is key is created and you have downloaded it 
     to be able to access the instance upon creation.
      ```
      Ec2Instance1a:
       Type: AWS::EC2::Instance
       DependsOn: AristaVPCIGW
       Properties:
        AvailabilityZone: ca-central-1a
        ImageId: ami-a954d1cd
        InstanceType: t2.micro
        KeyName: Arista-vEOS-Router
        UserData: 
         Fn::Base64: !Sub | 
          #!/bin/bash -xe 
          yum update -y 
          yum --enablerepo=epel install -y iperf iperf3
         NetworkInterfaces:
          - NetworkInterfaceId: !Ref VMIntf1a
            DeviceIndex: 0
         Tags:
          - Key: Name
            Value: !Sub
            - ${ID}-Host
            - {ID: !Ref ID}
      ```  
     P. A Resource for the Arista vEOS Router's Interfaces and name them **vEOSIntfSubnetA1** and **vEOSIntfSubnetA2**.
      ```
      vEOSIntfSubnetA1:
       Type: AWS::EC2::NetworkInterface
       Properties:
        SourceDestCheck: false
        SubnetId: !Ref 'subnetA1'
        GroupSet:
        - !Ref AristaSecurityGroup
        PrivateIpAddress: 10.1.1.6
        Tags:
         - Key: Name
           Value: !Sub
           - vRouter-${ID}-ETH1
           - {ID: !Ref ID}
      vEOSIntfSubnetA2:
       Type: AWS::EC2::NetworkInterface
       Properties:
        SourceDestCheck: false
        SubnetId: !Ref 'subnetA2'
        GroupSet:
        - !Ref AristaSecurityGroup
        PrivateIpAddress: 10.1.11.6
        Tags:
         - Key: Name
           Value: !Sub
           - vRouter-${ID}-ETH2
           - {ID: !Ref ID}
      ``` 
     Q. A Resource to Create and Associate an Elastic IP to the previously created Arista vEOS Router's Ethernet2 Interface 
     and name it **vEOSIntfSubnetA2EIP**.
      ```
      vEOSIntfSubnetA2EIP:
       Type: AWS::EC2::EIP
       Properties:
        Domain: vpc
      AssociatevEOSIntfSubnetA2EIP:
       Type: AWS::EC2::EIPAssociation
       Properties:
        AllocationId: !GetAtt vEOSIntfSubnetA2EIP.AllocationId
        NetworkInterfaceId: !Ref vEOSIntfSubnetA2
      ``` 
     R. A Resource to create the Arista vEOS Router instance and name it **AristavEOSRouter**.  Note that we are referencing a 
     pre-defined Key named *'Arista-vEOS-Router'* while creating this instance.  Ensure that this is key is created and you 
     have downloaded it to be able to access the instance upon creation.
      ```
      AristavEOSRouter:
       Type: AWS::EC2::Instance
       DependsOn: AristaVPCIGW
       Properties:
        AvailabilityZone: ca-central-1a
        ImageId: ami-42922926
        InstanceType: t2.medium
        KeyName: Arista-vEOS-Router
        UserData: 
         Fn::Base64: "%EOS-STARTUP-CONFIG-START%\nhostname Arista-vRouter\ninterface Ethernet1\nmtu 9001\nno switchport\nip 
         address 10.1.1.6/24\ninterface Ethernet2\nmtu 9001\nno switchport\nip address 10.1.11.6/24\nip route 0.0.0.0/0 
         Ethernet2 10.1.11.1\nip routing\n%EOS-STARTUP-CONFIG-END%\n"
        NetworkInterfaces:
        - NetworkInterfaceId: !Ref vEOSIntfSubnetA1
          DeviceIndex: 0
        - NetworkInterfaceId: !Ref vEOSIntfSubnetA2
          DeviceIndex: 1
        Tags:
         - Key: Name
           Value: !Sub
           - ${ID}-vRouter
           - {ID: !Ref ID}
      ``` 
# Building a Stack
We will build the Stack using the YAML file that comprises of the Parameters and Resources we defined in the previous section.  We will use the following AWS CLI command to launch the stack:
```
aws cloudformation create-stack --stack-name AristaVPCStack --template-body file://AristaVPC.yaml
```
