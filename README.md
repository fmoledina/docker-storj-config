# Docker Storj Node config with Loki, Prometheus, and Grafana

This config repo sets up the Storage node + Storj-Exporter + Loki log aggregation.

Included is a `docker-compose.yml` which incorporates the following services + minimal configuration files for each:

- Storj node
- Storj-Exporter
- Prometheus
- Loki
- Promtail
- Grafana

## Motivation

I've been interested in exploring [Grafana Loki](https://grafana.com/oss/loki/) with [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/) for log ingestion and metrics for a number of different services on my home server. Testing it out for Storj nodes seemed like a great way to get an understanding of how it works.

For Storj nodes, the main benefit is that a single Promtail listener can injest logs from multiple nodes and produce metrics that Prometheus can then scrape. Individual storj-log-exporter instances are not required.

Furthermore, once the logs are ultimately shipped to Loki, one can do LogQL queries against them in Grafana or using [LogCLI](https://grafana.com/docs/loki/latest/getting-started/logcli/). For example, to search for all `ERROR` log level entries:

![LogQL](logQL.PNG)

## Log Output Format

**Note: This method uses the `json` log output from the storagenode service rather than the familiar console output. The log output looks different than the typical `console` output. However, common tools such as [successrate.sh](https://github.com/ReneSmeekes/storj_success_rate) appear to be able parse JSON log files without issue.**

An example log line item in JSON format is as follows:

```json
{"L":"INFO","T":"2021-02-04T15:47:50.240Z","N":"piecestore","M":"download started","Piece ID":"H5MHLOAWQBOZKVABCF5SXROFFON2HSKZNSMFJSICIRDEFNRKZVBA","Satellite ID":"1wFTAgs9DP5RSnCqKV1eLf6N9wtk4EAtmN5DpSxcs8EjT69tGE","Action":"GET"}
```

To set the log output to JSON encoding, proceed with the following steps:
1. Stop the storagenode.
2. Edit the node `config.yaml` and set the following `log.encoding` parameter:

```bash
# configures log encoding. can either be 'console', 'json', or 'pretty'.
log.encoding: "json"
```

## Docker Nodes

This method works to ship logs directly from Docker to Loki via Promtail, without exporting logs to a separate file. Ensure that log output is unspecified or set to `stderr` in the node `config.yaml` as follows:

```bash
# can be stdout, stderr, or a filename
# log.output: stderr
```

### Install Loki Docker Driver
I was able to get this working with Loki 2.1.0 but had problems running with *latest* images. Therefore, Loki Docker Driver and Loki versions were explicitly specified.

Install the Loki Docker Driver (see the [Loki Documentation](https://grafana.com/docs/loki/latest/clients/docker-driver/) for more info)

For AMD64, use the following:

```bash
docker plugin install grafana/loki-docker-driver:2.1.0 --alias loki --grant-all-permissions
```

For ARM, use the following:

```bash
docker plugin install grafana/loki-docker-driver:arm-v7 --alias loki --grant-all-permissions
```

For ARM64, use the following:

```bash
docker plugin install grafana/loki-docker-driver:arm-64 --alias loki --grant-all-permissions
```

Check that the plugin was installed and is successfully started:
```bash
$ docker plugin ls
ID             NAME          DESCRIPTION           ENABLED
03fc82f3cabe   loki:latest   Loki Logging Driver   true
```

### Specify Paths and Ports

Copy the example env vars files:

```shell
cp example.env .env
cp storj-example.env storj01.env
```

Edit the variables for all Storj node paths, Storj node ports, and appdata paths in `.env`. Also specify STORJ wallet and email address in `.env`. Edit the address and storage capacity in `storj01.env`. For multiple nodes, create additional entries in `.env` and a separate `storj02.env` file, and adapt the `docker-compose.yml` to include additional `storj##` and `exporter##` services.

### Configure prometheus.yml

Edit `./appconfig/prometheus/prometheus.yml` with Storj-Exporter instances. Note that the label used is `nodename` rather than `instance` as in Storj-Exporter and storj-log-exporter instructions. This is to align with the corresponding `nodename` variable provided by Promtail for Storj node log files.

Add the following to your existing job for each node:
```yaml
  - job_name: 'storj-exporter'
    static_configs:
      ...
      - targets: ["exporter01:9651"]
        labels:
          nodename: "storj01"   # Same as the docker-compose service name
      # - targets: ["exporter02:9651"]
      #   labels:
      #     nodename: "storj02"
      ...
```

Also included is the connection to the Promtail metrics endpoint, where all the log metrics will be scraped from:

```yaml
  - job_name: 'promtail'
    static_configs:
      - targets: ['promtail:9080']
```

### Start the Stack

Start all the services in the `docker-compose.yml`:

```shell
docker-compose up -d
```

### Add dashboard to Grafana

Add the dashboard from the file [dashboard_exporter_promtail.json](./dashboard_exporter_promtail.json) in the same way as described in KevinK's [How-To monitor all nodes in your lan](https://forum.storj.io/t/how-to-monitor-all-nodes-in-your-lan-using-prometheus-grafana-linux-using-docker). The dashboard in this repo is virtually unchanged, apart from referring to `nodename` instead of `instance` for each node, and adding a link to this repo at the top.

### Result

Storj-Exporter Promtail dashboard:

![Storj-Exporter-Promtail](./dashboard-exporter-promtail.PNG)

Storj-Logs dashboard:

![Storj-Logs](./dashboard-logs.PNG)

## Native Binary Nodes

This method uses Promtail with logs exported to file and doesn't offer much benefit over the existing [Storj-Log-Exporter](https://github.com/kevinkk525/storj-log-exporter). 

TODO add instructions for file-based logs for completion.

## Notes

The [Notes section from Storj-Log-Exporter](https://github.com/kevinkk525/storj-log-exporter#notes) are valid for the Promtail log metrics methodology presented in this repo.

## TODO

- Separate README for native binary Storj nodes, including links to guides for non-Docker Loki/Promtail installation
- 2 week log retention limit by default with instructions on how to increase
- Optional drop all logs except loglevel ERROR or FATAL (i.e. drop INFO, WARN, DEBUG)