### Details
The `monitoring-config` folder contains a sample Flux Grafana configuration of dashboards grabbed from the `fluxcd/flux2` repository on [20/9/22 from the docs](https://fluxcd.io/flux/guides/monitoring/). <br />
The Flux controllers exposes the `/metrics` endpoint on port 8080. When using Prometheus Operator you need a `PodMonitor` object to configure scraping for the controllers.