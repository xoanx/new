# Demo 1: template-service

## Environment
- docker
- docker-compose
- kind
- kubectl
- helm
- gradle
- Java
- LENS

## Account
- Docker hub
## Build & Start project
### Option 1
### Build
```shell
# build
# ./gradlew clean build -x test
gradle clean build -x test
# start postgres
docker compose -f docker-compose-postgres.yml up -d
# run
java -jar build/libs/template-service-0.0.1-SNAPSHOT.jar 
```
### Option 2
### Stage 1 - Development with local postgres container
```shell
# build
./gradlew clean build -x test
gradle clean build -x test
# containerize & start service&db
docker compose -f docker-compose-build.yml up

## we might need to kill the process to free port 8080
# Clean stage:
docker compose -f docker-compose-postgres.yml down
``` 
### Stage 2 - Test image with local postgres container
```shell
# build docker image
docker-compose -f docker-compose-build.yml up -d --force-recreate --no-deps --build
# Clean stage:
docker compose -f docker-compose-build.yml down
```
## Test
## Test for 2 local stages
```shell
curl -X 'GET' \
  'http://localhost:8080/restapi/v1/products' \
  -H 'accept: application/json'
```
swagger: http://localhost:8080/swagger-ui/index.html
swagger: http://localhost:8080/swagger-ui/index.html


### Stage 3 - Deploy to a kubernetes cluster
## push image to docker hub
```shell
docker build . -t my-template-service
docker tag my-template-service tranhuuhoa/my-template-service:latest
docker login 
docker push tranhuuhoa/my-template-service:latest
```
## (Optional) Create a kubernetes with kind
```shell
cd k8s
kind create cluster --config ./kind-mscluster-config.yaml
kubectl cluster-info
``` 
### Deploy postgresql
```shell
helm install my-release oci://registry-1.docker.io/bitnamicharts/postgresql --set auth.enablePostgresUser=true --set auth.postgresPassword=postgres --set auth.database=template_db
```
## test password postgres
```shell
helm list
kubectl exec --stdin --tty my-release-postgresql-0 -- /bin/bash psql --host 127.0.0.1 -U postgres -d template_db -p 5432
```
### Deploy to kubernetes
```shell
kubectl run my-service --image=tranhuuhoa/my-template-service --env POSTGRES_DATASOURCE_URI=my-release-postgresql

kubectl logs -f my-service
## expose service to out side 
kubectl port-forward -n default my-service 8080:8080
```
# Demo 2: API and authentication
**Architecture:**

![Alt text](APISix-Keycloak.png)

```shell
# Build application
./gradlew clean build -x test

# Create data folder
mkdir postgres-data
# start environment
docker-compose -f docker-compose-idp.yml up -d

```

- Access Keycloak: http://localhost:8080/keycloak/ (admin/admin)
- Create client_secret of digital_realm/apisix_client

##### Step 1 - test API without authentication
```shell
curl -X 'GET' \
  'http://localhost:8000/restapi/v1/products' \
  -H 'accept: application/json'
```
##### Step 2 - Access APISIX dashboard
http://localhost:9000/
admin/admin

##### Step 3 - Define route
```shell
curl http://localhost:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
  "uri": "/restapi/*",
  "plugins":{
    "openid-connect":{
      "client_id": "apisix_client",
      "scope": "openid profile",
      "client_secret": "f2Z9kKqBxEBNYqpKv0NuGGQV6sSADz6Y",
      "discovery": "http://keycloak:8080/keycloak/realms/digital_realm/.well-known/openid-configuration",
      "introspection_endpoint": "http://keycloak:8080/keycloak/realms/digital_realm/protocol/openid-connect/token/introspect",
      "bearer_only": true,
      "realm": "digital_realm",
      "introspection_endpoint_auth_method": "client_secret_post"
    }
  },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "msservice:8080": 1
        }
    }
}'

```
- View route:
```shell
curl http://127.0.0.1:9180/apisix/admin/routes -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X GET
```
Or access APISIX dashboard

##### Step 4 - Login to get $ACCESS_TOKEN
```shell
curl -L -g 'http://localhost:8080/keycloak/realms/digital_realm/protocol/openid-connect/token' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-d 'grant_type=password' \
-d 'client_id=app_client' \
-d 'username=user_1' \
-d 'password=user_1' \
-d 'scope=openid'
```

##### Step 5 - Test API
```shell
#This should be fail
curl -X 'GET' \
  'http://localhost:9080/restapi/v1/products' \
  -H 'accept: application/json'
  
  
#this should be success
curl -X 'GET' \
  'http://localhost:9080/restapi/v1/products' \
  -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJHVVBrR3JoSnI1Q3JPXzE1elIwdlZrdHRpUTRkbXFjS3pDc3J6WTRxZkd3In0.eyJleHAiOjE2OTA0MzE0MTYsImlhdCI6MTY5MDQzMTExNiwianRpIjoiYTgyZWMwNGUtMTZhNi00YmI3LTg2YjUtZWYwZGEyOWU2ZmZiIiwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDAwL2tleWNsb2FrL3JlYWxtcy9kaWdpdGFsX3JlYWxtIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6ImVjM2Y4NjgwLWVjYTgtNDQwMy1hZjZjLWE3NDQzNjczODdhMSIsInR5cCI6IkJlYXJlciIsImF6cCI6ImFwcF9jbGllbnQiLCJzZXNzaW9uX3N0YXRlIjoiYmRlZmM0MzktNTc4ZC00MjA2LTg1MjItZTBkODM4OTEyNjBjIiwiYWNyIjoiMSIsImFsbG93ZWQtb3JpZ2lucyI6WyIvKiJdLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtZGlnaXRhbF9yZWFsbSJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIGVtYWlsIHByb2ZpbGUiLCJzaWQiOiJiZGVmYzQzOS01NzhkLTQyMDYtODUyMi1lMGQ4Mzg5MTI2MGMiLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwicHJlZmVycmVkX3VzZXJuYW1lIjoidXNlcl8xIiwiZ2l2ZW5fbmFtZSI6IiIsImZhbWlseV9uYW1lIjoiIiwiZW1haWwiOiJ1c2VyXzFAY2xhc3MuY29tIn0.EPSDHF_c_6Msf7pbU4LXKlyDJY_04_f3KF6U6FGsrDSB0JZSet8QmMlyvdUHSmZY_WNxjHts8BcLa_0fZwMwfPb0cQFpu7SNoWCdPkK3glrpqHXxO4RYSFPQoZ1OEtNu7Emf6hnM_96IrJzGCafCKJdQkujfZriY_EvdrSYwsH6H11T7_-bGADfpwpnPfZIObZ890CNhgHC9reO39ZJcCuUPbCx7BT-qMkjab6pmayuiTklrOmo8MGOx_xbcqtqRwGPgVmCKwPunVNbflhWe4z4vhac2lHzqDvV1K7R8bYjn2s1jbO9VsWoV9ngLGwewWaMJmzvXwxihtjH0dovF8w' \
  -H 'accept: application/json'
```
