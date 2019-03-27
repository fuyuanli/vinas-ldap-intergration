Please check Wiki for more detail

modify lam package to support groupOfNames.

It will make posixGroup unusable due to all reference to it are replaced or removed.

for Groups, Samba 3 extension must be added or otherwise gidNumber will error out.

the patch are use as it is and only test compatiblity with LAM 6.7-1

Copy following files to /usr/share/ldap-account-manager/lib/modules

** groupOfNames.inc

** inetOrgPerson.inc

** posixAccount.inc

** sambaGroupMapping.inc

copy group.inc to /usr/share/ldap-account-manager/lib/types

Guides are self-contained html files, so just please download them
