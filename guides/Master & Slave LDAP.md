# Master & Slave LDAP 

### (1)LDAP Setup 
Use Ubuntu 18.04 LXC template to create

```sh
systemctl stop ufw.service;
systemctl disable ufw.service;
apt update && apt upgrade;
apt -y install slapd ldap-utils
```

During the installation, you’ll be prompted
to set your base dn like dc=example,dc=com,
If you want to change your base dn, use ```dpkg-reconfigure slapd``` to change it.
But remember configeration LDAP use MDB.

to set LDAP admin password, provide your desired password, then press [OK]
![N|Solid](https://github.com/fuyuanli/vinas-ldap-intergration/raw/master/guides/images/ldap-001.png?raw=true)

Confirm the password and continue installation by selecting [ok] with TAB key.
![N|Solid](https://github.com/fuyuanli/vinas-ldap-intergration/raw/master/guides/images/ldap-002.png?raw=true)

### (2)Verify
use `slapcat` to test sladp

```
dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: example.com
dc: example
structuralObjectClass: organization
entryUUID: xxxxxxx-689b-1038-8d53-cd4ea0a9f2fa
creatorsName: cn=admin,dc=example,dc=com
createTimestamp: 20190320100850Z
entryCSN: 20190320100850.169668Z#000000#000#000000
modifiersName: cn=admin,dc=example,dc=com
modifyTimestamp: 20190320100850Z

dn: cn=admin,dc=example,dc=com
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: <ssha encrypted password>
structuralObjectClass: organizationalRole
entryUUID: e29b65e4-689b-1038-8d54-cd4ea0a9f2fa
creatorsName: cn=admin,dc=example,dc=com
createTimestamp: 20190320100850Z
entryCSN: 20190320100850.185122Z#000000#000#000000
modifiersName: cn=admin,dc=example,dc=com
modifyTimestamp: 20190320100850Z
```

### (3)Begin add schema
```
cd /etc/ldap/schema

ldapadd -Q -Y EXTERNAL -H ldapi:/// -f collective.ldif;
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f corba.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f cosine.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f duaconf.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f dyngroup.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f java.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f misc.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f openldap.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f ppolicy.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f ldapns.ldif; 
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f pmi.ldif; 

scp <samba server>:/usr/share/doc/samba/examples/LDAP/samba.ldif.gz /etc/ldap/schema/;

zcat samba.ldif.gz | ldapadd -Q -Y EXTERNAL -H ldapi:///
```

### (4)Indices
Make sure default ldap database has correct `olcDbIndex`, below example are for user/group only not meant for address and etc

```
cat '/etc/ldap/slapd.d/cn=config/olcDatabase={1}mdb.ldif'

# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 2dd0e81c
dn: olcDatabase={1}mdb
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: {1}mdb
olcDbDirectory: /var/lib/ldap
olcSuffix: dc=example,dc=com
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
olcAccess: {1}to attrs=shadowLastChange by self write by * read
olcAccess: {2}to * by * read
olcLastMod: TRUE
olcRootDN: cn=admin,dc=example,dc=com
olcRootPW:: <ssha encrypted password>
olcDbCheckpoint: 512 30
olcDbIndex: objectClass eq
olcDbIndex: cn,uid eq
olcDbIndex: uidNumber,gidNumber eq
olcDbIndex: member,memberUid eq
olcDbMaxSize: 1073741824
structuralObjectClass: olcMdbConfig
entryUUID: 3d6fde1e-76cf-1038-8f2b-3f4c412963b8
creatorsName: cn=config
createTimestamp: 20181107115143Z
entryCSN: 20190307115143.184607Z#000000#000#000000
modifiersName: cn=config
modifyTimestamp: 20190307115143Z
```

If database not correctly indexed, you need to use `ldapmodify`
```
Content of newindex.ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcDbIndex
olcDbIndex: objectClass eq
olcDbIndex: uidNumber,gidNumber eq
olcDbIndex: loginShell eq
olcDbIndex: uid,cn eq,sub
olcDbIndex: memberUid eq,sub
olcDbIndex: member,uniqueMember eq
olcDbIndex: sambaSID eq
olcDbIndex: sambaPrimaryGroupSID eq
olcDbIndex: sambaGroupType eq
olcDbIndex: sambaSIDList eq
olcDbIndex: sambaDomainName eq
olcDbIndex: default sub,eq
olcDbIndex: displayName sub,eq
```

````ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f newindex.ldif````

### (5)Configure Master LDAP

At master LDAP, use this module.ldif

```
dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/lib/ldap
olcModuleload: memberof.la
olcModuleload: refint.la
olcModuleLoad: syncprov.la
```
memberof.ldif
```
dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
```
refint.ldif
```
dn: olcOverlay={1}refint,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: {1}refint
olcRefintAttribute: memberof member manager owner
```

syncprov.ldif
```
dn: olcOverlay=syncprov,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: {2}syncprov
olcSpSessionLog: 100
```

add configuration at Master LDAP
```
ldapadd -Y EXTERNAL -H ldapi:/// -f module.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f memberof.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f refint.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov.ldif
```

### (6)Configure Slave LDAP

At Slave LDAP, use this module.ldif

```
dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/lib/ldap
olcModuleload: memberof.la
olcModuleload: refint.la
```
rp.ldif
```
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001                      # If you have muite slave,rid cannot be the same 
provider=ldap://master.example.com:389/   # Change to master domain or ip address 
bindmethod=simple
binddn="cn=admin,dc=example,dc=com"
credentials=password                      # Change to master LDAP admin password
searchbase="dc=example,dc=com "
scope=sub
schemachecking=on
type=refreshAndPersist
retry="30 5 300 3"
interval=00:00:05:00
```
```ldapadd -Y EXTERNAL -H ldapi:/// -f rp.ldif```

### (7)Initialize at Master LDAP
Initialize Users and Groups Databse -Create new_ldap.ldif
```
dn: ou=Users,dc=example,dc=com
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=example,dc=com
objectClass: organizationalUnit
ou: Groups
```
```slapadd -b "dc=example,dc=com" -v -l new_ldap.ldif```

### (8)Check Slave LDAP
Use this command to check slave Ldap have `ou=Users` and `ou=Groups`

```ldapsearch -x -b dc=example,dc=com```
### (9)Install and setup LAM
```
apt -y install apache2 php php-cgi libapache2-mod-php php-mbstring php-common php-pear php-monolog php-psr-log git;
a2enconf php7.2-cgi;
systemctl reload apache2;

apt -y install ldap-account-manager; #solve all dependancy;

wget https://nchc.dl.sourceforge.net/project/lam/LAM/6.7/ldap-account-manager_6.7-1_all.deb;  #or the latest version;
dpkg -i ldap-account-manager_6.7-1_all.deb;
apt --fix-broken install;

systemctl restart apache2;

cd /tmp/;
git clone https://github.com/fuyuanli/vinas-ldap-intergration.git;
mv /tmp/vinas-ldap-intergration/group.inc /usr/share/ldap-account-manager/lib/types/;
mv /tmp/vinas-ldap-intergration/user.inc /usr/share/ldap-account-manager/lib/types/;
mv /tmp/vinas-ldap-intergration/*.inc  /usr/share/ldap-account-manager/lib/modules/;
```

This is steps to Configure Master LDAP:
Use this guide (1),(2),(3),(4),(5),(7),(9)


This is steps to Configure Slave LDAP:
Use this guide (1),(2),(3),(4),(6),(8),(9)

