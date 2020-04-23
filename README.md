# PrometheusLab
 Part of the CICD module 

### :one: Installing Prometheus:

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus

cd tmp && wget https://github.com/prometheus/prometheus/releases/download/v2.7.1/prometheus-2.7.1.linux-amd64.tar.gz 
tar -xvf prometheus-2.7.1.linux-amd64.tar.gz && cd prometheus-2.7.1.linux-amd64 && ls # cd into the extracted prometheus directory and list contents

sudo mv -t /etc/prometheus console* prometheus.yml # move any file that begins with console and move prometheus.yml to /etc/prometheus

sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool /var/lib/prometheus

```

##### Systemd unit file for Prometheus:

`sudo vi /etc/systemd/system/prometheus.service`

```
[Unit]
Description=Prometheus
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

[Install]
WantedBy=multi-user.target

```

then reload systemd

` sudo systemctl daemon-reload && sudo systemctl enable prometheus && sudo systemctl start prometheus `

### :two: Installing Node exporter:
node exporter is installed on the OS that will be monitored using Prometheus and exposed through the `prometheus.yml` file located at `/etc/prometheus`

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter

cd /tmp && wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz && tar -xvf node_exporter-0.17.0.linux-amd64.tar.gz && cd node_exporter-0.17.0.linux-amd64

sudo mv node_exporter /usr/local/bin

sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

```

##### Systemd unit file for Node exporter:

`sudo vi /etc/systemd/system/node_exporter.service`

```
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target

```

reload systemd

`sudo systemctl daemon-reload && sudo systemctl enable node_exporter && sudo systemctl start node_exporter`

### :three: Monitoring [cAdvisor](https://github.com/google/cadvisor) with Prometheus:
[cAdvisor](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md) is used to analyze resource usage and performance characteristics of running containers.

running the cAdvisor docker container:
```bash

sudo docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  gcr.io/google-containers/cadvisor:v0.35.0

```
by default it uses port `8080`

### :four: Monitoring a simple nodeJS app:

<details><summary>NodeJS Code</summary>
<p>

```javascript

var express = require('express');
var bodyParser = require('body-parser');
var app = express();
const prom = require('prom-client');

app.use(bodyParser.urlencoded({ extended: true }));
app.set('view engine', 'ejs');

const collectDefaultMetrics = prom.collectDefaultMetrics;
collectDefaultMetrics({ prefix: 'nodeapp' });
app.use(express.static("public")); // use css

// placeholder tasks
var task = [];
var complete = [];

// add a task
app.post("/addtask", function(req, res) {
var newTask = req.body.newtask;
task.push(newTask);
res.redirect("/");
});
// remove a task
app.post("/removetask", function(req, res) {
var completeTask = req.body.check;
if (typeof completeTask === "string") {
complete.push(completeTask);
task.splice(task.indexOf(completeTask), 1);
}
else if (typeof completeTask === "object") {
for (var i = 0; i < completeTask.length; i++) {
complete.push(completeTask[i]);
task.splice(task.indexOf(completeTask[i]), 1);
}
}
res.redirect("/");
});


// get website files
app.get("/", function (req, res) {
res.render("index", { task: task, complete: complete });
});
// listen for connections
app.listen(7231, function() {
console.log('Testing app listening on port 7231')
});
app.get('/metrics', function (req, res) {
res.set('Content-Type', prom.register.contentType);
res.end(prom.register.metrics());
});

```

</p>
</details>

###

### :zap: prometheus.yml file:

```yaml

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
  - job_name: 'Prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'Nodeexporter-localhost'
    static_configs:
    - targets: ['localhost:9100']

  - job_name: 'Docker-cAdvisor'
    static_configs:
    - targets: ['localhost:8080']

  - job_name: 'NodeJSapp'
    static_configs:
    - targets: ['localhost:7231']

```

### Preview of localhost:9090/targets:
