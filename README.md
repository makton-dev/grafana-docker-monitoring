# grafana-docker-monitoring
The purpose of this repo is to maintain a central location for managing the docker configs for a fully operational On-Prem Grafana Log and Metric NMS/APM.

## How this came to be
I been running an 8 node K3S Rasberry Pie 4 cluster for years and one of the biggest issues was the about the ram and CPU being used when no apps were even installed. I've also started working with Amazon ECS as it's cheaper to run than EKS. However, I need a way to monitor all the apps I'm running and ensure all 8 nodes are healthy while having a small footprint.

To make this really flexible, I create a "server" config and an "agent" config. The agent config can be installed on any system with docker, but it will have access to host for host metrics.

## Included with server
The server includes the following:
* Grafana Dashboard
* Loki Logging data source
* Prometheus Metric data source

The setup is based on the docker compose from Grafana with modification for standardization and ease of use as I found many issues using the defaut conpose file from Grafana.

## Included with client
The following is included with the client:
* Node Exporter
* cAdvisor - Docker metric exporter
* Alloy

Alloy pulls from the logs and the metric exporters and sends to the Loki and Prometheus data sources. Yes, this is a "push-only" setup.
