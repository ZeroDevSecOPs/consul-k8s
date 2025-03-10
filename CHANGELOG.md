## UNRELEASED

BREAKING CHANGES:
* Helm Chart
  * The `kube-system` and `local-path-storage` namespaces are now _excluded_ from connect injection by default on Kubernetes versions >= 1.21. If you wish to enable injection on those namespaces, set `connectInject.namespaceSelector` to `null`. [[GH-726](https://github.com/hashicorp/consul-k8s/pull/726)]
IMPROVEMENTS:
* Helm Chart
  * Automatic retry for `gossip-encryption-autogenerate-job` on failure [[GH-789](https://github.com/hashicorp/consul-k8s/pull/789)]
  * `kube-system` and `local-path-storage` namespaces are now excluded from connect injection by default on Kubernetes versions >= 1.21. This prevents deadlock issues when `kube-system` components go down and allows Kind to work without changing the failure policy of the mutating webhook. [[GH-726](https://github.com/hashicorp/consul-k8s/pull/726)]
* CLI
  * Add `status` command. [[GH-768](https://github.com/hashicorp/consul-k8s/pull/768)]
  * Add `-verbose`, `-v` flag to the `consul-k8s install` command, which outputs all logs emitted from the installation. By default, verbose is set to `false` to hide logs that show resources are not ready. [[GH-810](https://github.com/hashicorp/consul-k8s/pull/810)]
  * Add Prometheus deployment and Consul K8s metrics integration with demo preset when installing via `-preset=demo`. [[GH-809](https://github.com/hashicorp/consul-k8s/pull/809)]

## 0.35.0 (October 19, 2021)

FEATURES:
* Control Plane
  * Add `gossip-encryption-autogenerate` subcommand to generate a random 32 byte Kubernetes secret to be used as a gossip encryption key. [[GH-772](https://github.com/hashicorp/consul-k8s/pull/772)]
  * Add support for `partition-exports` config entry. [[GH-802](https://github.com/hashicorp/consul-k8s/pull/802)], [[GH-803](https://github.com/hashicorp/consul-k8s/pull/803)]
* Helm Chart
  * Add automatic generation of gossip encryption with `global.gossipEncryption.autoGenerate=true`. [[GH-738](https://github.com/hashicorp/consul-k8s/pull/738)]
  * Add support for configuring resources for mesh gateway `service-init` container. [[GH-758](https://github.com/hashicorp/consul-k8s/pull/758)]
  * Add support for `PartitionExports` CRD. [[GH-802](https://github.com/hashicorp/consul-k8s/pull/802)], [[GH-803](https://github.com/hashicorp/consul-k8s/pull/803)]

IMPROVEMENTS:
* Control Plane
  * Upgrade Docker image Alpine version from 3.13 to 3.14. [[GH-737](https://github.com/hashicorp/consul-k8s/pull/737)]
  * CRDs: tune failure backoff so invalid config entries are re-synced more quickly. [[GH-788](https://github.com/hashicorp/consul-k8s/pull/788)]
* Helm Chart
  * Enable adding extra containers to server and client Pods. [[GH-749](https://github.com/hashicorp/consul-k8s/pull/749)]
  * ACL support for Admin Partitions. **(Consul Enterprise only)**
  **BETA** [[GH-766](https://github.com/hashicorp/consul-k8s/pull/766)]
    * This feature now enabled ACL support for Admin Partitions. The server-acl-init job now creates a Partition token. This token
      can be used to bootstrap new partitions as well as manage ACLs in the non-default partitions.
    * Partition to partition networking is disabled if ACLs are enabled.
    * Documentation for the installation can be found [here](https://github.com/hashicorp/consul-k8s/blob/main/docs/admin-partitions-with-acls.md).
* CLI
  * Add `version` command. [[GH-741](https://github.com/hashicorp/consul-k8s/pull/741)]
  * Add `uninstall` command. [[GH-725](https://github.com/hashicorp/consul-k8s/pull/725)]

## 0.34.1 (September 17, 2021)

BUG FIXES:
* Helm
  * Fix consul-k8s image version in values file. [[GH-732](https://github.com/hashicorp/consul-k8s/pull/732)]

## 0.34.0 (September 17, 2021)

FEATURES:
* CLI
  * The `consul-k8s` CLI enables users to deploy and operate Consul on Kubernetes.
    * Support `consul-k8s install` command. [[GH-713](https://github.com/hashicorp/consul-k8s/pull/713)]
* Helm Chart
  * Add support for Admin Partitions. **(Consul Enterprise only)**
  **ALPHA** [[GH-729](https://github.com/hashicorp/consul-k8s/pull/729)]
    * This feature allows Consul to be deployed across multiple Kubernetes clusters while sharing a single set of Consul
servers. The services on each cluster can be independently managed. This feature is an alpha feature. It requires:
      * a flat pod and node network in order for inter-partition networking to work.
      * TLS to be enabled.
      * Consul Namespaces enabled.

      Transparent Proxy is unsupported for cross partition communication.

To enable Admin Partitions on the server cluster use the following config.

```yaml
global:
  enableConsulNamespaces: true
  tls:
    enabled: true
  image: hashicorp/consul-enterprise:1.11.0-ent-alpha
  adminPartitions:
    enabled: true
server:
  exposeGossipAndRPCPorts: true
  enterpriseLicense:
    secretName: license
    secretKey: key
connectInject:
  enabled: true
  transparentProxy:
    defaultEnabled: false
  consulNamespaces:
    mirroringK8S: true
controller:
  enabled: true
```

Identify the LoadBalancer External IP of the `partition-service`

```bash
kubectl get svc consul-consul-partition-service -o json | jq -r '.status.loadBalancer.ingress[0].ip'
```

Migrate the TLS CA credentials from the server cluster to the workload clusters

```bash
kubectl get secret consul-consul-ca-key --context "server-context" -o yaml | kubectl apply --context "workload-context" -f -
kubectl get secret consul-consul-ca-cert --context "server-context" -o yaml | kubectl apply --context "workload-context" -f -
```

Configure the workload cluster using the following config.

```yaml
global:
  enabled: false
  enableConsulNamespaces: true
  image: hashicorp/consul-enterprise:1.11.0-ent-alpha
  adminPartitions:
    enabled: true
    name: "alpha" # Name of Admin Partition
  tls:
    enabled: true
    caCert:
      secretName: consul-consul-ca-cert
      secretKey: tls.crt
    caKey:
      secretName: consul-consul-ca-key
      secretKey: tls.key
server:
  enterpriseLicense:
    secretName: license
    secretKey: key
externalServers:
  enabled: true
  hosts: [ "loadbalancer IP" ] # external IP of partition service LB
  tlsServerName: server.dc1.consul
client:
  enabled: true
  exposeGossipPorts: true
  join: [ "loadbalancer IP" ] # external IP of partition service LB
connectInject:
  enabled: true
  consulNamespaces:
    mirroringK8S: true
controller:
  enabled: true
```

This should lead to the workload cluster having only Consul agents that connect with the Consul server. Services in this
cluster behave like independent services. They can be configured to communicate with services in other partitions by
configuring the upstream configuration on the individual services.

* Control Plane
  * Add support for Admin Partitions. **(Consul Enterprise only)** **
    ALPHA** [[GH-729](https://github.com/hashicorp/consul-k8s/pull/729)]
    * Add Partition-Init job that runs in Kubernetes clusters that do not have servers running to provision Admin
      Partitions.
    * Update endpoints-controller, config-entry controller and config entries to add partition config to them.

IMPROVEMENTS:
* Helm Chart
  * Add ability to specify port for ui service. [[GH-604](https://github.com/hashicorp/consul-k8s/pull/604)]
  * Use `policy/v1` for Consul server `PodDisruptionBudget` if supported. [[GH-606](https://github.com/hashicorp/consul-k8s/pull/606)]
  * Add readiness, liveness and startup probes to the connect inject deployment. [[GH-626](https://github.com/hashicorp/consul-k8s/pull/626)][[GH-701](https://github.com/hashicorp/consul-k8s/pull/701)]
  * Add support for setting container security contexts on client and server Pods. [[GH-620](https://github.com/hashicorp/consul-k8s/pull/620)]
  * Update Envoy image to 1.18.4 [[GH-699](https://github.com/hashicorp/consul-k8s/pull/699)]
  * Add configuration for webhook-cert-manager tolerations [[GH-712](https://github.com/hashicorp/consul-k8s/pull/712)]
  * Update default Consul version to 1.10.2 [[GH-718](https://github.com/hashicorp/consul-k8s/pull/718)]
* Control Plane
  * Add health endpoint to the connect inject webhook that will be healthy when webhook certs are present and not empty. [[GH-626](https://github.com/hashicorp/consul-k8s/pull/626)]
  * Catalog Sync: Fix issue registering NodePort services with wrong IPs when a node has multiple IP addresses. [[GH-619](https://github.com/hashicorp/consul-k8s/pull/619)]
  * Allow registering the same service in multiple namespaces. [[GH-697](https://github.com/hashicorp/consul-k8s/pull/697)]

BUG FIXES:
* Helm Chart
  * Disable [streaming](https://www.consul.io/docs/agent/options#use_streaming_backend) on Consul clients because it is currently not supported when
    doing mesh gateway federation. If you wish to enable it, override the setting using `client.extraConfig`:

    ```yaml
    client:
      extraConfig: |
        {"use_streaming_backend": true}
    ```
    [[GH-718](https://github.com/hashicorp/consul-k8s/pull/718)]

## 0.33.0 (August 12, 2021)

BREAKING CHANGES:
* The consul-k8s repository has been merged with consul-helm and now contains the `consul-k8s-control-plane` binary (previously named `consul-k8s`) and the Helm chart to deploy Consul on Kubernetes. The docker image previously named `hashicorp/consul-k8s` has been renamed to `hashicorp/consul-k8s-control-plane`. The binary and Helm chart will be released together with the same version. **NOTE: If you install Consul through the Helm chart and are not customizing the `global.imageK8S` value then this will not be a breaking change.** [[GH-589](https://github.com/hashicorp/consul-k8s/pull/589)]
  * Helm chart v0.33.0+ will support the corresponding `consul-k8s-control-plane` image with the same version only. For example Helm chart 0.33.0 will only be supported to work with the default value `global.imageK8S`: `hashicorp/consul-k8s-control-plane:0.33.0`.
  * The control-plane binary has been renamed from `consul-k8s` to `consul-k8s-control-plane` and is now invoked as `consul-k8s-control-plane` in the Helm chart. The first version of this newly renamed binary will be 0.33.0.
  * The Go module `github.com/hashicorp/consul-k8s` has been named to `github.com/hashicorp/consul-k8s/control-plane`.
  * The Helm chart is located under `consul-k8s/charts/consul`.
  * The control-plane source code is located under `consul-k8s/control-plane`.
* Minimum Kubernetes versions supported is 1.17+ and now matches what is stated in the `README.md` file.  [[GH-1053](https://github.com/hashicorp/consul-helm/pull/1053)]

IMPROVEMENTS:
* Control Plane
  * Add flags `-log-level`, `-log-json` to all subcommands to control log level and json formatting. [[GH-523](https://github.com/hashicorp/consul-k8s/pull/523)]
  * Execute Consul clients and servers using the Docker entrypoint for consistency. [[GH-590](https://github.com/hashicorp/consul-k8s/pull/590)]
* Helm Chart
  * Substitute `HOST_IP/POD_IP/HOSTNAME` variables in `server.extraConfig` and `client.extraConfig` so they are passed in to server/client config already evaluated at runtime. [[GH-1042](https://github.com/hashicorp/consul-helm/pull/1042)]
  * Set failurePolicy to Fail for connectInject mutating webhook so that pods fail to schedule when the webhook is offline. This can be controlled via `connectInject.failurePolicy`. [[GH-1024](https://github.com/hashicorp/consul-helm/pull/1024)]
  * Allow setting global.logLevel and global.logJSON and propogate this to all consul-k8s commands. [[GH-980](https://github.com/hashicorp/consul-helm/pull/980)]
  * Allow setting `connectInject.replicas` to control number of replicas of webhook injector. [[GH-1029](https://github.com/hashicorp/consul-helm/pull/1029)]
  * Add the ability to manually specify a k8s secret containing server-cert via the value `server.serverCert.secretName`. [[GH-1024](https://github.com/hashicorp/consul-helm/pull/1046)]
  * Allow setting `ui.pathType` for providers that do not support the default pathType "Prefix". [[GH-1012](https://github.com/hashicorp/consul-helm/pull/1012)]
  * Allow setting `client.nodeMeta` to specify arbitrary key-value pairs to associate with the node. [[GH-728](https://github.com/hashicorp/consul-helm/pull/728)]

BUG FIXES:
* Control Plane
  * Connect: Use `AdmissionregistrationV1` instead of `AdmissionregistrationV1beta1` API as it was deprecated in k8s 1.16. [[GH-558](https://github.com/hashicorp/consul-k8s/pull/558)]
  * Connect: Fix bug where environment variables `<NAME>_CONNECT_SERVICE_HOST` and
  `<NAME>_CONNECT_SERVICE_PORT` weren't being set when the upstream annotation was used. [[GH-549](https://github.com/hashicorp/consul-k8s/issues/549)]
  * Connect: Fix a bug with leaving around ACL tokens after a service has been deregistered. Note that this will not clean up existing leftover ACL tokens. [[GH-540](https://github.com/hashicorp/consul-k8s/issues/540)][[GH-599](https://github.com/hashicorp/consul-k8s/issues/599)]
  * CRDs: Fix ProxyDefaults and ServiceDefaults resources not syncing with Consul < 1.10.0 [[GH-1023](https://github.com/hashicorp/consul-helm/issues/1023)]
  * Connect: Skip service registration for duplicate services only on Kubernetes. [[GH-581](https://github.com/hashicorp/consul-k8s/pull/581)]
  * Connect: redirect-traffic command passes ACL token when ACLs are enabled. [[GH-576](https://github.com/hashicorp/consul-k8s/pull/576)]

## 0.26.0 (June 22, 2021)

FEATURES:
* Connect: Support Transparent Proxy. [[GH-481](https://github.com/hashicorp/consul-k8s/pull/481)]
  This feature enables users to use KubeDNS to reach other services within the Consul Service Mesh,
  as well as enforces the inbound and outbound traffic to go through the Envoy proxy.

  Using transparent proxy for your service mesh applications means:
  - Proxy service registrations will set `mode` to `transparent` in the proxy configuration
    so that Consul can configure the Envoy proxy to have an inbound and outbound listener.
  - Both proxy and service registrations will include the cluster IP and service port of the Kubernetes service
    as tagged addresses so that Consul can configure Envoy to route traffic based on that IP and port.
  - The `consul-connect-inject-init` container will run `consul connect redirect-traffic` [command](https://www.consul.io/commands/connect/redirect-traffic),
    which will apply rules (via iptables) to redirect inbound and outbound traffic to the proxy.
    To run this command the `consul-connect-inject-init` requires running as root with capability `NET_ADMIN`.

  This feature includes the following changes:
  * Add new `-enable-transparent-proxy` flag to the `inject-connect` command.
    When `true`, transparent proxy will be used for all services on the Consul Service Mesh
    within a Kubernetes cluster. This flag defaults to `true`.
  * Add new `consul.hashicorp.com/transparent-proxy` pod annotation to allow enabling and disabling transparent
    proxy for individual services.
* CRDs: Add CRD for MeshConfigEntry. Supported in Consul 1.10+ [[GH-513](https://github.com/hashicorp/consul-k8s/pull/513)]
* Connect: Overwrite Kubernetes HTTP readiness and/or liveness probes to point to Envoy proxy when
  transparent proxy is enabled. [[GH-517](https://github.com/hashicorp/consul-k8s/pull/517)]
* Connect: Allow exclusion of inbound ports, outbound ports and CIDRs, and additional user IDs when
  Transparent Proxy is enabled. [[GH-506](https://github.com/hashicorp/consul-k8s/pull/506)]

  The following annotations are supported:
  * `consul.hashicorp.com/transparent-proxy-exclude-inbound-ports` - Comma-separated list of inbound ports to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-outbound-ports` - Comma-separated list of outbound ports to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-outbound-cidrs` - Comma-separated list of IPs or CIDRs to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-uids` - Comma-separated list of Linux user IDs to exclude.
* Connect: Add the ability to set default tproxy mode at namespace level via label. [[GH-501](https://github.com/hashicorp/consul-k8s/pull/510)]
  * Setting the annotation `consul.hashicorp.com/transparent-proxy` to `true/false` will define whether tproxy is enabled/disabled for the pod.
  * Setting the label `consul.hashicorp.com/transparent-proxy` to `true/false` on a namespace will define the default behavior for pods in that namespace, which do not also have the annotation set.
  * The default tproxy behavior will be defined by the value of `-enable-transparent-proxy` flag to the `consul-k8s inject-connect` command. It can be overridden in a namespace by the the label on the namespace or for a pod using the annotation on the pod.
* Connect: support upgrades for services deployed before endpoints controller to
  upgrade to a version of consul-k8s with endpoints controller. [[GH-509](https://github.com/hashicorp/consul-k8s/pull/509)]
* Connect: A new command `consul-k8s connect-init` has been added.
  It replaces the existing init-container logic for ACL login and Envoy bootstrapping and introduces a polling wait for service registration,
  see `Endpoints Controller` for more information.
  [[GH-446](https://github.com/hashicorp/consul-k8s/pull/446)], [[GH-452](https://github.com/hashicorp/consul-k8s/pull/452)], [[GH-459](https://github.com/hashicorp/consul-k8s/pull/459)]
* Connect: A new controller `Endpoints Controller` has been added which is responsible for managing service endpoints and service registration.
  When a Kubernetes service references a deployed connect-injected pod, the endpoints controller will be responsible for managing the lifecycle of the connect-injected deployment. [[GH-455](https://github.com/hashicorp/consul-k8s/pull/455)], [[GH-467](https://github.com/hashicorp/consul-k8s/pull/467)], [[GH-470](https://github.com/hashicorp/consul-k8s/pull/470)], [[GH-475](https://github.com/hashicorp/consul-k8s/pull/475)]
  - This includes:
    - service registration and deregistration, formerly managed by the `consul-connect-inject-init`.
    - monitoring health checks, formerly managed by `healthchecks-controller`.
    - re-registering services in the events of consul agent failures, formerly managed by `consul-sidecar`.
  - The endpoints controller replaces the health checks controller while preserving existing functionality. [[GH-472](https://github.com/hashicorp/consul-k8s/pull/472)]
  - The endpoints controller replaces the cleanup controller while preserving existing functionality.
    [[GH-476](https://github.com/hashicorp/consul-k8s/pull/476)], [[GH-454](https://github.com/hashicorp/consul-k8s/pull/454)]
  - Merged metrics configuration support is now partially managed by the endpoints controller.
    [[GH-469](https://github.com/hashicorp/consul-k8s/pull/469)]

IMPROVEMENTS:
* Connect: skip service registration when a service with the same name but in a different Kubernetes namespace is found
  and Consul namespaces are not enabled. [[GH-527](https://github.com/hashicorp/consul-k8s/pull/527)]
* Connect: Leader election support for connect-inject deployment. [[GH-479](https://github.com/hashicorp/consul-k8s/pull/479)]
* Connect: the `consul-connect-inject-init` container has been split into two init containers. [[GH-441](https://github.com/hashicorp/consul-k8s/pull/441)]
 Connect: Connect webhook no longer generates its own certificates and relies on them being provided as files on the disk.
  [[GH-454](https://github.com/hashicorp/consul-k8s/pull/454)]]
* CRDs: Update `ServiceDefaults` with `Mode`, `TransparentProxy`, `DialedDirectly` and `UpstreamConfigs` fields. Note: `Mode` and `TransparentProxy` should not be set
  using this CRD but via annotations. [[GH-502](https://github.com/hashicorp/consul-k8s/pull/502)], [[GH-485](https://github.com/hashicorp/consul-k8s/pull/485)], [[GH-533](https://github.com/hashicorp/consul-k8s/pull/533)]
* CRDs: Update `ProxyDefaults` with `Mode`, `DialedDirectly` and `TransparentProxy` fields. Note: `Mode` and `TransparentProxy` should not be set
  using the CRD but via annotations. [[GH-505](https://github.com/hashicorp/consul-k8s/pull/505)], [[GH-485](https://github.com/hashicorp/consul-k8s/pull/485)], [[GH-533](https://github.com/hashicorp/consul-k8s/pull/533)]
* CRDs: update the CRD versions from v1beta1 to v1. [[GH-464](https://github.com/hashicorp/consul-k8s/pull/464)]
* Delete secrets created by webhook-cert-manager when the deployment is deleted. [[GH-530](https://github.com/hashicorp/consul-k8s/pull/530)]

BUG FIXES:
* CRDs: Update the type of connectTimeout and TTL in ServiceResolver and ServiceRouter from time.Duration to metav1.Duration.
  This allows a user to set these values as a duration string on the resource. Existing resources that had set a specific integer
  duration will continue to function with a duration with 'n' nanoseconds, 'n' being the set value.
* CRDs: Fix a bug where the `config` field in `ProxyDefaults` CR failed syncing to Consul because `apiextensions.k8s.io/v1` requires CRD spec to have structured schema. [[GH-495](https://github.com/hashicorp/consul-k8s/pull/495)]
* CRDs: make `lastSyncedTime` a pointer to prevent setting last synced time Reconcile errors. [[GH-466](https://github.com/hashicorp/consul-k8s/pull/466)]

BREAKING CHANGES:
* Connect: Add a security context to the init copy container and the envoy sidecar and ensure they
  do not run as root. If a pod container shares the same `runAsUser` (5995) as Envoy an error is returned.
  [[GH-493](https://github.com/hashicorp/consul-k8s/pull/493)]
* Connect: Kubernetes Services are required for all Consul Service Mesh applications.
  The Kubernetes service name will be used as the service name to register with Consul
  unless the annotation `consul.hashicorp.com/connect-service` is provided to the deployment/pod to override this.
  If using ACLs, the ServiceAccountName must match the service name used with Consul.

  *Note*: if you're already using a Kubernetes service, no changes required.

  Example Service:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: sample-app
  spec:
    selector:
      app: sample-app
    ports:
      - port: 80
        targetPort: 9090
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: sample-app
    name: sample-app
  spec:
    replicas: 1
    selector:
       matchLabels:
         app: sample-app
    template:
      metadata:
        annotations:
          'consul.hashicorp.com/connect-inject': 'true'
        labels:
          app: sample-app
      spec:
        containers:
        - name: sample-app
          image: sample-app:0.1.0
          ports:
          - containerPort: 9090
  ```
* Connect: `consul.hashicorp.com/connect-sync-period` annotation is no longer supported.
  This annotation used to configure the sync period of the `consul-sidecar` (aka `lifecycle-sidecar`).
  Since we no longer inject the `consul-sidecar` to keep services registered in Consul, this annotation has
  been removed. [[GH-467](https://github.com/hashicorp/consul-k8s/pull/467)]
* Connect: transparent proxy feature enabled by default. This may break existing deployments.
  Please see details of the feature.

## 0.26.0-beta3 (May 27, 2021)

IMPROVEMENTS:
* Connect: Overwrite Kubernetes HTTP readiness and/or liveness probes to point to Envoy proxy when
  transparent proxy is enabled. [[GH-517](https://github.com/hashicorp/consul-k8s/pull/517)]
* Connect: Don't set security context for the Envoy proxy when on OpenShift and transparent proxy is disabled.
  [[GH-521](https://github.com/hashicorp/consul-k8s/pull/521)]
* Connect: `consul-connect-inject-init` run with `privileged: true` when transparent proxy is enabled.
  [[GH-524](https://github.com/hashicorp/consul-k8s/pull/524)]

BUG FIXES:
* Connect: Process every Address in an Endpoints object before returning an error. This ensures an address that isn't reconciled successfully doesn't prevent the remaining addresses from getting reconciled. [[GH-519](https://github.com/hashicorp/consul-k8s/pull/519)]

## 0.26.0-beta2 (May 06, 2021)

BREAKING CHANGES:
* Connect: Add a security context to the init copy container and the envoy sidecar and ensure they
  do not run as root. If a pod container shares the same `runAsUser` (5995) as Envoy an error is returned
  on scheduling. [[GH-493](https://github.com/hashicorp/consul-k8s/pull/493)]

IMPROVEMENTS:
* CRDs: Update ServiceDefaults with Mode, TransparentProxy and UpstreamConfigs fields. Note: Mode and TransparentProxy should not be set
  using this CRD but via annotations. [[GH-502](https://github.com/hashicorp/consul-k8s/pull/502)], [[GH-485](https://github.com/hashicorp/consul-k8s/pull/485)]
* CRDs: Update ProxyDefaults with Mode and TransparentProxy fields. Note: Mode and TransparentProxy should not be set
  using the CRD but via annotations. [[GH-505](https://github.com/hashicorp/consul-k8s/pull/505)], [[GH-485](https://github.com/hashicorp/consul-k8s/pull/485)]
* CRDs: Add CRD for MeshConfigEntry. Supported in Consul 1.10+ [[GH-513](https://github.com/hashicorp/consul-k8s/pull/513)]
* Connect: No longer set multiple tagged addresses in Consul when k8s service has multiple ports and Transparent Proxy is enabled.
  [[GH-511](https://github.com/hashicorp/consul-k8s/pull/511)]
* Connect: Allow exclusion of inbound ports, outbound ports and CIDRs, and additional user IDs when
  Transparent Proxy is enabled. [[GH-506](https://github.com/hashicorp/consul-k8s/pull/506)]

  The following annotations are supported:

  * `consul.hashicorp.com/transparent-proxy-exclude-inbound-ports` - Comma-separated list of inbound ports to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-outbound-ports` - Comma-separated list of outbound ports to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-outbound-cidrs` - Comma-separated list of IPs or CIDRs to exclude.
  * `consul.hashicorp.com/transparent-proxy-exclude-uids` - Comma-separated list of Linux user IDs to exclude.

* Connect: Add the ability to set default tproxy mode at namespace level via label. [[GH-501](https://github.com/hashicorp/consul-k8s/pull/510)]

  * Setting the annotation `consul.hashicorp.com/transparent-proxy` to `true/false` will define whether tproxy is enabled/disabled for the pod.
  * Setting the label `consul.hashicorp.com/transparent-proxy` to `true/false` on a namespace will define the default behavior for pods in that namespace, which do not also have the annotation set.
  * The default tproxy behavior will be defined by the value of `-enable-transparent-proxy` flag to the `consul-k8s inject-connect` command. It can be overridden in a namespace by the the label on the namespace or for a pod using the annotation on the pod.

* Connect: support upgrades for services deployed before endpoints controller to
  upgrade to a version of consul-k8s with endpoints controller. [[GH-509](https://github.com/hashicorp/consul-k8s/pull/509)]

* Connect: add additional logging to the endpoints controller and connect-init command to help
  the user debug if pods arent starting right away. [[GH-514](https://github.com/hashicorp/consul-k8s/pull/514/)]

BUG FIXES:
* Connect: Use `runAsNonRoot: false` for connect-init's container when tproxy is enabled. [[GH-493](https://github.com/hashicorp/consul-k8s/pull/493)]
* CRDs: Fix a bug where the `config` field in `ProxyDefaults` CR was not synced to Consul because
  `apiextensions.k8s.io/v1` requires CRD spec to have structured schema. [[GH-495](https://github.com/hashicorp/consul-k8s/pull/495)]
* Connect: Fix a bug where health status in Consul is updated incorrectly due to stale pod information in cache.
  [[GH-503](https://github.com/hashicorp/consul-k8s/pull/503)]

## 0.26.0-beta1 (April 16, 2021)

BREAKING CHANGES:
* Connect: Kubernetes Services are now required for all Consul Service Mesh applications.
  The Kubernetes service name will be used as the service name to register with Consul
  unless the annotation `consul.hashicorp.com/connect-service` is provided to the deployment/pod to override this.
  If using ACLs, the ServiceAccountName must match the service name used with Consul.
  
  *Note*: if you're already using a Kubernetes service, no changes are required.

  Example Service:
  ```yaml
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: sample-app
  spec:
    selector:
      app: sample-app
    ports:
      - port: 80
        targetPort: 9090
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: sample-app
    name: sample-app
  spec:
    replicas: 1
    selector:
       matchLabels:
         app: sample-app
    template:
      metadata:
        annotations:
          'consul.hashicorp.com/connect-inject': 'true'
        labels:
          app: sample-app
      spec:
        containers:
        - name: sample-app
          image: sample-app:0.1.0
          ports:
          - containerPort: 9090
    ```
* Connect: `consul.hashicorp.com/connect-sync-period` annotation is no longer supported.
  This annotation was used to configure the sync period of the `consul-sidecar` (aka `lifecycle-sidecar`).
  Since we no longer inject the `consul-sidecar` to keep services registered in Consul, this annotation is
  now meaningless. [[GH-467](https://github.com/hashicorp/consul-k8s/pull/467)]
* Connect: transparent proxy feature is enabled by default. This may break existing deployments.
  Please see details of the feature below.

FEATURES:
* Connect: Support Transparent Proxy. [[GH-481](https://github.com/hashicorp/consul-k8s/pull/481)]
  This feature enables users to use KubeDNS to reach other services within the Consul Service Mesh,
  as well as enforces the inbound and outbound traffic to go through the Envoy proxy.
  Using transparent proxy for your service mesh applications means:
  - Proxy service registrations will set `mode` to `transparent` in the proxy configuration
    so that Consul can configure the Envoy proxy to have an inbound and outbound listener.
  - Both proxy and service registrations will include the cluster IP and service port of the Kubernetes service
    as tagged addresses so that Consul can configure Envoy to route traffic based on that IP and port.
  - The `consul-connect-inject-init` container will run `consul connect redirect-traffic` [command](https://www.consul.io/commands/connect/redirect-traffic),
    which will apply rules (via iptables) to redirect inbound and outbound traffic to the proxy.
    To run this command the `consul-connect-inject-init` requires running as root with capability `NET_ADMIN`.
  
  **Note: this feature is currently in beta.** 
  
  This feature includes the following changes:
  * Add new `-enable-transparent-proxy` flag to the `inject-connect` command.
    When `true`, transparent proxy will be used for all services on the Consul Service Mesh
    within a Kubernetes cluster. This flag defaults to `true`.
  * Add new `consul.hashicorp.com/transparent-proxy` pod annotation to allow enabling and disabling transparent
    proxy for individual services.

IMPROVEMENTS:
* CRDs: update the CRD versions from v1beta1 to v1. [[GH-464](https://github.com/hashicorp/consul-k8s/pull/464)]
* Connect: the `consul-connect-inject-init` container has been split into two init containers. [[GH-441](https://github.com/hashicorp/consul-k8s/pull/441)]
* Connect: A new internal command `consul-k8s connect-init` has been added.
  It replaces the existing init container logic for ACL login and Envoy bootstrapping and introduces a polling wait for service registration,
  see `Endpoints Controller` for more information.
  [[GH-446](https://github.com/hashicorp/consul-k8s/pull/446)], [[GH-452](https://github.com/hashicorp/consul-k8s/pull/452)], [[GH-459](https://github.com/hashicorp/consul-k8s/pull/459)]
* Connect: A new controller `Endpoints Controller` has been added which is responsible for managing service endpoints and service registration.
  When a Kubernetes service referencing a connect-injected pod is deployed, the endpoints controller will be responsible for managing the lifecycle of the connect-injected deployment. [[GH-455](https://github.com/hashicorp/consul-k8s/pull/455)], [[GH-467](https://github.com/hashicorp/consul-k8s/pull/467)], [[GH-470](https://github.com/hashicorp/consul-k8s/pull/470)], [[GH-475](https://github.com/hashicorp/consul-k8s/pull/475)]
  - This includes:
      - service registration and deregistration, formerly managed by the `consul-connect-inject-init`.
      - monitoring health checks, formerly managed by `healthchecks-controller`.
      - re-registering services in the events of consul agent failures, formerly managed by `consul-sidecar`.

  - The endpoints controller replaces the health checks controller while preserving existing functionality. [[GH-472](https://github.com/hashicorp/consul-k8s/pull/472)]

  - The endpoints controller replaces the cleanup controller while preserving existing functionality.
    [[GH-476](https://github.com/hashicorp/consul-k8s/pull/476)], [[GH-454](https://github.com/hashicorp/consul-k8s/pull/454)]

  - Merged metrics configuration support is now partially managed by the endpoints controller.
    [[GH-469](https://github.com/hashicorp/consul-k8s/pull/469)]
* Connect: Leader election support for connect webhook and controller deployment. [[GH-479](https://github.com/hashicorp/consul-k8s/pull/479)]
* Connect: Connect webhook no longer generates its own certificates and relies on them being provided as files on the disk.
  [[GH-454](https://github.com/hashicorp/consul-k8s/pull/454)]] 
* Connect: Connect pods and their Envoy sidecars no longer have a preStop hook as service deregistration is managed by the endpoints controller.
  [[GH-467](https://github.com/hashicorp/consul-k8s/pull/467)]

BUG FIXES:
* CRDs: make `lastSyncedTime` a pointer to prevent setting last synced time Reconcile errors. [[GH-466](https://github.com/hashicorp/consul-k8s/pull/466)]

## 0.25.0 (March 18, 2021)

FEATURES:
* Metrics: add metrics configuration to inject-connect and metrics-merging capability to consul-sidecar. When metrics and metrics merging are enabled, the consul-sidecar will expose an endpoint that merges the app and proxy metrics.

  The flags `-merged-metrics-port`, `-service-metrics-port` and `-service-metrics-path` can be used to configure the merged metrics server, and the application service metrics endpoint on the consul sidecar.

  The flags `-default-enable-metrics`, `-default-enable-metrics-merging`, `-default-merged-metrics-port`, `-default-prometheus-scrape-port` and `-default-prometheus-scrape-path` configure the inject-connect command.

IMPROVEMENTS:
* CRDs: add field Last Synced Time to CRD status and add printer column on CRD to display time since when the
  resource was last successfully synced with Consul. [[GH-448](https://github.com/hashicorp/consul-k8s/pull/448)]
  
BUG FIXES:
* CRDs: fix incorrect validation for `ServiceResolver`. [[GH-456](https://github.com/hashicorp/consul-k8s/pull/456)]

## 0.24.0 (February 16, 2021)

BREAKING CHANGES:
* Connect: the `lifecycle-sidecar` command has been renamed to `consul-sidecar`. [[GH-428](https://github.com/hashicorp/consul-k8s/pull/428)]
* Connect: the `consul-connect-lifecycle-sidecar` container name has been changed to `consul-sidecar` and the `consul-connect-envoy-sidecar` container name has been changed to `envoy-sidecar`. 
[[GH-428](https://github.com/hashicorp/consul-k8s/pull/428)]
* Connect: the `-default-protocol` and `-enable-central-config` flags are no longer supported.
  The `consul.hashicorp.com/connect-service-protocol` annotation on Connect pods is also
  no longer supported. [[GH-418](https://github.com/hashicorp/consul-k8s/pull/418)]

  Current deployments that have the annotation should remove it, otherwise they
  will get an error if a pod from that deployment is rescheduled.

  Removing the annotation will not change their protocol
  since the config entry was already written to Consul. If you wish to change
  the protocol you must migrate the config entry to be managed by a
  [`ServiceDefaults`](https://www.consul.io/docs/agent/config-entries/service-defaults) resource.
  See [Upgrade to CRDs](https://www.consul.io/docs/k8s/crds/upgrade-to-crds) for more
  information.

  To set the protocol for __new__ services, you must use the
  [`ServiceDefaults`](https://www.consul.io/docs/agent/config-entries/service-defaults) resource,
  e.g.

  ```yaml
  apiVersion: consul.hashicorp.com/v1alpha1
  kind: ServiceDefaults
  metadata:
    name: my-service-name
  spec:
    protocol: "http"
  ```
* Connect: pods using an upstream that references a datacenter, e.g.
  `consul.hashicorp.com/connect-service-upstreams: service:8080:dc2` will
  error during injection if Consul does not have a `proxy-defaults` config entry
  with a [mesh gateway mode](https://www.consul.io/docs/connect/config-entries/proxy-defaults#mode)
  set to `local` or `remote`. [[GH-421](https://github.com/hashicorp/consul-k8s/pull/421)]

  In practice, this would have already been causing issues since without that
  config setting, traffic wouldn't have been routed through mesh gateways and
  so would not be actually making it to the other service.

FEATURES:
* CRDs: support annotation `consul.hashicorp.com/migrate-entry` on custom resources
  that will allow an existing config entry to be migrated onto a Kubernetes custom resource. [[GH-419](https://github.com/hashicorp/consul-k8s/pull/419)]
* Connect: add new cleanup controller that runs in the connect-inject deployment. This
  controller cleans up Consul service instances that remain registered despite their
  pods being deleted. This could happen if the pod's `preStop` hook failed to execute
  for some reason. [[GH-433](https://github.com/hashicorp/consul-k8s/pull/433)]

IMPROVEMENTS:
* CRDs: give a more descriptive error when a config entry already exists in Consul. [[GH-420](https://github.com/hashicorp/consul-k8s/pull/420)]
* Set `User-Agent: consul-k8s/<version>` header on calls to Consul where `<version>` is the current
  version of `consul-k8s`. [[GH-434](https://github.com/hashicorp/consul-k8s/pull/434)]

## 0.23.0 (January 22, 2021)

BUG FIXES:
* CRDs: Fix issue where a `ServiceIntentions` resource could be continually resynced with Consul
  because Consul's internal representation had a different order for an array than the Kubernetes resource. [[GH-416](https://github.com/hashicorp/consul-k8s/pull/416)] 
* CRDs: **(Consul Enterprise only)** default the `namespace` fields on resources where Consul performs namespace defaulting to prevent constant re-syncing.
  [[GH-413](https://github.com/hashicorp/consul-k8s/pull/413)]

IMPROVEMENTS:
* ACLs: give better error if policy that consul-k8s tries to update was created manually by user. [[GH-412](https://github.com/hashicorp/consul-k8s/pull/412)]

FEATURES:
* TLS: add `tls-init` command that is responsible for creating and updating Server TLS certificates. [[GH-410](https://github.com/hashicorp/consul-k8s/pull/410)]

## 0.22.0 (December 21, 2020)

BUG FIXES:
* Connect: on termination of a connect injected pod the lifecycle-sidecar sometimes re-registered the application resulting in
  stale service entries for applications which no longer existed. [[GH-409](https://github.com/hashicorp/consul-k8s/pull/409)]

BREAKING CHANGES:
* Connect: the flags `-envoy-image` and `-consul-image` for command `inject-connect` are now required. [[GH-405](https://github.com/hashicorp/consul-k8s/pull/405)]

FEATURES:
* CRDs: add new CRD `IngressGateway` for configuring Consul's [ingress-gateway](https://www.consul.io/docs/agent/config-entries/ingress-gateway) config entry. [[GH-407](https://github.com/hashicorp/consul-k8s/pull/407)]
* CRDs: add new CRD `TerminatingGateway` for configuring Consul's [terminating-gateway](https://www.consul.io/docs/agent/config-entries/terminating-gateway) config entry. [[GH-408](https://github.com/hashicorp/consul-k8s/pull/408)]

## 0.21.0 (November 25, 2020)

IMPROVEMENTS:
* Connect: Add `-log-level` flag to `inject-connect` command. [[GH-400](https://github.com/hashicorp/consul-k8s/pull/400)]
* Connect: Ensure `consul-connect-lifecycle-sidecar` container shuts down gracefully upon receiving `SIGTERM`. [[GH-389](https://github.com/hashicorp/consul-k8s/pull/389)]
* Connect: **(Consul Enterprise only)** give more descriptive error message if using Consul namespaces with a Consul installation that doesn't support namespaces. [[GH-399](https://github.com/hashicorp/consul-k8s/pull/399)]

## 0.20.0 (November 12, 2020)

FEATURES:
* Connect: Support Kubernetes health probe synchronization with Consul for connect injected pods. [[GH-363](https://github.com/hashicorp/consul-k8s/pull/363)]
    * Adds a new controller to the connect-inject webhook which is responsible for synchronizing Kubernetes pod health checks with Consul service instance health checks.
      A Consul health check is registered for each connect-injected pod which mirrors the pod's Readiness status to Consul. This modifies connect routing to only
      pods which have passing Kubernetes health checks. See breaking changes for more information.
    * Adds a new label to connect-injected pods which mirrors the `consul.hashicorp.com/connect-inject-status` annotation.
    * **(Consul Enterprise only)** Adds a new annotation to connect-injected pods when namespaces are enabled: `consul.hashicorp.com/consul-namespace`. [[GH-376](https://github.com/hashicorp/consul-k8s/pull/376)]

BREAKING CHANGES:
* Connect: With the addition of the connect-inject health checks controller any connect services which have failing Kubernetes readiness
  probes will no longer be routable through connect until their Kubernetes health probes are passing.
  Previously, if any connect services were failing their Kubernetes readiness checks they were still routable through connect.
  Users should verify that their connect services are passing Kubernetes readiness probes prior to using health checks synchronization.

DEPRECATIONS:
* `create-inject-token` in the server-acl-init command has been un-deprecated.
  `-create-inject-auth-method` has been deprecated and replaced by `-create-inject-token`.
  
  `-create-inject-namespace-token` in the server-acl-init command has been deprecated. Please use `-create-inject-token` and `-enable-namespaces` flags
  to achieve the same functionality. [[GH-368](https://github.com/hashicorp/consul-k8s/pull/368)]

IMPROVEMENTS:
* Connect: support passing extra arguments to the envoy binary. [[GH-378](https://github.com/hashicorp/consul-k8s/pull/378)]
    
    Arguments can be passed in 2 ways:
    * via a flag to the consul-k8s inject-connect command,
      e.g. `consul-k8s inject-connect -envoy-extra-args="--log-level debug --disable-hot-restart"`
    * via pod annotations,
      e.g. `consul.hashicorp.com/envoy-extra-args: "--log-level debug --disable-hot-restart"`
      
* CRDs:
   * Add Age column to CRDs. [[GH-365](https://github.com/hashicorp/consul-k8s/pull/365)]
   * Add validations and field descriptions for ServiceIntentions CRD. [[GH-385](https://github.com/hashicorp/consul-k8s/pull/385)]
   * Update CRD sync status if deletion in Consul fails. [[GH-365](https://github.com/hashicorp/consul-k8s/pull/365)]

BUG FIXES:
* Federation: **(Consul Enterprise only)** ensure replication ACL token can replicate policies and tokens in Consul namespaces other than `default`. [[GH-364](https://github.com/hashicorp/consul-k8s/issues/364)]
* CRDs: **(Consul Enterprise only)** validate custom resources can only set namespace fields if Consul namespaces are enabled. [[GH-375](https://github.com/hashicorp/consul-k8s/pull/375)]
* CRDs: Ensure ACL token is global so that secondary DCs can manage custom resources.
  Without this fix, controllers running in secondary datacenters would get ACL errors. [[GH-369](https://github.com/hashicorp/consul-k8s/pull/369)]
* CRDs: **(Consul Enterprise only)** Do not attempt to create a `*` namespace when service intentions specify `*` as `destination.namespace`. [[GH-382](https://github.com/hashicorp/consul-k8s/pull/382)]
* CRDs: **(Consul Enterprise only)** Fix namespace support for ServiceIntentions CRD. [[GH-362](https://github.com/hashicorp/consul-k8s/pull/362)]
* CRDs: Rename field namespaces -> namespace in ServiceResolver CRD. [[GH-365](https://github.com/hashicorp/consul-k8s/pull/365)]

## 0.19.0 (October 12, 2020)

FEATURES:
* Add beta support for new commands `consul-k8s controller` and `consul-k8s webhook-cert-manager`. [[GH-353](https://github.com/hashicorp/consul-k8s/pull/353)]

  `controller` will start a Kubernetes controller that acts on Consul
  Custom Resource Definitions. The currently supported CRDs are:
    * `ProxyDefaults` - https://www.consul.io/docs/agent/config-entries/proxy-defaults
    * `ServiceDefaults` - https://www.consul.io/docs/agent/config-entries/service-defaults
    * `ServiceSplitter` - https://www.consul.io/docs/agent/config-entries/service-splitter
    * `ServiceRouter` - https://www.consul.io/docs/agent/config-entries/service-router
    * `ServiceResolver` - https://www.consul.io/docs/agent/config-entries/service-resolver
    * `ServiceIntentions` (requires Consul >= 1.9.0) - https://www.consul.io/docs/agent/config-entries/service-intentions
   
   See [https://www.consul.io/docs/k8s/crds](https://www.consul.io/docs/k8s/crds)
   for more information on the CRD schemas. **Requires Consul >= 1.8.4**.
   
   `webhook-cert-manager` manages certificates for Kubernetes webhooks. It will
   refresh expiring certificates and update corresponding secrets and mutating
   webhook configurations.

BREAKING CHANGES:
* Connect: No longer set `--max-obj-name-len` flag when executing `envoy`. This flag
  was [deprecated](https://www.envoyproxy.io/docs/envoy/latest/version_history/v1.11.0#deprecated)
  in Envoy 1.11.0 and had no effect from then onwards. With Envoy >= 1.15.0 setting
  this flag will result in an error, hence why we're removing it. [[GH-350](https://github.com/hashicorp/consul-k8s/pull/350)]

  If you are running any Envoy version >= 1.11.0 this change will have no effect. If you
  are running an Envoy version < 1.11.0 then you must upgrade Envoy to a newer
  version. This can be done by setting the `global.imageEnvoy` key in the
  Consul Helm chart.

IMPROVEMENTS:

* Add an ability to configure the synthetic Consul node name where catalog sync registers services. [[GH-312](https://github.com/hashicorp/consul-k8s/pull/312)]
  * Sync: Add `-consul-node-name` flag to the `sync-catalog` command to configure the Consul node name for syncing services to Consul.
  * ACLs: Add `-sync-consul-node-name` flag to the server-acl-init command so that it can create correct policy for the sync catalog.

BUG FIXES:
* Connect: use the first secret of type `kubernetes.io/service-account-token` when creating/updating auth method. [[GH-350](https://github.com/hashicorp/consul-k8s/pull/321)]
  
## 0.18.1 (August 10, 2020)

BUG FIXES:

* Connect: Reduce downtime caused by an alias health check of the sidecar proxy not being healthy for up to 1 minute
  when a Connect-enabled service is restarted. Note that this fix reverts the behavior of Consul Connect to the behavior
  it had before consul-k8s `v0.16.0` and Consul `v1.8.x`, where Consul can route to potentially unhealthy instances of a service
  because we don't respect Kubernetes readiness/liveness checks yet. Please follow [GH-155](https://github.com/hashicorp/consul-k8s/issues/155)
  for updates on that feature. [[GH-305](https://github.com/hashicorp/consul-k8s/pull/305)]

## 0.18.0 (July 30, 2020)

IMPROVEMENTS:

* Connect: Add resource request and limit flags for the injected init and lifecycle sidecar containers. These flags replace the hardcoded values previously included. As part of this change, the default value for the lifecycle sidecar container memory limit has increased from `25Mi` to `50Mi`. [[GH-298](https://github.com/hashicorp/consul-k8s/pull/298)], [[GH-300](https://github.com/hashicorp/consul-k8s/pull/300)]

BUG FIXES:

* Connect: Respect allow/deny list flags when namespaces are disabled. [[GH-296](https://github.com/hashicorp/consul-k8s/issues/296)]

## 0.17.0 (July 09, 2020)

BREAKING CHANGES:

* ACLs: Always update Kubernetes auth method created by the `server-acl-init` job. Previously, we would only update the auth method if Consul namespaces are enabled. With this change, we always update it to make sure that any configuration changes or updates to the `connect-injector-authmethod-svc-account` are propagated [[GH-282](https://github.com/hashicorp/consul-k8s/pull/282)].
* Connect: Connect pods have had the following resource settings changed: `consul-connect-inject-init` now has its memory limit set to `150M` up from `25M` and `consul-connect-lifecycle-sidecar` has its CPU request and limit set to `20m` up from `10m`. [[GH-291](https://github.com/hashicorp/consul-k8s/pull/291)]

IMPROVEMENTS:

* Extracted Consul's HTTP flags into our own package so we no longer depend on the internal Consul golang module. [[GH-259](https://github.com/hashicorp/consul-k8s/pull/259)]

BUG FIXES:

* Connect: Update resource settings to fix out of memory errors and CPU usage at 100% of limit. [[GH-283](https://github.com/hashicorp/consul-k8s/issues/283), [consul-helm GH-515](https://github.com/hashicorp/consul-helm/issues/515)]
* Connect: Creating a pod with a different service account name than its Consul service name will now result in an error when ACLs are enabled.
  Previously this would not result in an error, but the pod would not be able to send or receive traffic because its ACL token would be for a
  different service name. [[GH-237](https://github.com/hashicorp/consul-k8s/issues/237)]

## 0.16.0 (June 17, 2020)

FEATURES:

* ACLs: `server-acl-init` now supports creating tokens for ingress and terminating gateways [[GH-264](https://github.com/hashicorp/consul-k8s/pull/264)].
  * Add `-ingress-gateway-name` flag that takes the name of an ingress gateway that needs an acl token. May be specified multiple times. [Enterprise Only] If using Consul namespaces and registering the gateway outside of the default namespace, specify the value in the form `<GatewayName>.<ConsulNamespace>`.
  * Add `-terminating-gateway-name` flag that takes the name of a terminating gateway that needs an acl token. May be specified multiple times. [Enterprise Only] If using Consul namespaces and registering the gateway outside of the default namespace, specify the value in the form `<GatewayName>.<ConsulNamespace>`.
* Connect: Add support for configuring resource settings for memory and cpu limits/requests for sidecar proxies. [[GH-267](https://github.com/hashicorp/consul-k8s/pull/267)]

BREAKING CHANGES:

* Gateways: `service-address` command will now return hostnames if that is the address of the Kubernetes LB. Previously it would resolve the hostname to 1 IP. The `-resolve-hostnames` flag was added to preserve the IP resolution behavior. [[GH-271](https://github.com/hashicorp/consul-k8s/pull/271)]

IMPROVEMENTS:

* Sync: Add `-sync-lb-services-endpoints` flag to optionally sync load balancer endpoint IPs instead of load balancer ingress IP or hostname to Consul [[GH-257](https://github.com/hashicorp/consul-k8s/pull/257)].
* Connect: Add pod name to the consul connect metadata for connect injected pods. [[GH-231](https://github.com/hashicorp/consul-k8s/issues/231)]

BUG FIXES:

* Connect:
    * Fix bug where preStop hook was malformed. This caused Consul ACL tokens to never be deleted for connect services. [[GH-265](https://github.com/hashicorp/consul-k8s/issues/265)]
    * Fix bug where environment variable for upstream was not populated when using a different datacenter resulted. [[GH-246](https://github.com/hashicorp/consul-k8s/issues/246)]
    * Fix bug where the Connect health-check was defined with a service name instead of a service ID. This check was passing in consul version before 1.8, but will now fail with versions 1.8 and higher. [[GH-272](https://github.com/hashicorp/consul-k8s/pull/272)]

## 0.15.0 (May 13, 2020)

BREAKING CHANGES:

* The `service-address` command now resolves load balancer hostnames to the
  first IP. Previously it would use the hostname directly.
  This is a stop-gap measure because Consul currently only supports
  IP addresses for mesh gateways. [[GH-260](https://github.com/hashicorp/consul-k8s/pull/260)]

FEATURES:

* Add new `create-federation-secret` command that will create a Kubernetes secret
  containing data needed for secondary datacenters to federate. This command should be run only in the primary datacenter. [[GH-253](https://github.com/hashicorp/consul-k8s/pull/253)]

## 0.14.0 (April 23, 2020)

BREAKING CHANGES:

* ACLs: Remove `-expected-replicas`, `-release-name`, and `-server-label-selector` flags
  in favor of the new required `-server-address` flag [[GH-238](https://github.com/hashicorp/consul-k8s/pull/238)].

FEATURES:

* ACLs: The `server-acl-init` command can now run against Consul servers running outside of k8s [[GH-243](https://github.com/hashicorp/consul-k8s/pull/243)]:
  * Add `-bootstrap-token-file` flag to provide your own bootstrap token. If set, the command will
    skip ACL bootstrapping.
  * `-server-address` flag can also take a [cloud auto-join](https://www.consul.io/docs/agent/cloud-auto-join.html)
    string to discover server addresses.
  * Add `-inject-auth-method-host` flag to allow configuring the location of the Kubernetes API server
    for the Kubernetes auth method. This is useful because during the login workflow
    Consul servers are talking to the Kubernetes API to verify the service account token.
    When Consul servers are external to the Kubernetes cluster,
    we no longer know the address of the Kubernetes API server that is accessible
    from the external Consul servers.

IMPROVEMENTS:

* ACLs: Add `-server-address` and `-server-port` flags
  so that we don't need to discover server pod IPs and ports through the Kubernetes API [[GH-238](https://github.com/hashicorp/consul-k8s/pull/238)].

BUG FIXES:

* Connect: Fix upstream annotation parsing when multiple prepared queries are separated by spaces [[GH-224](https://github.com/hashicorp/consul-k8s/issues/224)]
* ACLs: Fix bug with `acl-init -token-sink-file` where running the command twice would fail [[GH-248](https://github.com/hashicorp/consul-k8s/pull/248)]

## 0.13.0 (April 06, 2020)

FEATURES:

* ACLs: Support new flag `server-acl-init -create-acl-replication-token` that creates
  an ACL token with permissions to perform ACL replication. [[GH-210](https://github.com/hashicorp/consul-k8s/pull/210)]
* ACLs: Support ACL replication from another datacenter. If `-acl-replication-token-file`
  is set, the `server-acl-init` command will skip ACL bootstrapping and instead
  will use the token in that file to create policies and tokens. This enables
  the `server-acl-init` command to be run in secondary datacenters. [[GH-226](https://github.com/hashicorp/consul-k8s/pull/226)]
* ACLs: Support new flag `acl-init -token-sink-file` that will write the token
  to the specified file. [[GH-232](https://github.com/hashicorp/consul-k8s/pull/232)]
* Commands: Add new command `service-address` that writes the address of the
  specified Kubernetes service to a file. If the service is of type `LoadBalancer`,
  the command will wait until the external address of the load balancer has
  been assigned. If the service is of type `ClusterIP` it will write the cluster
  IP. Services of type `NodePort` or `ExternalName` will result in an error.
  [[GH-234](https://github.com/hashicorp/consul-k8s/pull/234) and [GH-235](https://github.com/hashicorp/consul-k8s/pull/235)]
  
  Example usage:
  
      consul-k8s service-address \
        -k8s-namespace=default \
        -name=consul-mesh-gateway \
        -output-file=address.txt

* Commands: Add new `get-consul-client-ca` command that retrieves Consul clients' CA when auto-encrypt is enabled
  and writes it to a file [[GH-211](https://github.com/hashicorp/consul-k8s/pull/211)].

IMPROVEMENTS:

* ACLs: The following ACL tokens have been changed to local tokens rather than
  global tokens because they only need to be valid in their local datacenter:
  `client`, `enterprise-license`, `snapshot-agent`. In addition, if Consul
  Enterprise namespaces are not enabled, the `catalog-sync` token will be local. [[GH-226](https://github.com/hashicorp/consul-k8s/pull/226)]
* ACLs: If running with `-create-acl-replication-token=true` and `-create-inject-auth-method=true`,
  the anonymous policy will be configured to allow read access to all nodes and
  services. This is required for cross-datacenter Consul Connect requests to
  work. [[GH-230](https://github.com/hashicorp/consul-k8s/pull/230)].
* ACLs: The policy for the anonymous token has been renamed from `dns-policy` to `anonymous-token-policy`
  since it is used for more than DNS now (see above). [[GH-230](https://github.com/hashicorp/consul-k8s/pull/230)].

BUG FIXES:

* Sync: Fix a race condition where sync would delete services at initial startup [[GH-208](https://github.com/hashicorp/consul-k8s/pull/208)]

DEPRECATIONS:

* ACLs: The flag `-init-type=sync` for the command `acl-init` has been deprecated.
  Only the flag `-init-type=client` is supported. Previously, setting `-init-type=sync`
  had no effect so this is not a breaking change. [[GH-232](https://github.com/hashicorp/consul-k8s/pull/232)]
* Connect: deprecate the `-consul-ca-cert` flag in favor of `-ca-file` [[GH-217](https://github.com/hashicorp/consul-k8s/pull/217)]

## 0.12.0 (February 21, 2020)

BREAKING CHANGES:

* Connect Injector
  * Previously the injector would inject sidecars into pods in all namespaces. New flags `-allow-k8s-namespace` and `-deny-k8s-namespace` have been added. If no `-allow-k8s-namespace` flag is specified, the injector **will not inject sidecars into pods in any namespace**. To maintain the previous behavior, set `-allow-k8s-namespace='*'`.
* Catalog Sync
  * `kube-system` and `kube-public` namespaces are now synced from **unless** `-deny-k8s-namespace=kube-system -deny-k8s-namespace=kube-public` are passed to the `sync-catalog` command.
  * Previously, multiple sync processes could be run in the same Kubernetes cluster with different source Kubernetes namespaces and the same `-consul-k8s-tag`. This is no longer possible.
  The sync processes will now delete one-another's registrations. To continue running multiple sync processes, each process must be passed a different `-consul-k8s-tag` flag.
  * Previously, catalog sync would delete services tagged with `-consul-k8s-tag` (defaults to `k8s`) that were registered out-of-band, i.e. not by the sync process itself. It would delete services regardless of which node they were registered on.
  Now the sync process will only delete those services not registered by itself if they are on the `k8s-sync` node (the synthetic node created by the catalog sync process).

* Connect and Mesh Gateways: Consul 1.7+ now requires that we pass `-envoy-version` flag if using a version other than the default (1.13.0) so that it can generate correct bootstrap configuration. This is not yet supported in the Helm chart and consul-k8s, and as such, we require Envoy version 1.13.0.

IMPROVEMENTS:

* Support [**Consul namespaces [Enterprise feature]**](https://www.consul.io/docs/enterprise/namespaces/index.html) in all consul-k8s components [[GH-197](https://github.com/hashicorp/consul-k8s/pull/197)]
* Create allow and deny lists of k8s namespaces for catalog sync and Connect inject
* Connect Inject
  * Changes default Consul Docker image (`-consul-image`) to `consul:1.7.1`
  * Changes default Envoy Docker image (`-envoy-image`) to `envoyproxy/envoy-alpine:v1.13.0`

BUG FIXES:

* Bootstrap ACLs: Allow users to update their Connect ACL binding rule definition on upgrade
* Bootstrap ACLs: Fixes mesh gateway ACL policies to have the correct permissions
* Sync: Fixes a hot loop bug when getting an error from Consul when retrieving service information [[GH-204](https://github.com/hashicorp/consul-k8s/pull/204)]

DEPRECATIONS:
* `connect-inject` flag `-create-inject-token` is deprecated in favor of new flag `-create-inject-auth-method`

NOTES:

* Bootstrap ACLs: Previously, ACL policies were not updated after creation. Now, if namespaces are enabled, they are updated every time the ACL bootstrapper is run so that any namespace config changes can be adjusted. This change is only an issue if you are updating ACL policies after creation.

* Connect: Adds additional parsing of the upstream annotation to support namespaces. The format of the annotation becomes:

    `service_name.optional_namespace:port:optional_datacenter`

  The `service_name.namespace` is only parsed if namespaces are enabled. If they are not enabled and someone has added a `.namespace`, the upstream will not work correctly, as is the case when someone has put in an incorrect service name, port or datacenter. If namespaces are enabled and the `.namespace` is not defined, Consul will automatically fallback to assuming the service is in the same namespace as the service defining the upstream.


## 0.11.0 (January 10, 2020)

Improvements:

* Connect: Add TLS support [[GH-181](https://github.com/hashicorp/consul-k8s/pull/181)].
* Bootstrap ACLs: Add TLS support [[GH-183](https://github.com/hashicorp/consul-k8s/pull/183)].

Notes:

* Build: Our darwin releases for this version and up will be signed and notarized according to Apple's requirements.
Prior to this release, MacOS 10.15+ users attempting to run our software may see the error: "'consul-k8s' cannot be opened because the developer cannot be verified." This error affected all MacOS 10.15+ users who downloaded our software directly via web browsers, and was caused by changes to Apple's third-party software requirements.

  MacOS 10.15+ users should plan to upgrade to 0.11.0+.
* Build: ARM release binaries: Starting with 0.11.0, `consul-k8s` will ship three separate versions of ARM builds. The previous ARM binaries of Consul could potentially crash due to the way the Go runtime manages internal pointers to its Go routine management constructs and how it keeps track of them especially during signal handling (https://github.com/golang/go/issues/32912). From 0.11.0 forward, it is recommended to use:

  consul-k8s\_{version}\_linux_armelv5.zip for all 32-bit armel systems
  consul-k8s\_{version}\_linux_armhfv6.zip for all armhf systems with v6+ architecture
  consul-k8s\_{version}\_linux_arm64.zip for all v8 64-bit architectures
* Build: The `freebsd_arm` variant has been removed.


## 0.10.1 (December 17, 2019)

Bug Fixes:

* Connect: Fix bug where the new lifecycle sidecar didn't have permissions to
  read the ACL token file. [[GH-182](https://github.com/hashicorp/consul-k8s/pull/182)]

## 0.10.0 (December 17, 2019)

Bug Fixes:

* Connect: Fix critical bug where Connect-registered services instances would be deregistered
  when the Consul client on the same node was restarted. This fix adds a new
  sidecar that ensures the service instance is always registered. [[GH-161](https://github.com/hashicorp/consul-k8s/issues/161)]

* Connect: Fix bug where UI links between sidecar and service didn't work because
  the wrong service ID was being used. [[GH-163](https://github.com/hashicorp/consul-k8s/issues/163)]

* Bootstrap ACLs: Support bootstrapACLs for users setting the `nameOverride` config. [[GH-165](https://github.com/hashicorp/consul-k8s/issues/165)]

## 0.9.5 (December 5, 2019)

Bug Fixes:

* Sync: Add Kubernetes namespace as a suffix
  to the service names via `-add-k8s-namespace-suffix` flag.
  This prevents service name collisions in Consul when there
  are two services with the same name in different
  namespaces in Kubernetes [[GH-139](https://github.com/hashicorp/consul-k8s/issues/139)]

* Connect: Only write a `service-defaults` config during Connect injection if
  the protocol is set explicitly [[GH-169](https://github.com/hashicorp/consul-k8s/pull/169)]

## 0.9.4 (October 28, 2019)

Bug Fixes:

* Sync: Now changing the annotation `consul.hashicorp.com/service-sync` to `false`
  or deleting the annotation will un-sync the service. [[GH-76](https://github.com/hashicorp/consul-k8s/issues/76)]

* Sync: Rewrite Consul services to lowercase so they're valid Kubernetes services.
  [[GH-110](https://github.com/hashicorp/consul-k8s/issues/110)]

## 0.9.3 (October 15, 2019)

Bug Fixes:

* Add new delete-completed-job command that is used to delete the
  server-acl-init Kubernetes Job once it's completed. [[GH-152](https://github.com/hashicorp/consul-k8s/pull/152)]

* Fixes a bug where even if the ACL Tokens for the other components existed
  (e.g. client or sync-catalog) we'd try to generate new tokens and update the secrets. [[GH-152](https://github.com/hashicorp/consul-k8s/pull/152)]

## 0.9.2 (October 4, 2019)

Improvements:

* Allow users to set annotations on their Kubernetes services that get synced into
  Consul meta when using the Connect Inject functionality.
  To use, set one or more `consul.hashicorp.com/service-meta-<key>: <value>` annotations
  which will result in Consul meta `<key>: <value>`
  [[GH-141](https://github.com/hashicorp/consul-k8s/pull/141)]

Bug Fixes:

* Fix bug during connect-inject where the `-default-protocol` flag was being
  ignored [[GH-141](https://github.com/hashicorp/consul-k8s/pull/141)]

* Fix bug during connect-inject where service-tag annotations were
  being ignored [[GH-141](https://github.com/hashicorp/consul-k8s/pull/141)]

* Fix bug during `server-acl-init` where if any step errored then the command
  would exit and subsequent commands would fail. Now this command runs until
  completion, i.e. it retries failed steps indefinitely and is idempotent
  [[GH-138](https://github.com/hashicorp/consul-k8s/issues/138)]

Deprecations:

* The `consul.hashicorp.com/connect-service-tags` annotation is deprecated.
  Use `consul.hashicorp.com/service-tags` instead.

## 0.9.1 (September 18, 2019)

Improvements:

* Allow users to set tags on their Kubernetes services that get synced into
  Consul service tags via the `consul.hashicorp.com/connect-service-tags`
  annotation [[GH-115](https://github.com/hashicorp/consul-k8s/pull/115)]

Bug fixes:

* Fix bootstrap acl issue when Consul was installed into a namespace other than `default`
  [[GH-106](https://github.com/hashicorp/consul-k8s/issues/106)]
* Fix sync bug where `ClusterIP` services had their `Service` port instead
  of their `Endpoint` port registered. If the `Service`'s `targetPort` was different
  then `port` then the wrong port would be registered [[GH-132](https://github.com/hashicorp/consul-k8s/issues/132)]


## 0.9.0 (July 8, 2019)

Improvements:

* Allow creation of ACL token for Snapshot Agents
* Allow creation of ACL token for Mesh Gateways
* Allows client ACL token creation to be optional

## 0.8.1 (May 9, 2019)

Bug fixes:

* Fix central configuration write command to handle the case where the service already exists

## 0.8.0 (May 8, 2019)

Improvements:

* Use the endpoint IP address when generating a service id for NodePort services to prevent possible overlap of what are supposed to be unique ids
* Support adding a prefix for Kubernetes -> Consul service sync [[GH 140](https://github.com/hashicorp/consul-helm/issues/140)]
* Support automatic bootstrapping of ACLs in a Consul cluster that is run fully in Kubernetes.
* Support automatic registration of a Kubernetes AuthMethod for use with Connect (available in Consul 1.5+).
* Support central configuration for services, including proxy defaults (available in Consul 1.5+).

Bug fixes:

* Exclude Kubernetes system namespaces from Connect injection

## 0.7.0 (March 21, 2019)

Improvements:

* Use service's namespace when registering endpoints
* Update the Coalesce method to pass go vet tests
* Register Connect services along with the proxy. This allows the services to appear in the intention dropdown in the UI.[[GH 77](https://github.com/hashicorp/consul-helm/issues/77)]
* Add `-log-level` CLI flag for catalog sync

## 0.6.0 (February 22, 2019)

Improvements:

* Add support for prepared queries in the Connect upstream annotation
* Add a health endpoint to the catalog sync process that can be used for Kubernetes health and readiness checks

## 0.5.0 (February 8, 2019)

Improvements:

* Clarify the format of the `consul-write-interval` flag for `consul-k8s` [[GH 61](https://github.com/hashicorp/consul-k8s/issues/61)]
* Add datacenter support to inject annotation
* Update connect injector logging to remove healthcheck log spam and make important messages more visible

Bug fixes:

* Fix service registration naming when using Connect [[GH 36](https://github.com/hashicorp/consul-k8s/issues/36)]
* Fix catalog sync so that agents don't incorrectly deregister Kubernetes services [[GH 40](https://github.com/hashicorp/consul-k8s/issues/40)][[GH 59](https://github.com/hashicorp/consul-k8s/issues/59)]
* Fix performance issue for the k8s -> Consul catalog sync [[GH 60](https://github.com/hashicorp/consul-k8s/issues/60)]

## 0.4.0 (January 11, 2019)
Improvements:

* Supports a configurable tag for the k8s -> Consul sync [[GH 42](https://github.com/hashicorp/consul-k8s/issues/42)]

Bug fixes:

* Register NodePort services with the node's ip address [[GH 8](https://github.com/hashicorp/consul-k8s/issues/8)]
* Add the metadata/annotations field if needed before patching annotations [[GH 20](https://github.com/hashicorp/consul-k8s/issues/20)]

## 0.3.0 (December 7, 2018)
Improvements:

* Support syncing ClusterIP services [[GH 4](https://github.com/hashicorp/consul-k8s/issues/4)]

Bug fixes:

* Allow unnamed container ports to be used in connect-inject default
  annotations.

## 0.2.1 (October 26, 2018)

Bug fixes:

* Fix single direction catalog sync [[GH 7](https://github.com/hashicorp/consul-k8s/issues/7)]

## 0.2.0 (October 10, 2018)

Features:

* **New subcommand: `inject-connect`** runs a mutating admission webhook for
  automatic Connect sidecar injection in Kubernetes. While this can be setup
  manually, we recommend using the Consul helm chart.

## 0.1.0 (September 26, 2018)

* Initial release
