### Deploy HA cluster with Grafana and set up monitoring of systems that are part of the cluster.
#### Deploy HA Cluster with Grafana(x2), PostgeSQL and Nginx load balancing proxy
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
    domain = load_balancing.725c.kis.im
    root_url = http://load_balancing.725c.kis.im/  #DNS Record
    enable_gzip = true

- Install and setting `Nginx`
  - Install Nginx from official sources
  - Setup [config](https://github.com/vadim-davydchenko/grafana_final/blob/master/load_balancing.a951.kis.im.conf) file Nginx in `/etc/nginx/conf.d/load_balancing.a951.kis.im.conf`
