# Kafka Fleet Manager with Authorino (POC)

This is a Proof of Concept (PoC) on protecting the [Kafka Management API](https://api.openshift.com/?urls.primaryName=kafka%20service%20fleet%20manager%20service) (a.k.a. Kafka Fleet Manager service, or **kas-fleet-manager**) with [Authorino](https://github.com/kuadrant/authorino).

## Architecture

![Architecture](architecture.svg)

## Install

<details>
  <summary><b>Meet the prerequisites</b></summary><br/>

  - [KAS Installer](https://github.com/bf2fc6cc711aee1a0c2a/kas-installer) tool
  - A custom version of kaf-fleet-manager that includes the ext-authz metadata endpoints such as [guicassolato/kas-fleet-manager](https://github.com/guicassolato/kas-fleet-manager/tree/ext-authz)
  - [OpenShift](https://www.openshift.com) (tested on OpenShift v4.11.7)
  - A user with administrative privileges in the OpenShift cluster
  - Red Hat user with permissions to pull the required container images
  - Application and Data Services [Service Account](https://console.redhat.com/application-services/service-accounts)
  - Kubernetes CLI (`kubectl`)
  - [OpenShift CLI](https://docs.openshift.com/container-platform/4.11/cli_reference/openshift_cli/getting-started-cli.html) (`oc`)
  - [yq](https://mikefarah.gitbook.io/yq/) (v0.4.x)

  Additional prerequisites on MacOS:
  - envsubst (`brew install gettext`)
</details>

<details>
  <summary><b>① Login to the cluster</b></summary><br/>

  Export the cluster domain to a shell variable:

  ```sh
  export K8S_CLUSTER_DOMAIN=<your-openshift-cluster-domain>
  ```

  Login to the target OpenShift cluster where you want to run the Kafka Fleet Manager API:

  ```sh
  oc login --token=<secret-token> --server=https://api.$K8S_CLUSTER_DOMAIN:6443
  ```
</details>

<details>
  <summary><b>② Run the Kafka Fleet Manager service</b></summary><br/>

  Make sure to match the [prerequisites](https://github.com/bf2fc6cc711aee1a0c2a/kas-installer#prerequisites) to use KAS Installer and then execute the steps below.

  Download KAS Installer:

  ```sh
  git clone git@github.com:bf2fc6cc711aee1a0c2a/kas-installer.git && cd kas-installer
  ```

  Replace the placeholders below with the corresponding values and create your custom KAS installer configuration file `kas-installer.env`:

  ```sh
  cat <<EOF>kas-installer.env
  K8S_CLUSTER_DOMAIN="${K8S_CLUSTER_DOMAIN}"

  USER=authorino

  RH_USERNAME="<redhat-username>"
  RH_USER_ID="<redhat-user-id>"
  RH_ORG_ID="<redhat-org-id>"

  OBSERVABILITY_CONFIG_REPO="https://api.github.com/repos/<github-org>/observability-resources-mk/contents"
  OBSERVABILITY_CONFIG_ACCESS_TOKEN=<github-pat-with-read-permission-on-your-fork-of-observability-resources-mk>

  IMAGE_REPOSITORY_USERNAME=<quay.io-username>
  IMAGE_REPOSITORY_PASSWORD=<quay.io-encrypted-password>

  SSO_PROVIDER_TYPE=redhat_sso
  REDHAT_SSO_HOSTNAME=sso.redhat.com
  REDHAT_SSO_CLIENT_ID=<service-account-client-id>
  REDHAT_SSO_CLIENT_SECRET=<service-account-client-secret>

  KAS_FLEET_MANAGER_IMAGE_REPOSITORY=guicassolato/kas-fleet-manager
  KAS_FLEET_MANAGER_IMAGE_TAG=ext-authz
  EOF
  ```

  Install Kafka Fleet Manager:

  ```sh
  ./kas-installer.sh
  ```

  The step above may take several minutes.
</details>

<details>
  <summary><b>③ Install Authorino</b></summary><br/>

  Install the Authorino Operator:

  ```sh
  kubectl apply -f -<<EOF
  apiVersion: v1
  kind: Namespace
  metadata:
    name: authorino-operator
  ---
  apiVersion: operators.coreos.com/v1
  kind: OperatorGroup
  metadata:
    name: kuadrant-operators
    namespace: authorino-operator
  spec:
    upgradeStrategy: Default
  ---
  apiVersion: operators.coreos.com/v1alpha1
  kind: Subscription
  metadata:
    name: authorino-operator
    namespace: authorino-operator
  spec:
    channel: alpha
    installPlanApproval: Automatic
    name: authorino-operator
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: authorino-operator.v0.4.1
  EOF
  ```

  Request an Authorino instance:

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: operator.authorino.kuadrant.io/v1beta1
  kind: Authorino
  metadata:
    name: authorino
  spec:
    image: quay.io/kuadrant/authorino:latest
    authConfigLabelSelectors: authorino.kuadrant.io/tenant=kas-fleet-manager
    secretLabelSelectors: authorino.kuadrant.io/secret in (api-key, x509),authorino.kuadrant.io/tenant=kas-fleet-manager
    listener:
      tls:
        enabled: false
      maxHttpRequestBodySize: 65536
    oidcServer:
      tls:
        enabled: false
  EOF
  ```

  Get TLS certificates for the Authorino services inssued by the OpenShift certificate issuer:

  ```sh
  kubectl -n kas-fleet-manager-authorino annotate service/authorino-authorino-authorization service.alpha.openshift.io/serving-cert-secret-name=authorino-authorino-authorization-tls
  kubectl -n kas-fleet-manager-authorino annotate service/authorino-authorino-oidc service.alpha.openshift.io/serving-cert-secret-name=authorino-authorino-oidc-tls
  ```

  Enable TLS in the Authorino instance:

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: operator.authorino.kuadrant.io/v1beta1
  kind: Authorino
  metadata:
    name: authorino
  spec:
    image: quay.io/kuadrant/authorino:latest
    authConfigLabelSelectors: authorino.kuadrant.io/tenant=kas-fleet-manager
    secretLabelSelectors: authorino.kuadrant.io/secret in (api-key, x509),authorino.kuadrant.io/tenant=kas-fleet-manager
    listener:
      tls:
        certSecretRef:
          name: authorino-authorino-authorization-tls
      maxHttpRequestBodySize: 65536
    oidcServer:
      tls:
        certSecretRef:
          name: authorino-authorino-oidc-tls
  EOF
  ```
</details>

<details>
  <summary><b>④ Enable external authorization</b></summary><br/>

  **Apply the AuthConfigs**

  ```sh
  curl -sL https://raw.githubusercontent.com/guicassolato/kas-fleet-manager-authorino/main/kas-fleet-manager-authconfig.yaml | envsubst | kubectl -n kas-fleet-manager-authorino apply -f -
  kubectl -n kas-fleet-manager-authorino apply -f https://raw.githubusercontent.com/guicassolato/kas-fleet-manager-authorino/main/kas-fleet-manager-pip-authconfig.yaml
  ```

  **Patch the Envoy configuration**

  Patch the Envoy config with the ext-authz filter:

  ```sh
  kubectl -n kas-fleet-manager-authorino get configmap/kas-fleet-manager-envoy-config -o jsonpath='{.data.main\.yaml}' > /tmp/envoy-config.yaml
  yq -i '
    .static_resources.clusters += {"name":"authorino","connect_timeout":"0.25s","type":"strict_dns","lb_policy":"round_robin","http2_protocol_options":{},"load_assignment":{"cluster_name":"authorino","endpoints":[{"lb_endpoints":[{"endpoint":{"address":{"socket_address":{"address":"authorino-authorino-authorization.kas-fleet-manager-authorino.svc","port_value":50051}}}}]}]},"transport_socket":{"name":"envoy.transport_sockets.tls","typed_config":{"@type":"type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext","common_tls_context":{"validation_context":{"trusted_ca":{"filename":"/etc/ssl/certs/authorino-ca-cert.crt"}}}}}} |
    .static_resources.listeners[1].filter_chains[0].filters[0].typed_config.http_filters = [{"name":"envoy.filters.http.ext_authz","typed_config":{"@type":"type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz","transport_api_version":"V3","failure_mode_allow":true,"grpc_service":{"envoy_grpc":{"cluster_name":"authorino"},"timeout":"1s"}}}] + .static_resources.listeners[1].filter_chains[0].filters[0].typed_config.http_filters
  ' /tmp/envoy-config.yaml
  sed -e 's/\x1B\[[0-9;]*[JKmsu]//g' -i /tmp/envoy-config.yaml

  kubectl -n kas-fleet-manager-authorino delete configmap kas-fleet-manager-envoy-config
  kubectl -n kas-fleet-manager-authorino create configmap kas-fleet-manager-envoy-config --from-file=main.yaml=/tmp/envoy-config.yaml
  ```

  Mount Authorino's TLS certs into the Envoy deployment:

  ```sh
  kubectl -n kas-fleet-manager-authorino patch deployment/kas-fleet-manager --type='json' -p='[
    {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"authorino-ca-cert","secret":{"defaultMode":420,"secretName":"authorino-authorino-authorization-tls"}}},
    {"op":"add","path":"/spec/template/spec/containers/1/volumeMounts/-","value":{"mountPath":"/etc/ssl/certs/authorino-ca-cert.crt","name":"authorino-ca-cert","readOnly":true,"subPath":"tls.crt"}}]'
  ```

  The command above will restart the Kafka Fleet Manager service.
</details>

<details>
  <summary><b>⑤ Test the installation</b></summary><br/>

  Test the deployment by sending requests to the Kafka Fleet Manager service. Assert conformity of the deployment based on the expected response codes according to the permissions of your user.

  List Kafka clusters:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/kafkas
  ```

  Create a Kafka cluster:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      -H 'Content-Type: application/json' \
      -d '{"name":"my-kafka","cloud_provider":"aws","region":"us-east-1","multi_az":false}' \
      "https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/kafkas?async=true"
  ```

  List supported cloud providers:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/cloud_providers
  ```

  List supported cloud provider regions (AWS):

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/cloud_providers/aws/regions
  ```

  List supported instance types (AWS us-east-1):

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/instance_types/aws/us-east-1
  ```

  List Service Accounts:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/service_accounts
  ```
</details>

<details>
  <summary><b>Cleanup</b></summary><br/>

  Decommission the Kafka Fleet Manager service:

  ```sh
  ./uninstall.sh
  ```

  Uninstall the Authorino Operator:

  ```sh
  kubectl delete namespace authorino-operator
  ```
</details>

## Leverage

On an installation of kas-fleet-manager protected with Authorino as above, try the following editing of the `kas-fleet-manager` AuthConfig:

<details>
  <summary><b>Deny access to a user altogether</b></summary><br/>

  Edit the `kas-fleet-manager` AuthConfig:

  ```sh
  kubectl -n kas-fleet-manager-authorino edit authconfig/kas-fleet-manager
  ```

  Find the authorization policy called `deny-list` and add an email to the `list` array in Rego, e.g.:

  ```rego
  list := [
    "denied-test-user1@example.com",
    "denied-test-user2@example.com",
    "username@redhat.com" # <====== forbidden user
  ]
  denied { list[_] == input.auth.identity.email }
  allow { not denied }
  ```

  You can check your Red Hat OCM user email with the following command:

  ```sh
  ocm token --payload | jq -r .email
  ```

  Save and exit the editing of the AuthConfig.

  Test the change by sending a request to kas-fleet-manager authenticating with the now forbidden user. Authorino should respond with a `403` status code.

  Revert the steps above to reauthorize the user.
</details>

<details>
  <summary><b>Replace the Admin API RBAC with Kubernetes RBAC</b></summary><br/>

  Make possible for users authenticating with Red Hat SSO External to send requests to the admin endpoints of kas-fleet-manager, provided they are bound to required roles in the Kubernetes RBAC.

  Create the roles:

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: kas-fleet-manager-admin-read
  rules:
  - apiGroups: ["api.openshift.com/api/kafkas_mgmt/v1"]
    resources: ["admin"]
    verbs: ["get"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: kas-fleet-manager-admin-write
  rules:
  - apiGroups: ["api.openshift.com/api/kafkas_mgmt/v1"]
    resources: ["admin"]
    verbs: ["get", "patch"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: kas-fleet-manager-admin-full
  rules:
  - apiGroups: ["api.openshift.com/api/kafkas_mgmt/v1"]
    resources: ["admin"]
    verbs: ["get", "patch", "delete"]
  EOF
  ```

  Alter the AuthConfig:

  ```sh
  curl -sL https://raw.githubusercontent.com/guicassolato/kas-fleet-manager-authorino/main/kas-fleet-manager-authconfig-k8s-rbac.yaml | envsubst | kubectl -n kas-fleet-manager-authorino apply -f -
  ```

  Try the Admin API without a binding:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/admin/kafkas
  ```

  Create a role binding:

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: kas-fleet-manager-admin-readers
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: kas-fleet-manager-admin-read
  subjects:
  - kind: User
    name: $(ocm token --payload | jq -r .username)
  EOF
  ```

  Try the Admin API again with a binding:

  ```sh
  curl -H "Authorization: Bearer $(ocm token)" \
      https://kas-fleet-manager-kas-fleet-manager-authorino.apps.$K8S_CLUSTER_DOMAIN/api/kafkas_mgmt/v1/admin/kafkas
  ```
</details>

## Advanced topics

Harden the installation by setting up the following environment policies:

<details>
  <summary><b>Limit traffic to Authorino with a <code>NetworkPolicy</code></b></summary><br/>

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: kas-fleet-manager-authorino
  spec:
    podSelector:
      matchLabels:
        authorino-resource: authorino
    policyTypes:
    - Ingress
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            project: kas-fleet-manager-authorino
      - podSelector:
          matchLabels:
            app: kas-fleet-manager
  EOF
  ```
</details>

<details>
  <summary><b>Limit the features of Authorino that can be used in the kas-fleet-manager AuthConfig</b></summary><br/>

  Kubernetes [dynamic admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) can be used to trigger validations of arbitrary Kubernetes resources that cluster users try to persist, including `AuthConfig` custom resources. Provided the rules to validade such resources are expressed in another AuthConfig – one that validates all AuthConfigs –, Authorino itself can be hooked to perform the validation.

  The following steps configure Authorino to validate all requests to the Kubernetes API involving AuthConfig custom resources that are targeted to the dedicated instance of Authorino that serves the kas-fleet-manager authorization requests. As an example, the following validation rule is enforced:

  <table>
    <tbody>
      <tr>
        <td><i>Any AuthConfig linking the ingress host name used to access kas-fleet-manager must verify user access tokens issued by a Red Hat SSO server.</i></td>
      </tr>
    </tbody>
  </table>

  Create the AuthConfig to validate all AuthConfigs:

  ```sh
  curl -sL https://raw.githubusercontent.com/guicassolato/kas-fleet-manager-authorino/main/validating-webhook-authconfig.yaml | envsubst | kubectl -n kas-fleet-manager-authorino apply -f -
  ```

  Create a `ValidatingWebhookConfiguration`:

  ```sh
  kubectl -n kas-fleet-manager-authorino apply -f -<<EOF
  apiVersion: admissionregistration.k8s.io/v1
  kind: ValidatingWebhookConfiguration
  metadata:
    name: authconfig-authz
    annotations:
      service.beta.openshift.io/inject-cabundle: "true"
  webhooks:
  - name: check-authconfig.authorino.kuadrant.io
    clientConfig:
      service:
        namespace: kas-fleet-manager-authorino
        name: authorino-authorino-authorization
        port: 5001
        path: /check
    rules:
    - apiGroups: ["authorino.kuadrant.io"]
      apiVersions: ["v1beta1"]
      resources: ["authconfigs"]
      operations: ["CREATE", "UPDATE"]
      scope: Namespaced
    sideEffects: None
    admissionReviewVersions: ["v1"]
  EOF
  ```

  Test the validation by trying to edit the `kas-fleet-manager` AuthConfig adding any trusted source of identity that is not `oidc` or whose OIDC issuer endpoint is not a `redhat.com` subdomain.
</details>
