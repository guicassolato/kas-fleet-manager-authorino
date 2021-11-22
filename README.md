# Kafka Fleet Manager with Authorino

This is a PoC on protecting [Kafka Service Fleet Manager](https://api.openshift.com/?urls.primaryName=kafka%20service%20fleet%20manager%20service) with [Authorino](https://github.com/kuadrant/authorino).

## 1. Setup the environment

### Start the Kubernetes cluster
```sh
kind create cluster --name kas-fleet-manager-demo
```

### Deploy the Authorino Operator
```sh
git clone https://github.com/kuadrant/authorino-operator && cd authorino-operator
kubectl create namespace authorino-operator && make install deploy
```

### Create a namespace
```sh
kubectl create namespace authorino
```

### Deploy an API that pretends to be Kafka Service Fleet Manager
The _Talker API_ is just an echo API, included with the Authorino examples. We will use it to simulate Kafka Service Fleet Manager.

```sh
kubectl -n authorino apply -f https://raw.githubusercontent.com/Kuadrant/authorino/main/examples/talker-api/talker-api-deploy.yaml
```

### Deploy an Authorino instance
```sh
kubectl -n authorino apply -f -<<EOF
apiVersion: operator.authorino.kuadrant.io/v1beta1
kind: Authorino
metadata:
  name: authorino
spec:
  logLevel: debug
  listener:
    tls:
      enabled: false
  oidcServer:
    tls:
      enabled: false
EOF
```

### Setup Envoy
The following bundle from the Authorino examples (commands below) sets up the Envoy proxy, wiring up the Talker API behind the reverse-proxy and the Authorino instance to the external authorization HTTP filter.

```sh
curl -L https://raw.githubusercontent.com/Kuadrant/authorino/main/examples/envoy/overlays/notls/configmap.yaml | sed -E 's/authorino-authorization/authorino-authorino-authorization/g;s/ timeout: 1s/ timeout: 3s/g' | kubectl -n myapp apply -f -
kubectl -n authorino apply -f https://raw.githubusercontent.com/Kuadrant/authorino/main/examples/envoy/base/envoy.yaml
```

In this demo, Kafka Service Fleet Manager (actually the Talker API), Envoy and Authorino all run within their own separate pods. In a real-life scenario, Envoy and Authorino would likely be deployed as sidecars of Kafka Service Fleet Manager instead.

Forward requests on port 8000 to inside the cluster in order to actually reach the Envoy service:

```sh
kubectl -n authorino port-forward deployment/envoy 8000:8000 &
```

### Apply the `AuthConfig`
```sh
kubectl -n authorino apply -f -<<EOF
apiVersion: authorino.3scale.net/v1beta1
kind: AuthConfig
metadata:
  name: kas-fleet-manager-protection
spec:
  hosts:
  - api.openshift.com
  - api.stage.openshift.com
  - kas-fleet-manager.127.0.0.1.nip.io # local demo
  identity:
  - name: redhat-external
    oidc:
      endpoint: https://sso.redhat.com/auth/realms/redhat-external
  metadata:
  - name: ams-current-account
    http:
      endpoint: https://api.openshift.com/api/accounts_mgmt/v1/current_account
      method: GET
      headers:
        - name: Authorization
          valueFrom:
            authJSON: context.request.http.headers.authorization
  - name: quota-cost
    http:
      endpoint: https://api.openshift.com/api/accounts_mgmt/v1/organizations/{auth.metadata.ams-current-account.organization.id}/quota_cost?fetchRelatedResources=true&search=allowed%20%3E%200
      method: GET
      headers:
        - name: Authorization
          valueFrom:
            authJSON: context.request.http.headers.authorization
    priority: 1
  authorization:
  - name: quota-check
    opa:
      inlineRego: |
        import input.context.request.http as req

        create_kafka_instance {
          req.method == "POST"
          req.path == "/api/kafkas_mgmt/v1/kafkas"
        }

        quota = object.get(input.auth.metadata, "quota-cost", {})

        quota_type_std__billing_model_std {
          quota.items[i].related_resources[j].resource_name == "rhosak"
          quota.items[i].related_resources[j].product == "RHOSAK"
          quota.items[i].related_resources[j].billing_model == "standard"
        }

        quota_type_std__billing_model_marketplace {
          quota.items[i].related_resources[j].resource_name == "rhosak"
          quota.items[i].related_resources[j].product == "RHOSAK"
          quota.items[i].related_resources[j].billing_model == "marketplace"
        }

        allow { not create_kafka_instance }
        allow { create_kafka_instance; quota_type_std__billing_model_std }
        allow { create_kafka_instance; quota_type_std__billing_model_marketplace }
EOF
```

## 2. Consume Kafka Service Fleet Manager

### Get an offline token
You can request an offline token at https://console.redhat.com/openshift/token.

Check your existing offline tokens at https://sso.redhat.com/auth/realms/redhat-external/account/applications (though you will not be able to see the secret value of the token there).

### Login with OCM
```sh
~/.oc/bin/ocm login --token="<offline-token>"
```

### Send requests
```sh
curl -H "Authorization: Bearer $(~/.oc/bin/ocm token)" -X POST http://kas-fleet-manager.127.0.0.1.nip.io:8000/api/kafkas_mgmt/v1/kafkas
```

## 3. Cleanup
```sh
kind delete cluster --name kas-fleet-manager-demo
```
