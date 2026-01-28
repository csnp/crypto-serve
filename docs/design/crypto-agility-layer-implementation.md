# CryptoServe Crypto Agility Layer: Implementation Design

**Version:** 1.0
**Date:** January 2026
**Status:** Technical Design

---

## Overview

This document provides the technical implementation design for CryptoServe's Crypto Agility Layer — a suite of deployment modes that enable organizations to achieve cryptographic agility for legacy hardware, IoT devices, and mainframe systems without modifying the underlying infrastructure.

### The Three Modes

| Mode | Description | Best For |
|------|-------------|----------|
| **Transparent Proxy** | Network appliance intercepting traffic | Network-level protection, many devices |
| **API Gateway** | Protocol translation plugins for existing gateway | Application integration, API-first orgs |
| **Sidecar/Agent** | Lightweight process fronting legacy systems | Container/VM deployments, specific apps |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CryptoServe Crypto Agility Layer                      │
│                                                                          │
│   ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────────┐   │
│   │   Transparent    │ │    API Gateway   │ │    Sidecar/Agent     │   │
│   │      Proxy       │ │       Mode       │ │        Mode          │   │
│   │                  │ │                  │ │                      │   │
│   │  Network-level   │ │  Application     │ │  Process-level       │   │
│   │  interception    │ │  integration     │ │  companion           │   │
│   └────────┬─────────┘ └────────┬─────────┘ └──────────┬───────────┘   │
│            │                    │                      │               │
│            └────────────────────┼──────────────────────┘               │
│                                 │                                       │
│                    ┌────────────▼────────────┐                         │
│                    │   Shared Core Engine    │                         │
│                    │  • Translation Engine   │                         │
│                    │  • Device Profiles      │                         │
│                    │  • Policy Engine        │                         │
│                    │  • Audit & Metrics      │                         │
│                    └─────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

### Shared Components

All three modes share a common core:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Shared Core Library                             │
│                        (cryptoserve-agility-core)                        │
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │   Translation   │  │  Device Profile │  │     Policy Engine       │  │
│  │     Engine      │  │    Registry     │  │  (from existing core)   │  │
│  └────────┬────────┘  └────────┬────────┘  └───────────┬─────────────┘  │
│           │                    │                       │                 │
│  ┌────────▼────────┐  ┌────────▼────────┐  ┌──────────▼──────────────┐  │
│  │  Protocol       │  │  Capability     │  │   Algorithm Resolver    │  │
│  │  Handlers       │  │  Detector       │  │  (from existing core)   │  │
│  │  • TLS          │  │                 │  │                         │  │
│  │  • SSH          │  └─────────────────┘  └─────────────────────────┘  │
│  │  • Database     │                                                     │
│  │  • MQTT         │  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │  • Raw TCP      │  │  Audit Logger   │  │    Metrics Collector    │  │
│  └─────────────────┘  │ (existing SIEM) │  │   (Prometheus/OTEL)     │  │
│                       └─────────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility |
|-----------|----------------|
| Translation Engine | Perform real-time crypto protocol conversion |
| Protocol Handlers | Protocol-specific logic (TLS, SSH, etc.) |
| Device Profile Registry | Store/retrieve device capabilities |
| Capability Detector | Auto-probe devices for supported crypto |
| Policy Engine | Enforce translation rules (reuses existing) |
| Algorithm Resolver | Select appropriate algorithms (reuses existing) |
| Audit Logger | Log all translations (integrates with existing) |
| Metrics Collector | Performance and operational metrics |

---

## Mode 1: Transparent Proxy

### Concept

A network-level appliance (virtual or physical) that intercepts traffic transparently, requiring no client or server modifications.

```
┌─────────────┐                                           ┌─────────────┐
│   Client    │                                           │   Legacy    │
│  (Modern)   │                                           │   Server    │
└──────┬──────┘                                           └──────┬──────┘
       │                                                         │
       │ TLS 1.3 + PQC                              TLS 1.0 + RSA│
       │                                                         │
       ▼                                                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Network Infrastructure                            │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    Transparent Proxy                             │   │
│   │  ┌───────────┐   ┌───────────────┐   ┌───────────────────────┐  │   │
│   │  │  Traffic  │   │  Translation  │   │    Backend Pool       │  │   │
│   │  │ Intercept │──▶│    Engine     │──▶│  • mainframe:443      │  │   │
│   │  │  (eBPF)   │   │               │   │  • scada-gw:502       │  │   │
│   │  └───────────┘   └───────────────┘   │  • database:5432      │  │   │
│   │                                       └───────────────────────┘  │   │
│   │  ┌─────────────────────────────────────────────────────────────┐│   │
│   │  │ Intercept Rules:                                            ││   │
│   │  │ • 10.0.0.0/8:443 → translate (mainframe profile)            ││   │
│   │  │ • 10.0.0.0/8:502 → tunnel (modbus, wrap in TLS)             ││   │
│   │  │ • 10.0.0.0/8:5432 → translate (postgres profile)            ││   │
│   │  └─────────────────────────────────────────────────────────────┘│   │
│   └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### Traffic Interception Options

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **eBPF/XDP** | High performance, kernel-level | Linux only, complexity | High-throughput |
| **iptables REDIRECT** | Simple, portable | User-space overhead | Quick deployments |
| **VLAN/Port Mirror** | No host changes | Passive only | Monitoring mode |
| **Inline (bump-in-wire)** | True transparent | Single point of failure | Physical appliances |

#### Recommended: eBPF + User-space Proxy

```c
// bpf/intercept.c - eBPF program for traffic interception
SEC("xdp")
int xdp_intercept(struct xdp_md *ctx) {
    // Parse packet headers
    struct ethhdr *eth = (void *)(long)ctx->data;
    struct iphdr *ip = (void *)(eth + 1);
    struct tcphdr *tcp = (void *)(ip + 1);

    // Check if destination matches intercept rules
    if (should_intercept(ip->daddr, tcp->dest)) {
        // Redirect to user-space proxy via SOCKMAP
        return bpf_redirect_map(&sock_map, 0, 0);
    }

    return XDP_PASS;
}
```

```python
# backend/app/agility/proxy/transparent.py
class TransparentProxy:
    """Transparent proxy using eBPF for interception."""

    def __init__(self, config: ProxyConfig):
        self.config = config
        self.bpf = BPFLoader("intercept.o")
        self.translation_engine = TranslationEngine()

    async def start(self):
        # Attach eBPF program to interface
        self.bpf.attach_xdp(self.config.interface)

        # Start user-space proxy
        await self.serve_forever()

    async def handle_connection(
        self,
        client_reader: StreamReader,
        client_writer: StreamWriter
    ):
        # Determine original destination (from eBPF metadata)
        orig_dest = self.get_original_dest(client_writer)

        # Look up device profile for destination
        profile = await self.profile_registry.get_for_address(orig_dest)

        # Get translation policy
        policy = await self.policy_engine.get_policy(
            destination=orig_dest,
            profile=profile
        )

        # Perform translation
        await self.translation_engine.translate(
            ingress=client_reader,
            egress_dest=orig_dest,
            policy=policy
        )
```

#### Configuration

```yaml
# config/transparent-proxy.yaml
proxy:
  mode: transparent
  interface: eth0

intercept_rules:
  - name: mainframe-traffic
    match:
      destination: 10.0.1.0/24
      port: 443
    action: translate
    profile: ibm-zos
    context: mainframe-banking

  - name: scada-traffic
    match:
      destination: 10.0.2.0/24
      port: 502
    action: tunnel  # Wrap unencrypted Modbus in TLS
    context: industrial-critical

  - name: database-traffic
    match:
      destination: 10.0.3.0/24
      port: 5432
    action: translate
    profile: postgres-legacy
    context: general-internal

performance:
  connection_pool_size: 1000
  worker_threads: 4
  use_ebpf: true
```

#### Deployment

```yaml
# docker-compose.transparent-proxy.yaml
version: "3.8"
services:
  transparent-proxy:
    image: cryptoserve/agility-proxy:latest
    network_mode: host  # Required for transparent interception
    privileged: true    # Required for eBPF
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - BPF
    volumes:
      - ./config:/etc/cryptoserve
      - /sys/fs/bpf:/sys/fs/bpf
    environment:
      - CRYPTOSERVE_API_URL=https://api.cryptoserve.io
      - CRYPTOSERVE_API_KEY=${API_KEY}
```

---

## Mode 2: API Gateway Mode

### Concept

Extend the existing CryptoServe API with protocol translation plugins, allowing applications to route traffic through the gateway for crypto translation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CryptoServe API Gateway                             │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     Existing API Endpoints                         │  │
│  │  /api/v1/encrypt  /api/v1/decrypt  /api/v1/sign  /api/v1/verify   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    NEW: Protocol Translation                       │  │
│  │                                                                     │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │  │
│  │  │   /proxy    │  │  /tunnel    │  │  /connect   │  │ /transcode│ │  │
│  │  │   (HTTP)    │  │   (TCP)     │  │  (WebSocket)│  │  (Stream) │ │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │  │
│  │         │                │                │               │       │  │
│  │         └────────────────┼────────────────┼───────────────┘       │  │
│  │                          │                                         │  │
│  │                  ┌───────▼───────┐                                │  │
│  │                  │  Translation  │                                │  │
│  │                  │    Engine     │                                │  │
│  │                  └───────────────┘                                │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Plugin Architecture:                                                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐           │
│  │ TLS Plugin │ │ SSH Plugin │ │ MQTT Plugin│ │ DB Plugin  │           │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### New API Endpoints

```python
# backend/app/api/v1/endpoints/agility.py
from fastapi import APIRouter, WebSocket, Depends
from starlette.requests import Request
from starlette.responses import StreamingResponse

router = APIRouter(prefix="/agility", tags=["crypto-agility"])


@router.post("/proxy/{backend_id}")
async def proxy_request(
    backend_id: str,
    request: Request,
    current_user: User = Depends(get_current_user)
):
    """
    HTTP Reverse Proxy with crypto translation.

    Accepts modern TLS request, translates to legacy backend protocol.
    """
    backend = await get_backend(backend_id)
    policy = await get_translation_policy(backend)

    # Forward request with translation
    async with TranslationSession(policy) as session:
        response = await session.forward_http(
            method=request.method,
            url=backend.url + request.url.path,
            headers=request.headers,
            body=await request.body()
        )

    return StreamingResponse(
        response.stream(),
        status_code=response.status_code,
        headers=dict(response.headers)
    )


@router.websocket("/connect/{backend_id}")
async def websocket_tunnel(
    websocket: WebSocket,
    backend_id: str
):
    """
    WebSocket tunnel with crypto translation.

    Client connects via WebSocket, data tunneled to legacy TCP backend.
    """
    await websocket.accept()

    backend = await get_backend(backend_id)
    policy = await get_translation_policy(backend)

    async with TranslationSession(policy) as session:
        await session.tunnel_websocket(
            websocket=websocket,
            destination=backend.address
        )


@router.post("/transcode")
async def transcode_payload(
    request: TranscodeRequest,
    current_user: User = Depends(get_current_user)
):
    """
    One-shot payload transcoding.

    Re-encrypt data from one algorithm suite to another.
    Useful for batch migration or format conversion.
    """
    result = await translation_engine.transcode(
        data=request.data,
        source_context=request.source_context,
        target_context=request.target_context
    )

    return TranscodeResponse(
        data=result.data,
        source_algorithms=result.source_algorithms,
        target_algorithms=result.target_algorithms
    )
```

#### Protocol Plugin Architecture

```python
# backend/app/agility/plugins/base.py
from abc import ABC, abstractmethod
from typing import AsyncIterator

class ProtocolPlugin(ABC):
    """Base class for protocol translation plugins."""

    @property
    @abstractmethod
    def protocol_name(self) -> str:
        """Human-readable protocol name."""
        pass

    @property
    @abstractmethod
    def default_port(self) -> int:
        """Default port for this protocol."""
        pass

    @abstractmethod
    async def handshake_ingress(
        self,
        reader: StreamReader,
        writer: StreamWriter,
        policy: TranslationPolicy
    ) -> IngressSession:
        """Perform ingress (modern) handshake."""
        pass

    @abstractmethod
    async def handshake_egress(
        self,
        destination: Address,
        policy: TranslationPolicy
    ) -> EgressSession:
        """Perform egress (legacy) handshake."""
        pass

    @abstractmethod
    async def translate_stream(
        self,
        ingress: IngressSession,
        egress: EgressSession
    ) -> AsyncIterator[bytes]:
        """Translate data between sessions."""
        pass


# backend/app/agility/plugins/tls.py
class TLSPlugin(ProtocolPlugin):
    """TLS protocol translation plugin."""

    protocol_name = "TLS"
    default_port = 443

    async def handshake_ingress(
        self,
        reader: StreamReader,
        writer: StreamWriter,
        policy: TranslationPolicy
    ) -> IngressSession:
        # Create modern TLS context
        ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
        ctx.minimum_version = ssl.TLSVersion.TLSv1_3

        # Load PQC ciphersuites if available
        if policy.require_pqc:
            ctx.set_ciphers(PQC_CIPHERSUITES)

        # Perform handshake
        ssl_reader, ssl_writer = await asyncio.open_connection(
            ssl=ctx,
            server_side=True,
            reader=reader,
            writer=writer
        )

        return TLSIngressSession(ssl_reader, ssl_writer)

    async def handshake_egress(
        self,
        destination: Address,
        policy: TranslationPolicy
    ) -> EgressSession:
        # Create legacy TLS context based on device profile
        ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)

        profile = policy.device_profile
        if profile.max_tls_version == "1.0":
            ctx.maximum_version = ssl.TLSVersion.TLSv1
        elif profile.max_tls_version == "1.1":
            ctx.maximum_version = ssl.TLSVersion.TLSv1_1

        # Set legacy ciphers
        ctx.set_ciphers(profile.supported_ciphers)

        # Connect to backend
        reader, writer = await asyncio.open_connection(
            destination.host,
            destination.port,
            ssl=ctx
        )

        return TLSEgressSession(reader, writer)


# backend/app/agility/plugins/registry.py
class PluginRegistry:
    """Registry of available protocol plugins."""

    _plugins: dict[str, type[ProtocolPlugin]] = {}

    @classmethod
    def register(cls, plugin_class: type[ProtocolPlugin]):
        cls._plugins[plugin_class.protocol_name.lower()] = plugin_class

    @classmethod
    def get(cls, protocol: str) -> ProtocolPlugin:
        return cls._plugins[protocol.lower()]()

    @classmethod
    def list_protocols(cls) -> list[str]:
        return list(cls._plugins.keys())


# Auto-register plugins
PluginRegistry.register(TLSPlugin)
PluginRegistry.register(SSHPlugin)
PluginRegistry.register(PostgresPlugin)
PluginRegistry.register(MySQLPlugin)
PluginRegistry.register(MQTTPlugin)
PluginRegistry.register(ModbusPlugin)
PluginRegistry.register(RawTCPPlugin)
```

#### Client SDK Usage

```python
# sdk/python/packages/cryptoserve-agility/examples/gateway_mode.py
from cryptoserve import CryptoServeClient
from cryptoserve.agility import GatewayProxy

# Initialize client
client = CryptoServeClient(api_key="cs_live_xxx")

# Create a proxy to legacy mainframe
mainframe_proxy = client.agility.create_proxy(
    name="mainframe-banking",
    backend="mainframe.internal:443",
    profile="ibm-zos-2.4",
    context="financial-pci"
)

# Use like a normal HTTP client - translation happens automatically
response = mainframe_proxy.post(
    "/api/transfer",
    json={"amount": 1000, "to": "account123"}
)

# Or use the WebSocket tunnel for raw TCP
async with client.agility.tunnel("mainframe-banking") as tunnel:
    await tunnel.send(b"CICS TRANSACTION DATA")
    response = await tunnel.recv()
```

---

## Mode 3: Sidecar/Agent Mode

### Concept

A lightweight process deployed alongside legacy applications, intercepting local traffic and providing crypto translation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Host / Pod / VM                                │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        CryptoServe Agent                         │    │
│  │                                                                   │    │
│  │  ┌───────────────┐    ┌────────────────┐    ┌────────────────┐  │    │
│  │  │   Ingress     │    │  Translation   │    │    Egress      │  │    │
│  │  │   Listener    │───▶│    Engine      │───▶│   Connector    │  │    │
│  │  │  :8443 (TLS)  │    │                │    │   localhost    │  │    │
│  │  └───────────────┘    └────────────────┘    └───────┬────────┘  │    │
│  │         ▲                                           │            │    │
│  └─────────┼───────────────────────────────────────────┼────────────┘    │
│            │                                           │                  │
│            │ Modern clients                            ▼                  │
│            │ connect here         ┌─────────────────────────────────┐    │
│                                   │      Legacy Application          │    │
│                                   │      (no changes needed)         │    │
│                                   │      Listening on :443           │    │
│                                   └─────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### Agent Binary

```rust
// agent/src/main.rs
use tokio::net::{TcpListener, TcpStream};
use cryptoserve_agility_core::{TranslationEngine, Policy};

#[tokio::main]
async fn main() -> Result<()> {
    // Load configuration
    let config = Config::from_env()?;

    // Initialize translation engine
    let engine = TranslationEngine::new(config.clone()).await?;

    // Start metrics server
    tokio::spawn(metrics_server(config.metrics_port));

    // Start health check endpoint
    tokio::spawn(health_server(config.health_port));

    // Main listener
    let listener = TcpListener::bind(&config.listen_addr).await?;
    info!("Agent listening on {}", config.listen_addr);

    loop {
        let (stream, addr) = listener.accept().await?;
        let engine = engine.clone();
        let config = config.clone();

        tokio::spawn(async move {
            if let Err(e) = handle_connection(stream, addr, engine, config).await {
                error!("Connection error from {}: {}", addr, e);
            }
        });
    }
}

async fn handle_connection(
    ingress: TcpStream,
    addr: SocketAddr,
    engine: TranslationEngine,
    config: Config,
) -> Result<()> {
    // Perform modern TLS handshake on ingress
    let ingress_tls = engine.handshake_ingress(ingress, &config.ingress_policy).await?;

    // Connect to local backend
    let egress = TcpStream::connect(&config.backend_addr).await?;

    // Perform legacy handshake on egress
    let egress_tls = engine.handshake_egress(egress, &config.egress_policy).await?;

    // Bidirectional translation
    engine.translate_bidirectional(ingress_tls, egress_tls).await?;

    Ok(())
}
```

#### Configuration

```yaml
# agent-config.yaml
agent:
  name: mainframe-sidecar
  mode: sidecar

listen:
  address: 0.0.0.0
  port: 8443

backend:
  address: 127.0.0.1
  port: 443

ingress:
  tls_min_version: "1.3"
  ciphersuites:
    - TLS_AES_256_GCM_SHA384
    - TLS_CHACHA20_POLY1305_SHA256
  key_exchange:
    - ml-kem-768
    - x25519
  certificate: /etc/agent/certs/server.crt
  private_key: /etc/agent/certs/server.key

egress:
  profile: ibm-zos-2.4
  tls_version: "1.2"
  verify_backend: true

cryptoserve:
  api_url: https://api.cryptoserve.io
  api_key: ${CRYPTOSERVE_API_KEY}
  context_id: ctx_mainframe_banking
  sync_interval: 60s

observability:
  metrics_port: 9090
  health_port: 8080
  log_level: info
  log_format: json
```

#### Kubernetes Sidecar Deployment

```yaml
# k8s/sidecar-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app-with-agility
spec:
  replicas: 3
  selector:
    matchLabels:
      app: legacy-app
  template:
    metadata:
      labels:
        app: legacy-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
        # Legacy application container
        - name: legacy-app
          image: company/legacy-app:v1.2.3
          ports:
            - containerPort: 443
              name: legacy-https
          # App binds to localhost only, not exposed
          env:
            - name: BIND_ADDRESS
              value: "127.0.0.1"

        # CryptoServe Agent sidecar
        - name: cryptoserve-agent
          image: cryptoserve/agent:latest
          ports:
            - containerPort: 8443
              name: https
            - containerPort: 9090
              name: metrics
            - containerPort: 8080
              name: health
          env:
            - name: CRYPTOSERVE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: cryptoserve-secrets
                  key: api-key
            - name: BACKEND_ADDRESS
              value: "127.0.0.1:443"
            - name: CONTEXT_ID
              value: "ctx_mainframe_banking"
          volumeMounts:
            - name: agent-config
              mountPath: /etc/agent
            - name: tls-certs
              mountPath: /etc/agent/certs
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 5
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5

      volumes:
        - name: agent-config
          configMap:
            name: cryptoserve-agent-config
        - name: tls-certs
          secret:
            secretName: cryptoserve-agent-tls

---
apiVersion: v1
kind: Service
metadata:
  name: legacy-app
spec:
  selector:
    app: legacy-app
  ports:
    # External traffic goes to agent
    - name: https
      port: 443
      targetPort: 8443
```

#### Docker Compose Sidecar

```yaml
# docker-compose.sidecar.yaml
version: "3.8"
services:
  legacy-app:
    image: company/legacy-app:v1.2.3
    # No ports exposed - only agent is exposed
    environment:
      - BIND_ADDRESS=127.0.0.1
    networks:
      - internal

  cryptoserve-agent:
    image: cryptoserve/agent:latest
    ports:
      - "443:8443"
    environment:
      - CRYPTOSERVE_API_KEY=${CRYPTOSERVE_API_KEY}
      - BACKEND_ADDRESS=legacy-app:443
      - CONTEXT_ID=ctx_legacy_banking
    volumes:
      - ./agent-config.yaml:/etc/agent/config.yaml
      - ./certs:/etc/agent/certs
    networks:
      - internal
      - external
    depends_on:
      - legacy-app

networks:
  internal:
    internal: true  # No external access
  external:
```

#### Agent Sizing Guide

| Connections | Memory | CPU | Binary Size |
|-------------|--------|-----|-------------|
| 100 | 32MB | 0.1 core | 15MB |
| 1,000 | 64MB | 0.2 core | 15MB |
| 10,000 | 256MB | 0.5 core | 15MB |
| 100,000 | 1GB | 2 cores | 15MB |

---

## Data Models

### Device Profile

```python
# backend/app/models/device_profile.py
from sqlalchemy import Column, String, JSON, Enum
from sqlalchemy.dialects.postgresql import ARRAY

class DeviceProfile(Base):
    __tablename__ = "device_profiles"

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    vendor = Column(String)
    category = Column(Enum(
        "mainframe",
        "server",
        "network_equipment",
        "iot_device",
        "industrial",
        "medical",
        "custom"
    ))

    # Cryptographic capabilities
    tls_versions = Column(ARRAY(String))  # ["1.0", "1.1", "1.2"]
    key_exchange = Column(ARRAY(String))  # ["rsa", "dhe-rsa", "ecdhe"]
    symmetric_ciphers = Column(ARRAY(String))  # ["aes-128-cbc", "3des"]
    asymmetric_algorithms = Column(ARRAY(String))  # ["rsa-2048"]
    hash_algorithms = Column(ARRAY(String))  # ["sha1", "sha256"]

    # Limitations
    limitations = Column(ARRAY(String))  # ["no-tls-1.3", "no-ecdhe"]

    # Metadata
    compliance_notes = Column(ARRAY(String))
    auto_detected = Column(Boolean, default=False)
    last_probed = Column(DateTime)

    # Relationships
    tenant_id = Column(String, ForeignKey("tenants.id"))
    created_by = Column(String, ForeignKey("users.id"))
```

### Translation Policy

```python
# backend/app/models/translation_policy.py
class TranslationPolicy(Base):
    __tablename__ = "translation_policies"

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)

    # Links to existing models
    context_id = Column(String, ForeignKey("contexts.id"))
    device_profile_id = Column(String, ForeignKey("device_profiles.id"))

    # Ingress (modern) requirements
    ingress_min_tls = Column(String, default="1.3")
    ingress_key_exchange = Column(ARRAY(String))
    ingress_ciphers = Column(ARRAY(String))
    ingress_require_client_cert = Column(Boolean, default=False)

    # Egress (legacy) settings
    egress_tls_version = Column(String)
    egress_key_exchange = Column(String)
    egress_cipher = Column(String)
    egress_verify_cert = Column(Boolean, default=True)

    # Translation behavior
    mode = Column(Enum(
        "terminate_and_reencrypt",
        "tunnel",
        "passthrough"
    ))
    session_cache = Column(Boolean, default=True)
    session_timeout = Column(Integer, default=3600)

    # Audit settings
    log_connections = Column(Boolean, default=True)
    log_handshakes = Column(Boolean, default=True)
    sensitive_data_masking = Column(Boolean, default=True)
```

### Backend Registration

```python
# backend/app/models/agility_backend.py
class AgilityBackend(Base):
    __tablename__ = "agility_backends"

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)

    # Connection details
    address = Column(String, nullable=False)  # hostname or IP
    port = Column(Integer, nullable=False)
    protocol = Column(String, default="tls")

    # Associated profile and policy
    device_profile_id = Column(String, ForeignKey("device_profiles.id"))
    translation_policy_id = Column(String, ForeignKey("translation_policies.id"))

    # Health tracking
    health_status = Column(Enum("healthy", "degraded", "unhealthy", "unknown"))
    last_health_check = Column(DateTime)
    last_successful_connection = Column(DateTime)

    # Metadata
    tenant_id = Column(String, ForeignKey("tenants.id"))
    tags = Column(ARRAY(String))
```

### Bridge/Agent Instance

```python
# backend/app/models/agility_instance.py
class AgilityInstance(Base):
    __tablename__ = "agility_instances"

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)

    # Deployment mode
    mode = Column(Enum(
        "transparent_proxy",
        "api_gateway",
        "sidecar_agent"
    ))

    # Instance details
    version = Column(String)
    hostname = Column(String)
    ip_address = Column(String)
    listen_address = Column(String)
    listen_port = Column(Integer)

    # Associated backends
    backend_ids = Column(ARRAY(String))

    # Status
    status = Column(Enum("running", "stopped", "error", "unknown"))
    last_heartbeat = Column(DateTime)
    uptime_seconds = Column(Integer)

    # Metrics
    active_connections = Column(Integer, default=0)
    total_connections = Column(BigInteger, default=0)
    bytes_translated = Column(BigInteger, default=0)

    # Metadata
    tenant_id = Column(String, ForeignKey("tenants.id"))
    config_hash = Column(String)  # For detecting config drift
```

---

## API Endpoints

### Device Profiles

```
POST   /api/v1/agility/profiles              Create device profile
GET    /api/v1/agility/profiles              List device profiles
GET    /api/v1/agility/profiles/{id}         Get device profile
PUT    /api/v1/agility/profiles/{id}         Update device profile
DELETE /api/v1/agility/profiles/{id}         Delete device profile
POST   /api/v1/agility/profiles/detect       Auto-detect from address
GET    /api/v1/agility/profiles/presets      List built-in presets
```

### Translation Policies

```
POST   /api/v1/agility/policies              Create translation policy
GET    /api/v1/agility/policies              List translation policies
GET    /api/v1/agility/policies/{id}         Get translation policy
PUT    /api/v1/agility/policies/{id}         Update translation policy
DELETE /api/v1/agility/policies/{id}         Delete translation policy
POST   /api/v1/agility/policies/{id}/test    Test policy against backend
```

### Backends

```
POST   /api/v1/agility/backends              Register backend
GET    /api/v1/agility/backends              List backends
GET    /api/v1/agility/backends/{id}         Get backend details
PUT    /api/v1/agility/backends/{id}         Update backend
DELETE /api/v1/agility/backends/{id}         Remove backend
POST   /api/v1/agility/backends/{id}/probe   Probe backend capabilities
GET    /api/v1/agility/backends/{id}/health  Get backend health
```

### Instances (Bridges/Agents)

```
GET    /api/v1/agility/instances             List all instances
GET    /api/v1/agility/instances/{id}        Get instance details
DELETE /api/v1/agility/instances/{id}        Deregister instance
GET    /api/v1/agility/instances/{id}/config Get instance config
PUT    /api/v1/agility/instances/{id}/config Update instance config
GET    /api/v1/agility/instances/{id}/metrics Get instance metrics
GET    /api/v1/agility/instances/{id}/logs   Get instance logs
```

### Translation Operations (API Gateway Mode)

```
POST   /api/v1/agility/proxy/{backend_id}/*  HTTP reverse proxy
WS     /api/v1/agility/connect/{backend_id}  WebSocket tunnel
POST   /api/v1/agility/transcode             One-shot transcoding
```

### Inventory & Compliance

```
GET    /api/v1/agility/inventory             Get crypto inventory
GET    /api/v1/agility/inventory/quantum     Quantum vulnerability report
GET    /api/v1/agility/inventory/compliance  Compliance status
POST   /api/v1/agility/inventory/export      Export report (PDF/CSV)
```

---

## Translation Engine

### Core Translation Logic

```python
# backend/app/agility/engine/translation.py
class TranslationEngine:
    """Core engine for crypto protocol translation."""

    def __init__(
        self,
        plugin_registry: PluginRegistry,
        policy_engine: PolicyEngine,
        audit_logger: AuditLogger
    ):
        self.plugins = plugin_registry
        self.policy_engine = policy_engine
        self.audit = audit_logger

    async def translate(
        self,
        ingress_stream: StreamReader,
        ingress_writer: StreamWriter,
        backend: AgilityBackend,
        policy: TranslationPolicy
    ):
        """Perform bidirectional translation between ingress and egress."""

        # Get appropriate plugin
        plugin = self.plugins.get(policy.protocol)

        # Log connection start
        connection_id = await self.audit.log_connection_start(
            backend=backend,
            policy=policy,
            client_addr=ingress_writer.get_extra_info("peername")
        )

        try:
            # Perform ingress handshake (modern crypto)
            ingress_session = await plugin.handshake_ingress(
                reader=ingress_stream,
                writer=ingress_writer,
                policy=policy
            )

            await self.audit.log_handshake(
                connection_id=connection_id,
                direction="ingress",
                tls_version=ingress_session.tls_version,
                cipher=ingress_session.cipher,
                key_exchange=ingress_session.key_exchange
            )

            # Connect to backend
            egress_reader, egress_writer = await asyncio.open_connection(
                backend.address,
                backend.port
            )

            # Perform egress handshake (legacy crypto)
            egress_session = await plugin.handshake_egress(
                reader=egress_reader,
                writer=egress_writer,
                policy=policy
            )

            await self.audit.log_handshake(
                connection_id=connection_id,
                direction="egress",
                tls_version=egress_session.tls_version,
                cipher=egress_session.cipher,
                key_exchange=egress_session.key_exchange
            )

            # Bidirectional data translation
            await asyncio.gather(
                self._translate_direction(
                    source=ingress_session,
                    dest=egress_session,
                    connection_id=connection_id,
                    direction="client_to_server"
                ),
                self._translate_direction(
                    source=egress_session,
                    dest=ingress_session,
                    connection_id=connection_id,
                    direction="server_to_client"
                )
            )

        except Exception as e:
            await self.audit.log_error(connection_id, str(e))
            raise

        finally:
            await self.audit.log_connection_end(connection_id)

    async def _translate_direction(
        self,
        source: Session,
        dest: Session,
        connection_id: str,
        direction: str
    ):
        """Translate data in one direction."""
        bytes_translated = 0

        try:
            while True:
                data = await source.read(65536)
                if not data:
                    break

                await dest.write(data)
                bytes_translated += len(data)

        finally:
            await self.audit.log_bytes_translated(
                connection_id=connection_id,
                direction=direction,
                bytes=bytes_translated
            )
```

### Capability Detection

```python
# backend/app/agility/engine/detector.py
class CapabilityDetector:
    """Auto-detect cryptographic capabilities of legacy devices."""

    TLS_VERSIONS = ["1.3", "1.2", "1.1", "1.0"]
    CIPHERSUITES = [
        # Modern
        "TLS_AES_256_GCM_SHA384",
        "TLS_CHACHA20_POLY1305_SHA256",
        # Legacy
        "ECDHE-RSA-AES256-GCM-SHA384",
        "DHE-RSA-AES256-GCM-SHA384",
        "ECDHE-RSA-AES128-SHA256",
        "RSA-AES256-SHA256",
        "RSA-AES128-SHA",
        "DES-CBC3-SHA",
    ]

    async def detect(
        self,
        address: str,
        port: int,
        timeout: float = 10.0
    ) -> DeviceProfile:
        """Probe device and generate capability profile."""

        capabilities = {
            "tls_versions": [],
            "ciphersuites": [],
            "key_exchange": [],
            "certificates": []
        }

        # Test each TLS version
        for version in self.TLS_VERSIONS:
            if await self._test_tls_version(address, port, version, timeout):
                capabilities["tls_versions"].append(version)

        # Test ciphersuites
        for cipher in self.CIPHERSUITES:
            if await self._test_cipher(address, port, cipher, timeout):
                capabilities["ciphersuites"].append(cipher)

        # Extract key exchange from successful ciphers
        capabilities["key_exchange"] = self._extract_key_exchange(
            capabilities["ciphersuites"]
        )

        # Get certificate details
        capabilities["certificates"] = await self._get_certificate_chain(
            address, port, timeout
        )

        # Generate profile
        return DeviceProfile(
            id=f"auto_{address}_{port}",
            name=f"Auto-detected: {address}:{port}",
            auto_detected=True,
            last_probed=datetime.utcnow(),
            tls_versions=capabilities["tls_versions"],
            key_exchange=capabilities["key_exchange"],
            symmetric_ciphers=self._extract_symmetric(capabilities["ciphersuites"]),
            hash_algorithms=self._extract_hash(capabilities["ciphersuites"]),
            limitations=self._determine_limitations(capabilities)
        )

    async def _test_tls_version(
        self,
        address: str,
        port: int,
        version: str,
        timeout: float
    ) -> bool:
        """Test if device supports specific TLS version."""
        ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
        ctx.check_hostname = False
        ctx.verify_mode = ssl.CERT_NONE

        version_map = {
            "1.3": ssl.TLSVersion.TLSv1_3,
            "1.2": ssl.TLSVersion.TLSv1_2,
            "1.1": ssl.TLSVersion.TLSv1_1,
            "1.0": ssl.TLSVersion.TLSv1,
        }

        ctx.minimum_version = version_map[version]
        ctx.maximum_version = version_map[version]

        try:
            reader, writer = await asyncio.wait_for(
                asyncio.open_connection(address, port, ssl=ctx),
                timeout=timeout
            )
            writer.close()
            await writer.wait_closed()
            return True
        except:
            return False
```

---

## Frontend Components

### New Dashboard Pages

```typescript
// frontend/src/app/agility/page.tsx
export default function AgilityDashboard() {
  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold">Crypto Agility Layer</h1>
        <Button>
          <Plus className="mr-2 h-4 w-4" />
          Add Instance
        </Button>
      </div>

      {/* Summary Cards */}
      <div className="grid grid-cols-4 gap-4">
        <SummaryCard
          title="Active Instances"
          value={stats.instances}
          icon={<Server />}
          trend="+2 this week"
        />
        <SummaryCard
          title="Protected Devices"
          value={stats.devices}
          icon={<Shield />}
          trend="+15 this month"
        />
        <SummaryCard
          title="Active Connections"
          value={stats.connections}
          icon={<Activity />}
          realtime
        />
        <SummaryCard
          title="Quantum Ready"
          value={`${stats.quantumReadyPercent}%`}
          icon={<Atom />}
          trend="+5% this quarter"
        />
      </div>

      {/* Instance List */}
      <Card>
        <CardHeader>
          <CardTitle>Instances</CardTitle>
        </CardHeader>
        <CardContent>
          <InstanceTable instances={instances} />
        </CardContent>
      </Card>

      {/* Device Inventory Quick View */}
      <Card>
        <CardHeader>
          <CardTitle>Device Inventory</CardTitle>
          <CardDescription>
            Cryptographic posture of protected devices
          </CardDescription>
        </CardHeader>
        <CardContent>
          <DeviceInventoryChart data={inventoryData} />
        </CardContent>
      </Card>
    </div>
  );
}
```

```typescript
// frontend/src/components/agility/add-instance-wizard.tsx
export function AddInstanceWizard() {
  const [step, setStep] = useState(1);
  const [config, setConfig] = useState<InstanceConfig>({});

  return (
    <Dialog>
      <DialogContent className="max-w-2xl">
        <DialogHeader>
          <DialogTitle>Deploy New Instance</DialogTitle>
        </DialogHeader>

        {step === 1 && (
          <DeploymentModeStep
            onSelect={(mode) => {
              setConfig({ ...config, mode });
              setStep(2);
            }}
          />
        )}

        {step === 2 && (
          <BackendConfigStep
            onSubmit={(backend) => {
              setConfig({ ...config, backend });
              setStep(3);
            }}
            onBack={() => setStep(1)}
          />
        )}

        {step === 3 && (
          <ContextSelectionStep
            onSubmit={(context) => {
              setConfig({ ...config, context });
              setStep(4);
            }}
            onBack={() => setStep(2)}
          />
        )}

        {step === 4 && (
          <DeploymentStep
            config={config}
            onDeploy={handleDeploy}
            onBack={() => setStep(3)}
          />
        )}
      </DialogContent>
    </Dialog>
  );
}

function DeploymentModeStep({ onSelect }) {
  return (
    <div className="grid grid-cols-3 gap-4">
      <ModeCard
        icon={<Network />}
        title="Transparent Proxy"
        description="Network-level interception for multiple devices"
        onClick={() => onSelect("transparent_proxy")}
      />
      <ModeCard
        icon={<Globe />}
        title="API Gateway"
        description="Application-level integration via API"
        onClick={() => onSelect("api_gateway")}
      />
      <ModeCard
        icon={<Container />}
        title="Sidecar Agent"
        description="Lightweight companion for legacy apps"
        onClick={() => onSelect("sidecar_agent")}
      />
    </div>
  );
}
```

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-4)

**Deliverables:**
1. Device Profile model and CRUD API
2. Translation Policy model and CRUD API
3. Backend registration and health check
4. Basic TLS translation engine
5. Sidecar agent (Rust) MVP

**Tasks:**
```
[ ] Create database migrations for new models
[ ] Implement device profile API endpoints
[ ] Implement translation policy API endpoints
[ ] Implement backend registration API
[ ] Build TLS protocol plugin
[ ] Create Rust agent skeleton
[ ] Implement basic TLS termination in agent
[ ] Add agent-to-API heartbeat
[ ] Create Docker image for agent
[ ] Write integration tests
```

### Phase 2: Gateway Mode (Weeks 5-8)

**Deliverables:**
1. HTTP reverse proxy endpoint
2. WebSocket tunnel endpoint
3. Protocol plugin architecture
4. SSH protocol plugin
5. Database protocol plugins (Postgres, MySQL)

**Tasks:**
```
[ ] Implement /proxy endpoint
[ ] Implement /connect WebSocket endpoint
[ ] Create plugin registry system
[ ] Build SSH protocol plugin
[ ] Build Postgres protocol plugin
[ ] Build MySQL protocol plugin
[ ] Add connection pooling
[ ] Implement session caching
[ ] Create SDK helper for proxy mode
[ ] Write performance benchmarks
```

### Phase 3: Transparent Proxy (Weeks 9-12)

**Deliverables:**
1. eBPF traffic interception
2. Transparent proxy binary
3. Network appliance Docker image
4. Intercept rule configuration
5. High availability support

**Tasks:**
```
[ ] Develop eBPF interception program
[ ] Create user-space proxy handler
[ ] Implement intercept rule engine
[ ] Build network appliance container
[ ] Add VRRP/keepalived support
[ ] Implement connection draining
[ ] Create Helm chart for HA deployment
[ ] Performance tuning for 100k connections
[ ] Write operational documentation
```

### Phase 4: UI & Compliance (Weeks 13-16)

**Deliverables:**
1. Agility dashboard pages
2. Instance management UI
3. Device inventory views
4. Quantum readiness report
5. Compliance export

**Tasks:**
```
[ ] Create Agility dashboard page
[ ] Build instance list/detail views
[ ] Build device profile management UI
[ ] Build policy editor UI
[ ] Implement add-instance wizard
[ ] Create quantum vulnerability chart
[ ] Build compliance report generator
[ ] Add PDF/CSV export
[ ] Implement real-time metrics display
[ ] Create guided onboarding flow
```

### Phase 5: Advanced Features (Weeks 17-20)

**Deliverables:**
1. Auto-discovery and probing
2. MQTT protocol plugin
3. Modbus/industrial protocols
4. Fleet bulk operations
5. GitOps configuration

**Tasks:**
```
[ ] Implement capability detector
[ ] Build scheduled re-probing
[ ] Create MQTT protocol plugin
[ ] Create Modbus protocol plugin
[ ] Add bulk device import
[ ] Add bulk policy update
[ ] Implement GitOps config sync
[ ] Add Terraform provider
[ ] Create Ansible playbooks
[ ] Write migration guides
```

---

## Testing Strategy

### Unit Tests

```python
# tests/unit/agility/test_translation_engine.py
class TestTranslationEngine:
    async def test_tls_version_downgrade(self):
        """Test translation from TLS 1.3 to TLS 1.0."""
        policy = TranslationPolicy(
            ingress_min_tls="1.3",
            egress_tls_version="1.0"
        )

        engine = TranslationEngine(...)
        result = await engine.translate(
            mock_tls13_stream,
            mock_tls10_backend,
            policy
        )

        assert result.ingress_tls_version == "1.3"
        assert result.egress_tls_version == "1.0"

    async def test_pqc_to_rsa_translation(self):
        """Test translation from ML-KEM to RSA."""
        ...
```

### Integration Tests

```python
# tests/integration/agility/test_sidecar.py
class TestSidecarAgent:
    @pytest.fixture
    async def legacy_server(self):
        """Start a TLS 1.0 test server."""
        ...

    @pytest.fixture
    async def agent(self, legacy_server):
        """Start agent pointing to legacy server."""
        ...

    async def test_modern_client_to_legacy_server(
        self,
        agent,
        legacy_server
    ):
        """Modern client connects through agent to legacy server."""
        # Connect with TLS 1.3
        async with aiohttp.ClientSession() as session:
            resp = await session.get(
                f"https://localhost:{agent.port}/test",
                ssl=modern_ssl_context
            )
            assert resp.status == 200

        # Verify legacy server received TLS 1.0
        assert legacy_server.last_connection.tls_version == "TLSv1"
```

### Load Tests

```python
# tests/load/agility/test_throughput.py
@pytest.mark.load
async def test_agent_connection_capacity():
    """Test agent handles 10,000 concurrent connections."""
    agent = await start_test_agent()

    async def make_connection():
        async with aiohttp.ClientSession() as session:
            await session.get(f"https://localhost:{agent.port}/")

    # Open 10,000 connections
    tasks = [make_connection() for _ in range(10000)]
    await asyncio.gather(*tasks)

    assert agent.metrics.active_connections_peak >= 10000
    assert agent.metrics.errors == 0
```

---

## Deployment Artifacts

### Docker Images

```
cryptoserve/agility-proxy:latest      # Transparent proxy
cryptoserve/agility-agent:latest      # Sidecar agent
cryptoserve/agility-gateway:latest    # API gateway (included in main image)
```

### Helm Charts

```
charts/
├── cryptoserve-agility-proxy/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── rbac.yaml
├── cryptoserve-agility-agent/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml  # As sidecar injector
│       └── mutatingwebhook.yaml
└── cryptoserve-agility-full/
    └── ... # Umbrella chart
```

### Terraform Modules

```hcl
# terraform/modules/cryptoserve-agility/main.tf
module "cryptoserve_agility" {
  source = "cryptoserve/agility/aws"

  mode = "transparent_proxy"

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids

  backends = [
    {
      name    = "mainframe"
      address = "10.0.1.100"
      port    = 443
      profile = "ibm-zos"
      context = "financial"
    }
  ]

  instance_type = "c5.xlarge"
  desired_count = 3

  tags = var.common_tags
}
```

---

## Security Considerations

### Threat Model

| Threat | Mitigation |
|--------|------------|
| Key exposure in agent | Keys derived from master, never stored |
| Man-in-the-middle | Mutual TLS option, certificate pinning |
| Audit log tampering | Signed audit entries, immutable storage |
| Config injection | Signed config, integrity verification |
| Denial of service | Rate limiting, connection limits |

### Secrets Management

```yaml
# Agent secrets flow
1. Agent starts with API key only
2. Agent authenticates to CryptoServe API
3. API returns short-lived TLS certificate
4. Agent uses certificate for ingress
5. Certificate auto-rotates every 24h
6. No long-lived secrets on agent
```

### Audit Requirements

Every translation MUST log:
- Connection ID (unique)
- Timestamp (ISO 8601)
- Client address
- Backend address
- Ingress crypto (version, cipher, key exchange)
- Egress crypto (version, cipher, key exchange)
- Bytes translated
- Duration
- Errors (if any)

---

## Metrics & Observability

### Prometheus Metrics

```
# Instance metrics
cryptoserve_agility_connections_active{instance="...", backend="..."}
cryptoserve_agility_connections_total{instance="...", backend="..."}
cryptoserve_agility_bytes_translated_total{instance="...", direction="..."}
cryptoserve_agility_translation_latency_seconds{instance="...", quantile="..."}
cryptoserve_agility_handshake_failures_total{instance="...", reason="..."}

# Fleet metrics
cryptoserve_agility_instances_total{status="..."}
cryptoserve_agility_devices_total{risk_level="..."}
cryptoserve_agility_quantum_ready_ratio
```

### Grafana Dashboards

```
dashboards/
├── agility-overview.json       # Fleet summary
├── agility-instance.json       # Single instance detail
├── agility-compliance.json     # Compliance metrics
└── agility-performance.json    # Performance deep-dive
```

---

## Open Items for Development

1. **Language choice for agent**: Rust vs Go vs C++?
2. **eBPF kernel version requirements**: Minimum Linux 5.x?
3. **Windows support**: Agent on Windows servers?
4. **Offline mode**: Agent operation without API connectivity?
5. **Multi-region**: Geo-distributed backend pools?
6. **mTLS everywhere**: Require client certs on all modes?

---

## References

- [CryptoServe Bridge PRD](../prd/crypto-bridge-prd.md)
- [IoT Hardware Expansion](../exploration/iot-hardware-expansion.md)
- [Existing Crypto Engine](../../backend/app/core/crypto_engine.py)
- [Existing Policy Engine](../../backend/app/core/policy_engine.py)
