# Matchbox Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/matchbox)](https://artifacthub.io/packages/helm/matchbox/matchbox)

Helm chart for deploying [Matchbox](https://matchbox.health) on Kubernetes — an open-source FHIR server for validation and mapping of HL7 FHIR resources, developed by [ahdis ag](https://ahdis.ch).

Matchbox provides FHIR resource validation, FHIR Mapping Language endpoints, and Structured Data Capture (SDC) extraction, built on [HAPI FHIR](https://hapifhir.io).

## Installation

Available on [Artifact Hub](https://artifacthub.io/packages/helm/matchbox/matchbox).

```bash
helm repo add matchbox https://promptlylabs.github.io/matchbox-helm-chart
helm repo update
helm install matchbox matchbox/matchbox
```

See [charts/matchbox/README.md](charts/matchbox/README.md) for full configuration documentation.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

[Apache License 2.0](LICENSE)
