**AWS Networking Workshop**
--------------------

More and more customers have been migrating their production workloads from on-premises into AWS cloud. But in the mean time, it's a huge challenge on how to connect tens or hundreds of VPC together and still have ability to control routing. Otherwise, how to build a hybrid DNS architecture to allow all workloads communicating each other through DNS is a complicated question as well.

The objective of this workshop is to take you to build a hybrid architecture step by step by using three new AWS networking services AWS Transit Gateway, AWS Client VPN and Route53 Resolver. In the workshop, we will create 3 VPCs, 2 of them are spoke VPC and the other is shared service VPC where we will create AWS managed Microsoft Directory, AWS Client VPN endpoints and DNS service.The diagram below is the high level architecture we will build in this workshop.    

![Deployment Diagram](images/architecture.jpg)


**STEP 1 - Enviornment Set-up**
---------------------------

For avoiding misconfiguration, we will automatically create the 3 VPCs by using CloudFormation. Copy the link below and specify the path as an template URL. 

<https://s3.amazonaws.com/ykwang-networking-workshop/networking-workshop.yaml>

1) Choose **Creat new stack** on the CloudFormation page.
 
![Deployment Diagram](images/newstack.jpg)

2) Past the link under **Specify an Amazon S3 template URL** and click **Next**.

![Deployment Diagram](images/templateurl.jpg)

3) Specify your stack name and choose **KeyName** for accessing EC2 via SSH, then click **Next**.

![Deployment Diagram](images/stackname.jpg)

4) Click **Next** to skip **Option** page and click **Create** on **Review** page. CloudFormation will automatically create the VPCs and related resources. Wait about 25 ~ 30 minutes and make sure the status is **CREATE_COMPLETE**.

![Deployment Diagram](images/createcomplete.jpg)

5) Verify all resources CloudFormation created are identical as below.  

**VPC AND SUBNET**

* VPC4VPN - 10.1.0.0/16 (VPC4VPN-SN1 - 10.1.1.0/24, VPC4VPN-SN2 - 10.1.2.0/24)
* VPC10 - 10.10.0.0/16 (VPC10-SN1 - 10.10.1.0/24, VPC10-SN2 - 10.10.2.0/24)
* VPC20 - 10.20.0.0/16 (VPC20-SN1 - 10.20.1.0/24, VPC20-SN2 - 10.20.2.0/24)

**EC2 INSTANCE (Record your EC2 private IP)**

* ICMP Client in VPC10 - IP 10.10.1.X
* ICMP Client in VPC20 - IP 10.10.2.X

**Directory Service (Record MicrosoftAD DNS address)**

![Deployment Diagram](images/addns.jpg)

**STEP 2 - Create VPN Users on Microsoft AD**
---------------------------
To create users and groups in an AWS Directory Service directory, you must be connected to a EC2 instance that has been joined to your AWS Directory Service directory, and be logged in as a user that has privileges to create users and groups. You will also need to install the Active Directory Tools on your EC2 instance so you can add your users and groups with the Active Directory Users and Computers snap-in. 

1) Create a DHCP Options Set for Your Directory

Open the Amazon VPC console and choose **DHCP Options Sets** in the navigation pane. Then choose Create DHCP options set. Type a name you like and type **workshop.aws.com** for **Domain name**. For **Domain name servers**, type the IP addresses of your AWS provided directory's DNS servers you just recorded. Leave the settings blank for **NTP servers**, **NetBIOS name servers**, and **NetBIOS node type**. Choose **Create DHCP options set**. Make a note of the ID of the new set of DHCP options (dopt-xxxxxxxx).

![Deployment Diagram](images/dhcp.jpg)

2) Apply DHCP Options Set to VPC4VPN

Back to VPC console, select **VPC4VPN**, choose **Actions**, and then choose **Edit DHCP Options Set**. 

![Deployment Diagram](images/editdhcp.jpg)

In the Edit DHCP Options Set dialog box, select the options set that you recorded in last step. Then choose Save. 

![Deployment Diagram](images/editdhcp2.jpg)


3) Create a Windows Instance and Automatically Join the Directory

Open the Amazon EC2 console, choose Launch Instance and select **Microsoft Windows Server 2016 Base AMI**. On the page of **Configure Instance Details**, do the following:

* Choose **VPC4VPN** for **Network**
* Choose **VPC4VPN-SN1** for **Subnet**
* Choose **Use subnet setting(Enable)** for **Auto-assign Public IP**
* Choose **workshop.aws.com** for **Domain join directory**

![Deployment Diagram](images/instanceconfig.jpg)

Click **Create new IAM role** to create a new IAM role and attach the AmazonEC2RoleforSSM policy. Under **Select your use case**, choose **EC2**, and then choose **Next**. Type **AmazonEC2RoleforSSM** at search bar, select it and then choose **Next**.

![Deployment Diagram](images/roleforssm.jpg)

For **Role name**, enter a name for your new role (such as EC2DomainJoin). Then choose Create role.

![Deployment Diagram](images/roleforssm2.jpg)

Back to the page of **Configure Instance Details**, select **EC2DomainJoin** for **IAM role**.

![Deployment Diagram](images/instanceprofile.jpg)

Keep the setting default on the page of **Add Storage**, add key:Name and value:WinServer for tag, choose **Select an existing security group** and select the security group with name **Allow RDP and ICMP** and then click **Review and Launch**. Click **Launch** again and select a keypair for this workshop. 

![Deployment Diagram](images/ec2_sg.jpg)


4) Install the Active Directory Tools on Your EC2 Instance

Open EC2 console, download Remote Desktop File and get the administrator password.

![Deployment Diagram](images/rdp.jpg)

Open your RDP software and login Windows instance with username:Administrator and password you just decrypted and then do the following:

From the Start menu, choose Windows PowerShell and Copy the following command.

		Install-WindowsFeature -Name GPMC,RSAT-AD-PowerShell,RSAT-AD-AdminCenter,RSAT-ADDS-Tools,RSAT-DNS-Server

**IMPORTANT** After installation of AD tools, logout and re-login with domain administrator. Use **workshop.aws.com\admin** as username and **Passw0rd!** as password.  

5) Create a Group and a User

From the Start menu, choose **Active Directory Users and Computers**. In the directory tree, select **Users** OU under **workshop** directory and click **Action**, click **New** to create a **Group**. Type VPN Users for **Group name** and click **OK**. 

![Deployment Diagram](images/adgroup.jpg)

Click **Action** to create a **User**. Type **First name**, **Last name**, **User logon name** and click **Next**. On the second page of the wizard, type a password in **Password** and **Confirm Password**. Uncheck **User must change password at next logon**, select **Password never expires** and then click **Next** click **Finish**

![Deployment Diagram](images/aduser.jpg)

Right click at **VPN Users** group, choose **Properties** and click **Members** to add the user to this user group.

![Deployment Diagram](images/joingroup.jpg)


**STEP 3 - Create AWS Client VPN**
----------------------------------------------

1) Generate a Server Certificate and import into ACM

**IMPORTANT** - AWS CLI tool is essential in this step. If the laptop you use without AWS CLI tool, you can ssh to EC2 instance in VPC10 to complete it.  

Follow the instruction below to generate a server certificate and upload it into ACM.

1. Clone the OpenVPN easy-rsa repo to your local computer.

		$ git clone https://github.com/OpenVPN/easy-rsa.git
		
2. Navigate into the easy-rsa/easyrsa3 folder in your local repo. 

		$ cd easy-rsa/easyrsa3
		
3. Initialize a new PKI environment.

		$ ./easyrsa init-pki
		
4. Build a new certificate authority (CA).

		$ ./easyrsa build-ca nopass
		
5. Generate the server certificate and key.

		$ ./easyrsa build-server-full server nopass

6. Copy the server certificate and key to a custom folder and then navigate into the custom folder. 

		$ cp pki/ca.crt /custom_folder/
		$ cp pki/issued/server.crt /custom_folder/
		$ cp pki/private/server.key /custom_folder/
		$ cd /custom_folder/
		
7. Upload the server certificate and key to ACM.

		$ aws acm import-certificate --certificate file://server.crt --private-key file://server.key --certificate-chain file://ca.crt --region region



Open **Certificate Manage** console, expand the details page of certificate you just imported and record the resource ARN.

![Deployment Diagram](images/importcert.jpg)

2) Create a Client VPN Endpoint

Open the Amazon VPC console, choose **Client VPN Endpoints** and choose **Create Client VPN Endpoint**. Do the following:

* Type **Networking Workshop** for **Name Tag**
* Type **192.168.200.0/22** for **Client IPv4 CIDR**
* Select **arn:aws:acm:us-east-1:xxx:certificate:xxxx** for **Server certificate ARN**
* Choose **Use Active Directory authentication** for **Authentication Options**
* Select **d-xxxxxxxxxx** for **Directory ID**
* Select **No** for **Connection Logging**
* Leave **DNS IP address** as blank and select **UDP** for 
**Transport Protocol**

![Deployment Diagram](images/vpnendpoint.jpg)

3) Configure Client VPN Endpoint

Select the client VPN endpoint and choose **Associations**, click **Associate**. Select the vpc-id of **VPC4VPN** for **VPC** setting and select subnet-id with us-east-1a for **Choose a subnet to associate**.

![Deployment Diagram](images/targetnetwork.jpg)

The VPC's default security group is automatically applied for the subnet association. Make sure all traffic is allowed for inbound rules and outbound rules . Select the client VPN endpoint and choose **Authorization**, click **Authorize Ingress**. Type **0.0.0.0/0** for **Destination network to enable**, select **Allow access to all users** and click **Add authorization rule**.

![Deployment Diagram](images/authorizerule.jpg)

Click **Route Table** and **Create Route**. Type **0.0.0.0/0** for Route destination and select **subnet-xxxx** for **Target VPC Subnet ID**.

![Deployment Diagram](images/createroute.jpg)

Click **Summary** to review the setting and go through each tab.

![Deployment Diagram](images/settingreview.jpg)

4) Connect Your AWS Client VPN

**IMPORT** - Make sure you've already installed OpenVPN client in your laptop or mobile device. If not, check the link below to install.

* Windows - <https://docs.aws.amazon.com/vpn/latest/clientvpn-user/windows.html>
* MacOS - <https://docs.aws.amazon.com/vpn/latest/clientvpn-user/macos.html>
* Android and iOS - <https://docs.aws.amazon.com/vpn/latest/clientvpn-user/android.html>

Click **Download Client Configuration** and store the ovpn file in your loacl disk.

![Deployment Diagram](images/vpncfg.jpg)

In this step, the illustrations were captured from Tunnelblick whic is a free OpenVPN client for MacOS. You can reference the steps because the working process of most OpenVPN clients is similar. 
Open your OpenVPN client and import the OpenVPN config(.ovpn file). Click **connect**, type the username and password that you create for vpn user in Active Directory. The OpenVPN client will automatically establish VPN connection between AWS Client VPN enspoints and your laptop.

![Deployment Diagram](images/vpnlogin.jpg)

Back to **Client VPN Endpoints** console, click **Connections** to check the connection status. You can also check the status at your laptop by using command like **ifconfig**, **netstat -rn** for MacOS or **ipconfig** and **route print** for Windows.
 
![Deployment Diagram](images/vpncheck.jpg)

5) Test your VPN Connection

* Open you browser and link to <https://www.whatismyip.com/>
* Ping 168.95.1.1
* Ping www.amazon.com

Awesome!! Public IP is from Ashburn, VA US owned by Amazon. In term of ping test, we know the connectivity is ok, but the RTT is not good enough. The reason is taht all traffic routing to the destination is transimmited over OpenVPN tunnel, and we built the AWS Client VPN endpoints in US-WEST not in Asia.

![Deployment Diagram](images/pingcheck.jpg)

**Noted**

If you try to ping the Windows instance in VPC4VPN, don't forget to turn off the Windows firewall, otherwise the icmp will be blocked by Windows host firewall. 

Turn Windows Defender Firewall off - <https://support.microsoft.com/en-us/help/4028544/windows-10-turn-windows-defender-firewall-on-or-off>  

Ping the Linux instances located in VPC10 to verify whether the vpn connection can reach to other VPCs. The answer is no because there is no routing information between VPC4VPN and VPC10. 

![Deployment Diagram](images/pingvpc10.jpg)


**STEP 4 - Create Transit Gateway**
----------------------------------------------
What is a Transit Gateway?

A transit gateway acts as a regional virtual router for traffic flowing between your virtual private clouds (VPC) and VPN connections. A transit gateway scales elastically based on the volume of network traffic. Routing through a transit gateway operates at layer 3, where the packets are sent to a specific next-hop attachment, based on their destination IP addresses.

1) Create the Transit Gateway

Open the Amazon VPC console, choose **Transit Gateways** and **Create Transit Gateway**. Type **Networking Workshop** for **Name tag** and leave all settings as default.



