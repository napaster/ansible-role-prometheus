# ansible-prometheus

Deploy package-based [Prometheus](https://prometheus.io/) and optional
Alertmanager.

In this infrastructure the role is used in three modes:

* central receiver/storage: accepts remote write from site collectors;
* edge collector on a router: scrapes local/NAT-side targets and sends samples
  to the central receiver via `remote_write`;
* local collector on an internal server: scrapes localhost or nearby targets
  and sends samples to the router collector.

The role does not download Prometheus binaries. It installs OS packages through
the local package manager. This matches Archlinux and Debian deployments where
the `prometheus` package is available.

## Requirements

* Ansible 3.0.0+
* systemd
* `community.general` collection
* `promtool` and `amtool` from OS packages for configuration validation

## Security model

Inside a private NAT network, scrape endpoints may be plain HTTP if the network
is trusted enough.

Across WAN or any untrusted boundary, remote write must be protected:

* use HTTPS in front of the central receiver;
* require authentication on the central endpoint;
* store collector passwords in files deployed with Ansible Vault;
* reference secrets from Prometheus config with `password_file` or
  `credentials_file`, not inline plaintext.

Prometheus supports remote write authentication in the `remote_write` section
through `basic_auth`, `authorization`, OAuth2 and TLS settings. Prometheus 3.x
enables receiving remote write via the `--web.enable-remote-write-receiver`
command-line flag.

## Variables

The role is configured by the `prometheus` list variable. It is inactive when
`prometheus` is empty.

```yaml
prometheus: []
```

OS-specific defaults:

```yaml
prometheus_options_dest: '/etc/conf.d'
prometheus_service: 'prometheus'
prometheus_conf_dest: '/etc/prometheus'
prometheus_package: 'prometheus'
prometheus_bin: '/usr/bin/prometheus'
alertmanager_service: 'alertmanager'
alertmanager_conf_dest: '/etc/alertmanager'
alertmanager_package: 'alertmanager'
alertmanager_bin: '/usr/bin/alertmanager'
amtool_conf_dest: '/etc/amtool'
```

### Common keys

```yaml
prometheus:
  - enable:
      - prometheus: 'true'
        alertmanager: 'false'
    started:
      - prometheus: 'true'
        alertmanager: 'false'
    restart:
      - prometheus: 'true'
        alertmanager: 'false'
    reload:
      - prometheus: 'true'
        alertmanager: 'false'
    install_package:
      - prometheus: 'true'
        alertmanager: 'false'
    package_state: 'present'
    lint: 'all'
    secret_files: []
    prometheus_options: []
    prometheus_web_settings: {}
    prometheus_settings: {}
    prometheus_rules: {}
```

* `enable`: enable systemd services.
* `started`: start systemd services.
* `restart`: restart services when service options or web config change.
* `reload`: reload Prometheus when scrape/rule/secret files change and service
  options did not change.
* `install_package`: install `prometheus` and optionally `alertmanager`.
* `package_state`: `present` or `latest`.
* `lint`: `promtool` lint mode: `all`, `duplicate-rules` or `none`.
* `secret_files`: vault-backed files for passwords/tokens.
* `prometheus_options`: command-line flags rendered to the OS service options
  file.
* `prometheus_web_settings`: content of `/etc/prometheus/web.yml`.
* `prometheus_settings`: content of `/etc/prometheus/prometheus.yml`.
* `prometheus_rules`: content of `/etc/prometheus/prometheus_rules.yml`.

### Secret files

```yaml
secret_files:
  - path: '/etc/prometheus/remote_write.password'
    content: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    owner: 'root'
    group: 'root'
    mode: '0640'
```

The task uses `no_log: true`.

## Central receiver example

This host receives data from edge collectors. Put HTTPS and public access
control in a reverse proxy when possible. Prometheus web config can still hold
basic auth users for the receiver/UI.

Passwords in `basic_auth_users` must be bcrypt hashes.

```yaml
prometheus:
  - enable:
      - prometheus: 'true'
        alertmanager: 'true'
    started:
      - prometheus: 'true'
        alertmanager: 'true'
    restart:
      - prometheus: 'true'
        alertmanager: 'true'
    reload:
      - prometheus: 'true'
        alertmanager: 'true'
    install_package:
      - prometheus: 'true'
        alertmanager: 'true'
    package_state: 'present'
    lint: 'all'
    prometheus_options:
      - web:
          - listen_address: '127.0.0.1:9090'
            config_file: '/etc/prometheus/web.yml'
            enable_remote_write_receiver: 'true'
        storage:
          - tsdb_retention_time: '90d'
            tsdb_retention_size: '100GB'
        log_level: 'info'
        log_format: 'logfmt'
    prometheus_web_settings:
      basic_auth_users:
        gw_2110000_ru: '$2y$12$replace_with_bcrypt_hash'
    prometheus_settings:
      global:
        scrape_interval: '30s'
        evaluation_interval: '30s'
        external_labels:
          prometheus: 'central'
      scrape_configs:
        - job_name: 'central-prometheus'
          static_configs:
            - targets:
                - '127.0.0.1:9090'
    alertmanager_options:
      - web:
          - listen_address: '127.0.0.1:9093'
        cluster: ''
    alertmanager_settings:
      global:
        resolve_timeout: '5m'
      route:
        receiver: 'blackhole'
      receivers:
        - name: 'blackhole'
```

Expected remote write endpoint:

```text
https://metrics.example.com/api/v1/write
```

## Router edge collector example

The router scrapes itself and servers behind NAT, then writes to the central
receiver. Local scrapes are HTTP. Remote write to the central receiver is HTTPS
with basic auth and a password file.

```yaml
prometheus:
  - enable:
      - prometheus: 'true'
        alertmanager: 'false'
    started:
      - prometheus: 'true'
        alertmanager: 'false'
    restart:
      - prometheus: 'true'
        alertmanager: 'false'
    reload:
      - prometheus: 'true'
        alertmanager: 'false'
    install_package:
      - prometheus: 'true'
        alertmanager: 'false'
    package_state: 'present'
    lint: 'all'
    secret_files:
      - path: '/etc/prometheus/remote_write.password'
        content: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
        mode: '0640'
    prometheus_options:
      - web:
          - listen_address: '127.0.0.1:9090'
        storage:
          - tsdb_retention_time: '3d'
            tsdb_retention_size: '10GB'
            remote_flush_deadline: '5m'
        log_level: 'info'
        log_format: 'logfmt'
    prometheus_settings:
      global:
        scrape_interval: '30s'
        evaluation_interval: '30s'
        external_labels:
          site: 'karavan'
          collector: '{{ inventory_hostname }}'
      remote_write:
        - url: 'https://metrics.example.com/api/v1/write'
          basic_auth:
            username: '{{ inventory_hostname }}'
            password_file: '/etc/prometheus/remote_write.password'
          queue_config:
            capacity: 10000
            max_shards: 4
            min_shards: 1
      scrape_configs:
        - job_name: 'router-prometheus'
          static_configs:
            - targets:
                - '127.0.0.1:9090'
              labels:
                instance: '{{ inventory_hostname }}'
        - job_name: 'router-netdata'
          metrics_path: '/api/v1/allmetrics'
          params:
            format:
              - 'prometheus'
          static_configs:
            - targets:
                - '{{ ansible_host }}:19999'
              labels:
                instance: '{{ inventory_hostname }}'
        - job_name: 'nat-servers-node'
          static_configs:
            - targets:
                - '192.168.165.1:9100'
                - '192.168.165.2:9100'
              labels:
                site: 'karavan'
```

## Internal server collector example

Use this when a server inside NAT should run its own Prometheus and send data to
the router collector. The router collector must enable remote write receiver on
its LAN address or expose another authenticated endpoint if the network is not
trusted.

```yaml
prometheus:
  - enable:
      - prometheus: 'true'
        alertmanager: 'false'
    started:
      - prometheus: 'true'
        alertmanager: 'false'
    restart:
      - prometheus: 'true'
        alertmanager: 'false'
    reload:
      - prometheus: 'true'
        alertmanager: 'false'
    install_package:
      - prometheus: 'true'
        alertmanager: 'false'
    prometheus_options:
      - web:
          - listen_address: '127.0.0.1:9090'
        storage:
          - tsdb_retention_time: '1d'
            tsdb_retention_size: '3GB'
    prometheus_settings:
      global:
        scrape_interval: '30s'
        external_labels:
          site: 'karavan'
          collector: '{{ inventory_hostname }}'
      remote_write:
        - url: 'http://192.168.165.254:9090/api/v1/write'
      scrape_configs:
        - job_name: 'localhost'
          static_configs:
            - targets:
                - '127.0.0.1:9100'
```

## Playbook example

```yaml
---
- name: Deploy prometheus
  become: true
  gather_facts: true
  remote_user: root
  no_log: false
  strategy: free
  roles:
    - pacman
    - yum
    - dnf
    - prometheus
  hosts: empty
```

Run checks first:

```bash
ansible-playbook playbook/monitoring/deploy_prometheus.yml --syntax-check
ansible-playbook playbook/monitoring/deploy_prometheus.yml --check --diff --limit gw.2110000.ru
```

## Notes

Prometheus validates `prometheus.yml` with `promtool check config` and validates
rule files with `promtool check rules`. Alertmanager config is validated with
`amtool check-config`.

Remote write receiver is enabled by the `--web.enable-remote-write-receiver`
flag rendered from:

```yaml
prometheus_options:
  - web:
      - enable_remote_write_receiver: 'true'
```
