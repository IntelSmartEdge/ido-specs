```text
SPDX-License-Identifier: Apache-2.0
Copyright (c) 2020 Intel Corporation
```

# Telemetry support in OpenNESS

- [Telemetry support in OpenNESS](#telemetry-support-in-openness)
  - [Overview](#overview)
  - [Architecture](#architecture)
  - [Flavors and configuration](#flavors-and-configuration)
  - [Telemetry features](#telemetry-features)
    - [Prometheus](#prometheus)
      - [Usage](#usage)
    - [Grafana](#grafana)
      - [Usage](#usage-1)
    - [Node Exporter](#node-exporter)
      - [Usage](#usage-2)
      - [VCAC-A](#vcac-a)
    - [cAdvisor](#cadvisor)
      - [Usage](#usage-3)
    - [CollectD](#collectd)
      - [Plugins](#plugins)
      - [Usage](#usage-4)
    - [OpenTelemetry](#opentelemetry)
      - [Usage](#usage-5)
    - [PCM](#pcm)
      - [Usage](#usage-6)
    - [TAS](#tas)
      - [Usage](#usage-7)
  - [Summary](#summary)

## Overview

OpenNESS supports platform and application telemetry with the aid of multiple telemetry projects. This support allows users to retrieve information about the platform, the underlying hardware, cluster and applications deployed. The data gathered by telemetry can be used to visualize metric and resources consumption, set up alerts for certain events, and aid in making scheduling decisions based on the received telemetry. With the telemetry data at a user's disposal, a mechanism is also provided to schedule workloads based on the data available.

Currently, the support for telemetry is focused on metrics; support for application tracing telemetry is planned in the future.

## Architecture

The telemetry components used in OpenNESS are deployed from the Edge Controller as K8s (Kubernetes) pods. Among the components included for telemetry support are collectors, metric aggregators, schedulers, monitoring and visualization tools. Depending on the role of the component, it is deployed as either a `Deployment` or `Deamonset`. Generally global components receiving inputs from local collectors are deployed as a `Deployment` type with a single replica set, whereas local collectors running on each host are deployed as `Daemonsets`. Monitoring and visualization components such as Prometheus and Grafana along with TAS (Telemetry Aware Scheduler) are deployed on the Edge Controller while other components are generally deployed on Edge Nodes. Local collectors running on Edge Nodes collecting platform metrics are deployed as privileged containers. Communication between telemetry components is secured with TLS either using native TLS support for a given feature or using a reverse proxy running in a pod as a container. All the components are deployed as Helm charts.

![Telemetry Architecture](telemetry-images/architecture.svg)

## Flavors and configuration

The deployment of telemetry components in OpenNESS is easily configurable from the OEK (OpenNESS Experience Kit). The deployment of the Grafana dashboard and PCM (Performance Counter Monitoring) collector is optional (`telemetry_grafana_enable` enabled by default, `telemetry_pcm_enable` disabled by default). There are four distinctive flavors for the deployment of the CollectD collector, enabling the respective set of plugins (`telemetry_flavor`):

- common (default)
- flexran
- smartcity
- corenetwork

Further information on what plugins each flavor enables can be found in the [CollectD section](#collectd). All flags can be changed in `./group_vars/all/10-default.yml` for the default configuration or in `./flavors` in a configuration for a specific platform flavor.

## Telemetry features

The following section provides an overview of each of the components supported within OpenNESS along with a description on how to use the feature.

### Prometheus

Prometheus is an open-source, community-driven toolkit for systems monitoring and alerting. The main features includes PromQL query language, multi-dimensional time series data model and support for dashboards and graphs.

The main idea behind Prometheus is that it defines a unified metrics data format that can be hosted as part of any application that incorporates a simple web server. The data can be then scraped (downloaded) and processed by Prometheus using a simple HTTP/HTTPS connection.

In OpenNESS, Prometheus is deployed as a K8s Deployment with a single pod/replica on the Edge Controller node. It is configured out of the box to scrape all other telemetry endpoints/collectors enabled in OpenNESS and gather data from them. Prometheus is enabled in the OEK by default with the `telemetry/prometheus` role.

#### Usage

1. To connect to a Prometheus dashboard, start the internet browser on the same network as the OpenNESS cluster and enter the address of the dashboard (where IP address is address of the Edge Controller)

    ```shell
    From browser:
    http://<controller-ip>:30000
    ```

2. To list the targets/endpoints currently scraped by Prometheus, navigate to the `status` tab & select `targets` from drop down menu.
    ![Prometheus targets](telemetry-images/prometheus_targets.png)
3. To query a specific metric from one of the collectors, navigate to the `graph` tab, select a metric from the `insert-metric-at-cursor` list or type the value into field above and press `execute`.
    ![Prometheus metrics](telemetry-images/prometheus_metrics.png)
4. To graph the selected metrics press `graph` tab.
    ![Prometheus graph](telemetry-images/prometheus_graph.png)

### Grafana

Grafana is an open-source visualization and analytics software. It takes the data provided from external sources and displays relevant data to the user via dashboards. It enables the user to create customized dashboards based on the information the user wants to monitor and allows for the provision of additional data sources. In OpenNESS, the Grafana pod is deployed on the master node as a K8s `Deployment` type and is by default provisioned with data from Prometheus. It is enabled by default in OEK and can be enabled/disabled by changing the `telemetry_grafana_enable` flag. The password to gain access to the dashboard can be altered with `telemetry_grafana_pass` flag.

#### Usage

1. To connect to Grafana dashboard, start the internet browser on the same network as the OpenNESS cluster and enter the address of the dashboard (where IP address is address of the Edge Controller)

    ```shell
    From browser:
    http://<controller-ip>:32000
    ```

2. Log into the dashboard using the default credentials (login: admin, password: grafana)
    ![Grafana login](telemetry-images/grafana_login.png)
3. To create new dashboard, navigate to `http://<controller-ip>:32000/dashboards`.
    ![Grafana dash](telemetry-images/grafana-new-dash.png)
4. Navigate to dashboard settings.
    ![Grafana settings](telemetry-images/grafana-settings.png)
5. Change the name of dashboard in `General tab` and add new new variables as per following table from `Variables tab`:
    ![Dash settings](telemetry-images/grafana-dash-setting.png)
    | Field            | variable 1                                     | variable 2        | variable 3         |
    |------------------|:----------------------------------:|:----------------------------------------------------:|:-------------------------------------------------------------:|
    | Name             | job                                | hostname                                             | node                                                          |
    | Label            | JOB                                | Host                                                 | IP                                                            |
    | Data source      | Prometheus TLS                     | Prometheus TLS                                       | Prometheus TLS                                                |
    | Query            | label_values(node_uname_info, job) | label_values(node_uname_info{job=~"$job"}, nodename) | label_values(node_uname_info{nodename=~"$hostname"},instance) |
    | Sort             | Alphabetical(asc)                  | Disabled                                             | Alphabetical(asc)                                             |
    | Multi-value      |                                    | Enable                                               |                                                               |
    | Include All opt. |                                    | Enable                                               | Enable                                                        |

6. Save dashboard.
   ![Dash save](telemetry-images/grafana-save-dash.png)
7. Go to the newly created dashboard and add a panel.
   ![Dash save](telemetry-images/grafana-add-panel.png)
8. Configure the panel as per below configuration and press apply.
   ![Dash save](telemetry-images/grafana-panel-settings.png)

9. To query a specific metric in the panel select a data source (Prometheus) and pick a metric needed to be monitored. From visualization tab pick a desired style to display the metric. Give the panel a name and press apply in upper right corner.
   ![Grafana dash](telemetry-images/grafana-panel.png)
10. To save the dashboard click save in upper right corner.
   ![Grafana dash](telemetry-images/grafana-save.png)

### Node Exporter

Node Exporter is a Prometheus exporter that exposes hardware and OS metrics of *NIX kernels. The metrics are gathered within the kernel and exposed on a web server so they can be scraped by Prometheus. In OpenNESS, the Node Exporter pod is deployed as a K8s `Daemonset`, it is a privileged pod and runs on every Edge Node in the cluster. It is enabled by default by OEK.

#### Usage

1. To access metrics available from the Node Exporter connect to the Prometheus [dashboard](#prometheus).
2. Look up an example Node Exporter metric by specifying the metric name (ie. `node_memory_Active_anon_bytes`) and pressing `execute` under `graph` tab.
   ![Node Exporter Metric](telemetry-images/node_exporter_metric.png)

#### VCAC-A

Node Exporter also enables exposure of telemetry from Intel's VCAC-A card to Prometheus. The telemetry from the VCAC-A card is saved into a text file, this text file is used as an input to Node Exporter. More information on VCAC-A usage in OpenNESS is available [here](https://github.com/otcshare/x-specs/blob/master/doc/enhanced-platform-awareness/openness-vcac-a.md).

### cAdvisor

The cAdvisor is a running daemon providing information about containers running in the cluster.
It collects and aggregates data about running containers such as resource usage, isolation parameters and network statistics. Once the data is processed it is exported, the data can be easily obtained by metrics monitoring tools such as Prometheus. In OpenNESS, cAdvisor is deployed as a K8s `daemonset` on every Edge Node and scraped by Prometheus.

#### Usage

1. To access metrics available from cAdvisor, connect to the Prometheus [dashboard](#prometheus).
2. Look up an example cAdvisor metric by specifying the metric name (ie. `container_spec_cpu_quota`) and pressing `execute` under the `graph` tab.
   ![cAdvisor Metric](telemetry-images/cadvisor_metric.png)

### CollectD

CollectD is a daemon/collector enabling collection of hardware metrics from computers and network equipment. It provides support for CollectD plugins which extend its functionality for specific metrics collection such as Intel RDT, Intel PMU and ovs-dpdk. The metrics collected are easily exposed to the Prometheus monitoring tool via the usage of the `write_prometheus` plugin. In OpenNESS, CollectD is supported with the help of the [OPNFV Barometer project](https://wiki.opnfv.org/display/fastpath/Barometer+Home) - using its Docker image and available plugins. As part of OpenNESS release, a CollectD plugin for Intel FPGA PAC N3000 telemetry is now available from OpenNESS (power and temperature telemetry). In OpenNESS, CollectD pod is deployed as a K8s `Daemonset` on every available Edge Node, it is deployed as a privileged container.

#### Plugins

There are four distinctive sets of plugins (flavors) enabled for CollectD deployment which can be used depending on the use-case/workload being deployed on OpenNESS. `Common` flavor is the default flavor in OpenNESS. The flavors available are: `common`, `corenetwork`, `flexran`, `smartcity`. Below is a table specifying which CollectD plugins are enabled for each flavor.
The various OEK flavors are enabled for CollectD deployment as follows:


| Common           | Core Network      | FlexRAN           | SmartCity          |
|------------------|:-----------------:|------------------:|-------------------:|
| cpu              | cpu               | cpu               | cpu                |
| cpufreq          | cpufreq           | cpufreq           | cpufreq            |
| load             | load              | load              | load               |
| hugepages        | hugepages         | hugepages         | hugepages          |
| intel_pmu        | intel_pmu         | intel_pmu         | intel_pmu          |
| intel_rdt        | intel_rdt         | intel_rdt         | intel_rdt          |
| ipmi             | ipmi              | ipmi              | ipmi               |
| write_prometheus | write_prometheus  | write_prometheus  | write_prometheus   |
|                  | ovs_pmd_stats     | ovs_stats         | ovs_pmd_stat       |
|                  | ovs_stats         | fpga_telemetry    |                    |

#### Usage

1. Select the flavor for deployment of CollectD from the OEK during OpenNESS deployment - flavor is to be selected with `telemetry_flavor: <flavor name>`.

   In the event of using the `flexran` profile, `n3000-1-3-5-beta-cfg-2x2x25g-setup.zip` and `n3000-1-3-5-beta-rte-setup.zip` need to be available in `./openness-experience-kits/opae_fpga` directory; for more information about the packages see [FPGA support in OpenNESS](https://github.com/otcshare/x-specs/blob/master/doc/enhanced-platform-awareness/openness-fpga.md#edge-controller)
2. To access metrics available from CollectD, connect to the Prometheus [dashboard](#prometheus).
3. Look up an example CollectD metric by specifying the metric name (ie. `collectd_cpufreq`) and pressing `execute` under `graph` tab.
   ![CollectD Metric](telemetry-images/collectd_metric.png)

### OpenTelemetry

OpenTelemetry is a project providing APIs and libraries enabling collection of telemetry from edge services and network functions. This project provides a mechanism to capture application metrics and traces within applications with support for multiple programming languages. The captured metrics are exposed to OpenTelemetry agents and collectors responsible for handling of the data and forwarding it to desired back-end application(s) for processing and analysis. Among the supported back-ends are popular projects such as Prometheus (metrics), Jaeger and Zipkin (traces). A concept of receiver and exporter exists within the Open Telemetry agent and collector. The receiver provides a mechanism to receive the data into the agent or collector, multiple receivers are supported and at least one or more must be configured. On the opposite end, an exporter provides a mechanism to export data to the receiver, similarly multiple exporters are supported and one or more must be set up.

As part of OpenNESS release, only the support for collecting metrics and exposing them to Prometheus is enabled. The deployment model chosen for Open Telemetry is to deploy a global collector pod receiving data from multiple agents, the agents are to be deployed as side-car containers for each instance of application pod. The application exports data to the agent locally, the agent receives the data and exports it to the collector by specifying the collector’s K8s service name and port via TLS secured connection, the collector exposes all the available data received from multiple agents on an endpoint which is being scraped by Prometheus. OpenCensus exporter/receiver is used in the default OpenNESS configuration for agent and collector to collect metrics from an application. A sample application pod generating random numbers, aggregating them and exposing the metrics, containing a reference side-car OpenTelemetry agent is available in the Edge Apps repository.

#### Usage

1. Pull Edge Apps repository
2. Build the sample telemetry application Docker image and push to the local docker registry from the Edge Apps repo.

    ```shell
    cd edgeapps/applications/telemetry-sample-app/image/
    ./build.sh push <docker _registry_ip> <port>
    ```

3. Create a secret using a root-ca created as part of OEK telemetry deployment (this will authorize against the Collector certificates).

   ```shell
   cd edgeapps/applications/telemetry-sample-app/
   ./create-secret.sh
   ```

4. Configure and deploy the sample telemetry application with the side-car OpenTelemetry agent from the Edge Apps repo using Helm. Edit `edgeapps/applications/telemetry-sample-app/opentelemetry-agent/values.yaml` - change `app:image:repository: 10.0.0.1:5000/intel/metricapp` to IP address of Docker registry.
  
    ```shell
    cd edgeapps/applications/telemetry-sample-app/
    helm install otel-agent opentelemetry-agent
    ```

5. Check that the app/agent pod is successfully deployed:

    ```shell
    kubectl get pods otel-agent-sidecar

    NAME                 READY   STATUS    RESTARTS   AGE
    otel-agent-sidecar   2/2     Running   0          2m26s
    ```

6. Open your browser and go to `http://<controller_IP>:30000/graph`.
7. Input `opentelemetry_example_random_number_bucket` into query field and click execute.
8. Click `Graph` button and check if the graph is plotted (the amount of numbers in buckets should increase).

### PCM

Processor Counter Monitor (PCM) is an application programming interface (API) and a set of tools based on the API to monitor performance and energy metrics of Intel® Core™, Xeon®, Atom™ and Xeon Phi™ processors. In OpenNESS, the PCM pod is deployed as a K8s `Daemonset` on every available node. PCM metrics are exposed to Prometheus via the Host's NodePort on each EdgeNode. It is important to note that the PCM feature is intended to run on physical hardware (ie. no support for VM virtualized Edge Nodes in OpenNESS) therefore, this feature is disabled by default. The feature can be enabled by setting the `telemetry_pcm_enable` flag in OEK. Additionally a preset dashboard is created for PCM in Grafana visualizing the most crucial metrics.

#### Usage

1. To access metrics available from PCM connect to Prometheus [dashboard](#Prometheus).
2. Look up an example PCM metric by specifying metric name (ie. `L2_Cache_Hits`) and pressing `execute` under `graph` tab.
   ![PCM Metric](telemetry-images/pcm-metrics.png)
3. Log into [Grafana](#grafana).
4. Select dashboard to visualize metrics from desired node and click on it.
   ![PCM dashboard](telemetry-images/grafana-pcm-dashboard.png)
5. Metrics are visualized.
   ![PCM stats](telemetry-images/pcm-stats.png)

### TAS

[Telemetry Aware Scheduler](https://github.com/intel/telemetry-aware-scheduling) enables the user to make K8s scheduling decisions based on the metrics available from telemetry. This is crucial for a variety of Edge use-cases and workloads where it is critical that the workloads are balanced and deployed on the best suitable node based on hardware ability and performance. The user is able to create a set of policies defining the rules to which pod placement must adhere, functionality to de-schedule pods from given nodes if a rule is violated is also provided. TAS consists of a TAS Extender which is an extension to K8s scheduler, it correlates the scheduling policies with deployment strategies and returns decisions to the K8s Scheduler. It also consists of a TAS Controller which consumes TAS policies and makes them available to TAS components locally. A metrics pipeline which exposes metrics to a K8s API must be established for TAS to be able to read in the metrics. In OpenNESS, the metrics pipeline consists of Prometheus responsible for collecting and providing the metrics, and Prometheus Adapter exposing the metrics from Prometheus to a K8s API - this adapter is configured to provide metrics from Node Exporter and CollectD collectors. TAS is enabled by default in OEK, a sample scheduling policy for TAS is provided for [VCAC-A node deployment](https://github.com/otcshare/x-specs/blob/master/doc/enhanced-platform-awareness/openness-vcac-a.md#telemetry-support).

#### Usage

1. To check the status of the TAS Extender and Controller check the logs of respective containers.
   ```shell
   $ kubectl get pods | grep telemetry-aware-scheduling #Finds name of TAS pod

    telemetry-aware-scheduling-75596fd6b4-4qjvm   2/2     Running   2          5d3h
   
   $ kubectl logs <TAS POD name> -c tascont #Logs for Controller

    2020/05/23 15:11:47 Watching Telemetry Policies

   $ kubectl logs <TAS POD name> -c tasext # Logs for Extender

    2020/05/23 15:11:47 Extender Now Listening on HTTPS  9001
   ```
2. To check metrics available from the K8s API (A non-empty output expected)
   ```shell
   kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
   ```
3. To define a scheduling policy, create a `TASPolicy` definition file and deploy by running `kubectl apply -f <Policy file name>` - the following is an example for VCAC-A node.
   ```yaml
   apiVersion: telemetry.intel.com/v1alpha1
    kind: TASPolicy
    metadata:
      name: vca-vpu-policy
      namespace: default
    spec:
      strategies:
        deschedule:
          rules:
          - metricname: vpu_device_thermal
            operator: GreaterThan
            target: 60
        dontschedule:
          rules:
          - metricname: vpu_device_memory
            operator: GreaterThan
            target: 50
          - metricname: vpu_device_utilization
            operator: GreaterThan
            target: 50
        scheduleonmetric:
          rules:
          - metricname: vpu_device_utilization
            operator: LessThan
   ```
4. To link a created policy to a workload, add the required label (`telemetry-policy`), resources (`telemetry/scheduling: 1`) and node affinity (`affinity`) to the workload's specification file and deploy the workload.
   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: intelvpu-demo-tas
     labels:
       app: intelvpu-demo-tas
   spec:
     template:
       metadata:
         labels:
           jobgroup: intelvpu-demo-tas
           telemetry-policy: vpu-policy
       spec:
         restartPolicy: Never
         containers:
         -
             name: intelvpu-demo-job-2
             image: ubuntu-demo-openvino
             imagePullPolicy: IfNotPresent
             command: [ "/do_classification.sh" ]
             resources:
               limits:
                 vpu.intel.com/hddl: 1
                 telemetry/scheduling: 1
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
                 - matchExpressions:
                     - key: vpu-policy
                       operator: NotIn
                       values:
                         - violating
   ```

## Summary

In summary, the OpenNESS platform provides a comprehensive set of features and components enabling the users to collect, monitor and visualize a variety of platform metrics with strong focus on telemetry from Intel Architecture hardware - such as CPU, VCAC-A, and FPGA.
It also provides the means for making intelligent scheduling decisions for performance sensitive Edge workloads based on the performance and abilities of the hardware.
The support for collecting and visualizing metrics from within applications is also provided.

The telemetry component deployment is modular (each component running in its own pod) and easily configurable meaning that support for other telemetry collectors is possible in the future. Currently, only support for metrics telemetry is provided but support for logging and tracing telemetry/tools for applications is planned for future releases.
