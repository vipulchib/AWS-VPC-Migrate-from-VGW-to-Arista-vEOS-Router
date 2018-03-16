# Migrate an AWS VPC utilizing VGW to Arista vEOS Router 

# Overview
Purpose of this post is to outline the steps involved in migration of an AWS VPC running VGW for VPN termination / connectivit to Arista vEOS Router.

When utilizing the AWS VGW we are employing IPSec and VTI on the Arista vEOS Router running in the Transit VPC router as shown in the image below.  Our goal is to migrate from this setup:
![AWS-VPC-with-VGW](https://github.com/vipulchib/AWS-VPC-Migrate-from-VGW-to-Arista-vEOS-Router/blob/master/AWS-VPC-with-VGW.png)

To this setup:
![AWS-VPC-with-Arista](https://github.com/vipulchib/AWS-VPC-Migrate-from-VGW-to-Arista-vEOS-Router/blob/master/AWS-VPC-with-Arista.png)

# What is CloudFormation and creation of templates in YAML
I have broken down every section of the template and provided my thought process with regards to what I am doing and how in another post - https://github.com/vipulchib/aws-cloudformation

# Build VPC-1 with AWS VGW for VPN termination
In this exercise it is assumed that the Transit VPC is already provisioned with an Arista vEOS Router.  I will demonstrate the 
following in order:

1.  Creation of VPC-1 using the AWS CLI and a CloudFormation stack using the YAML file that comprises of the Parameters and 
     the following Resouces: 
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
     configuration.  Prior to generating this config you will need to 'Download Configuration' from the 'VPN Connections' 
     portion of AWS's VPC Dashboard.  Download this file as the 'Generic' vendor and save the following which will enable 
     constructing the EOS config: **Pre-Shared Key, Outside Virtual Private Gateway IP, Inside IP Address of the Customer 
     Gateway and the Virtual Private Gateway IP**.

     ```
     hostname Arista-Transit
     !
     interface Ethernet1
        mtu 9001
        no switchport
        ip address 10.100.1.6/24
     !
     interface Ethernet2
        mtu 9001
        no switchport
        ip address 10.100.11.6/24
     !
     interface Tunnel10
        mtu 1436
        ip address 169.254.60.182/30
        tunnel mode ipsec
        tunnel source 10.100.11.6
        tunnel destination 52.60.137.228 --> Outside Virtual Private Gateway IP
        tunnel mss ceiling 1379
        tunnel ipsec profile AWS-profile1
     !
     router bgp 65100
        neighbor 169.254.60.181 peer-group edge-routers --> Inside IP Address of the Virtual Private Gateway
        neighbor 169.254.60.181 remote-as 65102
        ip security
     !
     ike policy AWS-IKE1
        integrity sha1
        version 1
        local-id 35.182.194.13
     !
     sa policy AWS-SA1
        esp encryption aes128
        esp integrity sha1
        pfs dh-group 14
     !
     profile AWS-profile1
       ike-policy AWS-IKE1
       sa-policy AWS-SA1
       connection start
       shared-key aNfcMGMqW8FLjtC4mYs0cgiE1x2sdSk8  --> Pre-Shared Key
     ```
3.  We will do a simple connectivity and throughput test between Host-1a in VPC-1 and Host-Transit in the Transit VPC: 
     - Iperf3 Server running on Host-Transit.
     ```
     [ec2-user@ip-10-100-11-10 ~]$ iperf3 -s
     -----------------------------------------------------------
     Server listening on 5201
     -----------------------------------------------------------
     Accepted connection from 10.1.11.10, port 51208
     [  5] local 10.100.11.10 port 5201 connected to 10.1.11.10 port 51210
     [ ID] Interval           Transfer     Bandwidth
     [  5]   0.00-1.00   sec  34.8 MBytes   292 Mbits/sec
     [  5]   1.00-2.00   sec  37.2 MBytes   312 Mbits/sec
     [  5]   1.00-2.00   sec  37.2 MBytes   312 Mbits/sec
     ```  
     - Iperf3 Client running on Host-1a.
     ```
     [ec2-user@ip-10-1-11-10 ~]$ iperf3 -c 10.100.11.10
     Connecting to host 10.100.11.10, port 5201
     [  4] local 10.1.11.10 port 51210 connected to 10.100.11.10 port 5201
     [ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
     [  4]   0.00-1.00   sec  37.1 MBytes   311 Mbits/sec   29    168 KBytes
     [  4]   1.00-2.00   sec  36.9 MBytes   310 Mbits/sec   33    168 KBytes
     [  4]   2.00-2.79   sec  31.2 MBytes   333 Mbits/sec   18    214 KBytes
     ```

# Migrating the VPC from VGW to Arista vEOS Router
I will focus on how to migrate an existing VPC-1 with AWS VGW to Arista vEOS Router.

1.  In this exercise we will update the CloudFormation Stack we previously created 'AristaVPCStack' using the AWS CLI and 
referencing a new  CloudFormation YAML file that comprises of the Parameters and the following Resouces: 
     - Transit VPC facing Subnets.
     - Route Tables with the new Subnet Associations.
     - A VPC Peering connection between the Transit-VPC and VPC-1.
     - Routing entry for Transit VPC facing subnets in VPC-1's connectivity to the vEOS Router in Transit VPC via previously 
     created VPC Peering connection.
     - Routing entries for VPC-1 facing subnet in the Transit VPC's connectivity to the vEOS Router in VPC-1 via previously 
     created VPC Peering connection.
     - vEOS Router instances, their respective Interfaces and Elastic IP's for reachability from the Internet.
     - Remove and clean the following:
          - The Customer Gateway.
          - The VPN Gateway and its attachment to the VPC.
          - Disable Route Propagation from the VPN Gateway to the Subnets in the VPC
          - Lastly, delete the previously created VPN connection to the Transit VPN Arista vEOS Router.
     
     Here is the AWS CLI command referencing the YAML file:
     ```
     aws cloudformation update-stack --stack-name AristaVPCStack --template-body file://Edge1-VPC-updated-with-vEOS-CF.yaml
     ```
3.  We will do a simple connectivity and throughput test between Host-1a in VPC-1 and Host-Transit in the Transit VPC.  This 
     traffic should now traverse the VPC Peering connection without IPSec and across GRE tunnels: 
     - Iperf3 Server running on Host-Transit.
     ```
     [ec2-user@ip-10-100-11-10 ~]$ iperf3 -s
     -----------------------------------------------------------
     Server listening on 5201
     -----------------------------------------------------------
     Accepted connection from 10.1.11.10, port 51314
     [  5] local 10.100.11.10 port 5201 connected to 10.1.11.10 port 51316
     [ ID] Interval           Transfer     Bandwidth
     [  5]   0.00-1.00   sec   111 MBytes   935 Mbits/sec
     [  5]   1.00-2.00   sec   118 MBytes   994 Mbits/sec
     [  5]   2.00-3.00   sec   118 MBytes   989 Mbits/sec
     ```  
     - Iperf3 Client running on Host-1a.
     ```
     [ec2-user@ip-10-1-11-10 ~]$ iperf3 -c 10.100.11.10
     Connecting to host 10.100.11.10, port 5201
     [  4] local 10.1.11.10 port 51316 connected to 10.100.11.10 port 5201
     [ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
     [  4]   0.00-1.00   sec   119 MBytes   996 Mbits/sec   17    549 KBytes
     [  4]   1.00-2.00   sec   119 MBytes   995 Mbits/sec   31    819 KBytes
     [  4]   2.00-3.00   sec   117 MBytes   985 Mbits/sec   24   1019 KBytes
     ```
