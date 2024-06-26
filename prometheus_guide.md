## Monitor your node

This guide will walk you through how to set up Prometheus with Grafana to monitor your node with a basic dashbaord.

A Substrate-based chain like Aya exposes data such as the height of the chain, the number of connected peers to your node and more. To monitor this data, Prometheus is used to collect metrics and Grafana allows for displaying them on the dashboard.

It is good practice to install Prometheus and Grafana on separate servers to your node(s) - that way if your node goes down the system can still send you alerts.  These should ideally be connected by a VPN such as Tailscale, rather than just opening the relevant ports for Prometheus and Grafana on each server.

For testing purposes it is possible to run Prometheus and Grafana on the same sever as your node.  For brevity we will assume that this is the case and so the guide will use `localhost` as the address for the various servers.  For a secure setup you should replace localhost with your relevant server ip address.

`THE STEPS IN THIS GUIDE ARE FOR TESTING PURPOSES - DO NOT USE IN A PRODUCTION SETTING`


### Preparation

First, create a user for Prometheus by adding the --no-create-home flag to disallow prometheus from logging in.

```
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
```

Create the directories required to store the configuration and executable files.

```
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
```

Change the ownership of these directories to prometheus so that only prometheus can access them.

```
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

### Installing and Configuring Prometheus

After setting up the environment, update your OS, and install the latest Prometheus. You can check the latest release by going to their GitHub repository under the releases page.

```
sudo apt-get update && apt-get upgrade
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar xfz prometheus-*.tar.gz
cd prometheus-2.52.0.linux-amd64
```

There are two binaries in the directory: `prometheus` the Prometheus main binary file, and `promtool`

`consoles` and `console_libraries` directories contain the web interface, configuration files examples and the license.

Copy the executable files to the `/usr/local/bin/` directory.

```
sudo cp ./prometheus /usr/local/bin/
sudo cp ./promtool /usr/local/bin/
```

Change the ownership of these files to the prometheus user.

```
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Copy the consoles and console_libraries directories to `/etc/prometheus`

```
sudo cp -r ./consoles /etc/prometheus
sudo cp -r ./console_libraries /etc/prometheus
```

Change the ownership of these directories to the prometheus user.

```
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Once everything is done, run this command to remove prometheus directory.

```
cd .. && rm -rf prometheus*
```

Before using Prometheus, it needs some configuration. Create a YAML configuration file named `prometheus.yml` by running the command below.

```
sudo nano /etc/prometheus/prometheus.yml
```

Copy and Paste the following configuration into the editor and save with `CTRL + X`

`NOTE: If your aya node is on a different server to prometheus replace localhost:9615 with <your-nodes-ipaddress>:9615` <br>
`You would also need to ensure that --prometheus-external was added to your start_aya_validator.sh script`

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "aya_node"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9615"]
```

The configuration file is divided into three parts which are `global`, `rule_files`, and `scrape_configs`.

`global` / `scrape_interval` defines how often Prometheus scrapes targets <br>
`global` / `evaluation_interval` controls how often the software will evaluate rules <br>
`rule_files` block contains information of the location of any rules we want the Prometheus server to load <br>
`scrape_configs` contains the information on which resources Prometheus monitors.

With the above configuration file, the first exporter is the one that Prometheus exports to monitor itself. As we want to have more precise information about the state of the Prometheus server we reduced the `scrape_interval` to 5 seconds for this job. The parameters `static_configs` and `targets` determine where the exporters are running.  The second exporter is capturing the data from your aya node, and the port by default is `9615`

You can check the validity of this configuration file by running:
```
promtool check config /etc/prometheus/prometheus.yml
```

Save the configuration file and change the ownership of the file to prometheus user.

```
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```


## Starting Prometheus

To test that Prometheus is set up properly, execute the following command to start it as the prometheus user.

```
sudo -u prometheus /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/ --web.console.templates=/etc/prometheus/consoles --web.console.libraries=/etc/prometheus/console_libraries
```

The following messages indicate the status of the server. If you see the following messages, your server is set up properly.

```
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:380 msg="No time or size retention was set so using the default time retention" duration=15d
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:418 msg="Starting Prometheus" version="(version=2.26.0, branch=HEAD, revision=3cafc58827d1ebd1a67749f88be4218f0bab3d8d)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:423 build_context="(go=go1.16.2, user=root@a67cafebe6d0, date=20210331-11:56:23)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:424 host_details="(Linux 5.4.0-42-generic #46-Ubuntu SMP Fri Jul 10 00:24:02 UTC 2020 x86_64 ubuntu2004 (none))"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:425 fd_limits="(soft=1024, hard=1048576)"
level=info ts=2021-04-16T19:02:20.167Z caller=main.go:426 vm_limits="(soft=unlimited, hard=unlimited)"
level=info ts=2021-04-16T19:02:20.169Z caller=web.go:540 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2021-04-16T19:02:20.170Z caller=main.go:795 msg="Starting TSDB ..."
level=info ts=2021-04-16T19:02:20.171Z caller=tls_config.go:191 component=web msg="TLS is disabled." http2=false
level=info ts=2021-04-16T19:02:20.174Z caller=head.go:696 component=tsdb msg="Replaying on-disk memory mappable chunks if any"
level=info ts=2021-04-16T19:02:20.175Z caller=head.go:710 component=tsdb msg="On-disk memory mappable chunks replay completed" duration=1.391446ms
level=info ts=2021-04-16T19:02:20.175Z caller=head.go:716 component=tsdb msg="Replaying WAL, this may take a while"
level=info ts=2021-04-16T19:02:20.178Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=0 maxSegment=4
level=info ts=2021-04-16T19:02:20.193Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=1 maxSegment=4
level=info ts=2021-04-16T19:02:20.221Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=2 maxSegment=4
level=info ts=2021-04-16T19:02:20.224Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=3 maxSegment=4
level=info ts=2021-04-16T19:02:20.229Z caller=head.go:768 component=tsdb msg="WAL segment loaded" segment=4 maxSegment=4
level=info ts=2021-04-16T19:02:20.229Z caller=head.go:773 component=tsdb msg="WAL replay completed" checkpoint_replay_duration=43.716µs wal_replay_duration=53.973285ms total_replay_duration=55.445308ms
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:815 fs_type=EXT4_SUPER_MAGIC
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:818 msg="TSDB started"
level=info ts=2021-04-16T19:02:20.233Z caller=main.go:944 msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml
level=info ts=2021-04-16T19:02:20.234Z caller=main.go:975 msg="Completed loading of configuration file" filename=/etc/prometheus/prometheus.yml totalDuration=824.115µs remote_storage=3.131µs web_handler=401ns query_engine=1.056µs scrape=236.454µs scrape_sd=45.432µs notify=723ns notify_sd=2.61µs rules=956ns
level=info ts=2021-04-16T19:02:20.234Z caller=main.go:767 msg="Server is ready to receive web requests."
```

On a web browser go to `http://YOUR_SERVER_IP_ADDRESS:9090/graph` to check whether you are able to access the Prometheus interface or not. If it is working in your browser, exit the process by pressing on `CTRL + C`.

`NOTE - You will need to have access to port 9090 on your server to test this connection`

Next, we would like to automatically start the server during the boot process, so we have to create a new systemd configuration file:

```
sudo nano /etc/systemd/system/prometheus.service
```

Copy and Paste the following configuration into the editor and save with `CTRL + X`
```
[Unit]
  Description=Prometheus Monitoring
  Wants=network-online.target
  After=network-online.target

[Service]
  User=prometheus
  Group=prometheus
  Type=simple
  ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
  ExecReload=/bin/kill -HUP $MAINPID

[Install]
  WantedBy=multi-user.target
```

Once the file is saved, execute the command below to reload systemd and enable the service so that it will be loaded automatically during the operating system's startup.

```
sudo systemctl daemon-reload && systemctl enable prometheus && systemctl start prometheus
```

Prometheus should be running now, and you should be able to access its front again end by re-visiting `http://YOUR_SERVER_IP_ADDRESS:9090/graph`

## Prometheus Node Exporter Setup

`This is optional, but recommended as it enables us to gather system metrics like CPU and RAM usage from each server running Aya-Node.  If you do not want to be able to view your node system metrics on your Grafana dashboard than move on to the Grafana section`

The Node Exporter is an agent that gathers system metrics and exposes them in a format which can be ingested by Prometheus. The Node Exporter is a project that is maintained through the Prometheus project. This is a completely optional step and can be skipped if you do not wish to gather system metrics. The following will need to be performed on each server that you wish to monitor system metrics for.

### Download Node Exporter

Download the Node Exporter binary to each server that you want to monitor.

```
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
```
Visit the Prometheus downloads page for the latest version. (https://prometheus.io/download/)

### Create User

Create a Node Exporter user, required directories, and make Node Exporter user the owner of those directories.

```
sudo groupadd -f node_exporter
sudo useradd -g node_exporter --no-create-home --shell /bin/false node_exporter
sudo mkdir /etc/node_exporter
sudo chown node_exporter:node_exporter /etc/node_exporter
```
### Unpack Node Exporter Binary

Untar and move the downloaded Node Exporter binary
```
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
mv node_exporter-1.8.1.linux-amd64 node_exporter-files
```
### Install Node Exporter

Copy node_exporter binary from node_exporter-files folder to /usr/bin and change the ownership to prometheus user.

```
sudo cp node_exporter-files/node_exporter /usr/bin/
sudo chown node_exporter:node_exporter /usr/bin/node_exporter
```

### Setup Node Exporter Service

Create a node_exporter service file.

```
sudo nano /usr/lib/systemd/system/node_exporter.service
```

Copy and Paste the following configuration, then save with `CTRL + X`

```
[Unit]
Description=Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/bin/node_exporter \
  --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
```

Then run the following command
```
sudo chmod 664 /usr/lib/systemd/system/node_exporter.service
``` 


### Reload systemd and Start Node Exporter

Reload the systemd service to register the prometheus service and start the prometheus service.

```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
```

Check the node exporter service status using the following command.

```
sudo systemctl status node_exporter
```

Example output: ( press `CTRL + C` to exit )
```
• node_exporter. service - Node Exporter
  Loaded: loaded (/usr/lib.systemmd/system/node_expoerter_service; enabled; vender preset: disabled)
  Active: active (running) Tue 2024-05-18 17:04:17 UTC; since 17h ago
    Docs: https://prometheus.io/docs/guides/node-exporter/
Main PID: 27646 (node_exporter)
  CGroup: /system.slice/node_exporter.service
            -27646 /usr/local/bin/node_exporter --web.listen-address=:9100
```

Configure node_exporter to start at boot
```
sudo systemctl enable node_exporter.service
```
If firewalld is enabled and running, add a rule for port 9100
```
sudo ufw allow proto tcp from any to any port 9100
sudo ufw reload
```
### Verify Node Exporter is Running

Verify the exporter is running by visiting the `/metrics` endpoint on the node on port `9100`
```
http://YOUR_SERVER_IP_ADDRESS:9100/metrics
```
You should be able to see something similar to the following:

```
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 7
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.12.5"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 919280
...
```
### Clean Up

Remove the download and temporary files

```
rm -rf node_exporter-1.8.0.linux-amd64.tar.gz node_exporter-files
```

---
Now we must edit the Prometheus YAML configuration file so that Prometheus scrapes the Node Exporter on port 9100.  We will add another target onto our existing `aya-node` job

```
sudo nano /etc/prometheus/prometheus.yml
```

Add `,"localhost:9100"` to the `aya_nodes` target and save the configuration with `CTRL + X`

`NOTE: If your aya node is on a different server to prometheus replace localhost:9100 with <your-nodes-ipaddress>:9100`

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "aya_node"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9615","localhost:9100"]
```

Now restart the Prometheus service.

```
sudo systemctl restart prometheus
```

---

## Installing Grafana

In order to visualize your node metrics, you can use Grafana to query the Prometheus server. Run the following commands to install it first.

```
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/oss/release/grafana_11.0.0_amd64.deb
sudo dpkg -i grafana_11.0.0_amd64.deb
```

If everything is fine, configure Grafana to auto-start on boot and then start the service.

```
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

You can now access Grafana by going to the `http://YOUR_SERVER_IP_ADDRESS:3000/login` in your browser. The default user and password is admin/admin.  You should change this password!!

NOTE:
If you want to change the port on which Grafana runs (3000 is a popular port), edit the file `/usr/share/grafana/conf/defaults.ini` with a command like `sudo nano /usr/share/grafana/conf/defaults.ini` and change the http_port value to something else. Then restart grafana with `sudo systemctl restart grafana-server`.


### Setting up Grafana

First we need to select a data source.  Go to Connections > Add new connection.  Search for Prometheus and select it.

![AddDataSource](monitoring_assets/grafana_add_data.png)

On the next screen click the `Add New Data Source` button.

Now you will need to enter `http://<your_actual_node_IP_address:9090>` localhost will not work.

![EnterURL](monitoring_assets/prometheus_add_url.png)

Scroll to the bottom of the page and click `Save and Test`.  Hopefully you will get a message that the test was sucessful!

## Installing your first Dashboard.

I have provided a basic dashbaord to monitor your aya node to get you started, it is based on a substrate node template.

Look in the `monitoring_assets` folder in this repo and download the `aya_node_template.json` to your desktop

In Grafana click on `Dashboards` in the left hand side menu bar, and then click `New` and select `import` from the dropdown menu 

![EnterURL](monitoring_assets/dashboard_new_import.png)

Now upload the `aya_node_template.json` file by dragging it from your desktop into the grafana window.

![UploadJSON](monitoring_assets/upload_json.png)

Click `Load`

Here is what it should look like!

![EnterURL](monitoring_assets/aya_grafana_template.png)

`NOTE - If the panels show "no data" you need to update to your new Prometheus data source.`

`in this case edit a panel by clicking the 3 dots at the top right and choose Edit`
![EditPanel](monitoring_assets/edit_panel.png)

`then select your own Prometheus data source from the list in the Query section`<br>
`you may find that you need to click into the query, type a space and then delete the space to refresh the query`

![SelectDataSource](monitoring_assets/select_data_source.png)


`if your data source is still not loading try this process posted by catman on discord :)`

```
1. Edit each panel
2. See that the Run queries button is greyed out
3. Go to the Metrics browser section and just add a random character to the end of the value and remove it again
eg. substrate_block_height{status="best"} -> substrate_block_height{status="best"}x -> substrate_block_height{status="best"}
4. This should enable the Run queries button, turning it blue. Click it and you should see it work properly.
5. Apply on each panel and Save the dashboard at the end
```

Enjoy your new dashboard!  It is possible to set up alerts from Grafafa by various methods

Check out [How to integrate Grafana Alerting and Telegram](https://grafana.com/blog/2023/12/28/how-to-integrate-grafana-alerting-and-telegram)
