# Configurable Health Check Response Status

Add the ability to configure the HTTP response status code returned by the Kafka Bridge health check endpoints (`/ready` and `/healthy`) to accommodate load balancers that only accept HTTP 200 (OK) responses in health checks.

## Current situation

The Kafka Bridge returns HTTP 204 (No Content) for successful health check requests on the `/ready` and `/healthy` endpoints which is semantically correct (according to the HTTP standard) but is causing issues with various third-party load-balancers which only treat HTTP 200 (OK) as healthy response code.

## Motivation

Cloud load-balancers have varying requirements for health check HTTP status codes:

### Strict (require HTTP 200, not configurable)

- **[GCP](https://docs.cloud.google.com/load-balancing/docs/health-check-concepts#criteria-protocol-http)**
- **[Azure](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview#probe-protocol)**
- **[IBM Cloud](https://cloud.ibm.com/docs/loadbalancer-service?topic=loadbalancer-service-performing-health-checks-with-ibm-cloud-load-balancer)**

### Configurable (default 200, can be changed)

- **AWS**: Defaults to HTTP 200, manual configuration required for other response codes.
- **[OCI](https://docs.oracle.com/en-us/iaas/Content/Balance/Tasks/load_balancer_health_management.htm)**: Defaults to HTTP 200, supports custom status codes.
- **[OVHcloud](https://help.ovhcloud.com/csm/en-public-cloud-network-loadbalancer-create-health-monitor)**: Defaults to HTTP 200, supports custom status codes.

### Flexible (accept 2xx range by default)

- **[DigitalOcean](https://docs.digitalocean.com/products/networking/load-balancers/how-to/manage/)**: Accepts HTTP 2xx and 3xx.
- **[Linode](https://www.linode.com/docs/guides/configure-health-checks-nodebalancers-remove-nonworking-backends/)**: Accepts HTTP 2xx and 3xx.
- **[Alibaba Cloud](https://www.alibabacloud.com/help/en/slb/application-load-balancer/user-guide/health-check-management)**: Accepts HTTP 2xx.

The strict requirements prevent deploying the Kafka Bridge on major cloud platforms without workarounds (custom proxies, modified images, or complex routing). See Appendix for survey of affected open-source projects.

## Proposal

Introduce an environment variable `KAFKA_BRIDGE_HEALTH_CHECKS_RESPONSE_STATUS_200` that, when set to `true`, changes the successful response status code for the `/ready` and `/healthy` endpoints from 204 (No Content) to 200 (OK).

## Affected/not affected projects

This proposal only targets the Strimzi Kafka Bridge.

## Compatibility

Fully backward compatible. Default behavior unchanged (204), opt-in via environment variable. No API or configuration changes required.

## Rejected alternatives

### Changing the default to 200 (OK)

Changing the default from 204 to 200 would break existing deployments expecting 204. The opt-in approach provides flexibility without breaking changes.

### Making the response status fully configurable

Allowing arbitrary status codes (e.g., `KAFKA_BRIDGE_HEALTH_CHECKS_RESPONSE_STATUS=200`) adds unnecessary complexity. Only 200 vs. 204 has practical relevance for health checks.

### Using a configuration file property

Environment variables are standard for deployment-specific configuration in containerized environments and keep the configuration file focused on Kafka Bridge-specific settings.

## Appendix: Cloud Provider and Open-Source Project Survey

This survey demonstrates that HTTP 204 health check incompatibility is a systemic issue affecting both cloud platforms and open-source projects.

### Open-Source Projects Affected

1. **[InfluxData Telegraf](https://github.com/influxdata/telegraf/issues/4935)** - `/ping` endpoint incompatible with GCP health checks
2. **[InfluxData InfluxDB](https://github.com/influxdata/influxdb/issues/9772)** - Same issue, suggested making status code configurable
3. **[Authentik](https://github.com/goauthentik/authentik/pull/10554)** - Changed from 204 to 200 for multi-cloud compatibility
4. **[Meilisearch](https://github.com/meilisearch/meilisearch/issues/1282)** - Changed to 200 for GCP compatibility
5. **[Dapr](https://github.com/dapr/dapr/issues/3980)** - Google MultiClusterService requires 200
6. **[EventStore](https://github.com/EventStore/EventStore/issues/3468)** - `/health/live` incompatible with GCP
7. **[KairosDB](https://github.com/kairosdb/kairosdb/issues/271)** - 204 incompatible with AWS ELB
8. **[Thumbor](https://github.com/thumbor/thumbor/issues/218)** - `/healthcheck` not AWS ELB compliant
9. **[Kubernetes Ingress-Nginx](https://github.com/kubernetes/ingress-nginx/issues/573)** - Changed all 204 codes to 200 for GLBC
10. **[Eclipse MicroProfile Health](https://github.com/eclipse/microprofile-health/issues/3)** - Specification-level debate on 200 vs. 204
