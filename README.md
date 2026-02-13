# Matchbox Helm Chart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/matchbox)](https://artifacthub.io/packages/search?repo=matchbox)

Helm chart for [Matchbox](https://matchbox.health) — an open-source FHIR server for validation and mapping, developed by [ahdis ag](https://ahdis.ch).

## Usage

```bash
helm repo add matchbox https://promptlylabs.github.io/matchbox-helm-chart
helm repo update
helm install matchbox matchbox/matchbox
```

See [charts/matchbox/README.md](charts/matchbox/README.md) for full documentation.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

[Apache License 2.0](LICENSE)
