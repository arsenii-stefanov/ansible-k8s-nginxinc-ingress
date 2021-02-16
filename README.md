Ansible K8s Nginx.inc Ingress
=========

References:

* Official GitHub repo: https://github.com/nginxinc/kubernetes-ingress

* Official DockerHub repo: https://hub.docker.com/r/nginx/nginx-ingress/


### K8s NGINX Ingress Controller Example

`FILE: {{ playbook_dir }}/my-k8s-namespace-staging.yml`

```
env_name: staging

kube_api_url: "https://gke-master-proxy-staging.my-domain.local"
kube_validate_certs: true
kube_auth_api_key: "{{ kube_secrets_api_key }}"
kube_ca_path: "{{ playbook_dir }}/config-files/ca-certs/gke-staging.crt"


###############################################
###      KUBERNETES INGRESS CONTROLLER      ###
###############################################

### For some reason Role does not work for NGINX any more, it returns multiple RBAC errors until Role is replaced with ClusterRole, despite the -watch-namespace CLI option
nginx_ingress_ctl_deploy_globally: true
nginx_ingress_ctl_kube_namespace: "my-k8s-namespacee-{{ env_name }}"
nginx_ingress_ctl_name: "nginx-ingr-{{ nginx_ingress_ctl_kube_namespace }}"
nginx_ingress_ctl_version: "1.8.0"
nginx_ingress_ctl_enable_snippets: true

nginx_ingress_ctl_ip_addr_config:
  provider: "gcp"
  name: "{{ nginx_ingress_ctl_name }}"
  project: "my-awesome-project"
  region: "us-central1"

nginx_ingress_ctl_global_real_ip_header: "X-Forwarded-For"
nginx_ingress_ctl_global_real_ip_recursive: "True"
nginx_ingress_ctl_global_set_real_ip_from: "35.186.192.42/32,130.211.0.0/22,35.191.0.0/16" # GCP load balancers pool
```

nginx_ingress_http_snippets:
- "log_format graylog_json escape=json '{ \"timestamp\": \"$time_iso8601\",' '\"remote_addr\": \"$remote_addr\",' '\"body_bytes_sent\": $body_bytes_sent,' '\"request_time\": $request_time,' '\"response_status\": $status,' '\"request\": \"$request\",' '\"request_method\": \"$request_method\",' '\"host\": \"$host\",' '\"upstream_cache_status\": \"$upstream_cache_status\",' '\"upstream_addr\": \"$upstream_addr\",' '\"http_x_forwarded_for\": \"$http_x_forwarded_for\",' '\"http_referrer\": \"$http_referer\",' '\"http_user_agent\": \"$http_user_agent\" }';"

# Horizontal Pod Autoscaler
nginx_ingress_ctl_hpa_config:
  min_replicas: 2
  max_replicas: 10
  target_cpu_utilization_percentage: 40

### K8s NGINX Ingress Virtual Host Example

See YAML files in the 'config-templates' drectory for examples

`FILE: {{ playbook_dir }}/my-k8s-namespace-staging.yml`

```
##########################################
###      KUBERNETES INGRESS VHOST      ###
##########################################

app_name: "my-awesome-app"
port_app: 8080
tls_name: "{{ app_name }}-tls"

nginx_ingress_ctl_ingress_vhost_configs:
- path: "{{ playbook_dir }}/config-templates/k8s/nginxinc_ingress/nginxinc_Secret_TLS.yml"
  fail_on_error: true
  state: "present"
  no_log: true
- path: "{{ playbook_dir }}/config-templates/k8s/nginxinc_ingress/nginxinc_Service_NodePort.yml"
  fail_on_error: true
  state: "present"
  no_log: false
- path: "{{ playbook_dir }}/config-templates/k8s/nginxinc_ingress/nginxinc_Ingress.yml"
  fail_on_error: true
  state: "present"
  no_log: false

# NodePort
nginx_ingress_nodeport:
  name: "node-port-{{ app_name }}"
  ports:
    - name: "port-app"
      port: "{{ port_app }}"
      target_port: "{{ port_app }}"
      protocol: "TCP"
  selectors:
    - name: "app"
      value: '{{ app_name }}'

# Ingress
nginx_ingress_https_enabled: true    ### Just enables the 'tls' section in the Ingress config, no HTTP-to-HTTPS redirect

nginx_ingress_vhost_server_snippets:
- "access_log syslog:server=graylog-udp-1.clean.local:12311 graylog_json;"
- "error_log syslog:server=graylog-udp-1.clean.local:12312 warn;"

# Ingress
nginx_ingress_vhost_config:
  name: "ingress-{{ app_name }}"
  http_to_https_redirect: true
  nginx_opts:
    proxy_buffers: "64 256k"
    proxy_buffer_size: 512k
    proxy_read_timeout: 300s
    proxy_send_timeout: 300s
    client_max_body_size: 1m
#  redirect_rules:
#    - source_http_host: "my-app-staging.example.com"
#      dest_url: "https://my-app-staging.example.com$request_uri"
#      type: 301
  vhosts:
    - v_host_name: "my-app-staging.example.com"
      kube_tls_secret_name: "{{ tls_name }}"
      paths:
        - path: "/"
          ### "Exact", "Prefix", or "ImplementationSpecific"
          path_type: "Exact"
          service_name: "{{ nginx_ingress_nodeport['name'] }}"
          service_port: "{{ port_app }}"
  ip_allow:
    - ip: "69.69.42.0/32"
      comment: "VPN"
    - ip: "69.69.69.0/24"
      comment: "Office"
    - ip: "130.211.0.0/22"
      comment: "GCP load balancers pool"
    - ip: "35.191.0.0/16"
      comment: "GCP load balancers pool"
    - ip: "34.86.101.28"
      comment: "Load Balancer"
```
