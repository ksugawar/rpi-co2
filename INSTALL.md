# rpi-co2
Build a CO2 monitor using my Raspberry Pi (without writing a line of code)

## How to install

1. Install Fedora 40 Server (or Raspberry Pi OS Lite or whatever) to your Raspberry Pi

1. Create User, and enable linger:
```
$ loginctl enable-linger <your-login>
```

1. Install additional packages

Below example is required in case of RasPi OS Lite:
```
$ sudo apt install podman git python3-serial
```

1. Git clone this repo
```
$ git clone <URI>
```

1. Create a pod
```
$ podman pod create --network=host --name co2mon
```

1. Run customized prometheus container (change port to 19090 because 9090 is in use by cockpit)
```
$ cd prom
$ cat Containerfile
ARG OS="linux"
FROM prom/prometheus:latest

WORKDIR /prometheus

USER       nobody
EXPOSE     19090
VOLUME     [ "/prometheus" ]
ENTRYPOINT [ "/bin/prometheus" ]
CMD        [ "--config.file=/etc/prometheus/prometheus.yml", \
             "--storage.tsdb.path=/prometheus", \
             "--web.listen-address=:19090" ]
$ podman build -t myprometheus -f Containerfile
$ podman run -d --pod co2mon --name prometheus localhost/myprometheus
```

1. Build mhz19-exporter image
```
$ git clone https://github.com/mhansen/mhz19-exporter.git
$ cd mhz19-exporter
$ cat Containerfile
FROM golang:1.16 as builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY *.go ./
RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm go build -mod=readonly -v -a exporter.go

FROM scratch
COPY --from=builder /app/exporter /
EXPOSE 9672
ENTRYPOINT ["/exporter"]
CMD        [ "--portname=/dev/ttyS0", \
             "--port=:9672" ]

$ podman build -t mhz19-exporter -f Containerfile
```

1. Change the serial device's SELinux label and SELinux boolean, to allow containers to use the host serial devices (/dev/ttyS0 in Fedora)
```
$ sudo chcon -t container_file_t /dev/ttyS0
$ sudo setsebool -P container_use_devices 1
```

1. Run mhz19-exporter container
```
$ podman run -d --pod co2mon --device /dev/ttyS0 --group-add keep-groups --name mhz19 mhz19-exporter
```

1. Change prometheus.conf so that it scrapes mhz19 exporter
```
$ podman exec prometheus sh -c 'cat - >/etc/prometheus/prometheus.yml' <<EOF
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
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
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9672"]
EOF
```

1. Restart prometheus container
```
$ podman stop prometheus
$ podman start prometheus
```

1. Run grafana container
```
$ podman run -d --pod co2mon --name grafana grafana/grafana-oss
```

1. Install nginx
```
$ sudo dnf install -y nginx
```

1. Add reverse-proxy conf for prometheus and grafana
```
$ cat /etc/nginx/conf.d/co2mon.conf
server {
        listen 192.168.0.130:19091;
        server_name _;

        location / {
                proxy_pass http://127.0.0.1:19090/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}

server {
        listen 192.168.0.130:3001;
        server_name _;

        location / {
                proxy_pass http://127.0.0.1:3000/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

1. Change a couple of SELinux settings to let nginx listen to non-standard ports and connect to prometheus/grafana
```
$ sudo semanage port -a -t http_port_t -p tcp 19091
$ sudo semanage port -a -t http_port_t -p tcp 3001
$ sudo setsebool -P httpd_can_network_connect 1
```

1. Start nginx
```
$ sudo systemctl start nginx
```

1. Open a web browser, connect to http://192.168.0.130:3001, and configure a dashboard.

## TO-DOs
* ~Generate a systemd unit file to make this a service~
* Generate a quadlet file for the pod/containers
* Put together an ansible playbook to automate all of the above (except the OS installation, of course)
