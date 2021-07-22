# Object Storage Support

## Table of Contents

<!-- toc -->
  - [Release Signoff Checklist](#release-signoff-checklist)
  - [Introduction](#introduction)
  - [Motivation](#motivation)
  - [User Stories](#user-stories)
      - [Admin](#admin)
      - [DevOps](#devops)
    - [Object Storage Provider (OSP)](#object-storage-provider-osp)
  - [Goals](#goals)
      - [Functionality](#functionality)
      - [System Properties](#system-properties)
    - [Non-Goals](#non-goals)
  - [COSI architecture](#cosi-architecture)
  - [COSI API](#cosi-api)
  - [Bucket Creation](#bucket-creation)
  - [Generating Access Credentials for Buckets](#generating-access-credentials-for-buckets)
  - [Bucket Provisioning (for pods)](#bucket-provisioning-for-pods)
- [COSI API Reference](#cosi-api-reference)
  - [Bucket](#bucket)
  - [BucketRequest](#bucketrequest)
  - [BucketClass](#bucketclass)
  - [BucketAccess](#bucketaccess)
  - [BucketAccessRequest](#bucketaccessrequest)
  - [BucketAccessClass](#bucketaccessclass)
- [COSI Driver](#cosi-driver)
  - [COSI Driver gRPC API](#cosi-driver-grpc-api)
      - [ProvisionerGetInfo](#provisionergetinfo)
      - [ProvisionerCreateBucket](#provisionercreatebucket)
      - [ProvisionerGrantBucketAccess](#provisionergrantbucketaccess)
      - [ProvisionerDeleteBucket](#provisionerdeletebucket)
      - [ProvisionerRevokeBucketAccess](#provisionerrevokebucketaccess)
- [Test Plan](#test-plan)
- [Graduation Criteria](#graduation-criteria)
  - [Alpha](#alpha)
  - [Alpha -&gt; Beta](#alpha---beta)
  - [Beta -&gt; GA](#beta---ga)
- [Alternatives Considered](#alternatives-considered)
  - [Add Bucket Instance Name to BucketAccessClass (brownfield)](#add-bucket-instance-name-to-bucketaccessclass-brownfield)
    - [Motivation](#motivation-1)
    - [Problems](#problems)
    - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Version Skew Strategy](#version-skew-strategy)
  - [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
    - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
    - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
    - [Monitoring Requirements](#monitoring-requirements)
    - [Dependencies](#dependencies)
    - [Scalability](#scalability)
  - [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements][57] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] (R) Graduation criteria is in place
- [X] (R) Production readiness review completed
- [ ] Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website][58], for publication to [kubernetes.io][59]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Introduction

This proposal introduces the *Container Object Storage Interface* (COSI), a standard for provisioning and consuming object storage in Kubernetes.  


## Motivation

File and block storage are treated as first class citizens in the Kubernetes ecosystem via CSI.  Workloads using CSI volumes enjoy the benefits of portability across vendors and across Kubernetes clusters without the need to change application manifests. **An equivalent standard does not exist for Object storage.**

Object storage has been rising in popularity in the recent years as an alternative form of storage to filesystems and block devices. Object storage paradigm promotes disaggregation of compute and storage. This is done by making data available over the network, rather than locally. Disaggregated architectures allow compute workloads to be stateless, which consequently makes them easier to manage, scale and automate. 

## User Stories

We define 3 kinds of stakeholders:

+ **Admins** - Members with escalated privileges and authority to manage site wide/cluster wide policies to control access, assign storage quotas and other concerns related to hardware or application resources.
+ **Application Developer/Operator (User)** - Members with privileges to run workloads, request storage for them, and manage resources inside a specific namespace.
+ **Object Storage Provider (OSP)** - Vendors offering Object Storage capabilities

#### Admin

+ Establish cluster/site wide policies for data redundancy, durability and other data related parameters
+ Establish cluster/site wide policies for access control over buckets
+ Enable/restrict users who can create, delete and reuse buckets

#### DevOps

+ Request a bucket to be created and/or provisioned for workloads
+ Request access to be created/deleted for a specific bucket
+ Request deletion of a created bucket

### Object Storage Provider (OSP)

+ Integrate with the official standard for orchestrating Object Storage resources in Kubernetes

## Goals

#### Functionality

+ Support automated **Bucket Creation**
+ Support automated **Bucket Deletion**
+ Support automated **Access Credential Generation**
+ Support automated **Access Credential Revokation**
+ Support automated **Bucket Provisioning** for workloads (enabling pods to access buckets)
+ Support **Bucket Reuse** 

#### System Properties

+ Support **workload portability** across clusters
+ Achieve the above goals in a **vendor neutral** manner
+ Standardize mechanism for third-party vendors to integrate easily with COSI

### Non-Goals

**Data Plane** API standardization will not be addressed by this project

## COSI architecture

Since this is an entirely new feature, it is possible to implement this completely out of tree. The following components are proposed for this architecture:

 - COSI ControllerManager
 - COSI Sidecar
 - COSI Driver
 - COSI NodeAdapter

  ![COSI Architecture](images/cosi-architecture.jpg)

1. The COSI ControllerManager is the central controller that manages lifecycle of COSI objects. Only one active instance of ControllerManager should be present in a cluster.
2. The COSI Sidecar is the point of integration between COSI and drivers. All operations that require communication with the OSP is triggered by the Sidecar using gRPC calls to the driver. Only one active instance of Sidecar should be present **for a particular driver** in a cluster.
3. The COSI driver communicates with the OSP to conduct Bucket related operations.
4. The COSI NodeAdapter is responsible for presenting buckets and its associated access credentials to pods. One instance of NodeAdapter should be present on every node.

More information about COSI driver is [here](#cosi-driver)

## COSI API

COSI defines 6 new API types

 - [Bucket](#bucket)
 - [BucketRequest](#bucketrequest)
 - [BucketClass](#bucketclass)
 - [BucketAccess](#bucketaccess)
 - [BucketAccessRequest](#bucketaccessrequest)
 - [BucketAccessClass](#bucketaccessclass)

More information about usage of these API types is provided inline with user stories.

## Bucket Creation

The following stakeholders are involved in the lifecycle of bucket creation:

 - Users  - request buckets to be created for their workload
 - Admins - setup COSI drivers in the cluster, and specify their configurations

Here are the steps for creating a Bucket:

  ![COSI Bucket Creation](images/create-bucket.jpg)


###### 1.  Admin creates BucketClass, User requests Bucket to be created

The BucketClass represents a set of common properties shared by multiple buckets. It is used to specify the driver for creating Buckets, and also for configuring driver-specific parameters. More information about BucketClass is [here](#bucketclass)

The BucketRequest is a request to create a Bucket. This resource is designed to specify properties that the application requires, such as the data protocol that this bucket should satisfy (i.e. s3, gcs or azure). More information about BucketRequest is [here](#bucketrequest)

```
    BucketRequest - br1                                   BucketClass - bc1
    |------------------------------|                      |----------------------------------|
    | spec:                        |                      | spec:                            |
    |   bucketClassName: bc1       |                      |   provisioner: s3.amazonaws.com  |
    |   protocol: s3               |                      |   parameters:                    | 
    |   deletionPolicy: delete     |                      |     key: value                   |
    |------------------------------|                      |----------------------------------|
```

###### 2. COSI creates a intermediate Bucket object

The ControllerManager creates a Bucket object by copying fields over from both the BucketRequest and BucketClass. This step can only proceed if the BucketClass exists. This Bucket object is an intermediate object, with its status field `bucketAvailable` set to false. This indicates that the Bucket is yet to be created in the OSP.

More information about Bucket is [here](#bucket)

```
    Bucket - br-$uuid
    |--------------------------------------|
    | name: br-$uuid                       |
    | spec:                                |
    |   bucketClassName: bc1               |
    |   protocol: s3                       |
    |   parameters:                        |
    |     key: value                       |
    |   provisioner: s3.amazonaws.com      | 
    | status:                              |
    |   bucketAvailable: false             |
    |--------------------------------------|
```

###### 3. COSI calls driver to create Bucket

The COSI sidecar, which runs alongside each of the drivers, listens for Bucket events. All but the sidecar with the specified provisioner will ignore the Bucket object. Only the appropriate sidecar will make a gRPC request to its driver to create a Bucket in the OSP.

More information about COSI gRPC API is [here](#cosi-grpc-api)

```
    COSI Driver (s3.amazonaws.com)
    |------------------------------------------|
    | grpc ProvisionerCreateBucket({           |
    |      "name": "br-$uuid",                 |
    |      "protocol": "s3",                   |
    |      "parameters": {                     |
    |         "key": "value"                   |
    |      }                                   |
    | })                                       |
    |------------------------------------------|
```
Once the Bucket is successfully created in the OSP, sidecar sets `bucketAvailable=true` in the Bucket object. At this point, the Bucket is ready to be utilized by workloads.

## Generating Access Credentials for Buckets

The following stakeholders are involved in the lifecycle of access credential generation:

 - Users  - request access to buckets
 - Admins - establish cluster wide access policies

Access credentials are represented by BucketAccess objects. Here are the steps for creating a BucketAccess:

  ![COSI Access Credential Creation](images/create-bucket-access.jpg)

###### 1.  Admin creates BucketAccessClass, User requests BucketAccess to be created

The BucketAccessClass represents a set of common properties shared by multiple BucketAccesses. It is used to specify policies for creating access credentials, and also for configuring driver-specific access parameters. More information about BucketAccessClass is [here](#bucketclass)

The BucketAccessRequest is a request to create BucketAccess. This resource is designed to specify the Bucket for which the credenails provide access, and also for specifying access modes (rw, ro, wo) for that the application requires. More information about BucketAccessRequest is [here](#bucketaccessrequest)

```
    BucketAccessRequest - bar1                             BucketAccessClass - bac1
    |-------------------------------|                      |----------------------------------|
    | spec:                         |                      | spec:                            |
    |   bucketAccessClassName: bac1 |                      |   provisioner: s3.amazonaws.com  |
    |   bucketRequestName: br1      |                      |   parameters:                    | 
    |                               |                      |     key: value                   |
    |-------------------------------|                      |----------------------------------|
```

###### 2. COSI creates a intermediate BucketAccess object

The ControllerManager creates a BucketAccess object by copying fields over from both BucketAccessRequest and BucketAccessClass. This step can only proceed if the BucketAccessClass and Bucket exist. The Bucket's `bucketAvailable` field should be true. This BucketAccess object is an intermediate object, with its status field `bucketAvailable` set to false. This indicates that access credentials are yet to be generated in the OSP.

More information about BucketAccess is [here](#bucketaccess)

```
    Bucket Access - bar-$uuid
    |--------------------------------------|
    | name: bar-$uuid                      |
    | spec:                                |
    |   bucketAccessClassName: bac1        |
    |   parameters:                        |
    |     key: value                       |
    |   bucketName: br1-$uuid              | 
    | status:                              |
    |   credentialsSecret: nil             |
    |   accessGranted: false               |
    |--------------------------------------|
```

###### 3. COSI calls driver to create BucketAccess

The COSI sidecar, which runs alongside each of the drivers, listens for BucketAccess events. All but the sidecar with the specified provisioner will ignore the BucketAccess object. Only the appropriate sidecar will make a gRPC request to its driver to generate credentials for a Bucket in the OSP.

More information about COSI gRPC API is [here](#cosi-grpc-api)

```
    COSI Driver (s3.amazonaws.com)
    |------------------------------------------|
    | grpc ProvisionerGrantBucketAccess({      |
    |      "name": "bar-$uuid",                |
    |      "bucketID": "br-$uuid",             |
    |      "parameters": {                     |
    |         "key": "value"                   |
    |      }                                   |
    | })                                       |
    |------------------------------------------|
```
Each BucketAccess is meant to map to one account (or its equivalent) in the OSP, and grant access for that account to access the bucket. Once the credentials are successfully generated in the OSP, the credential values are stored in a secret. The secret resides in the namespace of the driver. Since each driver is expected to run in its own namespace, the secret containing access credentials cannot be accessed directly by users.

The sidecar sets `accessGranted=true` in the BucketAccess object and sets the `credentialsSecret` field to the name and namespace of the generated secret. At this point, the BucketAccess is ready to be utilized by workloads.

## Bucket Provisioning (for pods)

The following stakeholders are involved in the lifecycle of provisioning bucket for pods:

 - Users  - specify bucket in the pod definition

Once a valid BucketAccess is available, pods can use it to access buckets.

###### 1.  User creates pod with volume of type objectstorage.k8s.io

A BucketAccess with `accessGranted=true` holds information for accessing buckets. 

A Pod with a volume of type `objectstorage.k8s.io` and a pointer to the BucketAccessRequest is required for COSI to fetch information for a particular Bucket. 

```
    PodSpec - pod1
    |-------------------------------------------------|
    | spec:                                           |
    |   containers:                                   |
    |   - volumeMounts:                               |
    |       name: cosi-bucket                         | 
    |       mountPath: /home/ubuntu/cosi-bucket-info  |
    | volumes:                                        |
    | - name: cosi-bucket                             |         
    |   csi:                                          |
    |     driver: objectstorage.k8s.io                |
    |     volumeAttributes:                           |
    |       bar-name: bar1                            |
    |                                                 |
    |-------------------------------------------------|
```

The volume `mountPath` will be the directory where bucket credentials and other related information will be served. 

NOTE: the contents of the files served in mountPath will be COSI generated files for serving buckets. This is not a mount of the bucket's data.

###### 2.  COSI Node Adapter serves BucketInfo in the specified directory

The above volume definition will prompt kubernetes to invoke COSI Node Adapater (which is a CSI driver). The CSI driver mechanism is useful for preventing pods from being scheduled until the Bucket is ready, and the access credentials have been created. The CSI driver will look for the specified BucketAccessRequest in the same namespace as the Pod. BucketAccessRequests in other namespaces cannot be specified.

Once the preconditions are met, COSI Node Adapter will generate a file named `bucket.json` and write it into the `mountPath`. 

```
    bucket.json                                          
    |-----------------------------------------------|
    | {                                             |
    |   apiVersion: "v1alpha1"                      |
    |   "S3": {                                     |
    |     "bucketName": "br-$uuid",                 |
    |     "endpoint": "https://s3.amazonaws.com",   |
    |     "accessKeyID": "AKIAIOSFODNN7EXAMPLE",    |
    |     "accessSecretKey": "wJalrXUtnFEMI/K...",  |
    |     "region": "us-west-1"                     |
    |   }                                           |           
    | }                                             |
    |-----------------------------------------------|

```

The API of `bucket.json` follows the same versioning and lifecycle as the rest of Bucket* APIs. Workloads are expected to read the definition in this file to access a bucket.

# COSI API Reference

## Bucket 

Resource to represent a Bucket in OSP. Buckets are cluster scoped.

```yaml
Bucket {
  TypeMeta
  ObjectMeta
  
  Spec BucketSpec {
    // Provisioner is the name of driver associated with this bucket
    Provisioner string 

    // DeletionPolicy is used to specify how COSI should handle deletion of this 
    // bucket. There are 3 possible values:
    //  - Retain: Indicates that the bucket should not be deleted from the OSP
    //  - Delete: Indicates that the bucket should be deleted from the OSP
    //        once all the workloads accessing this bucket are done
    //  - ForceDelete: Indicates that the bucket and its contents should be 
    //        deleted from the OSP even if workloads are currently accessing 
    //        this bucket
    DeletionPolicy DeletionPolicy 
    
    // Name of the BucketClass specified in the BucketRequest
    // +optional
    BucketClassName string 
    
    // Name of the BucketRequest that resulted in the creation of this Bucket
    // +optional
    BucketRequest corev1.ObjectReference
    
    // AllowedNamespaces is the list of namespaces from which workloads  
    // can access this Bucket. If empty, all namespaces can access this access
    // +optional 
    AllowedNamespaces []string 

    // Protocol is the data API this bucket is expected to adhere to.
    // The possible values for protocol are:
    // -  S3: Indicates Amazon S3 protocol
    // -  Azure: Indicates Microsoft Azure BlobStore protocol
    // -  GCS: Indicates Google Cloud Storage protocol
    Protocol Protocol
    
    // Parameters is an opaque map for passing in configuration to a driver
    // for creating the bucket
    // +optional
    Parameters map[string]string 
  }    
  
  Status BucketStatus {
    // BucketAvailable indicates that the bucket is ready for consumption
    // by workloads
    BucketAvailable bool
    
    // BucketID is the unique id for the bucket in the OSP
    BucketID string
  }
}
```

## BucketRequest

A request to create a Bucket in OSP. BucketRequest is a namespaced resource

```yaml
BucketRequest {
  TypeMeta
  ObjectMeta
  
  Spec BucketRequestSpec {
    // Protocol is the data API this bucket is expected to adhere to.
    // The possible values for protocol are:
    // -  S3: Indicates Amazon S3 protocol
    // -  Azure: Indicates Microsoft Azure BlobStore protocol
    // -  GCS: Indicates Google Cloud Storage protocol
    Protocol Protocol
    
    // Name of the BucketClass 
    // +optional    
    BucketClassName string
  }
    
  Status BucketRequestStatus {
    // BucketAvailable indicates that the bucket is ready for consumpotion
    // by workloads      
    BucketAvailable bool 
  }
```
## BucketClass

Resouce for configuring common properties for multiple Buckets. BucketClass is a clustered resource

```yaml
BucketClass {
  TypeMeta
  ObjectMeta
  
  // Provisioner is the name of driver associated with this bucket
  Provisioner string
  
  // DeletionPolicy is used to specify how COSI should handle deletion of this 
  // bucket. There are 3 possible values:
  //  - Retain: Indicates that the bucket should not be deleted from the OSP
  //  - Delete: Indicates that the bucket should be deleted from the OSP
  //        once all the workloads accessing this bucket are done
  //  - ForceDelete: Indicates that the bucket and its contents should be 
  //        deleted from the OSP even if workloads are currently accessing 
  //        this bucket
  DeletionPolicy DeletionPolicy

  // AllowedNamespaces is the list of namespaces from which workloads     
  // can access this Bucket. If empty, all namespaces can access this access
  // +optional   
  AllowedNamespaces []string

  // Parameters is an opaque map for passing in configuration to a driver
  // for creating the bucket
  // +optional
  Parameters map[string]string 
}
```

## BucketAccess

```yaml
BucketAccess {
  TypeMeta
  ObjectMeta
  
  Spec BucketAccessSpec {
    // Parameters is an opaque map for passing in configuration to a driver
    // for creating the bucket
    // +optional
    BucketAccessRequest corev1.ObjectReference
    
    // Name of the BucketAccessClass specified in the BucketAccessRequest
    BucketAccessClassName string
    
    // Parameters is an opaque map for passing in configuration to a driver
    // for creating access credentials
    // +optional
    Parameters map[string]string
  }
  
  Status BucketAccessStatus{
    // AccessGranted indicates that the credentials provisioned have been
    // granted access to the bucket
    AccessGranted bool
  
    // AccountID is the unique ID for the account in the OSP
    AccountID string
  
    // CredentialsSecret is the secret which holds information the
    // accessKey and secretKey values of the granted credentials
    CredentialsSecret corev1.ObjectReference
  }
}
```

## BucketAccessRequest

```yaml
BucketAcessRequest {
  TypeMeta
  ObjectMeta
  
  Spec BucketAccessRequestSpec {
    // BucketRequestName is the name of the BucketRequest
    BucketRequestName string
    
    // BucketAccessClassName is the name of the BucketAccessClass
    // +optional    
    BucketAccessClassName string
  }
    
  Status BucketRequestStatus {
    // AccessGranted indicates that the credentials have been given access to 
    // bucket pointed-to by the specified BucketRequest
    AccessGranted bool 
  }
```

## BucketAccessClass

```yaml
BucketClass {
  TypeMeta
  ObjectMeta

  // Parameters is an opaque map for passing in configuration to a driver
  // for creating the bucket
  // +optional
  Parameters map[string]string 
}
```

# COSI Driver

A component that runs alongside COSI Sidecar and satisfies the COSI gRPC API specification. Sidecar and driver work together to conduct Bucket operations in OSP. The driver acts as a gRPC server to the COSI Sidecar. Each COSI driver is identified by a unique id.

The sidecar uses the unique id to direct requests to the appropriate drivers. Multiple instances of drivers with the same id will be added into a group and only one of them will act as the leader at any given time. 

## COSI Driver gRPC API

#### ProvisionerGetInfo

This gRPC call responds with the name of the provisioner. The name is used to identify the appropriate driver for a given BucketRequest or BucketAccessRequest.

```
    ProvisionerGetInfo                                           
    |------------------------------------------|       |---------------------------------------|
    | grpc ProvisionerGetInfoRequest{}         | ===>  | ProvisionerGetInfoResponse{           | 
    |------------------------------------------|       |   "name": "s3.amazonaws.com"          |
                                                       | }                                     |
                                                       |---------------------------------------|
```

#### ProvisionerCreateBucket

This gRPC call creates a bucket in the OSP, and returns information about the new bucket. This api must be idempotent. The input to this call is the name of the bucket and an opaque parameters field. 

The returned `bucketID` should be a unique identifier for the bucket in the OSP. This value could be the name of the bucket too. This value will be used by COSI to make all subsequent calls related to this bucket.
 
```
    ProvisionerCreateBucket
    |------------------------------------------|       |-----------------------------------------------|
    | grpc ProvisionerCreateBucketRequest{     | ===>  | ProvisionerCreateBucketResponse{              |
    |     "name": "br-$uuid",                  |       |   "bucketID": "br-$uuid",                     |
    |     "parameters": {                      |       |   "bucketInfo": {                             |
    |        "key": "value"                    |       |      "s3": {                                  |
    |     }                                    |       |        "bucketName": "br-$uuid",              |
    | }                                        |       |        "region": "us-west1",                  |
    | -----------------------------------------|       |        "endpoint": "s3.amazonaws.com"         |
                                                       |      }                                        |                                   
                                                       |   }                                           | 
                                                       | }                                             |
                                                       |-----------------------------------------------|
```

#### ProvisionerGrantBucketAccess

This gRPC call creates a set of access credentials for a bucket. This api must be idempotent. The input to this call is the id of the bucket, a set of opaque parameters and name of the account. This `accountName` field is used to ensure that multiple requests for the same BucketAccessRequest do not result in multiple credentials.

The returned `accountID` should be a unique identifier for the account in the OSP. This value could be the name of the account too. This value will be used by COSI to make all subsequent calls related to this account.

```
    ProvisionerGrantBucketAccess
    |---------------------------------------------|       |-----------------------------------------------|
    | grpc ProvisionerGrantBucketAccessRequest{   | ===>  | ProvisionerGrantBucketAccessResponse{         |
    |     "bucketID": "br-$uuid",                 |       |   "accountID": "bar-$uuid",                   |
    |     "accountName": "bar-uuid"               |       |   "credentials": {                            |
    |     "parameters": {                         |       |      "s3": {                                  |
    |        "key": "value"                       |       |        "accessKeyID": "AKIAODNN7EXAMPLE",     |
    |     }                                       |       |        "accessSecretKey": "wJaUtnFEMI/K..."   |
    | }                                           |       |      }                                        |
    |---------------------------------------------|       |   }                                           | 
                                                          | }                                             |
                                                          |-----------------------------------------------|
```

#### ProvisionerDeleteBucket

This gRPC call deletes a bucket in the OSP.

```
    ProvisionerDeleteBucket
    |---------------------------------------------|       |-----------------------------------------------|
    | grpc ProvisionerDeleteBucketRequest{        | ===>  | ProvisionerDeleteBucketResponse{}             |
    |     "bucketID": "br-$uuid"                  |       |-----------------------------------------------|
    | }                                           |
    |---------------------------------------------|
```

#### ProvisionerRevokeBucketAccess

This gRPC call revokes access granted to a particular account.

```
    ProvisionerDeleteBucket
    |---------------------------------------------|       |-----------------------------------------------|
    | grpc ProvisionerRevokeBucketAccessRequest{  | ===>  | ProvisionerRevokeBucketAccessResponse{}       |
    |     "bucketID": "br-$uuid",                 |       |-----------------------------------------------|
    |     "accountID": "bar-$uuid"                |
    | }                                           |
    |---------------------------------------------|

```

# Test Plan

- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- An end-to-end test suite will cover testing all the components together.
- Component tests will cover testing each controller in a blackbox fashion.
- Tests need to cover both correctly and incorrectly configured cases.


# Graduation Criteria

## Alpha
- API is reviewed and accepted
- Implement all COSI components to support Greenfield, Green/Brown Field, Brownfield and Static Driverless provisioning
- Evaluate gaps, update KEP and conduct reviews for all design changes
- Develop unit test cases to demonstrate that the above mentioned use cases work correctly

## Alpha -\> Beta
- Basic unit and e2e tests as outlined in the test plan.
- Metrics in kubernetes/kubernetes for bucket create and delete, and granting and revoking bucket access.
- Metrics in provisioner for bucket create and delete, and granting and revoking bucket access.

## Beta -\> GA
- Stress tests to iron out possible race conditions in the controllers.
- Users deployed in production and have gone through at least one K8s upgrade.

# Alternatives Considered
This KEP has had a long journey and many revisions. Here we capture the main alternatives and the reasons why we decided on a different solution.

## Add Bucket Instance Name to BucketAccessClass (brownfield)

### Motivation
1. To improve workload _portability_ user namespace resources should not reference non-deterministic generated names. If a `BucketAccessRequest` (BAR) references a `Bucket` instance's name, and that name is pseudo random (eg. a UID added to the name) then the BAR, and hence the workload deployment, is not portable to another cluser.

1. If the `Bucket` instance name is in the BAC instead of the BAR then the user is not burdened with knowledge of `Bucket` names, and there is some centralized admin control over brownfield bucket access.

### Problems
1. The greenfield -\> brownfield workflow is very awkward with this approach. The user creates a `BucketRequest` (BR) to provision a new bucket which they then want to access. The user creates a BAR pointing to a BAC which must contain the name of this newly created \``Bucket` instance. Since the `Bucket`'s name is non-deterministic the admin cannot create the BAC in advance. Instead, the user must ask the admin to find the new `Bucket` instance and add its name to new (or maybe existing) BAC.

1. App portability is still a concern but we believe that deterministic, unique `Bucket` and `BucketAccess` names can be generated and referenced in BRs and BARs.

1. Since, presumably, all or most BACs will be known to users, there is no real "control" offered to the admin with this approach. Instead, adding _allowedNamespaces_ or similar to the BAC may help with this.

### Upgrade / Downgrade Strategy

No changes are required on upgrade to maintain previous behaviour.

### Version Skew Strategy

COSI is out-of-tree, so version skew strategy is N/A

## Production Readiness Review Questionnaire

<!--

Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In som. e cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.
-->

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name:
  - Components depending on the feature gate:
- [X] Other
  - Describe the mechanism: Create Deployment and DaemonSet resources (along with supporting secrets, configmaps etc.) for the three controllers that COSI requires
  - Will enabling / disabling the feature require downtime of the control
	plane? No
  - Will enabling / disabling the feature require downtime or reprovisioning
	of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled). No

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->
No

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

Yes. Delete the resources created when installing COSI

###### What happens if we reenable the feature if it was previously rolled back?

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.
-->

N/A since we are only targeting alpha for this Kubernetes release

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade-\>downgrade-\>upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

No

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

Existing components will not make any new API calls.

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

Yes, the following cluster scoped resources

- Bucket
- BucketClass
- BucketAccess
- BucketAccessClass

and the following namespaced scoped resources

- BucketRequest
- BucketAccessRequest

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

Not by the framework itself. Calls to external systems will be made by vendor drivers for COSI.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

No

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

Yes. Containers requesting Buckets will not start until Buckets have been provisioned. This is similar to dynamic volume provisioning

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

Not likely to increase resource consumption in a significant manner

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

We need Linux VMs for e2e testing in CI.

[1]:	#release-signoff-checklist
[2]:	#summary
[3]:	#motivation
[4]:	#user-stories
[5]:	#goals
[6]:	#non-goals
[7]:	#vocabulary
[8]:	#proposal
[9]:	#apis
[10]:	#storage-apis
[11]:	#bucketrequest
[12]:	#bucket
[13]:	#bucketclass
[14]:	#access-apis
[15]:	#bucketaccessrequest
[16]:	#bucketaccess
[17]:	#bucketaccessclass
[18]:	#app-pod
[19]:	#topology
[20]:	#object-relationships
[21]:	#workflows
[22]:	#finalizers
[23]:	#create-bucket
[24]:	#sharing-cosi-created-buckets
[25]:	#delete-bucket
[26]:	#grant-bucket-access
[27]:	#revoke-bucket-access
[28]:	#delete-bucketaccess
[29]:	#delete-bucket-1
[30]:	#setting-access-permissions
[31]:	#dynamic-provisioning
[32]:	#static-provisioning
[33]:	#grpc-definitions
[34]:	#provisionergetinfo
[35]:	#provisonercreatebucket
[36]:	#provisonerdeletebucket
[37]:	#provisionergrantbucketaccess
[38]:	#provisionerrevokebucketaccess
[39]:	#test-plan
[40]:	#graduation-criteria
[41]:	#alpha
[42]:	#alpha---beta
[43]:	#beta---ga
[44]:	#alternatives-considered
[45]:	#add-bucket-instance-name-to-bucketaccessclass-brownfield
[46]:	#motivation-1
[47]:	#problems
[48]:	#upgrade--downgrade-strategy
[49]:	#version-skew-strategy
[50]:	#production-readiness-review-questionnaire
[51]:	#feature-enablement-and-rollback
[52]:	#rollout-upgrade-and-rollback-planning
[53]:	#monitoring-requirements
[54]:	#dependencies
[55]:	#scalability
[56]:	#infrastructure-needed-optional
[57]:	https://git.k8s.io/enhancements
[58]:	https://git.k8s.io/website
[59]:	https://kubernetes.io/

[image-1]:	images/cosi-architecture.jpg
[image-2]:	images/cosi-architecture-br-2-bc.jpg
i
