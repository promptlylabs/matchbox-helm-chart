# Matchbox Helm Chart

A Helm chart for deploying [Matchbox](https://matchbox.health), a FHIR server based on HAPI FHIR JPA server starter.

## Description

Matchbox is a FHIR server that provides:
- Validation support for FHIR resources conforming to loaded implementation guides
- FHIR Mapping Language endpoints for StructureMap creation and transformation
- SDC (Structured Data Capture) extraction support

## Prerequisites

- Kubernetes cluster (1.19+)
- Helm 3.0+
- PostgreSQL database (external or in-cluster)
- Storage class for persistent volumes (if needed)

## Installation

### Basic Installation

```bash
helm install matchbox charts/matchbox
```

### With Custom Values

```bash
helm install matchbox charts/matchbox -f custom-values.yaml
```

### Example Configuration with Ingress and Database

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: matchbox.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: matchbox-tls
      hosts:
        - matchbox.example.com

secretNames:
  - matchbox-db-secret

resources:
  requests:
    cpu: 500m
    memory: 2Gi
  limits:
    cpu: 1000m
    memory: 4Gi

matchbox:
  spring:
    datasource:
      url: "jdbc:postgresql://<host>:<port>/<database>"
      username: # set via environment variable from secret
      password: # set via environment variable from secret
      driverClassName: org.postgresql.Driver
    jpa:
      properties:
        hibernate:
          dialect: org.hibernate.dialect.PostgreSQLDialect 
  hapi:
    fhir:
      implementation-guides:
        hl7_fhir_r4_core:
          name: hl7.fhir.r4.core
          version: 4.0.1
        fhir_r4_wales_psom:
          name: fhir.r4.wales.psom
          version: 1.0.0-rc3

```

## Configuration

The following table lists the main configurable parameters of the Matchbox chart and their default values.

| Parameter           | Description                                                             | Default                                               |
|---------------------|-------------------------------------------------------------------------|-------------------------------------------------------|
| `replicaCount`      | Number of replicas                                                      | `1`                                                   |
| `image.repository`  | Matchbox image repository                                               | `europe-west6-docker.pkg.dev/ahdis-ch/ahdis/matchbox` |
| `image.tag`         | Matchbox image tag                                                      | `v4.0.18` (chart appVersion)                          |
| `image.pullPolicy`  | Image pull policy                                                       | `Always`                                              |
| `service.type`      | Kubernetes service type                                                 | `ClusterIP`                                           |
| `service.port`      | Service port                                                            | `8080`                                                |
| `ingress.enabled`   | Enable ingress                                                          | `false`                                               |
| `ingress.className` | Ingress class name                                                      | `""`                                                  |
| `ingress.hosts`     | Ingress hosts configuration                                             | See values.yaml                                       |
| `resources`         | CPU/Memory resource requests/limits                                     | `{}`                                                  |
| `secretNames`       | Names of additional secrets with env vars                               | `[]`                                                  |
| `matchbox`          | Matchbox-specific configuration (see https://ahdis.github.io/matchbox/) | See values.yaml                                       |

### Database Configuration

Matchbox can use a PostgreSQL database for persistence.
If none is configured, an in-memory H2 database will be used (not recommended for production).
Configure it using the typical Spring Boot properties under the `matchbox` section in values.yaml (see example there):

Ideally, the JDBC URL, username, and password should be provided via a Kubernetes Secret.
You can create the secret as follows:

```bash
kubectl create secret generic matchbox-db-secret \
  --from-literal=SPRING_DATASOURCE_URL="jdbc:postgresql://postgresql:5432/matchbox" \
  --from-literal=SPRING_DATASOURCE_USERNAME="matchbox" \
  --from-literal=SPRING_DATASOURCE_PASSWORD="matchbox"
```

and then reference `matchbox-db-secret` in the `secretNames` array in values.yaml:

```yaml
secretNames:
  - matchbox-db-secret
```

### Health Checks

The chart configures three types of health probes:

- **Startup Probe**: Checks if the application has started
- **Liveness Probe**: Ensures the application is running
- **Readiness Probe**: Determines if the application is ready to serve traffic


## Accessing Matchbox

Once deployed, Matchbox provides:

- **FHIR API**: `http://<host>/v4/fhir`
- **Web UI**: `http://<host>/v4/#/`

Note that the base path `/v4/` can be changed on deployment via the property `matchbox.server.servlet.context-path`.
Unfortunately, setting it as a root path (`/`) makes the web UI fail to correctly send requests to the backend.

## Uninstalling

```bash
helm uninstall matchbox
```

## Troubleshooting

### Network Requirements

Matchbox requires network access to the following external services:
- **tx.fhir.org**: FHIR terminology server for validation
- **simplifier.net**: Package server for downloading implementation guides at startup
- **PostgreSQL database**: Database connectivity (if using external database)

Ensure your network policies and firewall rules allow egress to these destinations.

### Pod fails to start

Check the logs:
```bash
kubectl logs -l app.kubernetes.io/name=matchbox
```

### Database connection issues

Verify database configuration and credentials:
```bash
kubectl get secret matchbox-db-secret -o yaml
```

### Implementation guides not loading

Check the configuration in the ConfigMap:
```bash
kubectl get configmap <release-name>-matchbox-config -o yaml
```

Verify network connectivity to simplifier.net for IG package downloads.

## Production Deployment Recommendations

For production deployments, consider:

1. **High Availability**: Set `replicaCount: 2` and enable PDB:
   ```yaml
   replicaCount: 2
   pdb:
     enabled: true
     minAvailable: 33%
   ```

2. **Resource Limits**: Configure appropriate resources based on load testing (Matchbox recommends minimum 2.5GB memory):
   ```yaml
   resources:
     limits:
       cpu: 2000m
       memory: 4Gi
     requests:
       cpu: 1000m
       memory: 2500Mi
   ```

3. **Image Repository**: Mirror the Matchbox image to your private container registry for better reliability and security.

4. **Network Policies**: Define explicit ingress/egress rules for production environments.

## References

- [Matchbox GitHub Repository](https://github.com/ahdis/matchbox/)
- [Matchbox Documentation](https://ahdis.github.io/matchbox/)
