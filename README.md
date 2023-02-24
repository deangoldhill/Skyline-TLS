# Grafana with Prometheus for Check Point Skyline

A monitoring solution for Checkpoint physical and virtual appliances.
this docker compose script will install and configure an instance of Prometheus and Grafana as seperate containers, including required configuration parameters and dashboard templates.
The script will pull the official images, and use the Grafana provisioning feature to automatically configure the Grafana datasource and dashboard template.
[Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [Skyline](https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&solutionid=sk178566), [Caddy](https://caddyserver.com)

This Branch (TLS) configures HTTPS comunication between the browser and the Grafana portal, and between the Checkpoint appliances and the Prometheus server.
If TLS is not required, rather follow the No-TLS branch which requires fewer steps.

This TLS branch uses a Caddyserver container, which acts as a reverse proxy and automatically manages the TLS configuration and certificate management including certificate renewal.

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
git clone https://github.com/deangoldhill/skyline.git
```
Natigate to the Caddy directory, and modify the file to change the placeholder hostname "DOCKERHOST.DOMAIN.COM" with the DNS hostname created in the first step.

**Modify "mydocker.mydomain.com" with your DNS hostname created in the first step.
```bash
cd caddy
sed -i 's/DOCKERHOST.DOMAIN.COM/mydocker.mydomain.com/' Caddyfile
cd ..
```
Deploy the containers

**Replace "secretpassword" with your desired password, and "passwordhash" with the hash generated in the previous step.
```bash
ADMIN_USER='admin' ADMIN_PASSWORD='secretpassword' ADMIN_PASSWORD_HASH='passwordhash' docker-compose up -d
```

Get the CA certificate data from the Caddy server - This is required by the Checkpoint machines to trust the server certificate.
*NOTE - The certificate data printed to the console contains line breaks which must be removed when adding it to the config file on the Checkpoint machines. Use Notepad or something to get rid of the line breaks so the entire string is just 1 line.

```bash
docker exec caddy cat /data/caddy/pki/authorities/local/root.crt
```


## Setup Grafana

Navigate to `http://<DOCKERHOST>:3000` and login with user ***admin*** password ***admin***. You can change the password at first login, or you can modify the credentials in the compose file.


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
1. Modify "DOCKERHOST" with the DNS hostname of your docker host
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
            "url": "https://DOCKERHOST:9090/api/v1/write"
        }
    ]}
} 

```

/opt/CPotelcol/REST.py --set_open_telemetry "$(cat payload.json)"

```