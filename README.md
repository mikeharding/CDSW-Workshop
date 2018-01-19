# CDSW Workshop Installation
Automated installation of CDH 5.13.1, CDSW 1.2.2 using Director 2.6.1 serving as the platform for the CDSW Hands-On Workshop.

These scripts are based on Toby Ferguson's CDSW Cloud Install scripts available at [https://github.com/TobyHFerguson/cdsw_install](https://github.com/TobyHFerguson/cdsw_install).

This repository is a subset of cdsw_install, simplified and designed to deploy a consistent cloud configuration on AWS to deliver a CDSW hands-on workshop.  


## File Organization  
### Overview
This system is a subset of cdsw_install. It only contains required pre-configured and tested cloud provider files for AWS.

The complete cdsw_install repository includes a set of configuration files comprised of common files used across cloud providers, and files associated with a particular cloud provider.  GCP and Azure specific files have been removed for simplicity.

### File Kinds
There are two kinds of files:
+ Property Files - You are expected to modify these. They match the `*.properties` shell pattern and use the [Java Properties format](https://docs.oracle.com/javase/8/docs/api/java/util/Properties.html#load-java.io.Reader-).
+ Conf files - You are not expected to modify these. They match the `*.conf` shell pattern and use the [HOCON format](https://github.com/typesafehub/config/blob/master/HOCON.md), a superset of JSON.

The intent is that those items that you need to edit are in format (`*.properties`) files that are easy to edit, whereas those items that you don't need to touch are in the harder to edit HOCON format (i.e. `*.conf` files).

### Directory Structure
The top level directory contains the main `conf` files (`common.conf` & `aws.conf`).

The `aws` directory contain files relevant to AWS.

The main configuration file is `aws.conf`. This file itself includes the files needed for an AWS deployment. We will only describe the properties files here:

* `aws/provider.properties` - a file containing the configuration for Amazon Web Services
* `aws/ssh.properties` - a file containing the details required to configure passwordless ssh access into the machines that Director will create.
* `aws/owner_tag.properties` - a file containing the mandatory value for the `owner` tag which is used to tag all VM instances. Within the Cloudera FCE account, a VM without an owner tag will be deleted. It is customary (but not enforced) to use your Cloudera id for this tag value.
* `aws/kerberos.properties` - an *optional* file containing the details of the Kerberos Key Distribution Center (KDC) to be used for kerberos authentication. (See Kerberos Tricks below for details on how to easily setup an MIT KDC and use it). *If* `kerberos.properties` is provided then a secure cluster is set up. If `kerberos.properties` is not provided then an insecure cluster will be setup.  


## SECRET file
The SECRET file is ignored by GIT and you must construct it yourself. We recommend setting its mode to 600, although that is not enforced anywhere.

The secret file you need to create for AWS is  `aws/SECRET.properties`. It is in Java Properties format and contains your AWS secret access key:
```
AWS_SECRET_ACCESS_KEY=<your secret key>
```
Mine, with dots hiding characters from the secret key, looks like:
```
AWS_SECRET_ACCESS_KEY=53Hrd................r0wiBbKn3
```
If you fail to set up the `AWS_SECRET_KEY` then you'll find that cloudera-director silently fails, but grepping for AWS_SECRET_KEY in the local log file will reveal all:

```sh
$ unset AWS_ACCESS_KEY_ID #just to make sure its undefined!
$ cloudera-director bootstrap-remote filetest.conf --lp.remote.username=admin --lp.remote.password=admin
Process logs can be found at /home/centos/.cloudera-director/logs/application.log
Plugins will be loaded from /var/lib/cloudera-director-plugins
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=256M; support was removed in 8.0
Cloudera Director 2.4.0 initializing ...
$
```
Looks like its failed, right, because it doesn't continue on. No error message! But if you execute:
```sh
$ grep AWS_SECRET ~/.cloudera-director/logs/application.log
com.typesafe.config.ConfigException$UnresolvedSubstitution: filetest.conf: 28: Could not resolve substitution to a value: ${AWS_SECRET_ACCESS_KEY}
```
You'll discover the problem! (Or there's another problem, and you should look in that log file for details).


## Workflow

### Preparation
+ Edit the `aws/*.properties` and `aws/SECRET` files appropriately.


### Cluster Creation
+ Execute the following director bootstrap command:
```sh
cloudera-director bootstrap-remote aws.conf --lp.remote.username=admin --lp.remote.password=admin
```

## Post Creation
+ Once completed, use the AWS console to find the public IP (`CDSW_PIB`) address of the CDSW instance. Its name in the AWS console will begin with `cdsw-`.
+ You can reach the CDSW at `cdsw.CDSW_PIB.nip.io`. See below for details.

All nodes in the cluster will contain the user `cdsw`. That user's password is `Cloudera1`. (If you used the mit kdc installation scripts from below then you'll also find that this user's kerberos username and password are `cdsw` and `Cloudera1` also).

## Troubleshooting
There are two logs of interest:
* client log: $HOME/.cloudera-director/logs/application.log on client machine
* server log: /var/log/cloudera-director-server/application.log on server machine

If the cloudera-director client fails before communicating with the server you should look in the client log. Otherwise look in the server log.

The server log can be large - it should be truncated frequently; especially before using a new conf file!

## Limitations & Issues
Relies on an [nip.io](http://nip.io) trick to make it work.

You'll need to set two YARN config variables by hand
+ `yarn.nodemanager.resource.memory-mb`
+ `yarn.scheduler.maximum-allocation-mb`

These are setup in the `common.conf` file, but if there's a problem (the values are inappropriate) then you'll see errors when you run a Spark job from CDSW in the CDSW project's console.

## NIP.io tricks
(NIP.io)[http://nip.io] is a public bind server that uses the FQDN given to return an address. A simple explanation is if you have your kdc at IP address `10.3.4.6`, say, then you can refer to it as `kdc.10.3.4.6.nip.io` and this name will be resolved to `10.3.4.6` (indeed, `foo.10.3.4.6.nip.io` will likewise resolve to the same actual IP address). (Note that earlier releases of this project used `xip.io`, but that's located in Norway and for me in the USA `nip.io`, located in the Eastern US, works better.)

This technique is used in two places:
+ In the director conf file to specify the IP address of the KDC - instead of messing around with bind or `/etc/hosts` in a bootstrap script etc. simply set the KDC_HOST to `kdc.A.B.C.D.xip.io` (choosing appropriate values for A, B, C & D as per your setup)
+ When the cluster is built you will access the CDSW at the public IP address of the CDSW instance. Lets assume that that address is `C.D.S.W` (appropriate, some might say) - then the URL to access that instance would be http://ec2.C.D.S.W.xip.io

This is great for hacking around with ephemeral devices such as VMs and Cloud images!

## Useful Scripts
[install_mit_kdc.sh](https://github.com/TobyHFerguson/director-scripts/blob/master/cloud-lab/scripts/install_mit_kdc.sh) to install an mit kdc. [install_mit_client.sh](https://github.com/TobyHFerguson/director-scripts/blob/master/cloud-lab/scripts/install_mit_client.sh) to create a client for testing purposes.  

## Kerberos Tricks
(Refer to the mit scripts linked above for details).

## MIT KDC
I setup an MIT KDC in the Director image and then create a `kerberos.conf` to use that:
```
krbAdminUsername: "cm/admin@HADOOPSECURITY.LOCAL"
krbAdminPassword: "Passw0rd!"

# KDC_TYPE is either "Active Directory" or "MIT KDC"
#KDC_TYPE: "Active Directory"
KDC_TYPE: "MIT KDC"

KDC_HOST: "hadoop-ad.hadoopsecurity.local"

# Need to use KDC_HOST_IP if KDC_HOST is not in DNS. Cannot use /etc/hosts because CDSW doesn't read it
# See DSE-1796 for details

KDC_HOST_IP: "10.0.0.82"

SECURITY_REALM: "HADOOPSECURITY.LOCAL"

KRB_MANAGE_KRB5_CONF: true

#KRB_ENC_TYPES: "aes128-cts-hmac-sha1-96 arcfour-hmac-md5"
KRB_ENC_TYPES: "aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5"

# Note use of aes256 - if you find you can get a ticket but not use it then this might be the problem
# You'll need to either remove aes256 or include the jce libraries
```
### Server Config
In `/var/kerberos/krb5kdc/kdc.conf`:
```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 HADOOPSECURITY.LOCAL = {
 acl_file = /var/kerberos/krb5kdc/kadm5.acl
 dict_file = /usr/share/dict/words
 admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
 supported_enctypes = aes256-cts-hmac-sha1-96:normal aes128-cts-hmac-sha1-96:normal arcfour-hmac-md5:normal
 max_renewable_life = 7d
}
```
In `/var/kerberos/krb5kdc/kadm5.acl` I setup any principal with the `/admin` extension as having full rights:

```
*/admin@HADOOPSECURITY.LOCAL	*
```

I then execute the following to setup the users etc:
```sh
sudo kdb5_util create -P Passw0rd!
sudo kadmin.local addprinc -pw Passw0rd! cm/admin
sudo kadmin.local addprinc -pw Cloudera1 cdsw

systemctl start krb5kdc
systemctl enable krb5kdc
systemctl start kadmin
systemctl enable kadmin
```

Note that the CM username and credentials are `cm/admin@HADOOPSECURITY.LOCAL` and `Passw0rd!` respectively.

### Client Config (Managed by Cloudera Manager)
In `/etc/krb5.conf` I have this:
```
[libdefaults]
 default_realm = HADOOPSECURITY.LOCAL
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
 default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
 permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5

[realms]
 HADOOPSECURITY.LOCAL = {
  kdc = 10.0.0.82
  admin_server = 10.0.0.82
 }
 ```
 (Note that the IP address used is that of the private IP address of the director server; this is stable over reboot
 so works well)


## Standard users and groups
I use the following to create standard users and groups, running this on each machine in the cluster:
```sh
sudo groupadd supergroup
sudo useradd -G supergroup -u 12354 hdfs_super
sudo useradd -G supergroup -u 12345 cdsw
echo Cloudera1 | sudo passwd --stdin cdsw
```
And then adding the corresponding hdfs directory from a single cluster machine:
```sh
kinit cdsw
hdfs dfs -mkdir /user/cdsw
```
