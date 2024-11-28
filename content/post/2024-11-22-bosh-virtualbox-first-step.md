+++
date = '2024-11-22T17:00:00+01:00'
draft = false
title = 'First steps with Bosh on Virtualbox'
keywords = ['open source', 'cloud', 'automation', 'bosh']
+++

## Introduction

[BOSH](https://bosh.io/docs/) is an open source project from [Cloud Foundry](https://www.cloudfoundry.org/) that allow release engineering, software deployment and application lifecycle management of large scale and distributed system.

![Cloud Foundry Bosh logo](/images/cf-bosh-logo.png)

I wanted to take a look at it and fortunately it is possible to play on a local playground environment on [VirtualBox](https://www.virtualbox.org/) using the [bosh-lite](https://bosh.io/docs/bosh-lite/) instruction.

## Not so easy

As I'm using Arch Linux, let's install [VirtualBox](https://archlinux.org/packages/extra/x86_64/virtualbox/) and [Bosh-Cli(aur)](https://aur.archlinux.org/packages/bosh-7-cli).

First clone the bosh-deployment repo:

```sh
$ git clone https://github.com/cloudfoundry/bosh-deployment.git bosh-deployment
```

Then create the Bosh environment, using the quite long create-env command:

```sh
$ bosh create-env ./bosh-deployment/bosh.yml \
  --state ./state.json \
  -o ./bosh-deployment/virtualbox/cpi.yml \
  -o ./bosh-deployment/virtualbox/outbound-network.yml \
  -o ./bosh-deployment/bosh-lite.yml \
  -o ./bosh-deployment/bosh-lite-runc.yml \
  -o ./bosh-deployment/uaa.yml \
  -o ./bosh-deployment/credhub.yml \
  -o ./bosh-deployment/jumpbox-user.yml \
  --vars-store ./creds.yml \
  -v director_name=bosh-lite \
  -v internal_ip=192.168.56.6 \
  -v internal_gw=192.168.56.1 \
  -v internal_cidr=192.168.56.0/24 \
  -v outbound_network_name=NatNetwork
```

So far so good

```text
<...>
Started validating
  Downloading release 'bosh'... Skipped [Found in local cache] (00:00:00)
  Validating release 'bosh'... Finished (00:00:00)
  Downloading release 'bpm'... Skipped [Found in local cache] (00:00:00)
  Validating release 'bpm'... Finished (00:00:00)
  Downloading release 'bosh-virtualbox-cpi'... Skipped [Found in local cache] (00:00:00)
  Validating release 'bosh-virtualbox-cpi'... Finished (00:00:03)
  Downloading release 'garden-runc'... Skipped [Found in local cache] (00:00:00)
  Validating release 'garden-runc'... Finished (00:00:01)
  Downloading release 'bosh-warden-cpi'... Skipped [Found in local cache] (00:00:00)
  Validating release 'bosh-warden-cpi'... Finished (00:00:00)
  Downloading release 'os-conf'... Skipped [Found in local cache] (00:00:00)
  Validating release 'os-conf'... Finished (00:00:00)
  Downloading release 'uaa'... Skipped [Found in local cache] (00:00:00)
  Validating release 'uaa'... Finished (00:00:01)
  Downloading release 'credhub'... Skipped [Found in local cache] (00:00:00)
  Validating release 'credhub'... Finished (00:00:00)
  Validating cpi release... Finished (00:00:00)
  Validating deployment manifest... Finished (00:00:00)
  Downloading stemcell... Skipped [Found in local cache] (00:00:00)
  Validating stemcell... Finished (00:00:05)
Finished validating (00:00:19)

Started installing CPI
  Compiling package 'golang-1-linux/79f531850e62e3801f1dfa4acd11c421aebe653cd4316f6e49061818071bb617'... Finished (00:00:16)
  Compiling package 'golang-1-darwin/42353203720fdc244cc16f881167bb0363f38ff6f991247c4356e38bc6b5fb81'... Finished (00:00:17)
  Compiling package 'virtualbox_cpi/80aa8cbfaa414f488b4f7821397f42edea6473638e50154b20db3f1ddf234b83'...
```

But after less than 2 minutes, it panicked while trying to get the VirtualBox version.

```text
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Stderr:
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Successful: true (0)
[driver.LocalRunner] 2024/11/25 14:34:02 DEBUG - Execute 'VBoxManage --version'
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Running command 'VBoxManage --version'
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Stdout: WARNING: Environment variable LOGNAME or USER does not correspond to effective user id.
7.1.4r165100
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Stderr:
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Successful: true (0)
panic: Internal inconsistency: Expected len(Interface '(.+)' was successfully created matches) >= 3:

goroutine 1 [running]:
bosh-virtualbox-cpi/vm/network.Networks.getVboxVersion({{0x6fd610, 0xc0000b2aa0}, {0x702b28, 0xc000190060}})
        /home/fs/.bosh/installations/8a86c2ee-8dce-477f-6ddf-cde281d5c02d/tmp/bosh-release-pkg2644627013/bosh-virtualbox-cpi/vm/network/system_info.go:61 +0x159
bosh-virtualbox-cpi/vm/network.Networks.NewSystemInfo({{0x6fd610, 0xc0000b2aa0}, {0x702b28, 0xc000190060}})
        /home/fs/.bosh/installations/8a86c2ee-8dce-477f-6ddf-cde281d5c02d/tmp/bosh-release-pkg2644627013/bosh-virtualbox-cpi/vm/network/system_info.go:18 +0x4c
<...>
```

And a [2 months issue](https://github.com/cloudfoundry/bosh-virtualbox-cpi-release/issues/39) is already opened.

## Let's fix it

### What is the problem ?

So, here is the faulty function, from [bosh-virtualbox-cpi-release](https://github.com/cloudfoundry/bosh-virtualbox-cpi-release/) repo in src/bosh-virtualbox-cpi/vm/network/system_info.go:

```go
func (n Networks) getVboxVersion() (string, string, error) {
	output, err := n.driver.Execute("--version")
	if err != nil {
		return "", "", err
	}

	output = strings.TrimSpace(output)
	matches := strings.Split(output, ".")

	if len(matches) > 3 {
		panic(fmt.Sprintf("Internal inconsistency: Expected len(%s matches) >= 3:", createdHostOnlyMatch))
	}

	return matches[0], matches[1], nil
}
```

Indeed, we can see the output contains a long warning line that ends with a dot. Therefore the split function returns 4 elements.

```text
[Cmd Runner] 2024/11/25 14:34:02 DEBUG - Stdout: WARNING: Environment variable LOGNAME or USER does not correspond to effective user id.
7.1.4r165100
```

Is it reproducible ?

```sh
$ export LOGNAME=bla
$ VBoxManage --version
WARNING: Environment variable LOGNAME or USER does not correspond to effective user id.
7.1.4r165100

$ VBoxManage
WARNING: Environment variable LOGNAME or USER does not correspond to effective user id.
Oracle VirtualBox Command Line Management Interface Version 7.1.4
Copyright (C) 2005-2024 Oracle and/or its affiliates

Usage - Oracle VirtualBox command-line interface:

  VBoxManage [-q | --nologo] [--settingspw=password] [--settingspwfile=pw-file] [@response-file] [subcommand]

  VBoxManage help [subcommand]

  VBoxManage commands

  VBoxManage [-V | --version]

<...>
```

Indeed, and it show this extra warning on any command, even when displaying usage help.

To avoid to check this in all instances of **n.driver.Execute**, let's patch the **Execute** function itself.

```go
func (d ExecDriver) Execute(args ...string) (string, error) {
        output, error := d.ExecuteComplex(args, ExecuteOpts{})
        if strings.HasPrefix(output, "WARNING:") { // cut the first WARNING line if present
                output = output[len(strings.Split(output, "\n")[0]):]
        }
        return output, error
}
```

But before I can try again, I need to create a release and update the deployment to use it.

### Create a Bosh release

That looks easy, just use the create-release

```sh
$ bosh create-release --force --tarball=~/tmp/vbox-cpi-test.tgz
-- Started downloading 'golang-1-darwin/42353203720fdc244cc16f881167bb0363f38ff6f991247c4356e38bc6b5fb81' (sha1=sha256:ad0052d2c45bf20672f58ed4d15c368554014632e8c051033f5b0caacd025237)
-- Started downloading 'guest-additions/8c4bb0eb2604cd2ec71c3ee0c4349d26b1da64635fbbfc61fdb4eccddee9f35a' (sha1=sha256:31fb7d6cd62a235d52b27ce73fa03edbdc5750f75f49432edd936b7c957029ac)
-- Started downloading 'golang-1-linux/79f531850e62e3801f1dfa4acd11c421aebe653cd4316f6e49061818071bb617' (sha1=sha256:329aecaedd48d3ca6d3e5cd1f66e626e5b93fda0664770dda698d5498e82a5d3)
-- Started downloading 'virtualbox_cpi/d4f4a419067339daecb3cb3e023ae3bd692a094cd7804d2febe5eb231261378e' (sha1=sha256:dbb6b3b955f7ea9c07be8a564e004297f8b757c028ebdc74e359d996a62745d4)
-- Started downloading 'virtualbox_cpi/80aa8cbfaa414f488b4f7821397f42edea6473638e50154b20db3f1ddf234b83' (sha1=sha256:25d5b825113e7b5ef91b4e76d3b99618d7d41991952a3940a581f2a96fc03dea)

-- Failed downloading 'golang-1-darwin/42353203720fdc244cc16f881167bb0363f38ff6f991247c4356e38bc6b5fb81' (sha1=sha256:ad0052d2c45bf20672f58ed4d15c368554014632e8c051033f5b0caacd025237)

<...>

Building a release from directory '/home/fs/tmp/bosh-virtualbox-cpi-release':
  - Downloading blob 'e984d783-46bd-47c5-4cd2-1fb653a1e702' with digest string 'sha256:ad0052d2c45bf20672f58ed4d15c368554014632e8c051033f5b0caacd025237':
      Getting blob from inner blobstore:
        Checking downloaded blob 'e984d783-46bd-47c5-4cd2-1fb653a1e702':
          Expected stream to have digest 'sha256:ad0052d2c45bf20672f58ed4d15c368554014632e8c051033f5b0caacd025237' but was 'sha256:2665bfdba2584e95a145a323afa69e8ed2c71c26ec45186dbe914cb7c1d60ed9'

<...>
```

But once again it is not that easy. The problem is that the repo uses [Git Large File Storage](https://git-lfs.com/). So let's install [git-lfs](https://archlinux.org/packages/extra/x86_64/git-lfs/) and install it on the repo


```sh
$ git lfs install
$ git lfs pull
```
Try again, and voila, the tgz release.

```sh
$ bosh create-release --force --tarball=~/tmp/vbox-cpi-test.tgz

<...>

Added dev release 'bosh-virtualbox-cpi/0.5.0+dev.6'

Name         bosh-virtualbox-cpi
Version      0.5.0+dev.6
Commit Hash  dcc799b+
Archive      /home/fs/tmp/vbox-cpi-test.tgz

Job                                                                               Digest                                                                   Packages
guest-additions/8c4bb0eb2604cd2ec71c3ee0c4349d26b1da64635fbbfc61fdb4eccddee9f35a  sha256:31fb7d6cd62a235d52b27ce73fa03edbdc5750f75f49432edd936b7c957029ac  -
virtualbox_cpi/d4f4a419067339daecb3cb3e023ae3bd692a094cd7804d2febe5eb231261378e   sha256:dbb6b3b955f7ea9c07be8a564e004297f8b757c028ebdc74e359d996a62745d4  virtualbox_cpi

2 jobs

Package                                                                           Digest                                                                   Dependencies
golang-1-darwin/42353203720fdc244cc16f881167bb0363f38ff6f991247c4356e38bc6b5fb81  sha256:ad0052d2c45bf20672f58ed4d15c368554014632e8c051033f5b0caacd025237  -
golang-1-linux/79f531850e62e3801f1dfa4acd11c421aebe653cd4316f6e49061818071bb617   sha256:329aecaedd48d3ca6d3e5cd1f66e626e5b93fda0664770dda698d5498e82a5d3  -
virtualbox_cpi/f7ac5a02effd72ac4c1d1d0e53722443bc22f0692063564d5ddb6025310b03e8   sha256:23e222d9b92efa5e32741601f2251b5ee57fd37683131a3495dcbd6434d2bd15  golang-1-darwin
                                                                                                                                                           golang-1-linux

3 packages

Succeeded
```

### And try again

Now I have my **vbox-cpi-test.tgz** release, let's update the bosh-deployment/virtualbox/cpi.yml to test it.

```yaml
- name: cpi
  path: /releases/-
  type: replace
  value:
    name: bosh-virtualbox-cpi
    url: file:///home/fs/tmp/vbox-cpi-test.tgz
```

That's look better

```text
<...>
Uploading stemcell 'bosh-vsphere-esxi-ubuntu-jammy-go_agent/1.639'... Finished (00:00:20)

Started deploying
  Creating VM for instance 'bosh/0' from stemcell 'sc-134454b8-442c-49b9-48d6-c5b52c9c4cb7'... Finished (00:00:02)
  Waiting for the agent on VM 'vm-326e4746-1005-4c29-6824-8255b6106696' to be ready...
```

And after less than 10 minutes it completed successfully.

```text
<...>
  Compiling package 'grootfs/4ec1e6a649cb2f3012ebb52b56f7892bd21521a8298afe658cc28430ca4354e4'... Skipped [Package already compiled] (00:00:00)
  Compiling package 'credhub/ccbb941ebc15f45f6248efa1b40194707c3155a7eb7e2ae99179d54233d3609f'... Skipped [Package already compiled] (00:00:00)
  Compiling package 'nginx/0520ac9ce5ea4ac2bf4e1daa650bb3c41b1156498247d01fbb4ba0a28d580b23'... Skipped [Package already compiled] (00:00:00)
  Compiling package 'bosh-gcscli/412eb42fcc4b8631620eb43ef09c6c126e103949ea5ae82812f5548d36e9770c'... Skipped [Package already compiled] (00:00:00)
  Updating instance 'bosh/0'... Finished (00:01:27)
  Waiting for instance 'bosh/0' to be running... Finished (00:02:42)
  Running the post-start scripts 'bosh/0'... Finished (00:00:01)
Finished deploying (00:06:22)

Cleaning up rendered CPI jobs... Finished (00:00:00)

Succeeded
```

Just setup the environment

```sh
$ export BOSH_CLIENT=admin
$ export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`
$ bosh alias-env vbox -e 192.168.56.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)


Name               bosh-lite
UUID               1f6e7843-bb32-4353-a666-c9a89a082e4f
Version            280.1.10 (00000000)
Director Stemcell  -/1.639
CPI                warden_cpi
Features           config_server: enabled
                   local_dns: enabled
                   snapshots: disabled
User               admin

Succeeded
```

Now I have a bosh director VM running on my VirtualBox, but I notice it have a constent CPU load, for a 4 CPUs VM.

![Bosh director VM with high CPU load](/images/vbox-bosh-director-load.png)

## Conclusion

Now That I have a working solution, commit it and push it to my forked repo and create a [PR](https://github.com/cloudfoundry/bosh-virtualbox-cpi-release/pull/41)

Then let's deploy the Zookeeper deployment as explained in the guide

```sh
$ bosh -e vbox \
-d zookeeper \
deploy <(curl -qs https://raw.githubusercontent.com/cppforlife/zookeeper-release/master/manifests/zookeeper.yml)
```

Oh no ....

```text
<...>
Task 3 | 15:25:27 | Creating missing vms: zookeeper/53330f7d-0ccd-4c30-847f-0a26a763ec74 (3) (00:00:10)
Task 3 | 15:25:27 | Creating missing vms: zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd (0) (00:00:10)
Task 3 | 15:25:28 | Creating missing vms: zookeeper/2a7b26ba-1eea-4e6e-8b4d-2729bf825ae6 (2) (00:00:11)
Task 3 | 15:25:28 | Updating instance zookeeper: zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd (0) (canary)
Task 3 | 15:25:29 | L executing pre-stop: zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd (0) (canary)
Task 3 | 15:25:29 | L executing drain: zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd (0) (canary)
Task 3 | 15:25:30 | L stopping jobs: zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd (0) (canary) (00:03:48)
                  L Error: Timed out sending 'get_task' to instance: 'zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd', agent-id: '8c8072c2-877a-4d1a-b39e-00d61f887760' after 45 seconds
Task 3 | 15:29:16 | Error: Timed out sending 'get_task' to instance: 'zookeeper/f7edbcb7-f7ef-40f5-80e0-5c5c1086f1fd', agent-id: '8c8072c2-877a-4d1a-b39e-00d61f887760' after 45 seconds

Task 3 Started  Mon Nov 25 15:23:13 UTC 2024
Task 3 Finished Mon Nov 25 15:29:16 UTC 2024
Task 3 Duration 00:06:03
Task 3 error

Updating deployment:
  Expected task '3' to succeed but state is 'error'

Exit code 1
```
