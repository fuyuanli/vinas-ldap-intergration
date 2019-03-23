modify lam package to support groupOfNames.

It will make posixGroup unusable due to all reference to it are replaced or removed.

for Groups, Samba 3 extension must be added or otherwise gidNumber will error out.

the patch are use as it is and only test compatiblity with LAM 6.6-1

put following files at /usr/share/ldap-account-manager/lib/modules

** groupOfNames.inc

** inetOrgPerson.inc

** posixAccount.inc

** sambaGroupMapping.inc
