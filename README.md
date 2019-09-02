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
```
oc adm groups sync --sync-config=/etc/origin/master/groupsync.yaml --confirm
```

You should see OpenShift groups now:

```
oc get groups
```
5. Assign role to the Group

E.g, to assign Cluster admin role to ocp-admin group

```
oc adm policy add-cluster-role-to-group cluster-admin  ocp-admin
```

# LimitRange
# ResourceQuota

Limit ranges are set by cluster administrators and are scoped to a given project.

LimitRange  Example
```
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "resource-limits" 1
spec:
  limits:
    -
      type: "Pod"
      max:
        cpu: "2" 2
        memory: "1Gi" 3
      min:
        cpu: "200m" 4
        memory: "6Mi" 5
    -
      type: "Container"
      max:
        cpu: "2" 6
        memory: "1Gi" 7
      min:
        cpu: "100m" 8
        memory: "4Mi" 9
      default:
        cpu: "300m" 10
        memory: "200Mi" 11
      defaultRequest:
        cpu: "200m" 12
        memory: "100Mi" 13
      maxLimitRequestRatio:
        cpu: "10" 14
```

 1. The name of the limit range object.
 2. The maximum amount of CPU that a pod can request on a node across all containers.
 3. The maximum amount of memory that a pod can request on a node across all containers.
 4. The minimum amount of CPU that a pod can request on a node across all containers.
 5. The minimum amount of memory that a pod can request on a node across all containers.
 6. The maximum amount of CPU that a single container in a pod can request.
 7. The maximum amount of memory that a single container in a pod can request.
 8. The minimum amount of CPU that a single container in a pod can request.
 9. The minimum amount of memory that a single container in a pod can request.
 10. The default amount of CPU that a container will be limited to use if not specified.
 11. The default amount of memory that a container will be limited to use if not specified.
 12. The default amount of CPU that a container will request to use if not specified.
 13. The default amount of memory that a container will request to use if not specified.
 14. The maximum amount of CPU burst that a container can make as a ratio of its limit over request.


## To create a Limit Range

```
oc create -f <LimitRange Template>
```

## To get the list of limit ranges defined in a project

```
oc get limits -n <project name>
NAME              AGE
resource-limits   2d
```

## To get the details of a specific LimitRange

```
oc describe limits resource-limits
```


# ClusterResourceQuota
# LimitRange
