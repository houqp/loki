# Installation instructions for Istio

The ingestor, querier, etc. might start, but if those changes are not made, you will see logs like

```
loki level=debug ts=2021-11-24T11:33:37.352544925Z caller=broadcast.go:48 msg="Invalidating forwarded broadcast" key=collectors/distributor version=123 oldVersion=122 content=[loki-distributor-59c4896444-t9t6g[] oldContent=[loki-distributor-59c4896444-t9t6g[]
```

This means that the pod is failing to join the ring.

If you try to add `loki` to `Grafana` data sources, you will see logs like (`empty ring`)

```
loki level=warn ts=2021-11-24T08:02:42.08262122Z caller=logging.go:72 traceID=3fc821042d8ada1a orgID=fake msg="GET /loki/api/v1/labels?end=1637740962079859431&start=1637740361925000000 (500) 97.4µs Response: \"empty ring\\n\" ws: false; X-Scope-Orgid: fake; uber-trace-id: 3fc821042d8ada1a:1feed8872deea75c:1180f95a8235bb6c:0; "
```

When istio-injection is enabled on the namespace running Loki, there are few changes needed. One of the main changes is around the `Service` `appProtocols`. 
Given that istio will not allow a pod to resolve another pod using an ip address, the pods part of the `memberlist` will fail.

## Changes Required

### Query Frontend Service

1. Change the name of `grpc` port to `grpclb`. This is used by the grpc load balancing strategy which relies on SRV records. Otherwise the `querier` will not be able to reach the `query-frontend`. See https://github.com/grafana/loki/blob/0116aa61c86fa983ddcbbd5e30a2141d2e89081a/production/ksonnet/loki/common.libsonnet#L19
and
https://grpc.github.io/grpc/core/md_doc_load-balancing.html
3. Set the `appProtocol` of `grpclb` to `tcp`
4. Set `publishNotReadyAddresses` to `true`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: loki-query-frontend
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-query-frontend
spec:
  ports:
  - appProtocol: http
    name: http
    port: 3100
    protocol: TCP
    targetPort: http
  - appProtocol: tcp
    name: grpclb
    port: 9095
    protocol: TCP
    targetPort: grpc
  publishNotReadyAddresses: true
  selector:
    app: loki-query-frontend
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-query-frontend
  type: ClusterIP
```

### Querier service

Set the `appProtocol` of the `grpc` service to `tcp`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: loki-querier
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-querier
  name: loki-querier
  namespace: observability
spec:
  ports:
  - appProtocol: http
    name: http
    port: 3100
    protocol: TCP
    targetPort: http
  - appProtocol: tcp
    name: grpc
    port: 9095
    protocol: TCP
    targetPort: grpc
  selector:
    app: loki-querier
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-querier
  type: ClusterIP

```

### Ingester Service and Ingester Headless Service

Set the `appProtocol` of the `grpc` port to `tcp` 

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: loki-ingester-(headless)
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-ingester
  name: loki-ingester-headless
spec:  
  clusterIP: None (if headless)
  ports:
  - name: http
    port: 3100
    protocol: TCP
    targetPort: http
  - appProtocol: tcp
    name: grpc
    port: 9095
    protocol: TCP
    targetPort: grpc
  selector:
    app: loki-ingester
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-ingester
  type: ClusterIP
```

### Distributor Service

Set the `appProtocol` of the `grpc` port to `tcp` 

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: loki-distributor
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-distributor
spec:
  ports:
  - name: http
    port: 3100
    protocol: TCP
    targetPort: http
  - name: grpc
    port: 9095
    protocol: TCP
    targetPort: grpc
    appProtocol: tcp
  selector:
    app: loki-distributor
    app.kubernetes.io/instance: observability
    app.kubernetes.io/name: loki-distributor
  sessionAffinity: None
  type: ClusterIP

```

## Memberlist Service

Set the `appProtocol` of the `http` port to `tcp`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: observability
  name: loki-memberlist
  namespace: observability
spec:
  clusterIP: None
  ports:
    - name: http
      port: 7946
      protocol: TCP
      targetPort: 7946
      appProtocol: tcp
  selector:
    app.kubernetes.io/instance: observability
    app.kubernetes.io/part-of: memberlist
```
