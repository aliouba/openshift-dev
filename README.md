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


Take note that there are two new sections added: *ResourceQuota* and
*LimitRange*.

### Background: ResourceQuota
The
link:https://docs.openshift.com/container-platform/3.11/admin_guide/quota.html[quota
documentation^] provides a great description of what *ResourceQuota* is about:

----
A resource quota, defined by a ResourceQuota object, provides constraints that
limit aggregate resource consumption per project. It can limit the quantity of
objects that can be created in a project by type, as well as the total amount of
compute resources and storage that may be consumed by resources in that
project.
----

In our case, we are setting a specific set of quota for CPU, memory, storage,
volume claims, and *Pods*. Take a look at the `ResourceQuota` section from the
`project_request_template.yaml` file:

[source,yaml]
----
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: ${PROJECT_NAME}-quota <1>
  spec:
    hard:
      pods: 10 <2>
      requests.cpu: 4000m <3>
      requests.memory: 8Gi <4>
      resourcequotas: 1
      requests.storage: 50Gi <5>
      persistentvolumeclaims: 5 <6>
      {{ CNS_STORAGECLASS }}.storageclass.storage.k8s.io/requests.storage: 25Gi <7>
      {{ CNS_BLOCK_STORAGECLASS }}.storageclass.storage.k8s.io/persistentvolumeclaims: 0 <8>
----

<1> While only one quota can be defined in a *Project*, it still needs a unique
name/id.
<2> The total number of pods in a non-terminal state that can exist in the project.
<3> CPUs are measured in "milicores". More information on how
Kubernetes/OpenShift calculates cores can be found in the
link:https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/[upstream
documentation^].
<4> There is a system of both `limits` and `requests` that we will discuss more
when we get to the *LimitRange* object.
<5> Across all persistent volume claims in a project, the sum of storage requested cannot exceed this value.
<6> The total number of persistent volume claims in a project.
<7> This setting limits the amount of storage that can be provisioned using the `glusterfs-storage` *StorageClass*.
<8> This setting limits the number of **PersistentVolumeClaims** for a **StorageClass** called `{{ CNS_BLOCK_STORAGECLASS }}`. A value of 0 means that creating **PersistentVolumeClaims** from this storage class is not allowed in this project.

For more details on the available quota options, refer back to the
link:https://docs.openshift.com/container-platform/3.11/admin_guide/quota.html[quota
documentation^].

### Background: LimitRange
The
link:https://docs.openshift.com/container-platform/3.11/admin_guide/limits.html[limit
range documentation^] provides some good background:

----
A limit range, defined by a LimitRange object, enumerates compute resource
constraints in a project at the pod, container, image, image stream, and
persistent volume claim level, and specifies the amount of resources that a pod,
container, image, image stream, or persistent volume claim can consume.
----

While the quota sets an upper bound on the total resource consumption within a
project, the `LimitRange` generally applies to individual resources. For
example, setting how much CPU an individual *Pod* or container can use.

Take a look at the sample `LimitRange` definition that we have provided in the
`project_request_template.yaml` file:

[source,yaml]
----
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: ${PROJECT_NAME}-limits
    creationTimestamp: null
  spec:
    limits:
      -
        type: Container
        max: <1>
          cpu: 4000m
          memory: 1024Mi
        min: <2>
          cpu: 10m
          memory: 5Mi
        default: <3>
          cpu: 4000m
          memory: 1024Mi
        defaultRequest: <4>
          cpu: 100m
          memory: 512Mi
----

The difference between requests and default limits is important, and is covered
in the link:https://docs.openshift.com/container-platform/3.11/admin_guide/limits.html[limit
range documentation^]. But, generally speaking:

<1> `max` is the highest value that may be specified for limits and requests
<2> `min` is the lowest value that may be specified for limits and requests
<3> `default` is the maximum amount (limit) that the container may consume, when
nothing is specified
<4> `defaultRequest` is the minimum amount that the container may consume, when
nothing is specified

In addition to these topics, there are things like *Quality of Service Tiers* as
well as a *Limit* : *Request* ratio. There is additionally more information in
the
link:https://docs.openshift.com/container-platform/3.11/dev_guide/compute_resources.html[compute
resources^] section of the documentation.

For the sake of brevity, suffice it to say that there is a complex and powerful
system of Quality of Service and resource management in OpenShift. Understanding
the types of workloads that will be run in your cluster will be important to
coming up with sensible values for all of these settings.

The settings we provide for you in these examples generally restrict projects to:

* A total CPU quota of 4 cores (`4000m`) where
** Individual containers
*** must use 4 cores or less
*** cannot be defined with less than 10 milicores
*** will default to a request of 100 milicores (if not specified)
*** may burst up to a limit of 4 cores (if not specified)
* A total memory usage of 8 Gibibyte (8192 Megabytes) where
** Individual containers
*** must use 1 Gi or less
*** cannot be defined with less than 5 Mi
*** will default to a request of 512 Mi
*** may burst up to a limit of 1024 Mi
* Total storage claims of 25 Gi or less
* A total number of 5 volume claims
* 10 or less *Pods*

In combination with quota, you can create very fine-grained controls, even
across projects, for how users are allowed to request and utilize OpenShift's
various resources.

NOTE: Remember that quotas and limits are applied at the *Project* level. *Users*
may have access to multiple *Projects*, but quotas and limits do not apply
directly to *Users*. If you want to apply one quota across multiple *Projects*,
then you should look at the
link:https://docs.openshift.com/container-platform/3.11/admin_guide/multiproject_quota.html[multi-project
quota^] documentation. We will not cover multi-project quota in these exercises.
