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
    root_url = http://load_balancing.724c.kis.im/  #DNS Record
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
