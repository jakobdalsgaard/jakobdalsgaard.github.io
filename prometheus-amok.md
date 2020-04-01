Prometheus Amok

Got my new setup with a Linux box as router (an old Dell PowerEdge, but with two
gigabit ports, and it handles gigabit traffic tremendously well, couldn't be
happier).  Besides the Linux box, I now run a managed switch and an Ubiquiti
access point. All this equipment can be monitored in very many ways; but I'd
like the metrics to be gathered in one uniform application, so I installed
[Prometheus](https://prometheus.io/) - a time series database to store all sorts
of metrics in; I then put [Grafana](https://grafana.com/) in front of it --
whilst Prometheus has a very nice UI itself, Grafana does come with a lot of
nice features, such as dashboards and alerting.

## The World of Prometheus Exporters

### Unifi Prometheous Exporter

To be fetched from here: [Unifi Scraper](http://github.com/mdlayher/unifi) &emdash; now 
author recommends a docker; I'm more a fan of running my stuff on a real machine. So
I opted for a plain normal install on the box.

    export SRCDIR=$PWD
    mkdir -p src/github.com/mdlayher
    cd src/github.com/mdlayher
    git clone https://github.com/mdlayher/unifi_exporter.git
    cd unifi_exporter
    go get
    GOPATH=$SRCDIR go build ./cmd/unifi_exporter

And voil&aacute; I have the static binary `unifi_exporter`; copy that one to `/usr/local/sbin`; then make a
service unit file:

    [Unit]
    Description=Prometheus Unifi Exporter

    [Service]
    ExecStart=/usr/local/sbin/unifi_exporter -config.file /etc/prometheus/prometheus-unifi-exporter.yml

    [Install]
    WantedBy=multi-user.target

Save that in `prometheus-unifi-exporter.service`, copy to `/etc/systemd/system/` and do a `systemctl daemon-reload`.

The yaml config file can look like this:

    listen:
      address: :9130
      metricspath: /metrics
    unifi:
      address: https://my.domain.name:9445
      username: PrometheusUnifiExporter
      password: DamnPrometheus
      site:
      insecure: true
      timeout: 5s

For this go program to accept the Unifi application, I need to install at least a
self signed certificate in Unifi, and Ubiqiti has not made that proces easy. I found
it to work this way:

First create a self signed certificate with your designed hostname as CN:

    # Make self signed certs
    openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365

Then convert it to pkcs12 to have something the Unifi java application wants to play with:

    # pacakge in pkcs12
    openssl pkcs12 -export -in cert.pem -inkey key.pem -out unifi.p12 -name unifi  -caname root

Now update the keystore with this very fine pkcs12 certificate:

    # use keytool to update keystore
    keytool -importkeystore -deststorepass aircontrolenterprise -destkeypass aircontrolenterprise -destkeystore new-keystore -srckeystore unifi.p12 -srcstoretype PKCS12 -srcstorepass temppass -alias ubnt -noprompt

I believe `keytool` can come from just about any java you have lying around. Now shut down your Unifi Controller application, take a backup copy of
`/usr/lib/unifi/data/keystore` file, copy `new-keystore` to `/usr/lib/unifi/data/keystore` and start your Unifi Controller again and your Unifi
Controller should be running with a brand new self signed certificate.

Now the prometheus unifi exporter service can be installed by doing `systemctl start prometheus-unifi-exporter`.

### Openhab Exporter

I run openhab2, so obviously I needed an exporter for that too -- found this one: https://github.com/baaym/openhab2-prometheus-exporter -- and
it looked fine, installed it in `/opt/openhab2-exporter` did the relevant gunicorn install:

    sudo apt install python3-gunicorn

Make sure you change line 7 in the python script to reflect your openhab setup, the line looks like this `url = urllib.request.urlopen('http://...`
-- the address there should match your openhab2 configuration.
Then did a systemctl module file by the name of `/etc/systemd/system/openhab2-exporter.service` -- with contents:

    [Unit]
    Description=OpenHAB2 Prometheus exporter
    After=openhab2.service
    
    [Service]
    WorkingDirectory=/opt/openhab2-exporter
    ExecStart=/usr/bin/gunicorn3 -w 4 -b 127.0.0.1:9195 openhab2-exporter:app
    Restart=on-failure
    
    [Install]
    WantedBy=multi-user.target
    Alias=openhab2-exporter.service

Then `sudo systemctl daemon-reload` and `sudo systemctl start openhab2-exporter` -- and adding this configuration to prometheus.yml:

      - job_name: 'openhab2_exporter'
        scrape_interval: 10s
        scheme: http
        static_configs:
          - targets:
            - 'localhost:9195'

A restart of prometheus and it's data is available.
Tags: computer, software
