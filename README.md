### Deploy HA cluster with Grafana and set up monitoring of systems that are part of the cluster.
#### 1.Deploy HA Cluster with Grafana(x2), PostgeSQL and Nginx load balancing proxy
- Install and setting *PostgreSQL* from official sources:
  - Create user, database and password for Grafana and give privileges
  ```sudo -u postgres createuser grafana
  sudo -u postgres createdb grafana
  sudo -u postgres psql
  alter user grafana with encrypted password 'admin123';
  grant all privileges on database grafana to grafana;

- Allow listening from all addresses in `pg_hba.conf`

  `host    all             all             0.0.0.0/0               md5`
- Installing `Grafana` and setting up a DB connection on both servers:
  - Install `Grafana` from official sources
  - Editing `grafana.ini`
  ```
    [database]
    url = postgres://grafana:admin123@<ip adress DB>/grafana
    [server]
    domain = load_balancing.724c.kis.im
    root_url = http://load_balancing.724c.kis.im/  #DNS Record Nginx
    enable_gzip = true
  ```
- Install and setting `Nginx`
  - Install Nginx from official sources
  - Setup [config](https://github.com/vadim-davydchenko/grafana_final/blob/master/load_balancing.a952.kis.im.conf) file Nginx in `/etc/nginx/conf.d/load_balancing.a951.kis.im.conf`

#### 2.Setup GitlabAuth
- Getting tokens in Gitlab:
  `Gitlab -> Profile -> Edit profile -> Applications`
  
  `Redirect URI - http://<You DNS Record>/login/gitlab`
  
  Ticks `api, read_user, read_api`
- Setup config `grafana.ini` on authorization via Gitlab
```
  [auth.gitlab]
  enabled = true
  allow_sign_up = true
  client_id =   #Code from Application ID
  client_secret = #Code from Secret
  scopes = api
  auth_url = https://gitlab.com/oauth/authorize
  token_url = https://gitlab.com/oauth/token
  api_url = https://gitlab.com/api/v4
  allowed_domains =
  allowed_groups = 
```

#### 3.Connect Promeheus to Grafana with authentication and TLS.
- Create certificates for Prometheus:
  ```sudo mkdir -p /etc/prometheus-certs && cd /etc/prometheus-certs
     sudo openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout prometheus.key -out prometheus.crt -subj "/C=RU/ST=Moscow/L=Moscow/O=Rebrain/CN=<DNS Record Prometheus>" -addext "subjectAltName = DNS:<DNS Record Prometheus>"
- Create [config Nginx](https://github.com/vadim-davydchenko/grafana_final/blob/master/prometheus.conf) for Prometheus
- Run Prometheus with arguments

  `sudo systemctl stop prometheus`
  
  `cd /usr/local/bin`
  ```
  sudo prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.external-url=https://prometheus_proxy.a951.kis.im \
  --web.route-prefix="/"
  ```
- Setup basic_auth

  `sudo apt update`
  
  `sudo apt install  -y apache2-utils`
  
  `sudo htpasswd -c .htpasswd admin`
  
- Add parameters `auth_basic` and `auth_basic_user_file` in `prometheus.conf`
- Check Prometheus metrics
  
  `sudo curl -u admin -k https://<DNS-Record Prometheus>/metrics`
- Add data source in `Grafana`
  
  ![alt text](https://github.com/vadim-davydchenko/grafana_final/blob/master/Data%20Source%20Grafana.jpg)
  
#### 4.Setup on the node nginx Prometheus-nginx-exporter and setting Prometheus to collect metrics Nginx

- Create in `conf.d` [exporter.conf](https://github.com/vadim-davydchenko/grafana_final/blob/master/exporter.conf) and delete `default.conf`:
- Run docker container with `nginx-prometheus-exporter`

  `docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest -nginx.scrape-uri http://<Ip nginx>:10080/nginx_stats`
#### 5.Setting collect metrics Grafana herselfs and nginx-exporter in Prometheus

- In `grafana.ini` include directives:
  ```
  [metrics]
  enabled             = true
  disable_total_stats = false
  ```
- In `Prometheus` add next targets
  
  ```
  - job_name: "grafana1"
    static_configs:
      - targets: ['<IP Grafana1>:3000']
  - job_name: "grafana2"
    static_configs:
      - targets: ['<IP Grafana2>:3000']
  - job_name: "nginx-exporter"
    static_configs:
      - targets: ['<IP Nginx>:9113']
  ```
#### 6.Create a cluster monitoring dashboard containing the following panels

- Prometheus
  - Total number of records in TSDB so far   `rate(prometheus_tsdb_head_samples_appended_total[1m])`
  - Rate requests to Prometheus (QPS)        `prometheus_http_request_duration_seconds_bucket`
  - Main parameters CPU, ROM, RAM, I/O, Network (use `node_exporter`)
    
    ```
    rate(node_network_receive_bytes_total[1m])
    rate(node_cpu_seconds_total{mode="system"}[1m])
    node_disk_write_time_seconds_total
    node_disk_read_time_seconds_total
    node_filesystem_files_free
    node_disk_io_now
    ```
- PostgreSQL
  - Install `PostgreSQl exporter`

```
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.9.0/postgres_exporter-0.9.0.linux-amd64.tar.gz
tar xvfz postgres_exporter-*.linux-amd64.tar.gz
cd postgres_exporter-*.linux-amd64
export DATA_SOURCE_NAME='postgres://grafana:admin123@<IP Postgres>:5432/grafana?sslmode=disable'
./postgres_exporter
```
  - Version PostgreSQL        `pg_static`
  - Current number of connections (sessions)     `sum(pg_stat_database_numbackends)`
  - Load rate (number of requests per unit of time)     `sum(irate(pg_stat_database_xact_commit[5m])) + sum(irate(pg_stat_database_xact_rollback[5m]))` 
 
- Grafana
  - Number of organizations, users, dashboards  
 
    `sum(grafana_stat_total_orgs)`
    `sum(grafana_stat_total_users)`
    `sum(grafana_stat_total_dashboards)`
    
  - Status distribution of response codes
  
    `grafana_api_response_status_total`
- Nginx
  - Current status service    `nginx_up`
  - Rate of received connections broken down by status     `nginx_connections_accepted` and `nginx_connections_handled`
  - Rate of the total number of client requests   `nginx_http_requests_total`
