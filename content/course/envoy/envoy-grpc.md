# envoy-grpc

## AggregatedDiscoveryService

- 代码路径：api/envoy/service/discovery/v3/ads.proto
- 请求：
  - DiscoveryRequest
  - DeltaDiscoveryRequest
- 响应：
  - DiscoveryResponse：包含全量的xds数据
  - DeltaDiscoveryResponse：包含增量的xds数据

```protobuf
service AggregatedDiscoveryService {
  // This is a gRPC-only API.
  // 全量 xds 数据下发
  rpc StreamAggregatedResources(stream DiscoveryRequest) returns (stream DiscoveryResponse) {
  }

  // 增量 xds 数据下发
  rpc DeltaAggregatedResources(stream DeltaDiscoveryRequest)
      returns (stream DeltaDiscoveryResponse) {
  }
}
```



