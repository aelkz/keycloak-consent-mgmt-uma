### resource server definition
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
```

### 1- Create realm `openbanking`
### 2- Create user `raphael` under realm `openbanking` with password `12345`
### 3- Create client `consent-mgmt-api` and `pisp-api`
### 4- Set authorization model on `consent-mgmt-api` (import the .json file)
### 5- Add `payment_initiation_service_provider` realm role to `pisp-api` client

##### PISP-API will request some `resource` to `raphael` user under `consent-mgmt-api` authorization subsystem (UMA).
Acquire pisp-api access-token as it will behave as a user (service account) This client uses it's own client-secret as password.
Clients need to authenticate to the token endpoint in order to obtain an RPT.

```
PISP_API_TKN=$(curl -v -k -X POST $SSO_OBK_TOKEN_URL \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "client_id=${SSO_OBK_PISP_CLIENT_ID}" \
 -d "client_secret=${SSO_OBK_PISP_CLIENT_SECRET}" \
 -d "grant_type=client_credentials" \
 | sed 's/.*access_token":"//g' | sed 's/".*//g')
 ```

##### As part of the authorization process, clients need first to obtain a permission ticket from a UMA protected resource server in order to exchange it with an RPT at the Keycloak Token Endpoint.
Create a authorization request for user raphael. This is also called as REQUEST PERMISSION TICKET.

audience definition:<br>
The client identifier of the resource server to which the client is seeking access. This parameter is mandatory in case the permission parameter is defined. It serves as a hint to Keycloak to indicate the context in which permissions should be evaluated.

```
PISP_API_RPT=$(curl -X POST $SSO_OBK_TOKEN_URL \
  -H "Authorization: Bearer ${PISP_API_TKN}" \
  --data "grant_type=urn:ietf:params:oauth:grant-type:uma-ticket" \
  --data "audience=${SSO_OBK_CONSENT_MGMT_CLIENT_ID}" \
  --data "response_include_resource_name=true" \
  --data "permission=View Bank Statement#bank-statement:view" \
  | sed 's/.*access_token":"//g' | sed 's/".*//g')
```

Open the encoded RPT in JWT.io. You'll get something like:

```
"authorization": {
    "permissions": [
      {
        "scopes": [
          "bank-statement:view"
        ],
        "rsid": "46dcfb33-7c6c-40ad-ae2c-25550699f5af",
        "rsname": "View Bank Statement"
      }
    ]
  },
```

##### Protection API
An important requirement for this API is that only resource servers are allowed to access its endpoints using a special OAuth2 access token called a protection API token (PAT). In UMA, a PAT is a token with the scope uma_protection.

A protection API token (PAT) is a special OAuth2 access token with a scope defined as uma_protection. When you create a resource server, Keycloak automatically creates a role, uma_protection, for the corresponding client application and associates it with the client’s service account.

```
CONSENT_MGMT_RESOURCE_SERVER_PAT=$(curl -X POST $SSO_OBK_TOKEN_URL \
    -H "Content-Type: application/x-www-form-urlencoded" \
    --data "grant_type=client_credentials" \
    --data "client_id=${SSO_OBK_CONSENT_MGMT_CLIENT_ID}" \
    --data "client_secret=${SSO_OBK_CONSENT_MGMT_CLIENT_SECRET}" \
    | sed 's/.*access_token":"//g' | sed 's/".*//g')
```

##### Creating a resource for a specific user (not for the resource-server itself)
By default, the owner of a resource is the resource server. If you want to define a different owner, such as an specific user, you can send a request as follows:

```
curl -X POST ${SSO_AUTH_URL}/realms/${SSO_OBK_REALM}/authz/protection/resource_set \
  -H "Authorization: Bearer ${CONSENT_MGMT_RESOURCE_SERVER_PAT}" \
  -H "Content-Type: application/json" \
  --data-raw '{
    "name":"view-bank-statatement-for-'${SSO_OBK_USER_NAME}'",
    "displayName":"View Bank Statatement for '${SSO_OBK_USER_NAME}'",
    "owner": "'${SSO_OBK_USER_NAME}'",
    "ownerManagedAccess": true,
    "resource_scopes" : ["bank-statement:view"]
  }'
```

Where `owner` is the username from Keycloak user.
It's also possible to create a resource with attributes using `attributes":{}`.
To create resources and allow resource owners to manage these resources, you must set `ownerManagedAccess` property.


##### Enabling Party-to-Party Authorization
Use case example:
As an example, consider a user Alice (resource owner) using an Internet Banking Service (resource server) to manage his Bank Account (resource). One day, Alice decides to open her bank account to Bob (requesting party), a accounting professional. However, Bob should only have access to view (scope) Alice’s account.

As a resource server, the Internet Banking Service must be able to protect Alice’s Bank Account. For that, it relies on Keycloak Resource Registration Endpoint to create a resource in the server representing Alice’s Bank Account.

A UMA protected resource server expects a bearer token in the request where the token is an RPT. When a client requests a resource at the resource server without a permission ticket:

In some situations, client applications may want to start an asynchronous authorization flow and let the owner of the resources being requested decide whether or not access should be granted. For that, clients can use the `submit_request` request parameter along with an authorization request to the token endpoint. This parameter only has effect if used together with the ticket parameter as part of a UMA authorization process.

--- request access to a protected resource in order to get a permission ticket:
```
```
--- then,
You can think about this functionality as a `Request Access button in your application, where users can ask other users for access to their resources.

```
curl -X POST \
  http://${host}:${port}/auth/realms/${realm}/protocol/openid-connect/token \
  -H "Authorization: Bearer ${access_token}" \
  --data "grant_type=urn:ietf:params:oauth:grant-type:uma-ticket" \
  --data "ticket=${permission_ticket}
  --data "submit_request=true"
```
