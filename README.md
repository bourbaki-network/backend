# Bourbaki.network backend

- [Bourbaki.network backend](#bourbakinetwork-backend)
- [Third-party OAuth providers](#third-party-oauth-providers)
  - [Install](#install)
- [Hosted OAuth 2.0](#hosted-oauth-20)
  - [Setup](#setup)
    - [Configure Keycloak](#configure-keycloak)
    - [Get auth token](#get-auth-token)


We use a backend data layer ([Hasura](https://hasura.io)) to connect to a postgresql database and expose graphQL APIs.

There are two types of install of this depending on the kind of auth desired.
1. Custom authentication, via keycloak
2. Using OAuth providers such as Google etc

# Third-party OAuth providers

We use [Hasura Backend Plus](https://github.com/nhost/hasura-backend-plus) here which supports integration with Google, GitHub, Facebook, Apple, Twitter, Microsoft Live, Linkedin, and Spotify.

## Install

Pass env variables, update `.env`:

```bash
PGPASSWORD="generate-a-password-for-postgres"
PGUSER="postgres-username"
ADMIN_SECRET="generate-an-admin-password"
AUTH_SERVER="auth-dev.setu.co"
REALM="whatsapp"
REGION="aws s3 region"
S3_BUCKET_NAME="aws s3 bucket name"
AWS_ACCESS_KEY_ID="aws access key"
S3_SECRET_ACCESS_KEY="aws secret"
```

```
source .env
```

Build and start the backend:

```bash
docker volume create --name=hasura-data

docker-compose up
```

Cleanup afterward:

```bash
docker volume rm hasura-data
```

# Hosted OAuth 2.0

Deploy openresty for token introspection before hasura. Generate the openresty introspection script:

```bash
export AUTH_SERVER="keycloak:8000"
export CLIENT_ID="god"
export SECRET="<keycloak-secret-here>"
export REALM="master"

cat openresty.lua.template | \
			awk '{gsub("{{AUTH_SERVER}}", ENVIRON["AUTH_SERVER"], $0); print}' | \
			awk '{gsub("{{CLIENT_ID}}", ENVIRON["CLIENT_ID"], $0); print}' | \
      awk '{gsub("{{REALM}}", ENVIRON["REALM"], $0); print}' | \
			awk '{gsub("{{SECRET}}", ENVIRON["SECRET"], $0); print}' \
			> openresty.lua
```

Build container:

```bash
docker volume create --name=hasura-data

docker-compose -f docker-compose-auth.yaml build --no-cache
docker-compose -f docker-compose-auth.yaml up
```

## Setup

### Configure Keycloak

1. Choose an appropriate realm or create a realm
2. Create a client with:
   1. Client Protocol: `openid-connect`
   2. Access Type: `Confidential`
   3. Client ID `whatsapp-openresty-nginx`
   4. Standard Flow Enabled: `Off`
   5. Implicit Flow Enabled: `Off`
   6. Direct Access Grants Enabled: `Off`
   7. Service Accounts Enabled: `On`
   8. Authorization Enabled: `Off`
   9. Roles
      1.  Add a new role with name `user`
  10. Add role mappers (ends up in JWT auth tokens)
      1. `x-hasura-allowed-roles`:
         - Token Claim Name: `https://hasura\.io/jwt/claims.x-hasura-allowed-roles`
         - Mapper Type: `User Client Role`
         - Client ID: `god`
         - Claim JSON Type: `String`
         - Multivalued: `ON`
         - Name: `x-hasura-allowed-roles`
      3. `x-hasura-default-role`:
         - Token Claim Name: `https://hasura\.io/jwt/claims.x-hasura-default-role`
         - Claim Value: `user`
      4. `x-hasura-user-id`:
         - Token Claim Name: `https://hasura\.io/jwt/claims.x-hasura-user-id`
         - Claim Value: `1`
      5. `x-hasura-org-id`: 0 [Optional]
      6. `x-hasura-custom`: "" [Optional]

Verify settings are correct:

1. Fetch a token
```bash
RESULT=`/usr/bin/curl --data "grant_type=client_credentials&client_id=${CLIENT_ID}&client_secret=${SECRET}" https://${AUTH_SERVER}/auth/realms/${REALM}/protocol/openid-connect/token`
TOKEN=`echo $RESULT | sed 's/.*access_token":"\([^"]*\).*/\1/'
```

2. Check it on [jwt.io](https://jwt.io)

Ensure this part exists and conforms to the specs [here](https://hasura.io/docs/latest/graphql/core/auth/authentication/jwt.html#the-spec):

```json
  "https://hasura.io/jwt/claims": {
    "x-hasura-default-role": "user",
    "x-hasura-user-id": 32,
    "x-hasura-allowed-roles": ["user"]
  },
```

Go to the Hasura dashboard. Under "Request Headers", you can now add an authorization Header with the value "Bearer " followed by the JWT you copied earlier:

### Get auth token

```bash
RESULT=`/usr/bin/curl --data "grant_type=client_credentials&client_id=${CLIENT_ID}&client_secret=${SECRET}" https://${AUTH_SERVER}/auth/realms/${REALM}/protocol/openid-connect/token`
TOKEN=`echo $RESULT | sed 's/.*access_token":"\([^"]*\).*/\1/'`
```
