## Using HashiCorp Vault in CloudFoundry  

#### Running locally

1. Start vault:

```bash
vault server -config inmemory.conf
```

2. In another terminal set the VAULT_ADDR before initializing Vault:

```bash
export VAULT_ADDR=http://127.0.0.1:8200
```   

2. Initialize vault:

```bash
vault init -key-shares=5 -key-threshold=2
```

3. Copy the `Initial Root Token` we will still need it.

```bash
export VAULT_TOKEN=<token>
```

Vault requires an authenticated access to proceed from here on. 
Vault uses tokens as generic authentication on its transport level.

4. Vault is in `sealed` mode, let's unseal it:

```
vault unseal <key>
vault unseal <key>
```

5. Verify that Vault is in `unsealed` mode:

```bash
vault status | grep Sealed

Sealed: false
```

6. Write a secret into the `secret` backend:

```bash
vault write secret/vault-demo message='I find your lack of faith disturbing.'
```

7. Start the application in another terminal 

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=<token>
java -jar target/cloudfoundry-with-vault-demo-0.0.1-SNAPSHOT.jar --spring.cloud.vault.token=`echo $VAULT_TOKEN`
```

8. Request the GET localhost:8080

```bash
http :8080

message:I find your lack of faith disturbing.
```

9. Update the secret inside Vault

```bash
vault write secret/vault-demo message='Now, young Skywalker, you will die.'
```

10. Verify that the application still has the old secret

```bash
http :8080

message:I find your lack of faith disturbing.
```

11. Send refresh command to the application

```bash
http post :8080/actuator/refresh
```

12. Verify that the application knows about the latest secret

```bash
http :8080

message:'Now, young Skywalker, you will die.'
```

## Running on CloudFoundry

1. Using [Swisscom's CloudFoundry environment](developer.swisscom.com)

```bash
cf login -a https://api.lyra-836.appcloud.swisscom.com -u <login>
cf target -o mimacom -s Development
```

2. Get the [Open Service Broker API](https://www.openservicebrokerapi.org/) implementation from HashiCorp:

```bash
git clone https://github.com/hashicorp/vault-service-broker
```

3. Needed to change the `DefaultServiceID` and `DefaultServiceName` in the `main.go` file

4. Deploy the broker
   
```bash
cf push my-vault-broker-service -m 256M --random-route --no-start 
```
   
The `--no-start` makes sure it is not started after it is deployed.

5. Expose the locally running vault via [ngrok](https://ngrok.com/)

```
ngrok http 8200

Forwarding http://3db1eef2.ngrok.io -> localhost:8200
Forwarding https://3db1eef2.ngrok.io -> localhost:8200
```

6. Verify on the web interface at `http://localhost:4040`

7. Set the following environment variables

```bash
VAULT_ADDR=<ngrok_url>
VAULT_TOKEN=<token>
```

The broker is configured to use basic authentication

```bash
VAULT_USERNAME=vault
VAULT_PASSWORD=secret
```

8. Configure the environment variables

```bash
cf set-env my-vault-broker-service VAULT_ADDR ${VAULT_ADDR}
cf set-env my-vault-broker-service VAULT_TOKEN ${VAULT_TOKEN}
cf set-env my-vault-broker-service SECURITY_USER_NAME ${VAULT_USERNAME}
cf set-env my-vault-broker-service SECURITY_USER_PASSWORD ${VAULT_PASSWORD}
```

9. Verify the configured environment variables 

```bash
cf env my-vault-broker-service
```

10. Start the broker:

```bash
cf start my-vault-broker-service
```

11. Check the logs to verify the succesfull start

```bash
cf logs --recent my-vault-broker-service
```

12. Verify in the Ngrok Inspect UI the activity requests sent to the exposed Vault broker

```bash
GET /v1/cf/broker
POST /v1/sys/mounts/cf/broker
PUT /v1/auth/token/renew-self
GET /v1/sys/mounts
```
 
13. The service broker created a new mount
 
```
vault mounts

...
cf/broker/  generic    generic_4c6ea7ec    n/a     system       system   false           replicated
...
``` 

14. View the running broker:

```
cf apps

name                      requested state   instances   memory   disk   urls
my-vault-broker-service   started           1/1         256M     1G     my-vault-broker-service-meroblastic-econ.scapp.io
```  

15. Get the broker url:

```bash
VAULT_BROKER_URL=$(cf app my-vault-broker-service | grep routes: | awk '{print $2}')
```

16. Get the catalog information:

```bash
curl ${VAULT_USERNAME}:${VAULT_PASSWORD}@${VAULT_BROKER_URL}/v2/catalog | jq
```

```json
{
  "services": [
    {
      "id": "42ff1ff1-244d-413a-87ab-b2334b801134",
      "name": "my-hashicorp-vault",
      "description": "HashiCorp Vault Service Broker",
      "bindable": true,
      "tags": [
        ""
      ],
      "plan_updateable": false,
      "plans": [
        {
          "id": "42ff1ff1-244d-413a-87ab-b2334b801134.shared",
          "name": "shared",
          "description": "Secure access to Vault's storage and transit backends",
          "free": true
        }
      ]
    }
  ]
}
```

17. Create a service broker:

```bash
cf service-brokers
cf create-service-broker my-vault-service-broker "${VAULT_USERNAME}" "${VAULT_PASSWORD}" "https://${VAULT_BROKER_URL}" --space-scoped
```

You need to specify the `--space-scoped` and the `service ids` and `service name` must be unique. See `https://docs.cloudfoundry.org/services/managing-service-brokers.html`

18. Create a service instance:

```bash
cf create-service my-hashicorp-vault shared my-vault-service
``` 

19. Verify the result:

```bash
cf services

name               service              plan     bound apps   last operation
my-vault-service   my-hashicorp-vault   shared                create succeeded
```

20. Verify the HTTP requests sent the exposed Vault service using the Ngrok Inspect UI:

```bash
PUT /v1/cf/broker/b6f97d4c-528e-433c-869a-e396069d0b94
POST /v1/sys/mounts/cf/b6f97d4c-528e-433c-869a-e396069d0b94/secret
POST /v1/sys/mounts/cf/b6f97d4c-528e-433c-869a-e396069d0b94/transit
POST /v1/sys/mounts/cf/cf4e1f52-79a1-42f1-9885-35484fbf0cd6/secret
POST /v1/sys/mounts/cf/dfc8434a-2ec5-40b1-bd67-86871b5026c0/secret
GET /v1/sys/mounts
PUT /v1/auth/token/roles/cf-b6f97d4c-528e-433c-869a-e396069d0b94
PUT /v1/sys/policy/cf-b6f97d4c-528e-433c-869a-e396069d0b94
```

When  a new service instance is provisioned using the broker, the following paths will be mounted:

Mount the generic backend at /cf/<organization_id>/secret/
Mount the generic backend at /cf/<space_id>/secret/
Mount the generic backend at /cf/<instance_id>/secret/
Mount the transit backend at /cf/<instance_id>/transit/

A policy named `cf-<instance_id>` is also created for this service instance which grants read-only access to `cf/<organization_id>/*`, read-write access to `cf/<space_id>/*` and full access to `cf/<instance_id>/*`

21. Create a service key: (This failed in Swisscom's CloudFoundry)

```bash
cf create-service-key my-vault-service my-vault-service-key
cf service-keys my-vault-service
```

18. Verify the received requests for Vault using the Ngrok Inspect UI

```bash
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/auth/token/renew-self                                                               200 OK
PUT  /v1/cf/broker/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/a4a878ba-60ef-4476-9862-b78b0bb514d3 204 No Content
POST /v1/auth/token/create/cf-41b2d6df-f7d1-453e-98e5-9a0bd1b2c347                           200 OK
```

19. Retrieve credentials for this instance:

```bash
cf service-key my-vault-service my-vault-service-key
```

```json
{
 "address": "http://43cb69ee.ngrok.io/",
 "auth": {
  "accessor": "d994b172-c116-1d56-39c4-139f9615abb4",
  "token": "e340fdbe-373a-0b24-0a9c-806316b379a6"
 },
 "backends": {
  "generic": "cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret",
  "transit": "cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/transit"
 },
 "backends_shared": {
  "organization": "cf/be7eedf8-c813-49e1-98f8-2fc19370ee4d/secret",
  "space": "cf/5f7b0811-d90a-47f2-a194-951eb324f867/secret"
 }
}
```

In the application, we can leverage these services using the following configuration in the bootstrap.yml file. Note that we are only able to access the exposed backends.
`
```bash
spring:
  application:
    name: vault-demo
  cloud:
    vault:
      token: ${vcap.services.my-vault-service.credentials.auth.token}
      uri: ${vcap.services.my-vault-service.credentials.address:http://localhost:8200}
      generic:
        backend: ${vcap.services.my-vault-service.credentials.backends.generic:secret}
```

After redeploying with the above changes, let's write the secret into the vault to the given generic backend.

```bash
vault write cf/41b2d6df-f7d1-453e-98e5-9a0bd1b2c347/secret/vault-demo message='Vault Rocks'
http post http://vault-demo-deleterious-geum.cfapps.io/application/refresh
```

We can verify that the secret is retrieved via

```bash
http get http://vault-demo-deleterious-geum.cfapps.io
```

--


How can I connect apps running in PCF Dev to services running on my workstation?

Note: Using localhost inside of app containers will not refer to your workstation.

PCF Dev provides a special hostname for addressing the host from inside of application containers. 
If the PCF Dev system domain is local.pcfdev.io, then the host will be routable at host.pcfdev.io.
Services running on the host must be listening on all network interfaces (not just localhost) for apps to access them.

 
Look into:
Spring Cloud project for creating Cloud Foundry service brokers
`https://github.com/spring-cloud/spring-cloud-cloudfoundry-service-broker`

 
 
Resources:
[https://spring.io/blog/2016/06/24/managing-secrets-with-vault](https://spring.io/blog/2016/06/24/managing-secrets-with-vault)
[https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry](https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry)