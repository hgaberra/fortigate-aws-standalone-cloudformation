# FortiOS FGCP AP HA (Dual AZ) in AWS


## Table of Contents
  - [Overview](./README.md#overview)
  - [Solution Components](./README.md#solution-components)
  - [Failover Process](./README.md#failover-process)
  - [CloudFormation Templates](./README.md#cloudformation-templates)
  - [Deployment](./README.md#deployment)
  - [FAQ \ Tshoot](./README.md#faq--troubleshoot)

## Overview
FortiOS supports using FGCP (FortiGate Clustering Protocol) in unicast form to provide an active-passive clustering solution for deployments in AWS.  This feature shares a majority of the functionality that FGCP on FortiGate hardware provides with key changes to support AWS SDN (Software Defined Networking).

This solution works with two FortiGate instances configured as a master and slave pair and that the instances are deployed in different subnets and different availability zones within a single VPC.  These FortiGate instances act as a single logical instance and do not share interface IP addressing as they are in different subnets.

The main benefits of this solution are:
  - Fast and stateful failover of FortiOS and AWS SDN without external automation\services
  - Automatic AWS SDN updates to EIPs and route targets
  - Native FortiOS session sync of firewall, IPsec\SSL VPN, and VOIP sessions
  - Native FortiOS configuration sync
  - Ease of use as the cluster is treated as single logical FortiGate

For further information on FGCP reference the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com).

**Note:**  Other Fortinet solutions for AWS such as FGCP HA (Single AZ), AutoScaling, and Transit Gateway are available.  Please visit [www.fortinet.com/aws](https://www.fortinet.com/aws) for further information.

**Reference Diagram:**
![Example Diagram](./content/net-diag1.png)

## Solution Components
FGCP HA provides AWS networks with enhanced reliability through device fail-over protection, link fail-over protection, and remote link fail-over protection. In addition, reliability is further enhanced with session fail-over protection for most IPv4 and IPv6 sessions including TCP, UDP, ICMP, IPsec\SSL VPN, and NAT sessions.

A FortiGate FGCP cluster appears as a single logical FortiGate instance and configuration synchronization allows you to configure a cluster in the same way as a standalone FortiGate unit. If a fail-over occurs, the cluster recovers quickly and automatically and can also send notifications to administrator so that the problem that caused the failure can be corrected and any failed resources restored.

The FortiGate instances will use multiple interfaces for data plane and control plane traffic to achieve FGCP clustering in an AWS VPC.  The FortiGate instances require four ENIs for this solution to work as designed so make sure to use an AWS EC2 instance type that supports this.   Reference AWS Documentation for further information on this.

For data plane functions the FortiGates will use two dedicated ENIs, one for a public interface (ie ENI0\port1) and another for a private interface (ie ENI1\port2).  These ENIs will utilize primary IP addressing and FortiOS should not sync the interface configuration (config system interface) or static routes (config router static) as these FGTs are in separate subnets.  This is controlled via the CLI (config system vdom-exception). Thus when configuring these items, you should do so individually on both FortiGates.

A cluster EIP will be associated to the primary IP of the public interface (ie ENI0\port1) of the current master FortiGate instance and will be reassociated to a new master FortiGate instance as well.

For control plane functions, the FortiGates will use a dedicated ENI (ie ENI2\port3) for HA management access to each instance and also allow each instance to independently and directly communicate with the public AWS EC2 API.  This dedicated interface is critical to failing over AWS SDN properly when a new FGCP HA master is elected and is the only method of access available to the current slave FortiGate instance.

Depending on the FortiOS version, the FortiGates will either reuse the HA management interface (ie ENI2\port3) or use another dedicated ENI (ie ENI3\port4) for FGCP HA communication to perform tasks such as heartbeat checks, configurationport4 sync, and session sync.  In FortiOS 7.0.x and newer versions, one ENI will be used for both HA management and communication channels.

The FortiGates are configured to use the unicast version of FGCP by applying the configuration below on both the master and slave FortiGate instances.  This configuration is automatically configured and bootstrapped to the instances when deployed by the provided CloudFormation Templates.

#### Example Master FGCP Configuration:
    config system vdom-exception
    edit 1
    set object sytem.interface
    next
    edit 2
    set object router.static
    next
    end
    config system ha
    set group-name "group1"
    set mode a-p
    set hbdev "port3" 50
    set session-pickup enable
    set ha-mgmt-status enable
    config ha-mgmt-interface
    edit 1
    set interface port3
    set gateway 10.0.3.1
    next
    end
    set override disable
    set priority 255
    set unicast-hb enable
    set unicast-hb-peerip 10.0.30.10
    end

#### Example Slave FGCP Configuration:
    config system vdom-exception
    edit 1
    set object sytem.interface
    next
    edit 2
    set object router.static
    next
    end
    config system ha
    set group-name "group1"
    set mode a-p
    set hbdev "port3" 50
    set session-pickup enable
    set ha-mgmt-status enable
    config ha-mgmt-interface
    edit 1
    set interface port3
    set gateway 10.0.30.1
    next
    end
    set override disable
    set priority 1
    set unicast-hb enable
    set unicast-hb-peerip 10.0.3.10
    end

The FortiGate instances will make calls to the public AWS EC2 API to update AWS SDN to failover both inbound and outbound traffic flows to the new master FortiGate instance.  There are a few components that make this possible.

FortiOS will assume IAM permissions to access the AWS EC2 API by using the IAM instance role attached to the FortiGate instances.  The instance role is what grants the required permissions for FortiOS to:
  - Reassign cluster EIPs assigned to primary IPs assigned to the data plane ENIs
  - Update existing routes to target the new master instance ENIs

The FortiGate instances will utilize their independent and direct internet access available through the FGCP HA management interface (ie ENI2\port3) to access the public AWS EC2 API.  It is critical that this ENI is in a public subnet with an EIP assigned so that each instance has independent and direct access to the internet or the AWS SDN will not reference the current master FortiGate instance which will break data plane traffic.

For further details on FGCP and its components, reference the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com).

## Failover Process
The following network diagram will be used to illustrate a failover event from the current master FortiGate (FortiGate 1), to the current slave FortiGate (FortiGate 2).

Inbound failover is provided by reassigning the EIPs associated to the primary IP address of ENI0\port1 from FortiGate 1's public interface to FortiGate 2's public interface.

Outbound failover is provided updating any route targets referencing FortiGate 1’s private interface reference FortiGate 2’s private interface.

The AWS SDN updates are performed by FortiGate 2 initiating API calls from the dedicated HA management interface (ie ENI2\port3) through the AWS Internet Gateway.

**Reference Diagram:**
![Example Diagram](./content/net-diag2.png)

## CloudFormation Templates
CloudFormation templates are available to simplify the deployment process and are available on the Fortinet GitHub repo.

The FGCP templates are organized into different folders based on the FortiOS version.  Once a FortiOS version is selected, templates are available to build a new base VPC if needed and an existing FGT template to deploy the clustered instances.

Here is a list of the FGCP HA templates currently available in this folder:
  - [BaseVPC_FGCP_DualAZ](./BaseVPC_FGCP_DualAZ.template.json)
  - [FGCP_DualAZ_ExistingVPC](./FGCP_DualAZ_ExistingVPC.template.json)

These templates not only deploy AWS infrastructure but also bootstrap the FortiGate instances with the relevant network and FGCP HA configuration to support the VPC.  Most of this information is gathered as variables in the templates when a stack is deployed.  These variables are organized into these main groups:
-	VPC Configuration
-	FortiGate Instance Configuration
-	Interface IP Configuration for FortiGate 1
-	Interface IP Configuration for FortiGate 2

### VPC Configuration
In this section the parameters will request general information for existing VPCs.  AWS resource IDs will need to be selected for the existing VPC and subnets.  **Routes need to be added manually to the existing private route table(s) manually to reference the current FGCP master.**  Below is an example of the template parameters.

![Example Diagram](./content/params2.png)

### FortiGate Instance Configuration
For this section the variables will request general instance information such as instance type and key pair to use for deploying the instances.  Also, FortiOS specific information will be requested such as if an S3 endpoint should be deployed, init S3 bucket, FortiOS version, and FortiGate License filenames.  **Ensure that an S3 gateway endpoint is deployed and assigned to the PublicSubnet's AWS route table or bootstrapping will fail.** Additional items such as the IP addresses for AWS resources within the VPC such as the IP of the AWS intrinsic router for the public, private, and HAmgmt subnets.  The AWS intrinsic router is always the first host IP for each subnet.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html#VPC_Sizing) for further information on host IPs used by AWS services within a subnet.

![Example Diagram](./content/params3a.png)
![Example Diagram](./content/params3b.png)

### Interface IP Configuration for FortiGate 1 & 2
The next two sections request IP addressing information to configure the primary IP addresses of ENIs of the FortiGate instances.  This information will also be used to bootstrap the configuration for both FortiGates.

![Example Diagram](./content/params4.png)


## Deployment
Before attempting to create a stack with the templates, a few prerequisites should be checked to ensure a successful deployment:
1.	An AMI subscription must be active for the FortiGate license type being used in the template.
  * [BYOL Marketplace Listing](https://aws.amazon.com/marketplace/pp/B00ISG1GUG)
  * [PAYG Marketplace Listing](https://aws.amazon.com/marketplace/pp/B00PCZSWDA)
2.	The solution requires 3 EIPs to be created so ensure the AWS region being used has available capacity.  Reference [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html) for more information on EC2 resource limits and how to request increases.
3.	If BYOL licensing is to be used, ensure these licenses have been registered on the support site.  Reference the VM license registration process PDF in this [KB Article](http://kb.fortinet.com/kb/microsites/search.do?cmd=displayKC&docType=kc&externalId=FD32312).
4.   **Create a new S3 bucket in the same region where the template will be deployed.  If the bucket is in a different region than the template deployment, bootstrapping will fail and the FGTs will be inaccessible**.
5.  If BYOL licensing is to be used, upload these licenses to the root directory of the same S3 bucket from the step above.
6.  **Ensure that an S3 gateway endpoint is deployed and assigned to both of the PublicSubnet's AWS route table.**  If you are using the 'BaseVPC_FGCP_DualAZ.template.json' template, then this is already deployed for you.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-gateway.html#create-gateway-endpoint) for further information.
7.  **Ensure that all of the PublicSubnet's and HAmgmtSubnet's AWS route tables have a default route to an AWS Internet Gateway.**  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html#route-tables-internet-gateway) for further information.

Once the prerequisites have been satisfied, login to your account in the AWS console and proceed with the deployment steps below.

----
1.  In the AWS services page under All Services > Management Tools, select CloudFormation.

![Example Diagram](./content/deploy1.png)

2.  Select Create Stack then select with new resources.

![Example Diagram](./content/deploy2.png)

3.  On the Select Template page, under the Choose a Template section select Upload a template to Amazon S3 and browse to your local copy of the chosen deployment template.

![Example Diagram](./content/deploy3.png)

4.  On the Specify Details page, you will be prompted for a stack name and parameters for the deployment.  We are using AWS resource IDs for a VPC created with the default values of the 'BaseVPC_FGCP_DualAZ.template.json' template.

![Example Diagram](./content/deploy4.png)

5.  In the FortiGate Instance Configuration parameters section, we have selected an Instance Type and Key Pair to use for the FortiGates as well as BYOL licensing.  Notice we are prompted for if an S3 endpoint should be deployed, the InitS3Bucket, License Types, FortiGate1LicenseFile, and FortiGate2LicenseFile parameters.  For the values we are going to reference the S3 bucket and relevant information from the deployment prerequisite step 4.

![Example Diagram](./content/deploy5.png)
![Example Diagram](./content/deploy6.png)

6.  In the Interface IP Configuration for the FortiGates parameters section, we are going with the defaults in this example as the subnet addressing matches.  These IPs will be the primary IPs assigned to the FortiGate ENIs.  These values will also be used as the static IPs in the FortiOS configuration for both FortiGates.

![Example Diagram](./content/deploy7.png)

7.  On the Options page, you can scroll to the bottom and select Next.

8.  On the Review page, scroll down to the capabilities section.  As the template will create IAM resources, you need to acknowledge this by checking the box next to ‘I acknowledge that AWS CloudFormation might create IAM resources’ and then click Create.

![Example Diagram](./content/deploy9.png)

9.  On the main AWS CloudFormation console, you will now see your stack being created.  You can monitor the progress by selecting your stack and then select the Events tab.

![Example Diagram](./content/deploy10.png)

10.  Once the stack creation has completed successfully, select the Outputs tab to get the login information for the FortiGate instances and cluster.

![Example Diagram](./content/deploy11.png)

11.  Using the login information in the stack outputs, login to the master FortiGate instance with the ClusterLoginURL.  This should put you on FortiGate 1.  You will also be prompted to change the initial password for the admin account.

![Example Diagram](./content/deploy13.png)

12.  Navigate to the HA status page on the master FortiGate by going to System > HA.  Now you should see both FortiGate 1 and FortiGate 2 in the cluster with FortiGate 2 as the current slave.

![Example Diagram](./content/deploy14.png)

13.  Give the HA cluster time to finish synchronizing their configuration and update files.  You can confirm that both the master and slave FortiGates are in sync by looking at the Status column and confirming there is a green check next to both FortiGates and the status is Synchronized.

*** **Note:** Due to browser caching issues, the icon for Synchronization status may not update properly after the cluster is in-sync.  So either close your browser and log back into the cluster or alternatively verify the HA config sync status with the CLI command ‘get system ha status’. ***

![Example Diagram](./content/deploy15.png)

14.  Navigate to the AWS EC2 console and reference the instance Detail tab for FortiGate 1.  Notice the primary IPs assigned to the instance ENIs as well as the 2 EIPs associated to the instance, the Cluster EIP and the HAmgmt EIP.

![Example Diagram](./content/deploy16.png)

15.  Now reference the instance Detail tab for FortiGate 2.  Notice the primary IPs assigned to the instance ENIs and only one EIP is the HAmgmt EIP.

![Example Diagram](./content/deploy17.png)

16.  Navigate to the AWS VPC console and create a default route in the default route table with a next hop targeting ENI1\port2 of FortiGate 1 which is the current master.

![Example Diagram](./content/deploy18.png)

17.  Navigate back to the AWS EC2 console and reference the instance Detail tab for FortiGate 1.  Now shutdown FortiGate 1 via the EC2 console and refresh the page after a few seconds.  Notice that the Cluster EIP is no longer assigned to FortiGate 1.

![Example Diagram](./content/deploy19.png)

18.  Now reference the instance Detail tab for FortiGate 2.  Notice that the Cluster EIP is now associated to FortiGate 2.

![Example Diagram](./content/deploy20.png)

19.  Navigate back to the AWS VPC console and look at the routes for the default route table.  The default route target is now pointing to ENI1\port2 of FortiGate 2.

![Example Diagram](./content/deploy21.png)

20.  Now log back into the cluster_login_url and you will be placed on the current master FortiGate, which should now be FortiGate 2.

![Example Diagram](./content/deploy22.png)

21.  Now power on FortiGate 1 and confirm that it joins the cluster successfully as the slave and FortiGate 2 continues to be the master FortiGate.

![Example Diagram](./content/deploy23.png)

22.  This concludes the template deployment example.
----

## FAQ \ Troubleshoot
  - **Does FGCP support having multiple Cluster EIPs and secondary IPs on ENI0\port1?**

Yes.  FGCP will move over any secondary IPs associated to ENI0\port1 and EIPs associated to those secondary IPs to the new master FortiGate instance.  You will need to configure secondary IPs on the ENI via the AWS EC2 Console and in FortiOS for port1.  The private IPs configured on the ENI and FortiOS must match.

  - **Does FGCP support having multiple routes for ENI1\port2?**

Yes.  FGCP will move any routes (regardless of the network CIDR) found in AWS route tables that are referencing any of the current master FortiGate instance’s data plane ENIs (ENI0\port1, ENI1\port2, etc) to the new master on a failover event.

  - **What VPC configuration is required when deploying either of the existing VPC CloudFormation templates?**

The existing VPC CloudFormation template is expecting the same VPC configuration that is provisioned in the new Base VPC template.  The existing customer VPC would need to have 4 subnets in each of the two availability zones to cover the required Public, Private, HAsync, and HAmgmt subnets.  Also ensure that an S3 gateway endpoint deployed and assigned to both of the PublicSubnet's AWS route table.  Another critical point is that all of the Public and HAmgmt subnets need to be configured as public subnets.  This means that an IGW needs to be attached to the VPC and a route table, with a default route using the IGW, needs to be associated to the Public and HAmgmt subnets.

  - **During a failover test we see successful failover to a new master FortiGate instance, but then when the original master is online, it becomes master again.**

The master selection process of FGCP will ignore HA uptime differences unless they are larger than 5 minutes.  The HA uptime is visible in the GUI under System > HA.  This is expected and the default behavior of FortiOS but can be changed in the CLI under the ‘config system ha’ table.  For further details on FGCP master selection and how to influence the process, reference primary unit selection section of the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com/).

  - **During a failover test we see FGCP select a new master but AWS SDN is not updated to point to the new master FortiGate instance.**

Confirm the FortiGates configuration are in-sync and are properly selecting a new master by seeing the HA role change as expected in the GUI under System > HA or CLI with ‘get sys ha status’.  However during a failover the routes, and Cluster EIPs are not updated, then your issue is likely to do with direct internet access via HAmgmt interface (ENI2\port3) of the FortiGates, failed DNS resolution, or IAM instance role permissions issues.

For details on the IAM instance profile configuration that should be used, reference the policy statement attached to the ‘iam-role-policy’ resource in any of the CloudFormation templates.

For the HAmgmt interface, confirm this is configured properly in FortiOS under the ‘config system ha’ section of the CLI.  Reference the example master\slave CLI HA configuration in the Solutions Components section of this document.

Also confirm that subnet the HAmgmt interface is associated to, is a subnet with public internet access and that this interface has an EIP associated to it.  This means that an IGW needs to be attached to the VPC, and a route table with a default route to the IGW needs to be associated to the HAmgmt subnet.

Finally, the AWS API calls can be debugged on the FortiGate instance that is becoming master with the following CLI commands:
```
diag deb app awsd -1
diag deb enable
```

This can be disabled with the following CLI commands:
```
diag deb app awsd 0
diag deb disable
```

  - **Is it possible to validate that both FortiGate instances are able to assume the IAM instance role and successfully and access AWS EC2 API endpoints without a failover event?.**

Yes.  You can run the following command on either FortiGate to confirm availability.  Remember that all DNS queries and HTTPS calls to AWS EC2 API endpoints will be sent out of the HAmgmt interfaces.  To get more verbose output, you can use the diag debug commands shown above.

```
diag test app awsd 4
```

  - **Is it possible to remove direct internet access from the HAmgmt subnet and provide private AWS EC2 API access via a VPC interface endpoint?**

Yes.  However, there are a few caveats to consider.

First, a dedicated method of access to the FortiGate instances needs to be setup to allow dedicated access to the HAmgmt interfaces.  This method of access should not use the master FortiGate instance so that either instance can be accessed regardless of the cluster status.  Examples of dedicated access are Direct Connect or IPsec VPN connections to an attached AWS VPN Gateway.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/SetUpVPNConnections.html) for further information.

Second, the FortiGates should be configured to use the ‘169.254.169.253’ IP address for the AWS intrinsic DNS server as the primary DNS server to allow proper resolution of AWS API hostnames during failover to a new master FortiGate.  Here is an example of how to configure this with CLI commands:
```
config system dns
set primary 169.254.169.253
end
```

Finally, the VPC interface endpoint needs to be deployed into both of the HAmgmt subnets and must also have ‘Private DNS’ enabled to allow DNS resolution of the default AWS EC2 API public hostname to the private IP address of the VPC endpoint.  This means that the VPC also needs to have both DNS resolution and hostname options enabled as well.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-private-dns) for further information.
