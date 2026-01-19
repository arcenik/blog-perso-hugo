+++
date = '2025-06-22T12:20:32+02:00'
draft = true
title = 'Observations and analysis about systemd-networkd-wait-online.service'
+++

Since a few time, I have to wait two minutes on each reboot, waiting for a faulty systemd service: **systemd-networkd-wait-online.service**

```sh
server# journalctl --boot -u systemd-networkd-wait-online.service
Jun 22 13:38:00 server systemd[1]: Starting systemd-networkd-wait-online.service - Wait for Network to be Configured...
Jun 22 13:40:00 server systemd-networkd-wait-online[472]: Timeout occurred while waiting for network connectivity.
Jun 22 13:40:00 server systemd[1]: systemd-networkd-wait-online.service: Main process exited, code=exited, status=1/FAILURE
Jun 22 13:40:00 server systemd[1]: systemd-networkd-wait-online.service: Failed with result 'exit-code'.
Jun 22 13:40:00 server systemd[1]: Failed to start systemd-networkd-wait-online.service - Wait for Network to be Configured.
```

The service is linked to the **network-online** systemd target.

```sh
server# ls -l /etc/systemd/system/network-online.target.wants/
total 4
lrwxrwxrwx 1 root root 42 Mar 23 12:02 networking.service -> /usr/lib/systemd/system/networking.service
lrwxrwxrwx 1 root root 60 Apr 26 18:26 systemd-networkd-wait-online.service -> /usr/lib/systemd/system/systemd-networkd-wait-online.service
```

## Identify the problem

Reproduce the problem, running the command without any additional argument. Real time is 2 minutes and return 1.

```sh
server# time /usr/lib/systemd/systemd-networkd-wait-online ; echo $?
Timeout occurred while waiting for network connectivity.

real    2m0.236s
user    0m0.004s
sys     0m0.011s
1
```

<!--more-->

## Attempt to fix

Testing the commmand with extra argument, the command exit immediately and returns 0.

```sh
server# time /usr/lib/systemd/systemd-networkd-wait-online --timeout=2 -i enp2s0f0 ; echo $?

real    0m0.017s
user    0m0.013s
sys     0m0.004s
0
```

Now fix the service

```sh
server# systemctl edit --full systemd-networkd-wait-online.service
```

The command is changed to specify the network interface to watch and to reduce the timeout.

```ini
[Service]
Type=oneshot
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --timeout=2 -i enp2s0f0
```

The service is still failing, but mutch faster.

```sh
server# journalctl --boot -u systemd-networkd-wait-online.service
Jun 22 15:08:24 server systemd[1]: Starting systemd-networkd-wait-online.service - Wait for Network to be Configured...
Jun 22 15:08:26 server systemd-networkd-wait-online[478]: Timeout occurred while waiting for network connectivity.
Jun 22 15:08:26 server systemd[1]: systemd-networkd-wait-online.service: Main process exited, code=exited, status=1/FAILURE
Jun 22 15:08:26 server systemd[1]: systemd-networkd-wait-online.service: Failed with result 'exit-code'.
Jun 22 15:08:26 server systemd[1]: Failed to start systemd-networkd-wait-online.service - Wait for Network to be Configured.
```
