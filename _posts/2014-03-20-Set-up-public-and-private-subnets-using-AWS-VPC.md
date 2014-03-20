---
layout: post
title:  "Set up public and private subnets using AWS VPC"
date:   2014-03-20 09:09:09
---

This is a step-by-step guide on how to set up public and private subnets for running a service on an internal network within AWS. This guide will also set up a bastion host (or jump host) and show you how you can easily ssh in to the hosts within your private subnet via the bastion host. All of this stuff can be done via the AWS web console, but I thought it would be helpful to show the specific commands and provide some commentary about what is happening on each step.

---

## Prerequisites
* Install and configure the [AWS CLI](http://aws.amazon.com/cli/)
* Set up at least one [EC2 Key Pair](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) that will be used to connect to your hosts
* The steps below assume you're using the bash shell

## Steps
1. __Create the VPC__. You can use a different CIDR range if you want, but it must be within the [RFC 1918](https://tools.ietf.org/html/rfc1918) range.

    ```sh
    export VPC_ID=`aws ec2 create-vpc --cidr-block '10.0.0.0/16' | grep VpcId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
    ```
2. __Create the internet gateway__. The internet gateway will be used within your VPC to allow your public subnet to have access to the Internet.

    ```sh
    export IGW_ID=`aws ec2 create-internet-gateway | grep InternetGatewayId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
3. __Attach the internet gateway to your VPC__.

    ```sh
    aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
    ```
4. __Create the public and private subnets__. The public subnet (10.0.0.0/24) will be used for services that need to be accessible from the internet and the private subnet (10.0.1.0/24) will be used for internal-only services.

    ```sh
    export PUBLIC_SUBNET_ID=`aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block '10.0.0.0/24' | grep SubnetId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    export PRIVATE_SUBNET_ID=`aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block '10.0.1.0/24' | grep SubnetId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
5. __Find the route table for your VPC__. A default route table was automatically created for you when you created your VPC.

    ```sh
    export MAIN_ROUTE_TABLE_ID=`aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" | grep RouteTableId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
6. __Create a route to the internet gateway for the public subnet__.

    ```sh
    aws ec2 create-route --route-table-id $MAIN_ROUTE_TABLE_ID --destination-cidr-block '0.0.0.0/0' --gateway-id $IGW_ID
    ```
7. __Associate your public subnet with the route table__.

    ```sh
    aws ec2 associate-route-table --route-table-id $MAIN_ROUTE_TABLE_ID --subnet-id $PUBLIC_SUBNET_ID
    ```
8. __Create the NAT host security group__. This security group will be the group that contains rules for your NAT host.

    ```sh
    export NATSG_ID=`aws ec2 create-security-group --group-name 'natsg' --description 'NAT security group for your bastion host.' --vpc-id $VPC_ID | grep GroupId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```

9. __Create a security group for your hosts in the private subnet__. In my case, I'll be running Solr in my private subnet so I'll name my security group 'solr'.

    ```sh
    export SERVICESG_ID=`aws ec2 create-security-group --group-name 'solr' --description 'Solr security group.' --vpc-id $VPC_ID | grep GroupId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
10. __Enable SSH inbound from the internet and HTTP & HTTPS inbound from the private subnet for the NAT security group__. NOTE: It's a good idea to run SSH on a port other than 22. Why? Bots on the Internet will constantly try to connect to port 22 looking for any known SSH vulnerabilities. Running sshd on a non-standard port doesn't increase your security per-se, but it reduces a large amount of botnet activity from showing up in your logs and gives you a fighting chance to patch any new known vulnerabilities before your host is exploited. Here I'm setting up support for both port 22 and port 20022 - we'll remove port 22 later after we update the sshd config on the bastion host later.

    ```sh
    aws ec2 authorize-security-group-ingress --group-id $NATSG_ID --protocol -1 --source-group $NATSG_ID
    aws ec2 authorize-security-group-ingress --group-id $NATSG_ID --protocol 'tcp' --port 20022 --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-ingress --group-id $NATSG_ID --protocol 'tcp' --port 22 --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-ingress --group-id $NATSG_ID --protocol 'tcp' --port 80 --source-group $SERVICESG_ID
    aws ec2 authorize-security-group-ingress --group-id $NATSG_ID --protocol 'tcp' --port 443 --source-group $SERVICESG_ID
    ```
11. __Restrict outbound protocols to only those needed__. Why? It's a security best practice and a PCI DSS requirement. If you want to increase security even further you can set the CIDR to restrict just those IP addresses that you need access to, like your source code repo, package management repo, time servers, etc. In this example I'm allowing HTTP, HTTPS, SSH and NTP to any IP address.

    ```sh
    aws ec2 revoke-security-group-egress --group-id $NATSG_ID --protocol '-1' --port all --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-egress --group-id $NATSG_ID --protocol 'tcp' --port 80 --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-egress --group-id $NATSG_ID --protocol 'tcp' --port 443 --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-egress --group-id $NATSG_ID --protocol 'tcp' --port 22 --cidr '0.0.0.0/0'
    aws ec2 authorize-security-group-egress --group-id $NATSG_ID --protocol 'udp' --port 123 --cidr '0.0.0.0/0'
    ```
12. __Allow users to SSH from the NAT host to the hosts in the 'solr' security group__. NOTE: For hosts on the private subnet I'm running SSH on the standard port (22), because they aren't accessible from the Internet.

    ```sh
    aws ec2 authorize-security-group-ingress --group-id $SERVICESG_ID --protocol -1 --source-group $SERVICESG_ID
    aws ec2 authorize-security-group-ingress --group-id $SERVICESG_ID --protocol 'tcp' --port 22 --source-group $NATSG_ID
    ```
13. __Create a new NAT instance__. This instance will serve two purposes: 1) act as an outbound NAT host so hosts within the private subnet can make requests to the Internet and 2) act as a bastion host (or jump host) so you can SSH in to hosts on your private subnet. In either case the load will probably be extremely light so a micro instance or small instance is probably fine. Amazon has some dedicated AMI's for this purpose, so you should go to the web console and search for 'VPC NAT' with the AMI search tool to find the latest AMI. In my case the latest is `ami-f032acc0` for the image with the name `amzn-ami-vpc-nat-pv-2013.09.0.x86_64-ebs`.

    ```sh
    export NAT_INSTANCE_ID=`aws ec2 run-instances --image-id ami-f032acc0 --count 1 --instance-type t1.micro --key-name MyKeyPair --security-group-ids $NATSG_ID --subnet-id $PUBLIC_SUBNET_ID --associate-public-ip-address --monitoring 'Enabled=true' | grep InstanceId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
14. __Remove the source/destination check on the NAT host__. Since you want the NAT host to route outbound traffic from the private subnet to the Internet, you'll need to disable the source/destination check for this host only so it will accept traffic originating from hosts other than itself.

    ```sh
    aws ec2 modify-instance-attribute --instance-id $NAT_INSTANCE_ID --no-source-dest-check
    ```
15. __Create a route table for the your private subnet__

    ```sh
    export CUSTOM_ROUTE_TABLE_ID=`aws ec2 create-route-table --vpc-id $VPC_ID | grep RouteTableId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
16. __Associate your private subnet with the route table__.

    ```sh
    aws ec2 associate-route-table --route-table-id $CUSTOM_ROUTE_TABLE_ID --subnet-id $PRIVATE_SUBNET_ID
    ```
17. __Create a default route from the private network to the NAT host__.

    ```sh
    aws ec2 create-route --route-table-id $CUSTOM_ROUTE_TABLE_ID --destination-cidr-block '0.0.0.0/0' --instance-id $NAT_INSTANCE_ID
    ```
18. __Create a new instance in your private subnet__. This step is optional - it's just to complete the example. Here I'll create an m1.small instance to test the connectivity.

    ```sh
    export PRIVATE_INSTANCE_ID=`aws ec2 run-instances --image-id ami-e04428d0 --count 1 --instance-type m1.small --key-name MyKeyPair --security-group-ids $SERVICESG_ID --subnet-id $PRIVATE_SUBNET_ID --monitoring 'Enabled=true' | grep InstanceId | head -1 | awk '{gsub(/\"/, "");gsub(/,/,""); print $2}'`
    ```
19. __Set up your ssh config file.__ On your local machine, you can set up an SSH config file that will allow you to SSH into the hosts in your private subnet via the bastion host (aka your NAT host). Edit or create the file `~/.ssh/config` and add the contents:

    ```
    Host bastion
      Hostname ec2-54-186-165-211.us-west-2.compute.amazonaws.com
      Port 22
      ProxyCommand none
      User ec2-user

    Host *.us-west-2.compute.internal
      User ubuntu
      ProxyCommand ssh -p 22 -W %h:%p ec2-user@ec2-54-186-165-211.us-west-2.compute.amazonaws.com
    ```
    Replace `ec2-54-186-165-211.us-west-2.compute.amazonaws.com` with the public DNS hostname for your NAT host on both lines 2 and 8.
20. __SSH to the bastion host__.

    ```sh
    ssh bastion
    ```
21. __Switch the sshd port on the bastion host__. Once you are connected to the bastion host, edit the sshd config file using:

    ```sh
    sudo vi /etc/ssh/sshd_config
    ```
    And uncomment the `Port` config option and set it to:

    ```sh
    Port 20022
    ```
    And then restart sshd:

    ```sh
    sudo service sshd restart
    ```
    NOTE: It's a good idea to leave your current session connected until you complete the next two steps just in case you lock yourself out.
22. __Update your ssh config file__. On your local machine, update the file you create in step 19 to use the new port:

    ```
    Host bastion
      Hostname ec2-54-186-145-78.us-west-2.compute.amazonaws.com
      Port 20022
      ProxyCommand none
      User ec2-user

    Host *.us-west-2.compute.internal
      User ubuntu
      ProxyCommand ssh -p 20022 -W %h:%p ec2-user@ec2-54-186-145-78.us-west-2.compute.amazonaws.com
    ```
    Replace `ec2-54-186-145-78.us-west-2.compute.amazonaws.com` with the public DNS hostname for your NAT host on both lines 2 and 8.
23. __Verify you can ssh to the bastion host__.

    ```sh
    ssh bastion
    ```
24. __Verify you can ssh to one of the internal hosts on the private subnet__. Replace the DNS name from my example below with the private DNS name for the host you created in step 18.

    ```sh
    ssh ip-10-10-1-18.us-west-2.compute.internal
    ```
25. __Revoke inbound access on port 22 to the NAT host__. Since you've changed sshd to run on port 20022, you no longer need port 22 open.

    ```sh
    aws ec2 revoke-security-group-ingress --group-id $NATSG_ID --protocol 'tcp' --port 22 --cidr '0.0.0.0/0'
    ```

That's it. You now have a VPC set up with public and private subnets. Additionally you can set up an internal ELB to load-balance the hosts within the private subnet and connect to the ELB from hosts within the public subnet (e.g. your app servers in the public subnet can connect to the ELB that load balances your Solr servers in your private subnet.)

## Next steps

Here are some things you should do next to complete your VPC setup.

* Run an NTP server on your NAT host and then edit the default [DHCP Option Set](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_DHCP_Options.html) so that all of the hosts on your private subnet sync their clocks with the NTP server on the NAT host.
* Use [OpsWorks](http://docs.aws.amazon.com/opsworks/latest/userguide/welcome.html) to deploy hosts and applications into your private subnet.
* Set up an internal ELB to load balance services within the private subnet.
* Use [IAM roles for EC2 instances](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) to automatically retrieve temporary, per-instance AWS security credentials that can be used by your application to make calls to other AWS services.

