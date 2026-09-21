+++
date = '2026-09-09T15:00:53+02:00'
draft = false
title = 'Logging with Victoria Logs'
+++

**Victoria Logs** is a scalable logging solution designed to be fast and highly efficient. It provides users and organizations with better observability of their systems and helpsthem identify issues as well as debug, understand and analyze product behaviour.

![VictoriaMetrics Logo](/images/victoriametrics.jpg)

## Single server setup, using Docker Compose

The setup to runs Victoria Logs on Docker Compose is very easy.
One port, one data volume and all the configuration is done through command line parameters.
For this simple example, only one parameter: the storage data path.

```yaml
services:
  vic-logs:
    image: docker.io/victoriametrics/victoria-logs:v1.52.0
    ports:
      - "9428:9428"
      - "5514:5514"
      - "5514:5514/udp"
    volumes:
      - "/srv/victoria:/victoria"
    command:
      - "-storageDataPath=/victoria/data/logs"
      - "-syslog.listenAddr.tcp=:5514"
      - "-syslog.listenAddr.udp=:5514"
```

Adding **syslog** listener is easy as adding some ports and command line options

<!--more-->

```yaml
services:
  vic-logs:
    image: docker.io/victoriametrics/victoria-logs:v1.52.0
    ports:
      - "9428:9428"
    volumes:
      - "/srv/victoria:/victoria"
    command:
      - "-storageDataPath=/victoria/data/logs"
```

## Configuring clients

### From Systemd Journald

You can send directly the **Systemd Journald** logs to **Victoria Logs** using the **systemd-journal-upload** service.

In /etc/systemd/journal-upload.conf :

```ini
[Upload]
URL=http://192.168.1.100:9428/insert/journald
```

The enable the service

```bash
$ sudo systemctl enable --now systemd-journal-upload.service
```

On **NixOS**, enable the service and add the upload URL in the configuration:

```nix
  services.journald.upload = {
    enable = true;
    settings = {
      Upload = {
        URL = "http://192.168.1.100:9428/insert/journald";
      };
    };
  };
```

### From Docker daemon

The Docker daemon can be configured to ship it logs to syslog server.
See [docs.docker.com/engine/logging/configure](https://docs.docker.com/engine/logging/configure/)

```json
{
  "log-driver": "syslog",
  "log-opts": {
    "syslog-address": "udp://192.168.1.100:5514",
    "syslog-format": "rfc5424",
    "tag": "{{.Name}}"
  }
}
```
