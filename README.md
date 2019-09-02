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

# ResourceQuota

	
Quotas are set by cluster administrators and are scoped to a given project.

```
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: ${PROJECT_NAME}-quota (1)
  spec:
    hard:
      pods: 10 (2)
      requests.cpu: 4000m (3)
      requests.memory: 8Gi (4)
      resourcequotas: 1
      requests.storage: 50Gi (5)
      persistentvolumeclaims: 5 (6)
      {{ storageclass name }}.storageclass.storage.k8s.io/requests.storage: 25Gi (7)
      {{ storageclass name  }}.storageclass.storage.k8s.io/persistentvolumeclaims: 0 (8)
```
1. While only one quota can be defined in a *Project*, it still needs a unique
name/id.
2. The total number of pods in a non-terminal state that can exist in the project.
3. CPUs are measured in "milicores". More information on how
Kubernetes/OpenShift calculates cores can be found in the
link:https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/[upstream
documentation^].
4. There is a system of both `limits` and `requests` that we will discuss more
when we get to the *LimitRange* object.
5. Across all persistent volume claims in a project, the sum of storage requested cannot exceed this value.
6. The total number of persistent volume claims in a project.
7. This setting limits the amount of storage that can be provisioned using the `storageclass name ` *StorageClass*.
8. This setting limits the number of **PersistentVolumeClaims** for a **StorageClass** called . A value of 0 means that creating **PersistentVolumeClaims** from this storage class is not allowed in this project.

For more details on the available quota options, refer back to the
link:https://docs.openshift.com/container-platform/3.11/admin_guide/quota.html[quota
documentation^].


## To create a ResourceQuota

```
oc create -f <ResourceQuota Template>
```

## To get the list of ResourceQuota defined in a project

```
oc get quota -n <project name>
NAME                AGE
besteffort          11m
compute-resources   2m
object-counts       29m
```

## To get the details of a specific ResourceQuota

```
oc describe quota object-counts -n <project name>
Name:			object-counts
Namespace:		demoproject
Resource		Used	Hard
--------		----	----
configmaps		3	10
persistentvolumeclaims	0	4
replicationcontrollers	3	20
secrets			9	10
services		2	10
```


# ClusterResourceQuota
A multi-project quota, defined by a ClusterResourceQuota object, allows quotas to be shared across multiple projects. Resources used in each selected project will be aggregated and that aggregate will be used to limit resources across all the selected projects.

## Selecting Projects

## For Request User

You can select projects based on annotation selection, label selection, or both. For example, to select projects based on annotations, run the following command:

```
oc create clusterquota for-user \
     --project-annotation-selector openshift.io/requester=<user-name> \
     --hard pods=10 \
     --hard secrets=20
```

It creates the following ClusterResourceQuota object:


```
apiVersion: v1
kind: ClusterResourceQuota
metadata:
  name: for-user
spec:
  quota: 1
    hard:
      pods: "10"
      secrets: "20"
  selector:
    annotations: 2
      openshift.io/requester: <user-name>
    labels: null 3
status:
  namespaces: 4
  - namespace: ns-one
    status:
      hard:
        pods: "10"
        secrets: "20"
      used:
        pods: "1"
        secrets: "9"
  total: 5
    hard:
      pods: "10"
      secrets: "20"
    used:
      pods: "1"
      secrets: "9"

```

 1. The ResourceQuotaSpec object that will be enforced over the selected projects.
 2. A simple key/value selector for annotations.
 3. A label selector that can be used to select projects.
 4. A per-namespace map that describes current quota usage in each selected project.
 5. The aggregate usage across all selected projects.
 
 
This multi-project quota document controls all projects requested by <user-name> using the default project request endpoint. You are limited to 10 pods and 20 secrets.


## Based on project label

To select projects based on labels, run this command:

```
oc create clusterresourcequota for-name \ 1
    --project-label-selector=name=frontend \ 2
    --hard=pods=10 --hard=secrets=20
```

 1. Both clusterresourcequota and clusterquota are aliases of the same command. for-name is the name of the clusterresourcequota object.
 
 2. To select projects by label, provide a key-value pair by using the format --project-label-selector=key=value.

# LimitRange

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

