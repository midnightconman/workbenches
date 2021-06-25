# Akamai Datastreams - Cribl - Prometheus Workbench

## Overview

This workbench allows engineers to spin up the various components in this pipeline for testing, prototyping, or proving. As of this writing the components in the pipeline are:

- AWS S3 - holds raw logs
- AWS SQS - receives AWS S3 create object events
- Cribl (AWS S3 Source) - pulls AWS SQS events to pull AWS S3 object creation (or update) events
- Cribl (Prometheus Destination) - pushes metrics (via remote_write) to prometheus
- Prometheus - receives metrics and allows visualization of received metrics

## How To Use This Workbench

### Requirements

This workbench requires that you have installed and know how to use the following components:

- linux (can be a VM, needed for docker / container runtimes)
- docker (or just containerd / cri-o -- mostly for kubernetes)
- kubernetes (not tested, but probably 1.12+)

### Deployment

#### LocalStack

As `localstack` provides some base infrastructure capabilities, it is a good idea to bring it up first. You can do this with a simple `apply` command via `kubectl`:

```
kubectl apply -f localstack.yaml
```

#### Prometheus

As prometheus receives metrics from cribl, it makes sense to bring it up before you bring up cribl. This is not a requirement, cribl has been configured to `drop` metrics in the event of back pressure from prometheus -- so this is a recommendation not a requirement.

```
kubectl apply -f prometheus.yaml
```

#### Cribl

The pipeline configurations have been baked into cribl configurations, so the pipeline will start processing once the cribl pod comes up. To generate mock data, please refer to the `Port-Forward To Components` or `Easily Create Mock Logs` sections.

```
kubectl apply -f cribl.yaml
```

### Port-Forward To Components

Sometimes you will need to access the UI / portal of the various components. This is very simple with kubernetes but can vary by kubernetes provider, so if you have any trouble -- look over the information for your kubernetes provider. Here is an example of a command that could be used to start a port-forwarding session with the prometheus pod running in your kubernetes cluster:

```
kubectl port-forward --address 0.0.0.0 svc/prometheus 9090:80
```

After this command is run, you would be able to pull up your browser and access this UI via `http://localhost:9090`.

### Easily Create Mock Logs

Here is an example of a command that will continually add simple log lines to a file and push the updated file to s3. This command can be run from inside the localstack pod, the aws command and credentials are available there.

```
while : ; do echo "$( date +%s%N ) - test" >> /tmp/s3.txt ; aws --endpoint-url=http://localhost:4566 s3 cp /tmp/s3.txt s3://cribl ; sleep 1 ; done
```

## Observations Made During Construction

### Cribl Metrics Dimensionality and Chronological Ordering

Cribl offers `batch` window (possibly sliding, need clarification from cribl engineering) evaluations for their aggregation functions... but it seems to prevent overlapping data, they add dimensions around batch window start and end times to all aggregation metrics. This type of dimensional data is normally referred to as `unbounded` and unbounded data should not be stored in label fields in prometheus, normally the only unbounded dimension in a TSDB should be the primary `time` dimension. There are cribl pipeline function configuration settings to drop / remove certain metrics dimensions, in this instance I removed `endtime` and `starttime`. Unfortunately after removing these dimensions, no further metrics are received from cribl, all posts to the prometheus remote write endpoint are returning an error like `level=error ts=2021-06-25T17:44:35.760Z caller=write_handler.go:53 component=web msg="Out of order sample from remote write" err="out of bounds"`.

Here is an example of what the `endtime` and `starttime` dimensions data looks like in prometheus (when enabled, remember we really need to remove these dimensions for prometheus):

```
count{cribl_host="cribl-64c64dc69-9vrc7", cribl_wp="w0", endtime="1624579100", starttime="1624579090"}
count{cribl_host="cribl-64c64dc69-9vrc7", cribl_wp="w0", endtime="1624579120", starttime="1624579110"}
count{cribl_host="cribl-64c64dc69-9vrc7", cribl_wp="w0", endtime="1624579130", starttime="1624579120"}
count{cribl_host="cribl-64c64dc69-9vrc7", cribl_wp="w0", endtime="1624579160", starttime="1624579150"}
```

## Links

- https://localstack.cloud
  - https://hub.docker.com/r/localstack/localstack
- https://developer.akamai.com/api/core_features/datastream2_config/v1.html#overview
- https://docs.cribl.io/docs/welcome
  - https://docs.cribl.io/docs/sources-s3 - Cribl Source - AWS S3 / SQS
  - https://docs.cribl.io/docs/destinations-prometheus - Cribl Destination - Prometheus
- https://prometheus.io/docs/prometheus/latest/getting_started/
  - https://prometheus.io/docs/prometheus/latest/storage/#remote-storage-integrations - Prometheus remote_write
