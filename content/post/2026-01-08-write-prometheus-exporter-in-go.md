+++
date = '2026-01-08T19:03:07+02:00'
title = 'Write a Prometheus exporter in Go'
+++

How to expose Prometheus metrics from a json blob returned by a server.

Here is a walk through, based on json retreived from an EthSwarm node.

## Retreive the json from EthSwarm status API

Here is a sample for the status json, from the API [documentation](https://docs.ethswarm.org/api/#tag/Node-Status),

```json
{
  "overlay": "36b7efd913ca4cf880b8eeac5093fa27b0825906c600685b6abdd6566e6cfe8f",
  "proximity": 0,
  "beeMode": "light",
  "reserveSize": 0,
  "reserveSizeWithinRadius": 0,
  "pullsyncRate": 0,
  "storageRadius": 0,
  "connectedPeers": 0,
  "neighborhoodSize": 0,
  "requestFailed": true,
  "batchCommitment": 0,
  "isReachable": true,
  "lastSyncedBlock": 0,
  "committedDepth": 0
}
```

## Prometheus exporter package

**ethswarm/status.go** start with the package declaration and imports

```go
package ethswarm

import (
	"encoding/json"
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
)
```

Then, the json and exporter types. The json type members must start with a capital, so the json decoder can access them.
Only a few fields are retreived and exported in this example to avoir repetition and keep the example short.

```go
type EthSwarmStatus struct {
	BeeMode                 string `json:"beeMode"`
	ConnectedPeers          int    `json:"connectedPeers"`
	IsReachable             bool   `json:"isReachable"`
}

type EthSwarmStatusExporter struct {
	apiURL                  string
	connectedPeers          *prometheus.Desc
	isReachable             *prometheus.Desc
	info                    *prometheus.Desc
}
```

The exporter builder function, passing the url parameter and settings metrics definitions.

```go
func NewEthSwarmStatusExporter(url string) *EthSwarmStatusExporter {
	return &EthSwarmStatusExporter{
		apiURL: url,

		connectedPeers: prometheus.NewDesc(
			"ethswarm_status_connected_peers",
			"connectedPeers field from status API endpoint",
			nil, nil,
		),

		isReachable: prometheus.NewDesc(
			"ethswarm_status_is_reachable",
			"isReachable field from status API endpoint",
			nil, nil,
		),

		info: prometheus.NewDesc(
			"ethswarm_status_info",
			"Infos from status API endpoint",
			[]string{"mode"}, nil,
		),
	}
}
```

Finally, the **Collect** and the **Describe** functions, that send the data through a channel.

```go
func (e *EthSwarmStatusExporter) Collect(ch chan<- prometheus.Metric) {
	resp, err := http.Get(e.apiURL + "/status")
	if err != nil { // FIXME: handle error
		log.Printf("Error fetching API: %v", err)
		return
	}
	defer resp.Body.Close()

	var data EthSwarmStatus
	if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
		log.Printf("Error decoding JSON: %v", err)
		return
	}

	ch <- prometheus.MustNewConstMetric(
		e.connectedPeers,
		prometheus.GaugeValue,
		float64(data.ConnectedPeers),
	)

	var fValue float64
	if data.IsReachable {
		fValue = 1
	} else {
		fValue = 0
	}
	ch <- prometheus.MustNewConstMetric(
		e.isReachable,
		prometheus.GaugeValue,
		float64(fValue),
	)

	ch <- prometheus.MustNewConstMetric(
		e.info,
		prometheus.GaugeValue,
		float64(1),
		data.BeeMode,
	)
}

func (e *EthSwarmStatusExporter) Describe(ch chan<- *prometheus.Desc) {
	ch <- e.connectedPeers
	ch <- e.isReachable
	ch <- e.info
}
```

## Main function with flag and http server

The main function is quite minimal. Register the exporter and start the server.

```go
package main

import (
	"flag"
	"fmt"
	"net/http"
	"strings"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"

	"myexporter/ethswarm"
)

func main() {
	argApiUrl := flag.String("n", "http://localhost:1633", "URL of the node API")
	argListenAddr := flag.String("l", ":8080", "Listen binding")
	flag.Parse()
	apiUrl := strings.Trim(*argApiUrl, "/")
	listenAddr := *argListenAddr

	prometheus.MustRegister(ethswarm.NewEthSwarmStatusExporter(apiUrl))

	fmt.Println("Get metrics from ", apiUrl, " and listen on ", listenAddr)
	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(listenAddr, nil)
}
```

## Docker image

Finally, a simple dual stage build for minimal size image. The **CGO_ENABLED=0** allow a static output, so it does not depend from anything and the final image can be based on **scratch** image.

```docker
FROM golang:1.25-trixie AS build

COPY . /root/esexporter

RUN set -xe ;\
  cd /root/esexporter ;\
  CGO_ENABLED=0 go build . ;\
  chmod -v 755 go-ethswarm-exporter

FROM scratch AS bin

COPY --from=build /root/esexporter/go-ethswarm-exporter /go-ethswarm-exporter

EXPOSE 8080
```

The result is only **13.1MB** large.

```
IMAGE                     ID             DISK USAGE   CONTENT SIZE   EXTRA
ethswarmexporter:latest   51f0eb282a6a       13.1MB             0B
```

To compage with the debian slim image that is 25-32Mb large

![linux/amd64 28.39 MB](/images/docker-debian-slim-size.png)
