# YAS Helm Charts Repository

This repository acts as the central Helm Registry for the YAS (Yet Another Shop) microservices and infrastructure components.

## Repository Structure

* **`.github/workflows/`**
  * **[`charts-ci.yaml`](.github/workflows/charts-ci.yaml)**: GitHub Actions workflow that automatically packages and publishes Helm charts to GitHub Pages/GitHub Container Registry (GHCR) when changes are pushed.
* **`charts/`**
  * **Core Templates (Shared Base Charts)**
    * **`backend/`**: Common base Helm templates (`Deployment`, `Service`, `Ingress`, `HPA`) inherited by all YAS backend microservices.
    * **`ui/`**: Common base Helm templates inherited by all YAS frontend user interfaces.
  * **System Configuration**
    * **`yas-configuration/`**: Defines system-wide configuration maps and secrets (such as connection strings, application profiles, etc.).
  * **YAS Microservices**
    * **`product/`**, **`cart/`**, **`customer/`**, **`order/`**, **`inventory/`**, **`location/`**, **`media/`**, **`payment/`**, **`payment-paypal/`**, **`promotion/`**, **`rating/`**, **`recommendation/`**, **`search/`**, **`tax/`**, **`webhook/`**, **`sampledata/`**: Helm charts for individual microservices.
    * **`storefront-bff/`**, **`storefront-ui/`**, **`backoffice-bff/`**, **`backoffice-ui/`**: Helm charts for frontends and Backend-For-Frontends (BFF).
    * **`swagger-ui/`**: Chart to deploy Swagger UI for API documentation.
  * **Infrastructure Services**
    * **`postgresql/`**: Local PostgreSQL database setup (Zalando Postgres Operator).
    * **`pgadmin/`**: Database administration console.
    * **`kafka-cluster/`**: Strimzi Kafka cluster configuration and Debezium Connectors.
    * **`zookeeper/`**: Apache Zookeeper for Kafka orchestration.
    * **`elasticsearch-cluster/`**: ECK Elasticsearch and Kibana setup.
    * **`keycloak/`**: Identity and Access Management server.
    * **`opentelemetry/`**: OpenTelemetry Collector for observability logs and metrics.
    * **`grafana/`**: Grafana dashboards for metrics and tracing.

---

## How to Package and Publish Charts Locally

If you need to manually package the charts:

1. **Build dependencies** for a specific chart:
   ```bash
   helm dependency build charts/<chart-name>
   ```
2. **Package** the chart:
   ```bash
   helm package charts/<chart-name>
   ```
