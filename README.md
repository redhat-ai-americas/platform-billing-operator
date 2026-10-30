# platform-billing-operator

A Kubernetes operator that adds billing, metering, and FinOps to OpenShift AI's Models-as-a-Service. It compiles billing Plans into Kuadrant enforcement policies, meters per-request token usage, manages prepaid wallets with automatic balance enforcement, integrates with payment processors (Stripe), and provides a billing console for providers and tenants.

For a full walkthrough, see [Getting Started](docs/getting-started.md). For how the components fit together, see [Architecture](docs/architecture.md). For CRD field details, see [API Reference](docs/api-reference.md).

## Quick start (local development)

```bash
# Create a kind cluster with the full Kuadrant stack.
make dev-cluster

# Build and deploy the operator.
make docker-build IMG=controller:latest
make deploy IMG=controller:latest

# Run the M0 end-to-end demo.
make demo-m0
```

To stand up a full demo on OpenShift (RHOAI, Connectivity Link, a model behind the gateway), see [docs/demo-cluster-setup.md](docs/demo-cluster-setup.md).

## Install via OLM

```bash
operator-sdk olm install
operator-sdk run bundle quay.io/wjackson/platform-billing-operator-bundle:v0.0.1
```

## CRDs

| CRD | Scope | Purpose |
|-----|-------|---------|
| `PriceBook` | Namespaced | SKU pricing catalog with immutability after effectiveFrom |
| `Plan` | Namespaced | Rate limits and model-scope access, compiled to Kuadrant CRs |
| `TenantAccount` | Namespaced | Billing tenant with wallet, billing mode, and lifecycle state machine |
| `BillingConnector` | Namespaced | Payment service provider integration (Stripe, generic webhook) |
| `BudgetPolicy` | Namespaced | Spending caps with threshold-based actions (Notify, Throttle, Freeze) |
| `BillingConsole` | Cluster | Operator-managed billing console UI deployment (singleton) |
| `Reservation` | Namespaced | GPU committed-use reservations with calendar windows |

## Contributing

Run `make help` for available targets. Run `make test && make lint && make money-safety` before committing.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the development environment, the rules CI enforces, and what a reviewable change looks like.

## License

Copyright 2026 Red Hat, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

