# Tempo + OTel Research

This repo runs Grafana Tempo with MinIO object storage and an OpenTelemetry Collector.

## Current Setup In This Repo

- Tempo image: `grafana/tempo:2.10.1`
- MinIO S3-compatible object storage
- OTel Collector forwarding traces to Tempo
- Kubernetes manifests:
  - `tempo-config.yaml`
  - `tempo-deployment.yaml`
  - `tempo-service.yaml`
  - `minio.yaml`
  - `otel-collector-config.yaml`
  - `otel-collector-deployment.yaml`
  - `otel-collector-service.yaml`

## OTel Trace Flow (Current)

```mermaid
flowchart LR
    A[App / SDK] -->|OTLP gRPC or HTTP| B[OTel Collector]
    B -->|OTLP gRPC| C[Tempo Distributor]
    C --> D[Tempo Ingester]
    D -->|WAL + Blocks| E[MinIO S3 Bucket]
    F[Grafana / API Query] --> G[Tempo Query Frontend + Querier]
    G --> E
    G --> D
```

## Four Tempo Deployment Types

Below are four practical Tempo architecture types used in real deployments.

### 1) Single Binary (Monolithic)

- One Tempo process handles all roles (receiver/distributor, ingester, querier, query-frontend, compactor).
- Best for local dev, testing, and small workloads.

```mermaid
flowchart LR
    A[App / SDK] -->|OTLP| R1[Receiver/Distributor]
    R1 --> I1[Ingester]
    I1 --> S1[(Local Disk or S3)]
    G1[Grafana] --> QF1[Query Frontend]
    QF1 --> Q1[Querier]
    Q1 --> I1
    Q1 --> S1
    C1[Compactor] --> S1
    subgraph T1[Tempo Single Binary Pod]
      R1
      I1
      QF1
      Q1
      C1
    end
```

### 2) Scalable Single Binary

- Multiple Tempo single-binary replicas behind a service (each replica includes receiver, ingester, querier, query-frontend, compactor).
- Keeps simple operations but allows horizontal scale.

```mermaid
flowchart LR
    A[App / SDK] -->|OTLP| LBW[Write Service / LB]
    LBW --> R2A[Receiver]
    LBW --> R2B[Receiver]
    LBW --> R2C[Receiver]
    R2A --> I2A[Ingester]
    R2B --> I2B[Ingester]
    R2C --> I2C[Ingester]
    I2A --> S2[(Shared S3 Object Storage)]
    I2B --> S2
    I2C --> S2

    G2[Grafana] -->|Query| LBQ[Query Service / LB]
    LBQ --> QF2A[Query Frontend]
    LBQ --> QF2B[Query Frontend]
    QF2A --> Q2A[Querier]
    QF2B --> Q2B[Querier]
    Q2A --> I2A
    Q2A --> S2
    Q2B --> I2B
    Q2B --> S2

    C2A[Compactor] --> S2
    C2B[Compactor] --> S2
```

### 3) Microservices (Distributed Tempo)

- Tempo components are split: distributor, ingester, querier, query-frontend, compactor.
- Best for large scale and independent component scaling.

```mermaid
flowchart LR
    A[App / SDK] -->|OTLP| D[Distributor]
    D --> I[Ingester]
    I -->|Flush Blocks| S[(S3 / MinIO)]
    C[Compactor] --> S
    G[Grafana] --> QF[Query Frontend]
    QF --> Q[Querier]
    Q --> S
    Q --> I
```

### 4) Agent/Gateway Pattern (OTel + Tempo)

- OTel agents run near apps, optional central OTel gateway layer, then Tempo backend (receiver/distributor, ingester, querier, query-frontend, compactor).
- Best for centralized processing, filtering, retries, and multi-signal pipelines.

```mermaid
flowchart LR
    A1[App Node A] --> OA1[OTel Agent]
    A2[App Node B] --> OA2[OTel Agent]
    OA1 --> GW[OTel Gateway]
    OA2 --> GW
    GW -->|OTLP| TD[Tempo Receiver/Distributor]
    TD --> TI[Tempo Ingester]
    TI --> S[(S3 / MinIO)]
    TC[Tempo Compactor] --> S
    GR[Grafana] -->|Query| TQF[Tempo Query Frontend]
    TQF --> TQ[Tempo Querier]
    TQ --> TI
    TQ --> S
```

## Quick Apply

```bash
kubectl apply -f minio.yaml
kubectl apply -f tempo-config.yaml
kubectl apply -f tempo-deployment.yaml
kubectl apply -f tempo-service.yaml
kubectl apply -f otel-collector-config.yaml
kubectl apply -f otel-collector-deployment.yaml
kubectl apply -f otel-collector-service.yaml
```

## Notes

- For in-cluster OTLP receivers, bind endpoints to `0.0.0.0` (not `127.0.0.1`).
- Ensure bucket `tempo` exists in MinIO before Tempo starts.
