# Install prometheus and node exporter on Debian 10 & 11

## Step 1 - Create Prometheus system user / group

Create a dedicated Prometheus system user and group

`sudo groupadd --system prometheus`

`sudo useradd -s /sbin/nologin --system -g prometheus prometheus`

## Step 2 - Create configuration and data directories

> Prometheus needs directories to store data and configuration files. Create all required directories using the commands below

`sudo mkdir /var/lib/prometheus`

`for i in rules rules.d files_sd; do sudo mkdir -p /etc/prometheus/${i}; done`

## Step 3 - Download and Install Prometheus onDebian 11 / Debian 10

Download the latest release of Prometheus archive, extract it to get binary files. 

You can check releases from [Prometheus releases Github](https://github.com/prometheus/prometheus/releases) page

`sudo apt-get update`

`sudo apt-get -y install wget curl`

`mkdir -p /tmp/prometheus && cd /tmp/prometheus`

`curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest|grep browser_download_url|grep linux-amd64|cut -d '"' -f 4|wget -qi -`

Now, extract

`tar xvf prometheus*.tar.gz`

`cd prometheus*/`

### Move the prometheus binary files to /usr/local/bin/

`sudo mv prometheus promtool /usr/local/bin/`

### Move prometheus configuration template to /etc directory

`sudo mv prometheus.yml  /etc/prometheus/prometheus.yml`

Also move consoles and console_libraries to /etc/prometheus directory:

`sudo mv consoles/ console_libraries/ /etc/prometheus/`

`cd ~/`

`rm -rf /tmp/prometheus`


## Step 4 - Create/Edit a Prometheus configuration file

> Prometheus configuration file will be located under /etc/prometheus/prometheus.yml

`cat /etc/prometheus/prometheus.yml`

The default configuration file looks similar to below

```bash
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```


## Step 5 - Create a Prometheus systemd Service unit file

To be able to manage Prometheus service with systemd, you need to explicitly define this unit file

```bash
sudo tee /etc/systemd/system/prometheus.service<<EOF
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

### Change directory permissions

`for i in rules rules.d files_sd; do sudo chown -R prometheus:prometheus /etc/prometheus/${i}; done`

`for i in rules rules.d files_sd; do sudo chmod -R 775 /etc/prometheus/${i}; done`

`sudo chown -R prometheus:prometheus /var/lib/prometheus/`


### Reload systemd daemon and start the service.

`sudo systemctl daemon-reload`

`sudo systemctl start prometheus`

`sudo systemctl enable prometheus`

Confirm that the service is running

```
$ systemctl status prometheus
● prometheus.service - Prometheus
     Loaded: loaded (/etc/systemd/system/prometheus.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2021-08-31 19:14:07 UTC; 5s ago
       Docs: https://prometheus.io/docs/introduction/overview/
   Main PID: 1890 (prometheus)
      Tasks: 8 (limit: 2340)
     Memory: 19.1M
        CPU: 66ms
     CGroup: /system.slice/prometheus.service
             └─1890 /usr/local/bin/prometheus --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/var/lib/prometheus --web.console.templates=/etc/prometheus/consoles --web.console.lib>

Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.186Z caller=head.go:816 component=tsdb msg="Replaying on-disk memory mappable chunks if any"
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.187Z caller=head.go:830 component=tsdb msg="On-disk memory mappable chunks replay completed" duration=15.972µs
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.187Z caller=head.go:836 component=tsdb msg="Replaying WAL, this may take a while"
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.187Z caller=head.go:893 component=tsdb msg="WAL segment loaded" segment=0 maxSegment=0
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.188Z caller=head.go:899 component=tsdb msg="WAL replay completed" checkpoint_replay_duration=63.191µs wal_repl>
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.190Z caller=main.go:839 fs_type=EXT4_SUPER_MAGIC
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.190Z caller=main.go:842 msg="TSDB started"
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.191Z caller=main.go:969 msg="Loading configuration file" filename=/etc/prometheus/prometheus.yml
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.192Z caller=main.go:1006 msg="Completed loading of configuration file" filename=/etc/prometheus/prometheus.yml>
Aug 31 19:14:07 debian-bullseye-01 prometheus[1890]: level=info ts=2021-08-31T19:14:07.192Z caller=main.go:784 msg="Server is ready to receive web requests."
```

Access Prometheus web interface on URL http://[ip_hostname]:9090

![image](https://computingforgeeks.com/wp-content/uploads/2019/08/install-prometheus-debian-10-01-1024x419.png?ezimgfmt=ng:webp/ngcb23)

## Step 6 - Install node_exporter onDebian 11 / Debian 10

Download node_exporter archive

`curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest| grep browser_download_url|grep linux-amd64|cut -d '"' -f 4|wget -qi -`

Extract downloaded file and move the binary file to /usr/local/bin

`tar -xvf node_exporter*.tar.gz`

`cd  node_exporter*/`

`sudo cp node_exporter /usr/local/bin`

Confirm installation

```bash
$ node_exporter --version
node_exporter, version 1.2.2 (branch: HEAD, revision: 26645363b486e12be40af7ce4fc91e731a33104e)
  build user:       root@b9cb4aa2eb17
  build date:       20210806-13:44:18
  go version:       go1.16.7
  platform:         linux/amd64
```

Create node_exporter service

```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF
```

Reload systemd and start the service

`sudo systemctl daemon-reload`

`sudo systemctl start node_exporter`

`sudo systemctl enable node_exporter`

Confirm status

```bash
$ systemctl status node_exporter.service
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2021-08-31 19:16:11 UTC; 7s ago
   Main PID: 1969 (node_exporter)
      Tasks: 4 (limit: 2340)
     Memory: 3.2M
        CPU: 10ms
     CGroup: /system.slice/node_exporter.service
             └─1969 /usr/local/bin/node_exporter

Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=thermal_zone
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=time
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=timex
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=udp_queues
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=uname
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=vmstat
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=xfs
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:115 collector=zfs
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=node_exporter.go:199 msg="Listening on" address=:9100
Aug 31 19:16:11 debian-bullseye-01 node_exporter[1969]: level=info ts=2021-08-31T19:16:11.906Z caller=tls_config.go:191 msg="TLS is disabled." http2=false
```

Once we confirm the service to be running, let’s add the node_exporter to the Prometheus server

`sudo vim /etc/prometheus/prometheus.yml`

Add new job under `scrape_config` section

```
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Now restart Prometheus

`sudo systemctl restart prometheus`

You now have Prometheus installed onDebian 11 / Debian 10 Linux system. Check our Prometheus monitoring guides
