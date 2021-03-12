# Docker Storj Node config with Loki, Prometheus, and Grafana

This config repo builds on the companion [fmoledina/storj-exporter-logs-dashboard](https://github.com/fmoledina/storj-exporter-logs-dashboard) repo and includes a full docker-compose stack that sets up the Storage node + Storj-Exporter + Loki log aggregation. See the above repo for Grafana dashboards that go with this setup.

Included is a `docker-compose.yml` which incorporates the following services + minimal configuration files for each:

- Storj node
- Storj-Exporter
- Prometheus
- Loki
- Promtail
- Grafana

## Storagenode Installation
### Specify Paths and Ports

Copy the example env vars files:

```shell
cp example.env .env
cp storj-example.env storj01.env
```

Edit the variables for all Storj node paths, Storj node ports, and appdata paths in `.env`. Also specify STORJ wallet and email address in `.env`. Edit the address and storage capacity in `storj01.env`. For multiple nodes, create additional entries in `.env` and a separate `storj02.env` file, and adapt the `docker-compose.yml` to include additional `storj##` and `exporter##` services.

### First Run

If this is the first time running the storagenode, it needs to be run with `SETUP=true` to create the directory structure and `config.yaml` file.

```shell
docker-compose run --rm --no-deps -e SETUP=true storj01
```

Repeat for additional nodes `storj02`, etc.

## Monitoring Stack Installation
### Log Output Format

As discussed in the [companion repo](https://github.com/fmoledina/storj-exporter-logs-dashboard), the log format in the storagenode config needs to be set to `json`.

> **Note: This method uses the `json` log output from the storagenode service rather than the familiar console output. The log output looks different than the typical `console` output. However, common tools such as [successrate.sh](https://github.com/ReneSmeekes/storj_success_rate) appear to be able parse JSON log files without issue.**
> 
> An example log line item in JSON format is as follows:
> 
> ```json
> {"L":"INFO","T":"2021-02-04T15:47:50.240Z","N":"piecestore","M":"download started","Piece ID":"H5MHLOAWQBOZKVABCF5SXROFFON2HSKZNSMFJSICIRDEFNRKZVBA","Satellite ID":"1wFTAgs9DP5RSnCqKV1eLf6N9wtk4EAtmN5DpSxcs8EjT69tGE","Action":"GET"}
> ```
> 
> To set the log output to JSON encoding, proceed with the following steps:
> 1. Stop the storagenode.
> 2. Edit the node `config.yaml` and set the following `log.encoding` parameter:
> 
> ```bash
> # configures log encoding. can either be 'console', 'json', or 'pretty'.
> log.encoding: "json"
> ```

The configuration in this repo works by sending logs directly from Docker to Loki via Promtail, without first exporting logs to a separate file. Ensure that log output is unspecified or set to `stderr` in the node `config.yaml` as follows:

```bash
# can be stdout, stderr, or a filename
# log.output: stderr
```

### Install Loki Docker Driver
I was able to get this working with Loki 2.1.0 but had problems running with *latest* images. Therefore, Loki Docker Driver and Loki versions were explicitly specified.

Install the Loki Docker Driver (see the [Loki Documentation](https://grafana.com/docs/loki/latest/clients/docker-driver/) for more info)

For x86_64, use the following:

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

N.B. The ARM images appear to be one-off builds rather than following a regular build cycle. For what it's worth, the `arm-v7` image has been successfully tested with this stack.

Check that the plugin was installed and is successfully started:
```shell
$ docker plugin ls
ID             NAME          DESCRIPTION           ENABLED
03fc82f3cabe   loki:latest   Loki Logging Driver   true
```

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
          instance: "storj01"   # Allows compatibility with Storj-Exporter-Dashboard
      # - targets: ["exporter02:9651"]
      #   labels:
      #     nodename: "storj02"
      #     instance: "storj02"
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

Add the dashboards from the [companion repo](https://github.com/fmoledina/storj-exporter-logs-dashboard#add-dashboard-to-grafana) to Grafana. See that repo for additional notes.