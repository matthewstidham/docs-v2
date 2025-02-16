---
title: Step 1 - Install InfluxDB Enterprise meta nodes
aliases:
    - /enterprise_influxdb/v1.8/production_installation/meta_node_installation/
menu:
  enterprise_influxdb_1_8:
    name: Step 1 - Install meta nodes
    weight: 10
    parent: Install in your environment
---

InfluxDB Enterprise offers highly scalable clusters on your infrastructure
and a management UI ([via Chronograf](/{{< latest "chronograf" >}}) for working with clusters.
The Production Installation process is designed for users looking to
deploy InfluxDB Enterprise in a production environment.
The following steps will get you up and running with the first essential component of
your InfluxDB Enterprise cluster: the meta nodes.

## Meta node setup description and requirements

The Production Installation process sets up three [meta nodes](/enterprise_influxdb/v1.8/concepts/glossary/#meta-node), with each meta node running on its own server.
<br>
You **must** have a minimum of three meta nodes in a cluster.
InfluxDB Enterprise clusters require at least three meta nodes and an __**odd number**__
of meta nodes for high availability and redundancy.
InfluxData does not recommend having more than three meta nodes unless your servers
or the communication between the servers have chronic reliability issues.
<br>
Note: Deploying multiple meta nodes on the same server is strongly discouraged since it creates a larger point of potential failure if that particular server is unresponsive.
InfluxData recommends deploying meta nodes on relatively small footprint servers.

See the
[Clustering Guide](/enterprise_influxdb/v1.8/concepts/clustering#optimal-server-counts)
for more on cluster architecture.

### Other requirements

#### License key or file

InfluxDB Enterprise requires a license key **OR** a license file to run.
Your license key is available at [InfluxPortal](https://portal.influxdata.com/licenses).
Contact support at the email we provided at signup to receive a license file.
License files are required only if the nodes in your cluster cannot reach
`portal.influxdata.com` on port `80` or `443`.

#### Ports

Meta nodes communicate over ports `8088`, `8089`, and `8091`.

For licensing purposes, meta nodes must also be able to reach `portal.influxdata.com`
on port `80` or `443`.
If the meta nodes cannot reach `portal.influxdata.com` on port `80` or `443`,
you'll need to set the `license-path` setting instead of the `license-key`
setting in the meta node configuration file.

#### User account

The installation package creates an `influxdb` user used to run the influxdb meta service. The `influxdb` user also owns certain files needed to start the service. In some cases, local policies may prevent the local user account from being created and the service fails to start. Contact your systems administrator for assistance with this requirement.

## Meta node setup
### Step 1: Add appropriate DNS entries for each of your servers

Ensure that your servers' hostnames and IP addresses are added to your network's DNS environment.
The addition of DNS entries and IP assignment is usually site and policy specific; contact your DNS administrator for assistance as necessary.
Ultimately, use entries similar to the following (hostnames and domain IP addresses are representative).

| Record Type |               Hostname                |                IP |
|:------------|:-------------------------------------:|------------------:|
| A           | ```enterprise-meta-01.mydomain.com``` | ```<Meta_1_IP>``` |
| A           | ```enterprise-meta-02.mydomain.com``` | ```<Meta_2_IP>``` |
| A           | ```enterprise-meta-03.mydomain.com``` | ```<Meta_3_IP>``` |


> **Verification steps:**
>
Before proceeding with the installation, verify on each server that the other
servers are resolvable. Here is an example set of shell commands using `ping`:
>
    ping -qc 1 enterprise-meta-01
    ping -qc 1 enterprise-meta-02
    ping -qc 1 enterprise-meta-03
>

We highly recommend that each server be able to resolve the IP from the hostname alone as shown here.
Resolve any connectivity issues before proceeding with the installation.
A healthy cluster requires that every meta node can communicate with every other
meta node.

### Step 2: Set up, configure, and start the meta services

Perform the following steps on each meta server.

#### I. Download and install the meta service

##### Ubuntu & Debian (64-bit)

```
wget https://dl.influxdata.com/enterprise/releases/influxdb-meta_1.8.4-c1.8.4_amd64.deb
sudo dpkg -i influxdb-meta_1.8.4-c1.8.4_amd64.deb
```

##### RedHat & CentOS (64-bit)

```
wget https://dl.influxdata.com/enterprise/releases/influxdb-meta-1.8.4_c1.8.4.x86_64.rpm
sudo yum localinstall influxdb-meta-1.8.4_c1.8.4.x86_64.rpm
```

##### Verify the authenticity of release download (recommended)

For added security, follow these steps to verify the signature of your InfluxDB download with `gpg`.

1. Download and import InfluxData's public key:

    ```
    curl -s https://repos.influxdata.com/influxdb.key | gpg --import
    ```

2. Download the signature file for the release by adding `.asc` to the download URL.
   For example:

    ```
    wget https://dl.influxdata.com/enterprise/releases/influxdb-meta-1.8.4_c1.8.4.x86_64.rpm.asc
    ```

3. Verify the signature with `gpg --verify`:

    ```
    gpg --verify influxdb-meta-1.8.4_c1.8.4.x86_64.rpm.asc influxdb-meta-1.8.4_c1.8.4.x86_64.rpm
    ```

    The output from this command should include the following:

    ```
    gpg: Good signature from "InfluxDB Packaging Service <support@influxdb.com>" [unknown]
    ```

#### II. Edit the configuration file

In `/etc/influxdb/influxdb-meta.conf`:

* Uncomment `hostname` and set to the full hostname of the meta node.
* Uncomment `internal-shared-secret` in the `[meta]` section and set it to a long pass phrase to be used in JWT authentication for intra-node communication. This value must the same for all of your meta nodes and match the `[meta] meta-internal-shared-secret` settings in the configuration files of your data nodes.
* Set `license-key` in the `[enterprise]` section to the license key you received on InfluxPortal **OR** `license-path` in the `[enterprise]` section to the local path to the JSON license file you received from InfluxData.

{{% warn %}}
The `license-key` and `license-path` settings are mutually exclusive and one must remain set to the empty string.
{{% /warn %}}

```
# Hostname advertised by this host for remote addresses.  This must be resolvable by all
# other nodes in the cluster
hostname="<enterprise-meta-0x>"

[enterprise]
  # license-key and license-path are mutually exclusive, use only one and leave the other blank
  license-key = "<your_license_key>" # Mutually exclusive with license-path

  # license-key and license-path are mutually exclusive, use only one and leave the other blank
  license-path = "/path/to/readable/JSON.license.file" # Mutually exclusive with license-key
```

#### III. Start the meta service

On sysvinit systems, enter:
```
service influxdb-meta start
```

On systemd systems, enter:
```
sudo systemctl start influxdb-meta
```

> **Verification steps:**
>
Check to see that the process is running by entering:
>
    ps aux | grep -v grep | grep influxdb-meta
>
You should see output similar to:
>
    influxdb  3207  0.8  4.4 483000 22168 ?        Ssl  17:05   0:08 /usr/bin/influxd-meta -config /etc/influxdb/influxdb-meta.conf

<br>


> **Note:** It is possible to start the cluster with a single meta node but you
must pass the `-single-server flag` when starting the single meta node.
Please note that a cluster with only one meta node is **not** recommended for
production environments.

### Step 3: Join the meta nodes to the cluster

From one and only one meta node, join all meta nodes including itself.
In our example, from `enterprise-meta-01`, run:
```
influxd-ctl add-meta enterprise-meta-01:8091

influxd-ctl add-meta enterprise-meta-02:8091

influxd-ctl add-meta enterprise-meta-03:8091
```

> **Note:** Please make sure that you specify the fully qualified host name of
the meta node during the join process.
Please do not specify `localhost` as this can cause cluster connection issues.

The expected output is:
```
Added meta node x at enterprise-meta-0x:8091
```

> **Verification steps:**
>
Issue the following command on any meta node:
>
    influxd-ctl show
>
The expected output is:
>
    Data Nodes
    ==========
    ID      TCP Address      Version
>
    Meta Nodes
    ==========
    TCP Address               Version
    enterprise-meta-01:8091   1.8.4-c1.8.4
    enterprise-meta-02:8091   1.8.4-c1.8.4
    enterprise-meta-03:8091   1.8.4-c1.8.4


Note that your cluster must have at least three meta nodes.
If you do not see your meta nodes in the output, please retry adding them to
the cluster.

Once your meta nodes are part of your cluster move on to [the next steps to
set up your data nodes](/enterprise_influxdb/v1.8/install-and-deploy/production_installation/data_node_installation/).
Please do not continue to the next steps if your meta nodes are not part of the
cluster.
