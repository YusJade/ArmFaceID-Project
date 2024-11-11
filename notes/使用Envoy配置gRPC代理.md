# 如何使用 Envoy 为 gRPC 配置代理

## 什么是 Envoy

> Originally built at **Lyft**, Envoy is a high performance C++ distributed proxy designed for single services and applications, as well as a communication bus and “universal data plane” designed for large microservice “service mesh” architectures.

最初由 Lyft 构建，Envoy 是一款高性能的 C++ 分布式代理，专为单个服务和应用设计，同时也是为大型微服务“服务网格”架构设计的通信总线和“通用数据平面”。

## 安装 Envoy

```sh
wget -O- https://apt.envoyproxy.io/signing.key | sudo gpg --dearmor -o /etc/apt/keyrings/envoy-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/envoy-keyring.gpg] https://apt.envoyproxy.io jammy main" | sudo tee /etc/apt/sources.list.d/envoy.list
sudo apt-get update
sudo apt-get install envoy
envoy --version
```

## 配置 Envoy 配置文件

启动 Envoy 时可以向其指定一个配置文件，Envoy 将根据文件扩展名解析配置文件，这里使用 `yaml` 格式的配置文件：

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
	# address 为本机的 IP 地址
    socket_address: { address: 192.168.3.18, port_value: 9901 }

static_resources:
  listeners:
    - name: listener_0
      address:
	    # address 为本机的 IP 地址，port_value 为外部访问 gRPC 的端口号
        socket_address: { address: 192.168.3.18, port_value: 9092 }
      filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              codec_type: auto
              stat_prefix: ingress_http
              route_config:
                name: local_route
                virtual_hosts:
                  - name: local_service
                    domains: ["*"]
                    routes:
                      - match: { prefix: "/" }
                        route:
                          cluster: echo_service
                          timeout: 0s
                          max_stream_duration:
                            grpc_timeout_header_max: 0s
                    cors:
                      allow_origin_string_match:
                        - prefix: "*"
                      allow_methods: GET, PUT, DELETE, POST, OPTIONS
                      allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,custom-header-1,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                      max_age: "1728000"
                      expose_headers: custom-header-1,grpc-status,grpc-message
              http_filters:
                - name: envoy.filters.http.grpc_web
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                - name: envoy.filters.http.cors
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
                - name: envoy.filters.http.router
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
    - name: echo_service
      connect_timeout: 0.25s
      type: logical_dns
      # HTTP/2 support
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      lb_policy: round_robin
      load_assignment:
        cluster_name: cluster_0
        endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
	                # 配置 gRPC 服务的 IP 地址与端口号
                    address: 192.168.3.18
                    port_value: 25575

```

## 在命令行使用配置文件运行 Envoy

```sh
envoy -c ./envoy.yaml  --log-level debug
```

检查 Envoy 是否在 [http://192.168.3.18:9092](http://192.168.3.18:9092/) 上进行代理：

```sh
curl -v 192.168.3.18:9092
```

```log
*   Trying 192.168.3.18:9092...
* Connected to 192.168.3.18 (192.168.3.18) port 9092 (#0)
> GET / HTTP/1.1
> Host: 192.168.3.18:9092
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/grpc
< grpc-status: 2
< grpc-message: Bad method header
< x-envoy-upstream-service-time: 186
< date: Mon, 11 Nov 2024 03:02:00 GMT
< server: envoy
< content-length: 0
<
* Connection #0 to host 192.168.3.18 left intact
```

在运行的 Envoy 的终端可以看到对应的日志：

```log
[2024-11-11 10:59:56.749][26968][debug][http] [source/common/http/conn_manager_impl.cc:1878] [Tags: "ConnectionId":"2","StreamId":"689563732152126502"] encoding headers via codec (end_stream=true):
':status', '200'
'content-type', 'application/grpc'
'grpc-status', '2'
'grpc-message', 'Bad method header'
'x-envoy-upstream-service-time', '102'
'date', 'Mon, 11 Nov 2024 02:59:56 GMT'
'server', 'envoy'
```

然后就能在前端通过代理服务器访问 gRPC 服务：

```js
const FaceService = new FaceRpcClient('http://192.168.3.18:9092', null, null)
```