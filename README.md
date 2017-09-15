# AWS VPN and Strongswan

If in the same region, you can use vpc-peering. If cross region, you need to create VPN connections to build up site-to-site vpn.

There are two ways to setup site-to-site vpn connections: Strongswan-to-strongswan, or Strongswan-to-AWS-CGW(customer gateway).

## Architecture diagram

![](/images/dg-strongswan.png)

## Strongswan VPC configuration

### VPC Settings
  * region: us-west-1
  * VPC: 172.50.0.0/16
  * Subnet: 172.50.0.0/24
  * RouteTable

  ![](/images/rtb-172-50-0-0.png)

  * Security Groups: sg-172-50-0-0

  ![](/images/sg-172-50-0-0.png)

### Strongswan server configuration
  * instance type: t2.micro
  * OS: ubuntu 16.04
  * **Elastic IPs**: 13.56.167.238 (must create one because we use static VPN CGW)
  * Security Groups: sg-172-50-0-0
  * **Disable Source/Dest. Check**

#### Install strongswan packages
  <pre>
  sudo su
  apt-get update
  apt-get install strongswan
  echo 1 > /proc/sys/net/ipv4/ip_forward && sysctl -p
  </pre>

After you download VPN configuration from AWS, you need to config strongswan two files: `/etc/ipsec.conf` and `/etc/ipsec.secrets`

#### ipsec.conf
   * **Important**
      * `keyexchange=ikev1` Currently, AWS does not support ikev2
      * `authby=psk` PSK: pre shared key. The key get from AWS VPC > VPN Connections > Download configuration. The PSK key is at `/etc/ipsec.secrets`
      * `debug=2` : The log is at `cat /var/log/syslog | grep charon`
      * `auto=add` : use `ipsec up vpg-34.252.133.10` to manually connect VPN. After stable, remember change to `auto=start`
      * `left` : left means (L)ocal. Put the strongswan private ip
      * `leftid`: put the EIP, ie static public ip
      * `right`: right means (R)emote. Put the AWS VPN tunnel ip.

<pre>
config setup
       debug=2

conn %default
       mobike=no
       type=tunnel
       compress=no
       keyexchange=ikev1
       ike=aes128-sha1-modp1024!
       ikelifetime=28800s
       esp=aes128-sha1-modp1024!
       lifetime=3600s
       rekeymargin=3m
       keyingtries=3
       installpolicy=yes
       dpdaction=restart
       authby=psk
       left=172.50.0.140
       leftsubnet=172.50.0.0/16
       leftid=13.56.167.238
       rightsubnet=172.60.0.0/16

conn vpg-34.252.133.10
       right=34.252.133.10
       dpdaction=restart
       auto=add

conn vpg-52.18.222.29
       right=52.18.222.29
       dpdaction=restart
       auto=add
</pre>

#### ipsec.secrets
   * first column is AWS VPN outside IP Address. AWS will have two tunnels.

<pre>
34.252.133.10 : PSK "place PSK key for tunnel 1"
52.18.222.29  : PSK "place PSK key for tunnel 2"
</pre>

#### Start strongswan service
 After setup the `ipsec.conf` and `ipsec.secrets`, run the commands

<pre>
  systemctl restart strongswan
  ipsec up vpg-34.252.133.10
  ipsec up vpg-52.18.222.29
  systemctl status strongswan
  ipsec status
</pre>

## AWS VPN Connections Configuration

### VPC Settings
  * region: eu-west-1
  * VPC: 172.60.0.0/16
  * Subnet: 172.60.0.0/24
  * Route Tables: need to set 172.50.0.0/16 target to VGW.

  ![](/images/rtb-172-60-0-0.png)

  * Security Groups: sg-172-60-0-0. Please note SSH limit the from source from VPC_1 and VPC_2. You can also limit PING, Echo, and Traceroute.

  ![](/images/sg-172-60-0-0.png)

### AWS VPN Connections setup

  * Setup sequences:

  `Virtual private gateways > Customer Gateways > VPN Connections`

  * Virtual Private gateways: myVPG-172-60-
0-0. Then attach to VPC 172.60.0.0/16

  ![](/images/vpg-172-60-0-0.png)

  * Customer gateways: CGW-13-56-167-238

  ![](/images/cgw-13-56-167-238.png)

  * VNP Connections: VPN-172-60-0-0

  Please notice that the static ip prefixes is VPC_1 `172.50.0.0/16`. If PING can't work, must check here and routes table.

  ![](/images/vpn-172-60-0-0.png)

   * Download the configuration. Save as `vpn-config-172-60-0-0.txt`

   ![](/images/vpn-config-download.png)

   * Follow the instruction `vpn-config-172-60-0-0.txt` to configure strongswan server two files: `/etc/ipsec.conf` and `/etc/ipsec.secrets`. Then restart strongswan services. See the [ipsec.conf](#### ipsec.conf) and [ipsec.secrets](#### ipsec.secrets)

   * If successfully, you will see strongswan server with the following messages:

   ![](/images/strongswan-status-screenshot.png)

   And, the AWS console

   ![](/images/vpn-status-screenshot.png)

   * Create an EC2 instance in eu-west-1 with security groups `sg-172-60-0-0`. Try to ping from your `172.50.0.0/24` network.

# Lab2 - vpc-peering, NAT Gateway, NAT Instance

  * You can't NAT throught vpc-peering. Please see this [support ticket](https://forums.aws.amazon.com/thread.jspa?threadID=157603)

  * Default Extra Packages for Enterprise Linux (EPEL) repository is disable. You need to following the [instruction](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-enable-epel/) to enable it.

  <pre>
  sudo yum-config-manager --enable epel
  sudo yum repolist
  </pre>

  * Amazon Linux is Centos 6 version. There is no `systemctl` command. Only `service`. Reference [this](https://www.reddit.com/r/aws/comments/5fkf40/why_doesnt_the_amazon_linux_distro_have_systemctl/)

  * Enable the [iptables](https://forums.aws.amazon.com/thread.jspa?threadID=56024)

  <pre>
  touch /etc/sysconfig/iptables
  chkconfig iptables on
  service iptables start
  </pre>

## iptables

  * Add the "masquerade" NAT rule for the entire VPC:
  <pre>
    iptables -t nat -A POSTROUTING -s 10.0.3.0/24 -o eth0 -m policy --dir out --pol ipsec -j ACCEPT
    iptables -t nat -A POSTROUTING -s 10.0.3.0/24 -o eth0 -j MASQUERADE
  </pre>

  * Save the iptables configuration so that it will survive a reboot.
     * RHEL/CentOS machines
       * `/sbin/service iptables save` This will write the rules to `/etc/sysconfig/iptables`.
     * Debian/Ubuntu machines
       * Install iptables-persistent, if not already installed: `apt-get install iptables-persistent`. This during install it will prompt to save current rules. Say yes, to have it create `/etc/iptables/rules.v4` and `/etc/iptables/rules.v6` for you.
       * In the future, after making changes, run `iptables-save > /etc/iptables/rules.v4` to save.
  * check configuration. Run `iptables -L` and `iptables -t nat -L`

## Others
  * If you setup strongswan at centos 7. The installation command is

<pre>
yum install -y epel-release; yum reposlist; yum update -y; yum install -y strongswan vim ntp;
</pre>

  * In centos, the strongswan configuration is at `/etc/strongswan/ipsec.secrets` and `/etc/strongswan/ipsec.conf`
  * recommend to config ntp to sync server time.
  * We does not cover HA yet. Please reference the [Strongswan HA](https://wiki.strongswan.org/projects/strongswan/wiki/HighAvailability) and [active-passive strongswan](https://www.strongswan.org/testing/testresults/ha/active-passive/)
  * [EC2 instance recover](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-recover.html)


## Reference

* [AWS Generic Config](http://docs.aws.amazon.com/AmazonVPC/latest/NetworkAdminGuide/GenericConfig.html)
* [Connect VPCs with Strongswan]( https://asieira.github.io/connecting-aws-vpcs-with-strongswan.html)
* [Strongswan configuration for AWS-CGW](http://blog.xk72.com/post/155816502544/aws-vpc-vpn-strongswan-configuration)
* [ForwardingAndSplitTunneling](https://wiki.strongswan.org/projects/strongswan/wiki/ForwardingAndSplitTunneling)
* [AwsVpc](https://wiki.strongswan.org/projects/strongswan/wiki/AwsVpc)
