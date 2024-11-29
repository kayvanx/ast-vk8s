## Overview


This project provides a custom Kubernetes deployment of the Application Study Tool (AST). AST is a powerful tool that helps analyze and monitor the performance of F5 BIG-IP devices.  For detailed configuration, troubleshooting information, and more, see the [AST Docsite](https://f5devcentral.github.io/application-study-tool/).

This solution enables F5 SEs to quickly set up dedicated AST instances for each customer. Since AST requires iControl access to BigIP devices, this deployment simplifies access by targeting an LB (`*.bigip.f5`) where each BigIP is defined as a route. This allows access to the BigIP devices either through the customer's network (via CE) or over the internet.


**Architecture:**

The application consists of three main deployments:

* **Custom instance of Grafana:**  Provides the user interface and dashboards. This image includes pre-built dashboards for BigIP devices.
* **Prometheus:** A time-series database used for storing and querying the collected data.
* **Custom Instance of OpenTelemetry Collector:**  Enhanced with BIG-IP data receivers to fetch data via iControl REST API.


## Workflow

Admin > LB(RE) > Grafana Dashboard

AST (otel-collector) > LB (`*.bigip.f5`) > BigIP (iControl)


**Recommendation:**

For deployments with multiple customers, you can reuse the LBs for different customers while deploying a separate AST instance (including Grafana, Prometheus, and OpenTelemetry Collector) within a dedicated namespace for each customer. 

To achieve this:

* **BigIP Access:**  Define an origin pool for each customer to access their respective BigIP devices. Create a route in the BigIP LB that maps the BigIP name (e.g., `[bigip_name].bigip.f5`) to the corresponding pool.
* **Grafana Access:**  Create a route in the Grafana LB that points to the dedicated Grafana service within the customer's namespace.

This approach provides isolation and simplifies management for individual customers.


## Installation

**Prerequisites:**

* A running Virtual Kubernetes Site in XC (can run on RE or CE vSites)
* `kubectl` command-line tool installed in your computer and configured to interact with your vK8s instance.

**Steps:**

1. **Configure the ConfigMap:**

   * Review the `ast-configmap.yaml` file. This ConfigMap in the beginning of the file stores essential configuration data for the application, including Grafana credentials and BigIP access details.
   * Set the following variables:
      *  `GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD`: Credentials for accessing the Grafana UI.
      *  `BIGIP_USER_1` and `BIGIP_PASSWORD_1`:  Credentials for accessing your first BigIP device.
      *  `BIGIP_URL_1`: The URL of your BigIP device (e.g., `[bigipname].bigip.f5`). 
      *  **Note:** You can add multiple BigIPs by updating the `receivers` section in the `ast-configmap.yaml` file. 
      *  **Note:** You can also configure the AST to export the telemetry data to F5 Data Fabric. For this you require to set `SENSOR_ID` (a unique string used to associate the metrics with the Organization) and a `SENSOR_SECRET_TOKEN` (used to authenticate to the F5 Datafabric as an authorized data sender for the Org).

   * Apply the ConfigMap:

     ```bash
     kubectl apply -f ast-configmap.yaml 
     ```

2. **Deploy the application components:**

   * Deploy the OpenTelemetry Collector: This component collects metrics from your BigIP devices.

     ```bash
     kubectl apply -f otel-collector-deployment.yaml 
     ```

   * Deploy Prometheus: This is the time-series database that stores the collected metrics.

     ```bash
     kubectl apply -f prometheus-deployment.yaml 
     ```

   * Deploy Grafana: This provides the user interface for visualizing the data.

     ```bash
     kubectl apply -f grafana-deployment.yaml
     ```


