## Project Goals

- **Ready-to-use configurations** with step-by-step guides for new OpenShift Virtualization adopters
- **Digestible examples** for functional validation with minimal OpenShift experience required
- **Progressive complexity** from basic setups to advanced networking and storage configurations
- **Comprehensive coverage** of all OpenShift Virtualization topics using bite-sized approaches
- **Multiple automation strategies** including Ansible, Helm, Kustomize, GitOps, and Hashicorp tools

## Project Structure

### Raw Manifests (`manifests/`)
Raw Kubernetes YAML manifests organized by component type that serve as building blocks for all other deployment methods.

- `network/` - Networking configurations including overlay, SR-IOV, bridges, and UDN
- `storage/` - Storage solutions for LVM, NFS, iSCSI, and hostpath
- `operators/` - Platform operators for CNV, NMState, and LVM
- `security/` - RBAC, SCCs, and network policies
- `virtual-machines/` - VM definitions organized by operating system
- `templates/` - Reusable VM and storage templates
- `namespaces/` - Namespace definitions
- `system-images/` - Custom image configurations

### Automation (`automation/`)

- `kustomize/` - Environment-specific overlays and reusable components built on raw manifests
- `helm/` - Templated charts for parameterized deployments
- `ansible/` - End-to-end automation and orchestration with environment-specific inventories
- `gitops/` - Declarative deployment configurations for ArgoCD and Flux

### Documentation (`docs/`)
Comprehensive documentation covering architecture, tutorials, and troubleshooting.

- `architecture/` - System design and component relationships
- `tutorials/` - Step-by-step deployment guides
- `troubleshooting/` - Common issues and solutions
