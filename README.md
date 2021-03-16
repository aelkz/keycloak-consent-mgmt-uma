# KEYCLOAK CONSENT MANAGEMENT WITH UMA 2.0 AND AUTHORIZATION SERVICES

### Resource Server Definition
According to the OAuth2 specification, a resource server is a server hosting the protected resources and capable of accepting and responding to protected resource requests. Any client application can be configured to support fine-grained permissions. In doing so, you are conceptually turning the client application into a resource server.

### Requesting Party Token
In Keycloak Authorization Services the access token with permissions is called a Requesting Party Token or RPT for short.

### Well-known for UMA2
curl -X GET http://${host}:${port}/auth/realms/${realm}/.well-known/uma2-configuration

### Create a new namespace and setup the environment variables
```
SSO_HTTP_SCHEME=http
SSO_URL=127.0.0.1:8080

SSO_MASTER_REALM=master
SSO_MASTER_REALM_USERNAME=root
SSO_MASTER_REALM_PASSWORD=12345

SSO_OBK_REALM=openbanking
SSO_OBK_CONSENT_MGMT_CLIENT_ID=consent-mgmt-resource-server
SSO_OBK_CONSENT_MGMT_CLIENT_SECRET=e19682d2-03c2-40fb-ae45-b1237e29131d
SSO_OBK_PISP_CLIENT_ID=pisp-api # nubank, mercadopago, cielo, etc.
SSO_OBK_PISP_CLIENT_SECRET=2e475cc4-0faf-476d-a527-0c72536493ea
SSO_OBK_USER_NAME=raphael
SSO_OBK_USER_PWD=12345

SSO_AUTH_URL=${SSO_HTTP_SCHEME}://${SSO_URL}/auth
SSO_MASTER_TOKEN_URL=${SSO_HTTP_SCHEME}://${SSO_URL}/auth/realms/master/protocol/openid-connect/token

SSO_OBK_TOKEN_URL=${SSO_HTTP_SCHEME}://${SSO_URL}/auth/realms/${SSO_OBK_REALM}/protocol/openid-connect/token
SSO_OBK_REALM_KEYS_URL=${SSO_HTTP_SCHEME}://${SSO_URL}/auth/admin/realms/${SSO_OBK_REALM}/keys
SSO_OBK_CLIENTS_URL=${SSO_HTTP_SCHEME}://${SSO_URL}/auth/admin/realms/${SSO_OBK_REALM}/clients
```

### 1- Setup initial configurations

01.png

##### 1-1 Create realm `openbanking` with UMA option enabled
##### 1-2 Create user `raphael` under realm `openbanking` with password `12345`
##### 1-3 Create the `openid-connect` clients with the names `consent-mgmt-resource-server` and `pisp-api`
##### 1-4 Set `Service Accounts Enabled` to `true` and `Access Type` to `Confidential` on `pisp-api` client
##### 1-5 Set `Authorization Enabled` to `true` and `Access Type` to `Confidential` on `consent-mgmt-resource-server` client 
##### 1-6 Create the following realm roles: `account_information_service_provider`, `payment_initiation_service_provider`, `user` and `account_owner`
##### 1-7 Add `payment_initiation_service_provider` realm role to `pisp-api` client

### 2- Configure `consent-mgmt-resource-server` Authorization details
Open the `Authorization` tab on `consent-mgmt-resource-server` client.

##### 2-1 Enable the following options on `Settings`

02.png

##### 2-2 Open `Authorization Scopes` tab. Create the following `Scope`s: `bank-account:basic`, `bank-account:detail`, `bank-balance:read`, `bank-statement:basic`, `bank-statement:detail`

03.png

##### 2-3 Open `Resources` tab. That's the place we're creating the resources for every `customer` for our banking solution.
PS. As we cannot create `resources` to a specific owner but `consent-mgmt-resource-server` itself, we'll need to rely on Keycloak RESTful API. For this we will use the `Protection API` from Keycloak

Protection API:
```
An important requirement for this API is that only resource servers are allowed to access its endpoints using a special OAuth2 access token called a protection API token (PAT). In UMA, a PAT is a token with the scope uma_protection.
A protection API token (PAT) is a special OAuth2 access token with a scope defined as uma_protection. When you create a resource server, Keycloak automatically creates a role, uma_protection, for the corresponding client application and associates it with the client’s service account.

By default, the owner of a resource is the resource server. If you want to define a different owner, such as an specific user, you need to change the owner through the API.
```

Execute the following steps on a terminal:
```
CONSENT_MGMT_RESOURCE_SERVER_PAT=$(curl -k -v -X POST $SSO_OBK_TOKEN_URL \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data "grant_type=client_credentials" \
  --data "client_id=${SSO_OBK_CONSENT_MGMT_CLIENT_ID}" \
  --data "client_secret=${SSO_OBK_CONSENT_MGMT_CLIENT_SECRET}" \
  | sed 's/.*access_token":"//g' | sed 's/".*//g')
```

```
curl -k -v -X POST ${SSO_AUTH_URL}/realms/${SSO_OBK_REALM}/authz/protection/resource_set \
  -H "Authorization: Bearer ${CONSENT_MGMT_RESOURCE_SERVER_PAT}" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "name":"'${SSO_OBK_USER_NAME}'-bank-account",
    "displayName":"bank account resource for '${SSO_OBK_USER_NAME}' user",
    "owner": "'${SSO_OBK_USER_NAME}'",
    "ownerManagedAccess": true,
    "resource_scopes" : ["bank-account:basic","bank-account:detail"]
  }'

curl -k -v -X POST ${SSO_AUTH_URL}/realms/${SSO_OBK_REALM}/authz/protection/resource_set \
  -H "Authorization: Bearer ${CONSENT_MGMT_RESOURCE_SERVER_PAT}" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "name":"'${SSO_OBK_USER_NAME}'-bank-balance",
    "displayName":"bank balance resource for '${SSO_OBK_USER_NAME}' user",
    "owner": "'${SSO_OBK_USER_NAME}'",
    "ownerManagedAccess": true,
    "resource_scopes" : ["bank-balance:read"]
  }'

curl -k -v -X POST ${SSO_AUTH_URL}/realms/${SSO_OBK_REALM}/authz/protection/resource_set \
  -H "Authorization: Bearer ${CONSENT_MGMT_RESOURCE_SERVER_PAT}" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "name":"'${SSO_OBK_USER_NAME}'-bank-statement",
    "displayName":"bank statement resource for '${SSO_OBK_USER_NAME}' user",
    "owner": "'${SSO_OBK_USER_NAME}'",
    "ownerManagedAccess": true,
    "resource_scopes" : ["bank-statement:basic","bank-statement:detail"]
  }'   
```

Where `owner` is the username from Keycloak user.
It's also possible to create a resource with attributes using `attributes":{}`.
To create resources and allow resource owners to manage these resources, you must set `ownerManagedAccess` property.

PS. Don't remove the `Default Resource` resource. 

04.png

05.png

07.png

### 3- User-Managed Access: Authorization Process

##### 3-1 PISP-API will request some `resource` to `raphael` user under `consent-mgmt-api` UMA authorization subsystem.
Acquire pisp-api access-token as it will behave as a user (service account) This client uses it's own `client-secret` as password.
Clients need to authenticate to the token endpoint in order to exchange it to an RPT.

```
In Keycloak Authorization Services the access token with permissions is called a Requesting Party Token or RPT for short.
The RPT is a access-token with:

 "authorization" : {
   "permissions": [
     {
       rsid -> the resource id
       rsname -> the resource name
     }
       ..
   ]
 }
 ```

##### 3-2 Acquire PISP-API access token and then exchange it to a new 'RAW' RPT (with keycloak default permissions)
```
PISP_API_TKN=$(curl -v -k -X POST $SSO_OBK_TOKEN_URL \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "client_id=${SSO_OBK_PISP_CLIENT_ID}" \
 -d "client_secret=${SSO_OBK_PISP_CLIENT_SECRET}" \
 -d "grant_type=client_credentials" \
 | sed 's/.*access_token":"//g' | sed 's/".*//g')
 ```

```
PISP_API_RAW_RPT=$(curl -v -k -X POST $SSO_OBK_TOKEN_URL \
  -H "Authorization: Bearer ${PISP_API_TKN}" \
  --data "grant_type=urn:ietf:params:oauth:grant-type:uma-ticket" \
  --data "audience=${SSO_OBK_CONSENT_MGMT_CLIENT_ID}" \
  --data "response_include_resource_name=true" \
  | sed 's/.*access_token":"//g' | sed 's/".*//g')
```

Analyze the encoded RPT with [JWT.io](https://jwt.io). At this time you'll get only the `rsname`:"Default Resource".

##### 3-3 Acquire PISP-API access token and then exchange it to a new RPT (with `consent-mgmt-resource-server` additional permission requests)

<b>Use case example:</b>
Consider a user Alice (resource owner) using an Internet Banking Service (resource server) to manage his Bank Account (resource). One day, Alice decides to open her bank account to Bob (requesting party), a accounting professional. However, Bob should only have access to view (scope) Alice’s account.<br>
As a resource server, the Internet Banking Service must be able to protect Alice’s Bank Account. For that, it relies on Keycloak Resource Registration Endpoint to create a resource in the server representing Alice’s Bank Account.

###### 3-3-A Acquire the `SSO_OBK_CONSENT_MGMT_CLIENT_ID` specific UUID using the credentials from realm `master` user:
```
MASTER_ADMIN_TKN=$(curl -v -k -X POST $SSO_MASTER_TOKEN_URL \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=$SSO_MASTER_REALM_USERNAME" \
 -d "password=$SSO_MASTER_REALM_PASSWORD" \
 -d "grant_type=password" \
 -d "client_id=admin-cli" \
 | sed 's/.*access_token":"//g' | sed 's/".*//g')

SSO_OBK_CONSENT_MGMT_CLIENT_UUID=$(curl -v -k -X GET "$SSO_OBK_CLIENTS_URL?clientId=$SSO_OBK_CONSENT_MGMT_CLIENT_ID&first=0&max=1&search=true" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $MASTER_ADMIN_TKN" \
  | jq -r '.[0].id')
```

###### 3-3-B Also, lets acquire the `SSO_OBK_PISP_CLIENT_ID` specific UUID using also the credentials from realm `master` user:

```  
SSO_OBK_PISP_CLIENT_UUID=$(curl -v -k -X GET "$SSO_OBK_CLIENTS_URL?clientId=$SSO_OBK_PISP_CLIENT_ID&first=0&max=1&search=true" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $MASTER_ADMIN_TKN" \
  | jq -r '.[0].id')
```

###### 3-3-C Now, with the client `SSO_OBK_CONSENT_MGMT_CLIENT_ID` UUID's, let's acquire all `resource`s UUIDs by its own unique `name`:
```
# The name of the resource that PISP wants to be granted
SSO_OBK_RESOURCE_1=${SSO_OBK_USER_NAME}-bank-account
SSO_OBK_RESOURCE_2=${SSO_OBK_USER_NAME}-bank-statement

# Also, lets define the scopes for each resource that PISP wants to be granted
SSO_OBK_RESOURCE_1_SCOPE=bank-account:basic
SSO_OBK_RESOURCE_2_SCOPE=bank-statement:basic

# The resource search URI that will be located under SSO_OBK_CONSENT_MGMT_CLIENT_UUID (It's also possible to use the &owner=raphael attribute)
SSO_OBK_RESOURCE_1_URL="${SSO_HTTP_SCHEME}://${SSO_URL}/auth/admin/realms/${SSO_OBK_REALM}/clients/${SSO_OBK_CONSENT_MGMT_CLIENT_UUID}/authz/resource-server/resource?deep=false&first=0&max=1&name=${SSO_OBK_RESOURCE_1}"
SSO_OBK_RESOURCE_2_URL="${SSO_HTTP_SCHEME}://${SSO_URL}/auth/admin/realms/${SSO_OBK_REALM}/clients/${SSO_OBK_CONSENT_MGMT_CLIENT_UUID}/authz/resource-server/resource?deep=false&first=0&max=1&name=${SSO_OBK_RESOURCE_2}"

# You might want to acquire the MASTER_ADMIN_TKN again if it has been expired (step 3-3-A)
SSO_OBK_CONSENT_MGMT_RESOURCE_1_UUID=$(curl -v -k -X GET "$SSO_OBK_RESOURCE_1_URL" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $MASTER_ADMIN_TKN" \
  | jq -r '.[0]._id')

SSO_OBK_CONSENT_MGMT_RESOURCE_2_UUID=$(curl -v -k -X GET "$SSO_OBK_RESOURCE_2_URL" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json;charset=UTF-8" \
  -H "Authorization: Bearer $MASTER_ADMIN_TKN" \
  | jq -r '.[0]._id')
```

06.png

###### 3-3-D With all resources UUID's, let's request the RPT in order to acquire the `permission ticket`:

TODO

### EXTERNAL REFERENCES

https://stackoverflow.com/questions/54568012/keycloak-include-permission-into-the-access-token-along-with-the-roles<br>
https://stackoverflow.com/questions/64073855/how-to-get-requesting-party-token-rpt-by-api-in-keycloak<br>
https://stackoverflow.com/questions/42186537/resources-scopes-permissions-and-policies-in-keycloak<br>
https://github.com/CarrettiPro/keycloak-uma-delegation-poc
https://github.com/lbroudoux/spring-boot-keycloak-authz
https://www.youtube.com/watch?v=yosg4St0iUw&ab_channel=DevConf
https://stackoverflow.com/questions/58791070/keycloak-authorization-services-dont-deny-scopes-in-resource
https://medium.com/@ravthiru/rest-service-protected-using-keycloak-authorization-services-a6ad2d8ecb9f  
