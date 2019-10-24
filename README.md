# Cohesity Tenant Setup through APIs
The following demonstrates sequence of APIs to setup a working Cohesity tenant. The expected result will be an organisation with assigned

 - Storage domain
 - Protection Policies
 - Users
 - Protection Sources
 - Protection Jobs

## Fetch API Token
[Reference Documentation](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/access-tokens/create-generate-access-token)

> curl -X POST \
  https://FQDN/irisservices/api/v1/public/accessTokens \
  -H 'Accept: */*' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/json' \
  -d '{
	"username": "user-name",
	"password": "password",
	"domain": "LOCAL"
}'

 - Use the API Token in the JSON Response for further API calls.
 - `username`, `password` are mandatory fields.


## Create An Organisation
[Reference Document](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/tenant/create-tenant)

>curl -X POST \
  https://`FQDN`/irisservices/api/v1/public/tenants \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json' \
  -d '{
  "description": "New Cohesity Tenant",
  "name": "XYZ Org",
  "orgSuffix": "xyzOrg"
}'

 - `name` and `orgSuffix` are required fields.
 - On success, 201 Response code is returned with newly assigned tenantId. Use this tenant id to carry further actions on the tenant.

## Assigning an existing Protection policy to a Tenant
Before assigning a protection policy to a tenant, one must fetch all the policies to figure out the policy id.
### Fetching List of Protection Policies
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/protection-policies/get-protection-policies)
>curl -X GET \
  https://`FQDN`/irisservices/api/v1/public/protectionPolicies \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json'

 - No Query params are required.

### Assigning policy to a tenant
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/tenant/update-tenant-protection-policy)
>curl -X PUT \
  https://`FQDN`/irisservices/api/v1/public/tenants/policy \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/json' \
  -H 'cache-control: no-cache' \
  -d '{
  "policyIds": [
    "2425993104764203:1568297549728:1"
  ],
  "tenantId": "xyzOrg/"
}'

 - Policies can be shared among multiple tenants.
 - `policyId` and `tenantId` are mandatory fields here.

## Assigning Storage Domain (ViewBox) to a Tenant

Storage domains must be created in System view and then assigned to a tenant. A tenant administrator cannot create new storage domains.
To assign a storage domain, first fetch the list of storage domains using a `GET API` and then associate the storage domain `Id` with the tenant.

### Fetch all storage domains
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/view-boxes/get-view-boxes)
>curl -X GET \
  https://`FQDN`/irisservices/api/v1/public/viewBoxes \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json' 

 - The API will only return those storage domains which are available in the System view and not mapped to any tenant (In case sharing of storage domain is disabled at the time of cluster config)
 - Use the query param `allUnderHierarchy=true` to fetch all storage domains under the organisation hierarchy.

### Assigning domain to a tenant
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/tenant/update-tenant-view-box)
>curl -X PUT \
  https://`FQDN`/irisservices/api/v1/public/tenants/viewBox \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json' \
  -d '{
  "tenantId": "testOrg2/",
  "viewBoxIds": [
    5
  ]
}'

 - If sharing of storage domain across multiple tenants is disabled, a storage domain once assigned cannot further be assigned to a different organisation.
 - `tenantId` and `viewBoxIds` are mandatory fields.
 - A storage domain once assigned cannot be un-assigned. New storage domains can however be added further.

## Adding users to tenant
### Adding new user to tenant
>curl -X POST \
  https://`FQDN`/irisservices/api/v1/public/users \
  -H 'Accept: application/json, text/plain, */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Connection: keep-alive' \
  -H 'Content-Type: application/json' \
  -H 'x-impersonate-tenant-id: `TENANTID`' \
  -d '{
    "domain": "LOCAL",
    "roles": [
        "COHESITY_ADMIN"
    ],
    "restricted": false,
    "type": "user",
    "username": "test-user3",
    "emailAddress": "testuser3@cohesity.com",
    "password": "Password",
    "passwordConfirm": "Password"
}'

 - Note: `x-impersonate-tenant-id` should contain the tenantId into which the new user should be created. The header field is used to impersonate the tenant and carry actions on its behalf.
 - Roles must be one available here `https://FQDN/irisservices/api/v1/public/roles`
 - Users once assigned cannot be mapped to a different org. It must be deleted for this purpose.

### Adding existing LOCAL users to Tenant

 - Fetch the SID of the user
 - Assign the user to the tenant
#### Fetching user SID
>curl -X GET \
  https://FQDN/irisservices/api/v1/public/users \
  -H 'Accept: */*' \
  -H 'Authorization:`TOKEN`' \
  -H 'Content-Type: application/json' 
 - This will return list of all users which are not assigned to any org and are available in the System view.
 - User query param `allUnderHierarchy=true` to fetch all users under the org hierarchy.

#### Assign SID to a tenant
>curl -X PUT \
  https://`FQDN`/irisservices/api/v1/public/tenants/users \
  -H 'Authorization: `TOKEN`' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'content-type: application/json;charset=UTF-8' \
  -d '{
  "sids": [
  	"S-1-21-21-20753707-18075057-16"
  ],
  "tenantId": "testOrg2/"
}'

 - `tenantId` and `sids` are required fields.
 - A user which is already mapped to an organisation cannot be mapped to a different organisation.

## Adding a Protection Source
1. New protection sources can be registered in a tenant. In this case the tenant becomes the owner of the source and can carry all actions against it.
2. A source can have only one tenant owner. 
3. A protection source or entities in it can also be assigned to a tenant. If the entire source is assigned then the Organisation will become the owner of that source and can manage it. If only a subset of the source is assigned then the Organisation cannot perform source management operations.

### Adding a new vCenter Hypervisor Source
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/protection-sources/create-register-protection-source)
>curl -X POST \
  https://`FQDN`/irisservices/api/v1/public/protectionSources/register \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json' \
  -H 'x-impersonate-tenant-id: `TENANTID`' \
  -d '{
  "endpoint": "`vCenter-FQDN`",
  "environment": "kVMware",
  "username": "`username`",
  "password":"`password`",
  "vmwareType": "kVCenter"
}'

 - Note: The header `x-impersonate-tenant-id` must contain the tenant id.
 - Follow the reference doc to add different kind of sources.

### Assigning objects from existing source to the tenant
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/tenant/update-tenant-entity)
>curl -X PUT \
  https://`FQDN`/irisservices/api/v1/public/tenants/entity \
  -H 'Authorization: `TOKEN`' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'content-type: application/json;charset=UTF-8' \
  -d '{"entityIds":[167, 201, 215, 216],"tenantId":"xyzOrg/"}'

 - `entityIds` contains list of objects which will be assigned to the entity.
 - An object cannot be shared across multiple tenants.
 - If an entire source is assigned to a tenant, the tenant becomes the owner of the source and it cannot be unassigned. The tenant must in this case unregister the source.

## Creating a Tag Based Protection Job
[Reference Doc](https://developer.cohesity.com/apidocs-64.html#/rest/api-endpoints/protection-jobs/create-protection-job)
>curl -X POST \
  https://`FQDN`/irisservices/api/v1/public/protectionJobs \
  -H 'Accept: */*' \
  -H 'Authorization: `TOKEN`' \
  -H 'Content-Type: application/json' \
  -H 'x-impersonate-tenant-id: xyzOrg/' \
  -d '{
    "name": "Tag Based Job Name",
    "environment": "kVMware",
    "parentSourceId": 1,
    "vmTagIds": [
        [
            1707
        ]
    ],
    "priority": "kMedium",
    "alertingPolicy": [
        "kFailure"
    ],
    "incrementalProtectionSlaTimeMins": 60,
    "fullProtectionSlaTimeMins": 120,
    "qosType": "kBackupHDD",
    "environmentParameters": {
        "vmwareParameters": {
            "fallbackToCrashConsistent": true
        }
    },
    "indexingPolicy": {
        "disableIndexing": false,
        "allowPrefixes": [
            "/"
        ],
        "denyPrefixes": [
            "/$Recycle.Bin",
            "/Windows",
            "/Program Files",
            "/Program Files (x86)",
            "/ProgramData",
            "/System Volume Information",
            "/Users/*/AppData",
            "/Recovery",
            "/var",
            "/usr",
            "/sys",
            "/proc",
            "/lib",
            "/grub",
            "/grub2"
        ]
    },
    "policyId": "2425993104764203:1568297549728:1",
    "viewBoxId": 2487759
}'

 - `x-impersonate-tenant-id` must be set in the headers.
 - `viewBoxId` is the storage domain id (set earlier)
 - `policyId` is the policy for the protection job.

# Using POSTMAN collection
 - Import the Cohesity-tenant.postman_collection.json JSON file in POSTMAN
 - Set environment variables, `host=https://CLUSTER-FQDN/irisservices/api/v1` and `auth=<BEARER-TOKEN>`
 - Start consuming the APIs
