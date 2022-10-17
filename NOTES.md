Server:

2. venv and packages and basic: https://grpc.io/docs/languages/python/quickstart/
2. https://grpc.io/docs/languages/python/quickstart/
grpc-server.yogeshsharma.me:80


Certificate:

Please visit https://certbot.eff.org/ to check for certbot installation.
https://certbot.eff.org/instructions?ws=other&os=ubuntufocal

certbot certonly --standalone


Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/grpc-server.yogeshsharma.me/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/grpc-server.yogeshsharma.me/privkey.pem


## Envoy Conf

```yaml
static_resources:

  listeners:
  - name: grpc_listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: service_grpc_backend
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain: {filename: "/etc/letsencrypt/live/grpc-server.yogeshsharma.me/fullchain.pem"}
              private_key: {filename: "/etc/letsencrypt/live/grpc-server.yogeshsharma.me/privkey.pem"}

  clusters:
  - name: service_grpc_backend
    type: LOGICAL_DNS
    http2_protocol_options: {}
    connect_timeout: 1s
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: service_grpc_backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 50051
```


## Start Services

### Start GRPC Server:

cd /app/grpc-service/grpc/examples/python/route_guide/
python3 route_guide_server.py

### Start Envoy:

cd /app/envoy
envoy -c envoy-ssl.yaml 
