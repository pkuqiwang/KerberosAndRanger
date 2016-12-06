# KerberosAndRanger

###Kerberos the cluster
Log into Ambari Web UI and go to admin=>Kerberos, click Enable Kerberos
Use the following entry values

KDC hosts: KDC_HOST
Realm name: FIELD.HORTONWORKS.COM
Domains: .field.hortonworks.com,field.hortonworks.com

Kadmin host: KDC_HOST
Admin principal: hadoopadmin@FIELD.HORTONWORKS.COM

After installation, login to on of the node
```
--check keytabs
ll /etc/security/keytabs

--list keytab content
klist -kt /etc/security/keytabs/hive.service.keytab
kinit -kt /etc/security/keytabs/hive.service.keytab hive/_HOST@FIELD.HORTONWORKS.COM
klist

--login into Hive
beeline -u "jdbc:hive2://zk1:2181,zk2:2181,zk3:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2;principal=hive/_HOST@FIELD.HORTONWORKS.COM"

show tables;
show databases;
!quit
```

###Install NSCD for user group mapping
This step need to be done on all HDP nodes.

Modify /etc/openldap/ldap.conf to the following
```
# See ldap.conf(5) for details
# This file should be world readable but not world writable.

BASE    dc=field,dc=hortonworks,dc=com
URI     ldap://LDAP_FQND

#SIZELIMIT      12
#TIMELIMIT      15
#DEREF          never

TLS_CACERTDIR   /etc/openldap/certs

# Turning this off breaks GSSAPI used with krb5 when rdns = false
SASL_NOCANON    on
TLS_REQCERT never
```
Test with the following show return all the LDAP users
```
ldapsearch -W -D "cn=admin,dc=field,dc=hortonworks,dc=com" -b "dc=field,dc=hortonworks,dc=com"

ldapsearch -W -H ldaps://LDAP_FQND:636 -D "cn=admin,dc=field,dc=hortonworks,dc=com" -b "dc=field,dc=hortonworks,dc=com"
```
Modify /etc/nsswitch.conf, so passwd and group map to ldap
```
# Example:
#passwd:    db files nisplus nis
#shadow:    db files nisplus nis
#group:     db files nisplus nis

passwd:     files ldap
shadow:     files sss
group:      files ldap
#initgroups: files
```
Modify /etc/nslcd.conf to look like following
```
# See the manual page nslcd.conf(5) for more information.
ignorecase yes

# The user and group nslcd should run as.
uid nslcd
gid root

# Note: %2f encodes the '/' used as directory separator
uri ldap://qwang-kdc-ldap.field.hortonworks.com/

# The distinguished name of the search base.
base dc=field,dc=hortonworks,dc=com

# Customize certain database lookups.
base   group  ou=Groups,dc=field,dc=hortonworks,dc=com
base   passwd ou=Users,dc=field,dc=hortonworks,dc=com

# Mappings for AIX SecureWay
filter passwd (objectClass=posixaccount)
filter group  (objectClass=posixgroup)
```
Then start nslcd
```
systemctl start nslcd.service
systemctl enable nslcd.service

--test the service
getent passwd 
id ali
id hadoopadmin
```
The step for all nodes is completed.

###Update HDFS and YARN groups
exexute the following on resource manager node
```
sudo -u hdfs kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs@FIELD.HORTONWORKS.COM
sudo -u hdfs hdfs dfsadmin -refreshUserToGroupsMappings
sudo -u yarn kinit -kt /etc/security/keytabs/yarn.service.keytab yarn/qwang-hdp1.field.hortonworks.com@FIELD.HORTONWORKS.COM
sudo -u yarn yarn rmadmin -refreshUserToGroupsMappings
```

###Ranger installation
Add service => Ranger 
under ranger admin

Ranger DB host: qwang-kdc-ldap.field.hortonworks.com
Ranger DB password: password
Database Administrator (DBA) username: rangerdba

ranger user info

Sync Source: LDAP/AD
LDAP/AD URL: ldap://qwang-kdc-ldap.field.hortonworks.com:389
Bind User: cn=admin,dc=field,dc=hortonworks,dc=com
Bind User Password: password
User Search Base: ou=Users,dc=field,dc=hortonworks,dc=com
User Search Filter: (objectclass=person)
Enable Group Sync: yes
Group Search Base: ou=Groups,dc=field,dc=hortonworks,dc=com
Group Search Filter: (objectclass=posixgroup)

ranger plugin 
set all to On

ranger audit
Audit to Solr: On
SolrCloud: On
note: enable log search will enable solr cloud automaticly

ranger tag sync
no change

ranger advanced
LDAP setting
ranger.ldap.base.dn: dc=field,dc=hortonworks,dc=com
ranger.ldap.user.dnpattern: uid={0},ou=Users,dc=field,dc=hortonworks,dc=com

--after install
check ranger audit/user/group


