# ansible-prometheus

Deploy [Prometheus](//prometheus.io/) - monitoring system and time series
database and Alertmanager

## Requirements

* Ansible 3.0.0+;

## Extra

Configuration pass as-is from yaml to pretty yaml with validation via
`promtool` for Prometheus and `amtool` for Alertmanager


## Example configuration

```yaml
---
prometheus:
- enable:
# Enable alertmanager service or not
    - alertmanager: 'true'
# Enable prometheus service or not
      prometheus: 'true'
  restart:
# Restart alertmanager service after service override deploy or not
    - alertmanager: 'true'
# Restart prometheus service after service override deploy or not
      prometheus: 'true'
  reload:
# Reload alertmanager service after configuration deploy or not
    - alertmanager: 'true'
# Reload prometheus service after configuration deploy or not
      prometheus: 'true'
  install_package:
# Install prometheus package or not
    - prometheus: 'true'
# Install alertmanager package or not
      alertmanager: 'true'
# Linting checks to apply. Available options are: 'all', 'duplicate-rules',
# 'none'. Use 'none' to disable linting
  lint: 'all'
# Prometheus startup options
  prometheus_options:
    - web:
# Address to listen on for UI, API, and telemetry
        - listen_address: '0.0.0.0:9090'
# Path to configuration file that can enable TLS or authentication
          config_file: ''
# Maximum duration before timing out read of the request, and closing idle
# connections
          read_timeout: '5m'
# Maximum number of simultaneous connections
          max_connections: '512'
# The URL under which Prometheus is externally reachable (for example, if
# Prometheus is served via a reverse proxy). Used for generating relative and
# absolute links back to Prometheus itself. If the URL has a path portion, it
# will be used to prefix all HTTP endpoints served by Prometheus. If omitted,
# relevant URL components will be derived automatically
          external_url: ''
# Prefix for the internal routes of web endpoints. Defaults to path of
# 'web_external_url'
          route_prefix: ''
# Path to static asset directory, available at '/user'
          user_assets: ''
# Enable shutdown and reload via HTTP request
          enable_lifecycle: 'false'
# Enable API endpoints for admin control actions
          enable_admin_api: 'true'
# Enable API endpoint accepting remote write requests
          enable_remote_write_receiver: 'true'
# List of the remote write protobuf messages to accept when receiving the remote
# writes. Supported values:
# * prometheus.WriteRequest
# * io.prometheus.write.v2.Request
          remote_write_receiver_accepted_protobuf_messages:
            - 'prometheus.WriteRequest',
            - 'io.prometheus.write.v2.Request'
# Enable API endpoint accepting OTLP write requests
          enable_otlp_receiver: 'true'
# Path to the console template directory, available at '/consoles'
          console_templates: 'consoles'
# Path to the console library directory
          console_libraries: 'console_libraries'
# Document title of Prometheus instance
          page_title: 'Prometheus Time Series Collection and Processing Server'
# Regex for CORS origin. It is fully anchored
# Example: 'https?://(domain1|domain2)\.com'
          cors_origin: '.*'
      storage:
# How long to retain samples in storage. Defaults to 15d. Units supported:
# 'y', 'w', 'd', 'h', 'm', 's', 'ms'
        - tsdb_retention_time: '15d'
# Maximum number of bytes that can be stored for blocks. Units supported:
# 'KB', 'MB', 'GB', 'TB', 'PB'
          tsdb_retention_size: '100GB'
# Do not create lockfile in data directory
          tsdb_no_lockfile: 'false'
# Allow overlapping blocks, which in turn enables vertical compaction and
# vertical query merge
          tsdb_allow_overlapping_blocks: 'false'
# Compress the tsdb WAL
          tsdb_wal_compression: 'false'
# How long to wait flushing sample on shutdown or config reload
          remote_flush_deadline: ''
# Maximum overall number of samples to return via the remote read interface, in
# a single query. '0' means no limit. This limit is ignored for streamed
# response types
          remote_read_sample_limit: '5e7'
# Maximum number of concurrent remote read calls. 0 means no limit
          remote_read_concurrent_limit: '10'
# Maximum number of bytes in a single frame for streaming remote read response
# types before marshalling. Note that client might have limit on frame size as
# well. 1MB as recommended by protobuf by default
          remote_read_max_bytes_in_frame: '1048576'
      rules:
# Max time to tolerate prometheus outage for restoring "for" state of alert
        - alert_for_outage_tolerance: '1h'
# Minimum duration between alert and restored "for" state. This is maintained
# only for alerts with configured "for" time greater than grace period
          alert_for_grace_period: '10m'
# Minimum amount of time to wait before resending an alert to Alertmanager
          alert_resend_delay: '1m'
# Global concurrency limit for independent rules that can run concurrently. When
# set, 'query: max_concurrency' may need to be adjusted accordingly
          max_concurrent_evals: '4'
      alermanager:
# The capacity of the queue for pending Alertmanager notifications
        - notification_queue_capacity: '10000'
# Send any outstanding Alertmanager notifications when shutting down. If
# 'false', any outstanding Alertmanager notifications will be dropped when
# shutting down
          drain_notification_queue_on_shutdown: 'true'
      query:
# The delta difference allowed for retrieving metrics during expression
# evaluations
        - lookback_delta: '5m'
# Maximum time a query may take before being aborted
          timeout: '2m'
# Maximum number of queries executed concurrently
          max_concurrency: '20'
# Maximum number of samples a single query can load into memory. Note that
# queries will fail if they try to load more samples than this into memory, so
# this also limits the number of samples a query can return
          max_samples: '50000000'
# Only log messages with the given severity or above.
# One of: 'debug', 'info', 'warn', 'error'
      log_level: 'info'
# Output format of log messages. One of: 'logfmt' or 'json'
      log_format: 'logfmt'
# Alertmanager startup options
  alertmananger_options:
    - data:
# How long to keep data for
        - retention: '120h'
# Interval between garbage collection and snapshotting to disk of the silences
# and the notification logs
          maintenance_interval: '15m'
      silences:
# Maximum number of silences, including expired silences. If negative or zero,
# no limit is set
        - max: '0'
# Maximum silence size in bytes. If negative or zero, no limit is set
          max_size_bytes: '0'
# Interval between alert GC
      alerts_gc_interval: '30m'
      web:
# The URL under which Alertmanager is externally reachable (for example, if
# Alertmanager is served via a reverse proxy). Used for generating relative and
# absolute links back to Alertmanager itself. If the URL has a path portion, it
# will be used to prefix all HTTP endpoints served by Alertmanager. If omitted,
# relevant URL components will be derived automatically
        - external_url: ''
# Prefix for the internal routes of web endpoints. Defaults to path of
# --web.external-url
          route_prefix: ''
# Address to listen on for the web interface and API
          listen_address: ':9093'
# Maximum number of GET requests processed concurrently. If negative or zero,
# the limit is GOMAXPROC or 8, whichever is larger
          get_concurrency: '0'
# Timeout for HTTP requests. If negative or zero, no timeout is set
          timeout: '15'
      cluster:
# Path to config yaml file that can enable mutual TLS within the gossip
# protocol. See 'alertmanager_gossip' settings below. Should be set to
# '/etc/alertmanager/gossip.yml'
        - tls_config: '/etc/alertmanager/gossip.yml'
# Listen address for cluster. Set to empty string to disable HA mode
          listen_address: '0.0.0.0:9094'
# Explicit address to advertise in cluster
          advertise_address: ''
# Cluster initial peers (may be string or list of peers)
          peer: ''
# Time to wait between peers to send notifications
          peer_timeout: '15s'
# Interval between sending gossip messages. By lowering this value (more
# frequent) gossip messages are propagated across the cluster more quickly at
# the expense of increased bandwidth
          gossip_interval: ''
# Interval for gossip state syncs. Setting this interval lower (more frequent)
# will increase convergence speeds across larger clusters at the expense of
# increased bandwidth usage
          pushpull_interval: '1m0s'
# Timeout for establishing a stream connection with a remote node for a full
# state sync, and for stream read and write operations
          tcp_timeout: '10s'
# Timeout to wait for an ack from a probed node before assuming it is unhealthy.
# This should be set to 99-percentile of RTT (round-trip time) on your network
          probe_timeout: '500ms'
# Interval between random node probes. Setting this lower (more frequent) will
# cause the cluster to detect failed nodes more quickly at the expense of
# increased bandwidth usage
          probe_interval: '1s'
# Maximum time to wait for cluster connections to settle before evaluating
# notifications
          settle_timeout: '1m0s'
# Interval between attempting to reconnect to lost peers
          reconnect_interval: '10s'
# Length of time to attempt to reconnect to a lost peer
          reconnect_timeout: '6h0m0s'
# The cluster label is an optional string to include on each packet and stream.
# It uniquely identifies the cluster and prevents cross-communication issues
# when sending gossip messages
          label: 'default'
# Only log messages with the given severity or above. One of: 'debug', 'info',
# 'warn', 'error'
      log_level: ''
# Output format of log messages. One of: 'logfmt', 'json'
      log_format: ''
# Alertmanager Cluster TLS Gossip settings
# https://prometheus.io/docs/alerting/latest/https/#gossip-traffic
  alertmanager_gossip:
    tls_server_config:
      cert_file: '/etc/pki/tls/private/le/fullchain.pem'
      key_file: '/etc/pki/tls/private/le/privkey.pem'
      client_auth_type: 'RequireAndVerifyClientCert'
      min_version: 'TLS13'
    tls_client_config:
      cert_file: '/etc/pki/tls/private/le/fullchain.pem'
      key_file: '/etc/pki/tls/private/le/privkey.pem'
      server_name: 'prometheus.example.com'
# Alertmanager settings
  alertmanager_settings:
    global:
      resolve_timeout: 5m
  # This path should be defined at least, if you use templates
    templates:
      - '/etc/alertmanager/alertmanager.tmpl'
    route:
      receiver: 'telegram'
      group_by:
        - 'cluster'
        - 'group'
        - 'instance'
        - 'job'
        - 'severity'
      repeat_interval: '8737h'
    receivers:
      - name: 'telegram'
        telegram_configs:
          - chat_id: -1000000000000
            bot_token: "token"
            message: '{{ template "telegram.default.message" . }}'
            parse_mode: 'Markdown'
  alertmanager_templates: |
    {%- raw -%}
    {{ define "__text_alert_list" }}{{ range . }}Labels:
    {{ range .Labels.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Annotations:
    {{ range .Annotations.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Source: {{ .GeneratorURL }}
    {{ end }}{{ end }}

    {{ define "telegram.default.message" }}
    {{ if gt (len .Alerts.Firing) 0 }}
    Alerts Firing:
    {{ template "__text_alert_list" .Alerts.Firing }}
    {{ end }}
    {{ if gt (len .Alerts.Resolved) 0 }}
    Alerts Resolved:
    {{ template "__text_alert_list" .Alerts.Resolved }}
    {{ end }}
    {{ end }}
    {%- endraw -%}
  amtool_settings:
    alertmanager.url: 'htts://alermanager.example.com'
    tls.insecure.skip.verify: 'true'
  prometheus_settings:
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - '127.0.0.1:9093'
    scrape_configs:
    - job_name: 'example'
      static_configs:
      - targets:
        - '192.168.1.1:9283'
  prometheus_rules:
    groups:
      - name: 'alert_rules'
        rules:
          - alert: 'NetdataSmartdPendingSectors'
            expr: 'netdata:smartd:current_pending_sector'
            for: '1h'
            labels:
              severity: 'warning'
            annotations:
              summary: '{% raw %}Host {{ $labels.instance }} have drive with pending sectors{% endraw %}'
              description: '{% raw %}Drive "{{ $labels.dimension }}" on host "{{ $labels.instance }}" have {{ $value }} pending sectors{% endraw %}'
      - name: 'record_rules'
        rules:
          - record: 'netdata:smartd:current_pending_sector'
            expr: 'netdata_smartd_log_current_pending_sector_count_sectors_average > 1'
```
