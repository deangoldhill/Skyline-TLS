# Grafana with Prometheus for Check Point Skyline

A monitoring solution for Checkpoint physical and virtual appliances.
this docker compose script will install and configure an instance of Prometheus and Grafana as seperate containers, including required configuration parameters and dashboard templates.
The script will pull the official images, and use the Grafana provisioning feature to automatically configure the Grafana datasource and dashboard template.
[Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [Skyline](https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&solutionid=sk178566), [Caddy](https://caddyserver.com)

**NOTE - This configuration (Skyline-TLS) configures HTTPS comunication between the browser and the Grafana portal, and between the Checkpoint appliances and the Prometheus server. For testing purposes, or if TLS is not required, rather please refer to the Skyline repository which requires fewer steps [Skyline](https://github.com/deangoldhill/Skyline)

This configuration uses a Caddyserver container, which acts as a reverse proxy and automatically manages the TLS configuration and certificate management including certificate renewal.

## DNS Record
Caddy requires that a hostname is used when connecting to the Prometheus and Grafana servers rarther than the IP address, otherwsie the TLS client hello will be rejected.
```bash
Add a DNS A record pointing to your docker host, which your browser and the Checkpoint gateways will be able to resolve.
```


## Install Docker

Install Docker-CE and docker-compose:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## Generate password hash
Generate a password hash which will be used to authenticate the Checkpoint gateways connecting to the Prometheus server.
This command will launch a caddy container, generate the password hash and then delete the container. 

**Replace "secretpassword" with your desired password.
```bash
docker run --rm caddy:2.6.4 caddy hash-password --plaintext 'secretpassword'
```

## Install Containers

Clone this repository on your Docker host:

```bash
git clone https://github.com/deangoldhill/skyline-TLS.git
```
Natigate to the Caddy directory, and modify the file to change the placeholder hostname "DOCKERHOST.DOMAIN.COM" with the DNS hostname created in the first step.

**Modify "mydocker.mydomain.com" with your DNS hostname created in the first step.
```bash
cd skyline-TLS
sed -i 's/DOCKERHOST.DOMAIN.COM/mydocker.mydomain.com/' caddy/Caddyfile
```
Deploy the containers

**Replace "passwordhash" with the hash generated in the previous step.
```bash
ADMIN_USER='admin' ADMIN_PASSWORD_HASH='passwordhash' docker-compose up -d
```

Get the CA certificate data from the Caddy server - This is required by the Checkpoint machines to trust the server certificate.
*NOTE - The certificate data printed to the console contains line breaks which must be removed when adding it to the config file on the Checkpoint machines. Use Notepad or something to get rid of the line breaks so the entire string is just 1 line.

```bash
docker exec caddy cat /data/caddy/pki/authorities/local/root.crt
```


## Setup Grafana

Navigate to `http://<DOCKERHOST.DOMAIN.COM>:3000` and login with user ***admin*** password ***admin***. You can change the password at first login.
**Note - you must navigate to the Grafana portal using the hostname, not the IP address. Otherwsie you will get an SSL error.


Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: SKYLINE_PROMETHEUS
* Type: Prometheus
* Url: http://prometheus:9090

There are 4 Skyline dashboards

***Machines Overview***

Machines with top CPU / Memory utilisations
Vital stastics of selected (filter) machine.


***Single Machine***
In-depth information and statistics for a single machine,
Hardware resource consumption, version info, policy info, network throughput, connections,Drops, errors etc.

***Scalable Platforms***
As above but for Maestro platforms

***VSX***
VSX server and indavidual virtual system stastics



## Setup Skyline on Check Point machines

Create a configuration payload file on each Check Point machine with the following contents, and import it.

***paste the following and modify the following values*** 
1. Modify "DOCKERHOST.DOMAIN.COM" with the DNS hostname of your docker host
2. Modify "SECRETPASSWORD" with the desired password
3. Modify "CERTIFICATE" with the CA certificate data from the previous step

```bash
cd /var/tmp
vi payload.json


 {
    "enabled": true,
    "export-targets": {"add": [
        {
            "client-auth": {
                "basic": {
                    "username": "admin",
                    "password": "SECRETPASSWORD"
                }
            },
            "enabled": true,
            "server-auth": {
                "ca-public-key": {
                    "type": "PEM-X509",
                    "value": "CERTIFICATE"
                }
            },
            "type": "prometheus-remote-write",
            "url": "https://DOCKERHOST.DOMAIN.COM:9090/api/v1/write"
        }
    ]}
} 



/opt/CPotelcol/REST.py --set_open_telemetry "$(cat payload.json)"

```

At this point, you should be able to log into the Grafana portal and see graphs being populated.

## Troubleshooting
If all the above steps completed succesfully but Grafana graphs are not populating, it's probably an issue with the Checkpoint gateways connecting to the Prometheus server.
Review the OpenTelemtry agent log on the Checkpoint machine:
```bash
tail -f /opt/CPotelcol/otelcol.log
```
Most common errors are:

Permanent error: Post \"https://DOCKERHOST:9090/api/v1/write\": dial tcp DOCKERHOST:9090: i/o **timeout**", "dropped_items"
<br>*Reason - This indicates the TCP communication is failing - Checkpoint machine is not getting a response
<br>*Action - Check basic traffic flow, routing, firewall rules, DNS resolution etc.

Permanent error: Post \"https://DOICKERHOST:9090/api/v1/write\": http: **server gave HTTP response to HTTPS client**", "dropped_items"
<br>*Reason - The Caddy server failed to configure TLS, and is responding with plain HTTP.
<br>*Action - This is probably to do with the DNS hostname used in the Checkpoint configuration not matching what is specified in the Caddyfile. It must martch exactly otherwsie Caddy will reject the TLS client hello.

Permanent error: Post \"https://DOCKERHOST.DOMAIN.COM:9090/api/v1/write\": x509: **certificate signed by unknown authority**
<br>*Reason - The checkpoint machine is failing to validate the Caddy server certificate. 
<br>*Action - get the CA certificate data from the Caddy container, update the Checkpoint configuration payload and run /opt/CPotelcol/REST.py --set_open_telemetry "$(cat payload.json)" again. be sure to remove the line break in the certificate data - The certificate data must be one single line including the '-----BEGIN CERTIFICATE-----' and '-----END CERTIFICATE-----' parts.

Once everything is working correctly, the log files should end with "Everything is ready. Begin running and processing data"

If you have tried these steps but it's stil not working, please open a Github issue ticket.
