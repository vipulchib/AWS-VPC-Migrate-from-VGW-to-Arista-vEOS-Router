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

|   |   |   |   |   |
|---|---|---|---|---|
|   |   |   |   |   |
|   |   |   |   |   |
|   |   |   |   |   |
     
