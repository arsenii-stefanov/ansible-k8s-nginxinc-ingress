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

nginx_ingress_ctl_deploy_globally: false
nginx_ingress_ctl_kube_namespace: "my-k8s-namespacee-{{ env_name }}"
nginx_ingress_ctl_name: "nginx-ingr-{{ env_name }}"
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

### K8s NGINX Ingress Virtual Host Example

See YAML files in the 'config-templates' drectory for examples

`FILE: {{ playbook_dir }}/my-k8s-namespace-staging.yml`

```
##########################################
###      KUBERNETES INGRESS VHOST      ###
##########################################

app_name: "my-awesome-app"
port_app: 8080

nginx_ingress_ctl_ingress_vhost_configs:
- "{{ playbook_dir }}/config-templates/nginxinc_ingress/nginxinc_Service_NodePort.yml"
- "{{ playbook_dir }}/config-templates/nginxinc_ingress/nginxinc_Ingress.yml"

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
      value: '{{ app_name }}-{{ env_name }}'

# Ingress
nginx_ingress_vhost_config:
  name: "ingress-{{ app_name }}-{{ env_name }}"
  http_to_https_redirect: true
  nginx_opts:
    proxy_buffers: "64 256k"
    proxy_buffer_size: 512k
    proxy_read_timeout: 300s
    proxy_send_timeout: 300s
    client_max_body_size: 1m
  redirect_rules:
    - source_http_host: "my-app-staging.example.com"
      dest_url: "https://my-app-staging.example.com$request_uri"
      type: 301
  vhosts:
    - v_host_name: "my-app-staging.example.com"
      kube_tls_secret_name: "{{ app_name }}-{{ env_name }}-tls"
      paths:
        - path: "/"
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
