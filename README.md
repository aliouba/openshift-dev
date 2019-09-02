# LDAP GROUP IMPORTS

1. Create your LDAPSyncConfig file

```
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://ldaphostname:389
insecure: true
bindPassword: xxxx
bindDN: cn=admin,dc=dakar,dc=io
rfc2307:
    groupsQuery:
        baseDN: "ou=groups,dc=dakar,dc=io"
        scope: sub
        derefAliases: never
        filter: (objectclass=groupOfNames)
    groupUIDAttribute: dn
    groupNameAttributes: [ cn ]
    groupMembershipAttributes: [ member ]
    usersQuery:
        baseDN: "ou=users,dc=dakar,dc=io"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn
    userNameAttributes: [ mail ]
    tolerateMemberNotFoundErrors: false
    tolerateMemberOutOfScopeErrors: false
```
2. Copy LDAPSyncConfig into  /etc/origin/master folder
3. Check groups
```
oc adm groups sync --sync-config=/etc/origin/master/groupsync.yaml
```

This is a test run, it prints to the terminal the group objects that the command would create in the cluster

You should see four group definitions, as listed above.

```
apiVersion: v1
items:
- apiVersion: user.openshift.io/v1
  kind: Group
  metadata:
    annotations:
      openshift.io/ldap.sync-time: 2018-12-08T15:13:13Z
      openshift.io/ldap.uid: cn=ocp-admin,ou=groups,dc=dakar,dc=io
      openshift.io/ldap.url: 10.142.0.6:389
    creationTimestamp: null
    labels:
      openshift.io/ldap.host: 10.142.0.6
    name: ocp-admin
  users:
  - aliou.ba@dakar.io
  - boris.kodjoe@dakar.io
- apiVersion: user.openshift.io/v1
  kind: Group
  metadata:
    annotations:
      openshift.io/ldap.sync-time: 2018-12-08T15:13:13Z
      openshift.io/ldap.uid: cn=paymentapp,ou=groups,dc=dakar,dc=io
      openshift.io/ldap.url: 10.142.0.6:389
    creationTimestamp: null
    labels:
      openshift.io/ldap.host: 10.142.0.6
    name: paymentapp
  users:
  - mohamadou.diallo@dakar.io

```

4. Import groups if you are happy
5. Assign role to the Group

