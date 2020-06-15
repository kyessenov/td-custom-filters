# Using TrafficDirector to configure Istio extensions

This is a preliminary guide to use TrafficDirector instead of istiod for Istio extensions.
The goal is to enable service-to-service Prometheus stats, which is provided via two
extensions:

- `metadata_exchange` for sending and receiving peer metadata;
- `stats` for emitting metrics keyed on the peer attributes.

## Step 0: Set-up TrafficDirector

Follow the [official documentation](https://cloud.google.com/traffic-director/docs/set-up-gke-pods)
to set-up a basic mesh with TrafficDirector on GKE. This requires creating several GCP resources:
- target HTTP proxy `td-gke-proxy`;
- URL map `td-gke-url-map`;
- forwarding rule `td-gke-forwarding-rule`;
- (optional) firewall rule to permit GCP health checks to reach GKE pods.

You should have a service `service-test` and a deployment `app1`.

## Step 1: Inject Istio 1.6.1 proxies

Download and install `istioctl` from 1.6.1 version. You only need to install two configmaps `istio` and `istio-sidecar-injector`. Copy the two files from istio installation manifests, and make the following changes:

- change `discoveryAddress` to `traffidirector.googleapis.com:443`;
- change `caAddress` to `meshca.googleapis.com:443` (this makes sure that proxy readiness succeeds; without it, it would attempt to reach `istiod`, even though we do not need a CA in this exercise).

The two config maps are in [configmap.yaml](configmap.yaml). You can apply them as follows:

```
kubectl create namespace istio-system
kubectl apply -f configmap.yaml
```

### Step 1a: Inject into the service

Inject the proxy into TrafficDirector sample service and deployment:

```
istioctl kube-inject -f trafficdirector_service_sample.yaml > trafficdirector_service_sample.yaml.injected
```

There are additional changes needed to the injected templates:

- Remove `istio-ca-cert` volume and volume mounts.
- Remove readiness probe on port :15021. TD does not generate a health endpoint on this port, which makes the proxy fail readiness.
- Insert TrafficDirector bootstrap overrides:

```yaml
        - name: ISTIO_BOOTSTRAP
          value: "/var/lib/istio/envoy/gcp_envoy_bootstrap_tmpl.json"
        - name: ISTIO_META_TRAFFICDIRECTOR_INBOUND_INTERCEPTION_PORT
          value: "15006"
        - name: ISTIO_META_TRAFFICDIRECTOR_INBOUND_BACKEND_PORTS
          value: "8000"
```

See the resulting manifest [here](trafficdirector_service_sample.yaml.injected).

### Step 1b: Inject into a client

We are going to be using a simple [curl deployment](curl-deployment.yaml) as a client. Repeat the previous steps:

- Inject proxy with `kube-inject`.
- Remove `istio-ca-cert` volume and volume mounts.
- Remove readiness probe on port :15021 which is missing from GCP-specific bootstrap.
- Insert TrafficDirector bootstrap overrides:

```yaml
        - name: ISTIO_BOOTSTRAP
          value: "/var/lib/istio/envoy/gcp_envoy_bootstrap_tmpl.json"
        - name: ISTIO_META_TRAFFICDIRECTOR_INTERCEPTION_PORT
          value: "15001"

```

See the resulting manifest [here](curl-deployment.yaml.injected).

### Step 1c: Validate

Issue a request from `curl` pod to `app1` pod:

```
kubectl exec -it curl-deployment-5548cfdb5c-5cnwk -c curlpod curl service-test
```

You should see something like this:

```
app1-54658fdb7c-68dqw
```

## Step 2: Configure Istio extensions

Create HTTP custom filter GCP resources:

```
gcloud alpha network-services http-filters import istio-metadata-exchange --source=metadata_exchange.yaml --location=global
gcloud alpha network-services http-filters import istio-stats-inbound --source=stats_inbound.yaml --location=global
gcloud alpha network-services http-filters import istio-stats-outbound --source=stats_outbound.yaml --location=global
```

For each resource, record the full URL of the resources, which should look like https://networkservices.googleapis.com/v1alpha1/projects/XXXPROJECT/locations/global/httpFilters/istio-metadata-exchange.

### Step 2a: Enable on outbound proxies

Retrieve the existing HTTP proxy configuration and copy it:

```
gcloud alpha compute target-http-proxies export td-gke-proxy --destination thp.yaml
cp thp.yaml thp-with-mx-stats.yaml
```

Append the following snippet to the of `thp-with-mx-stats.yaml`:

```yaml
httpFilters:
  - https://networkservices.googleapis.com/v1alpha1/projects/XXXPROJECT/locations/global/httpFilters/istio-metadata-exchange
  - https://networkservices.googleapis.com/v1alpha1/projects/XXXPROJECT/locations/global/httpFilters/istio-stats-outbound

```

Apply the modified configuration:

```
gcloud alpha compute target-http-proxies import td-gke-proxy-mx-stats --source=thp-with-mx-stats.yaml
```

Enable the modified configuration:

```
gcloud compute forwarding-rules set-target td-gke-forwarding-rule --global --target-http-proxy=td-gke-proxy-mx-stats
```

### Step 2b: Enable on inbound proxies

Create and apply endpoint selector configuration:

```
gcloud alpha network-services endpoint-config-selectors import istio-config-selector --source=endpoint_config_selector.yaml --location=global
```

## Step 3: Validate

Issue a request again from curl pod to app1 pod:

```
kubectl exec -it curl-deployment-5548cfdb5c-5cnwk -c curlpod curl service-test
```

You should see again something like `app1-54658fdb7c-68dqw`. 

Extract metrics from the server proxy:

```
kubectl exec app1-54658fdb7c-68dqw -c istio-proxy curl localhost:15000/stats
```

You should see encoded service metrics, similar to this:

```
reporter=.=destination;.;source_workload=.=curl-deployment;.;source_workload_namespace=.=default;.;source_principal=.=unknown;.;source_app=.=curlpod;.;source_version=.=unknown;.;source_canonical_service=.=curl-deployment;.;source_canonical_revision=.=latest;.;destination_workload=.=app1;.;destination_workload_namespace=.=default;.;destination_principal=.=unknown;.;destination_app=.=unknown;.;destination_version=.=unknown;.;destination_service=.=service-test;.;destination_service_name=.=service-test;.;destination_service_namespace=.=default;.;destination_canonical_service=.=app1;.;destination_canonical_revision=.=latest;.;request_protocol=.=http;.;response_code=.=200;.;grpc_response_status=.=;.;response_flags=.=-;.;connection_security_policy=.=none;.;_istio_requests_total: 5
```

## Wrap-up

See the xDS configuration generated by TrafficDirector for [inbound](inbound.json) and [outbound](outbound.json).

