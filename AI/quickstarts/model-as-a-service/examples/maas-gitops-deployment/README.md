# MaaS Platform GitOps Deployment

Complete GitOps deployment for the Model-as-a-Service (MaaS) platform using ArgoCD. This deployment automates the installation of operators, platform setup, and runtime configuration that was previously done via the `install-runtime.sh` script.

## Overview

This GitOps deployment replaces the manual helm installation process with a declarative, automated approach using ArgoCD. It deploys all three phases of the MaaS platform:

1. **Phase 1 (Wave 0)**: Operator Subscriptions - `maas-operators`
2. **Phase 2 (Wave 10)**: Platform Setup - `maas-platform` 
3. **Phase 3 (Wave 20)**: Runtime Resources - `maas-runtime`

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     ArgoCD Project                          │
│                    (maas-platform)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ maas-operators│    │ maas-platform │    │ maas-runtime  │
│   (Wave 0)    │───▶│   (Wave 10)   │───▶│   (Wave 20)   │
└───────────────┘    └───────────────┘    └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ • RHOAI       │    │ • DSC         │    │ • Gateway     │
│ • Kuadrant    │    │ • Kuadrant    │    │ • Model Reg   │
│ • Cert-Mgr    │    │ • LWS         │    │ • RBAC        │
│ • LWS         │    │               │    │ • Tiers       │
│ • Grafana     │    │               │    │ • Storage     │
└───────────────┘    └───────────────┘    └───────────────┘
```

## Prerequisites

- OpenShift cluster (4.14+)
- ArgoCD installed and configured
- Cluster admin access
- Git repository for storing configurations

## Quick Start

### 1. Fork/Clone Repository

```bash
# Clone the Fusion-AI repository
git clone https://github.com/your-org/Fusion-AI.git
cd Fusion-AI
```

### 2. Update Git Repository URLs

Edit the ArgoCD application files to point to your Git repository:

```bash
# Update repoURL in all application files
sed -i 's|https://github.com/your-org/Fusion-AI.git|https://github.com/YOUR_ORG/Fusion-AI.git|g' \
  quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/*.yaml
```

### 3. Deploy ArgoCD Project and Applications

```bash
# Apply all ArgoCD resources
oc apply -k quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/

# Or apply individually
oc apply -f quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/appproject.yaml
oc apply -f quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/01-maas-operators-application.yaml
oc apply -f quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/02-maas-platform-application.yaml
oc apply -f quickstarts/model-as-a-service/examples/maas-gitops-deployment/argocd/03-maas-runtime-application.yaml
```

### 4. Monitor Deployment

```bash
# Watch ArgoCD applications
argocd app list | grep maas

# Get detailed status
argocd app get maas-operators
argocd app get maas-platform
argocd app get maas-runtime

# Watch operator installation
watch oc get csv -A

# Check DataScienceCluster
oc get datasciencecluster default-dsc -o yaml

# Check runtime resources
oc get all -n maas-models
```

## Deployment Phases

### Phase 1: Operator Subscriptions (Wave 0)

**Application**: `maas-operators`  
**Chart**: `quickstarts/model-as-a-service/deploy/maas-operators`

Installs operator subscriptions:
- Red Hat OpenShift AI (RHOAI)
- Red Hat Connectivity Link (Kuadrant)
- Cert-manager Operator
- Leader Worker Set Operator
- Grafana Operator

**Wait Conditions**:
- All operator CSVs reach `Succeeded` state
- CRDs are available (DataScienceCluster, Kuadrant, etc.)

### Phase 2: Platform Setup (Wave 10)

**Application**: `maas-platform`  
**Chart**: `quickstarts/model-as-a-service/deploy/maas-platform`

Creates operator instances:
- DataScienceCluster (DSC) with KServe, Workbenches, Pipelines, etc.
- Kuadrant instance for API management
- LeaderWorkerSet operator instance

**Wait Conditions**:
- DataScienceCluster reaches `Ready` condition
- All DSC components are healthy

### Phase 3: Runtime Resources (Wave 20)

**Application**: `maas-runtime`  
**Chart**: `quickstarts/model-as-a-service/deploy/maas-runtime`

Deploys runtime infrastructure:
- Gateway for model service routing
- Model Registry for ML model versioning
- RBAC roles and service accounts
- Tier groups for resource allocation
- Workbench storage configuration

## Configuration

### Customizing Operator Versions

Edit `01-maas-operators-application.yaml`:

```yaml
spec:
  source:
    helm:
      values: |
        operators:
          openshiftAI:
            channel: stable-3.x  # Change channel
            installPlanApproval: Manual  # For production
```

### Customizing DataScienceCluster

Edit `02-maas-platform-application.yaml`:

```yaml
spec:
  source:
    helm:
      values: |
        dataScienceCluster:
          components:
            kserve:
              managementState: Managed
            modelmeshserving:
              managementState: Removed  # Disable ModelMesh
```

### Customizing Runtime Configuration

Edit `03-maas-runtime-application.yaml`:

```yaml
spec:
  source:
    helm:
      values: |
        gateway:
          enabled: true
        modelRegistry:
          enabled: true
        tierGroups:
          groups:
            - name: premium
              priority: 100
              resources:
                requests:
                  nvidia.com/gpu: "4"  # Increase GPU allocation
```

## Sync Waves Explained

Sync waves ensure proper deployment order:

| Wave | Application | Purpose | Dependencies |
|------|-------------|---------|--------------|
| -1 | AppProject | Create project first | None |
| 0 | maas-operators | Install operators | AppProject |
| 10 | maas-platform | Create operator instances | Operators ready |
| 20 | maas-runtime | Deploy runtime resources | DSC ready |

ArgoCD waits for each wave to be healthy before proceeding to the next.

## Verification

### Check Application Status

```bash
# All applications should be Healthy and Synced
argocd app list | grep maas

# Expected output:
# maas-operators    default    https://kubernetes.default.svc    Synced    Healthy
# maas-platform     default    https://kubernetes.default.svc    Synced    Healthy
# maas-runtime      maas-models https://kubernetes.default.svc   Synced    Healthy
```

### Verify Operators

```bash
# All CSVs should be in Succeeded phase
oc get csv -A | grep -E 'rhods-operator|kuadrant|cert-manager|leader-worker-set|grafana'

# Check operator pods
oc get pods -n redhat-ods-operator
oc get pods -n kuadrant-system
oc get pods -n cert-manager-operator
```

### Verify Platform

```bash
# DataScienceCluster should be Ready
oc get datasciencecluster default-dsc

# Check DSC components
oc get pods -n redhat-ods-applications
oc get pods -n redhat-ods-monitoring

# Verify Kuadrant
oc get kuadrant kuadrant -n kuadrant-system
```

### Verify Runtime

```bash
# Check runtime namespace
oc get all -n maas-models

# Verify Gateway
oc get gateway maas-gateway -n maas-models

# Verify Model Registry
oc get modelregistry -n rhoai-model-registries

# Check RBAC
oc get sa,role,rolebinding -n maas-models
```

## Troubleshooting

### Application Not Syncing

```bash
# Check application details
argocd app get maas-operators --show-operation

# View sync errors
oc describe application maas-operators -n argocd

# Force sync
argocd app sync maas-operators --force
```

### Operator Installation Stuck

```bash
# Check subscription status
oc describe subscription rhods-operator -n redhat-ods-operator

# Check install plan
oc get installplan -n redhat-ods-operator

# View operator logs
oc logs -n redhat-ods-operator -l name=rhods-operator --tail=100
```

### DataScienceCluster Not Ready

```bash
# Check DSC status
oc get datasciencecluster default-dsc -o yaml

# View DSC events
oc get events -n redhat-ods-operator --sort-by='.lastTimestamp'

# Check component pods
oc get pods -n redhat-ods-applications
```

### Wave Timing Issues

If applications sync too quickly before dependencies are ready:

```bash
# Increase sync wave gaps
# Edit application annotations:
argocd.argoproj.io/sync-wave: "0"   # maas-operators
argocd.argoproj.io/sync-wave: "20"  # maas-platform (was 10)
argocd.argoproj.io/sync-wave: "40"  # maas-runtime (was 20)
```

## Advanced Configuration

### Using External Values Files

Instead of inline values, reference external files:

```yaml
spec:
  source:
    helm:
      valueFiles:
        - values.yaml
        - ../../examples/Fusion-Agentic-Assistance-Platform/values.yaml
```

### Sealed Secrets for Sensitive Data

For production, use Sealed Secrets or External Secrets:

```bash
# Create sealed secret for Keycloak passwords
kubectl create secret generic keycloak-passwords \
  --from-literal=admin-password=<password> \
  --from-literal=user-password=<password> \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > sealed-keycloak-passwords.yaml
```

### Multi-Cluster Deployment

Use ApplicationSet for deploying to multiple clusters:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: maas-platform-set
spec:
  generators:
    - list:
        elements:
          - cluster: prod-east
            url: https://prod-east.example.com
          - cluster: prod-west
            url: https://prod-west.example.com
  template:
    metadata:
      name: '{{cluster}}-maas-operators'
    spec:
      project: maas-platform
      source:
        repoURL: https://github.com/your-org/Fusion-AI.git
        path: quickstarts/model-as-a-service/deploy/maas-operators
      destination:
        server: '{{url}}'
```

## Comparison with Script Installation

| Aspect | Script (`install-runtime.sh`) | GitOps (ArgoCD) |
|--------|-------------------------------|-----------------|
| **Deployment** | Manual execution | Automated, declarative |
| **State Management** | No tracking | Git as source of truth |
| **Rollback** | Manual | Automatic via Git |
| **Drift Detection** | None | Automatic reconciliation |
| **Multi-Cluster** | Run script per cluster | ApplicationSet |
| **Audit Trail** | Script logs | Git history |
| **Secrets** | Environment variables | Sealed Secrets/External Secrets |
| **Updates** | Re-run script | Git commit + auto-sync |

## Best Practices

1. **Use Git Branches**: Separate branches for dev/staging/prod
2. **Pin Versions**: Use specific operator channels and CSVs
3. **Manual Approval**: Set `installPlanApproval: Manual` in production
4. **Health Checks**: Enable and monitor application health
5. **Sync Waves**: Maintain proper wave gaps for dependencies
6. **Secrets Management**: Use Sealed Secrets or External Secrets Operator
7. **RBAC**: Restrict ArgoCD project permissions appropriately
8. **Monitoring**: Set up alerts for application sync failures
9. **Documentation**: Keep values and configurations documented in Git
10. **Testing**: Test changes in dev environment before production

## Next Steps

After successful deployment:

1. **Deploy Models**: Use the model service chart to deploy LLMs
   ```bash
   oc apply -f quickstarts/model-as-a-service/examples/Fusion-Agentic-Assistance-Platform/models/
   ```

2. **Configure Model Registry**: Add models to the registry
   - See: `docs/02-model-catalog-and-registry/ADDING_MODELS_TO_REGISTRY.md`

3. **Set Up Monitoring**: Configure Grafana dashboards
   ```bash
   oc get route -n grafana grafana-route
   ```

4. **Deploy Applications**: Deploy your AI applications
   - See: `fusion-AgenticAssistanceSampleApp/`

## Related Documentation

- [MaaS Operators Guide](../../docs/01-setup/MAAS_OPERATORS_GUIDE.md)
- [MaaS Platform Customization](../../docs/01-setup/MAAS_PLATFORM_CUSTOMIZATION_GUIDE.md)
- [MaaS Runtime Customization](../../docs/01-setup/MAAS_RUNTIME_CUSTOMIZATION_GUIDE.md)
- [Deployment Order](../../docs/01-setup/DEPLOYMENT_ORDER.md)
- [Getting Started](../../docs/GETTING_STARTED.md)

## Support

For issues or questions:
- Check the troubleshooting section above
- Review ArgoCD application logs
- Consult the related documentation
- Open an issue in the repository

## License

This deployment configuration is part of the Fusion-AI project.