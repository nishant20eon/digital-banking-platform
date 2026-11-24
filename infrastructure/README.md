# Infrastructure

This directory is reserved for infrastructure-as-code, environment-specific deployment manifests, dashboards, alert rules and operational scripts.

Suggested future layout:

```text
infrastructure/
├── docker/
├── prometheus/
├── grafana/
├── otel/
├── k8s/
└── scripts/
```

Do not commit credentials, private keys or production configuration.
