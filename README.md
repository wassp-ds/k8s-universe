# `Kubernetes` Cluster Design

Secure `Kubernetes` cluster design document for production-ready infrastructure.

## Table of Contents

*   [Goal](#goal)
*   [Documentation](#documentation)
*   [Architecture](#architecture)
    *   [Infrastructure Layout](#infrastructure-layout)
    *   [Network Architecture](#network-architecture)
    *   [Security Boundaries](#security-boundaries)
*   [Format](#format)
    *   [README Requirements](#readme-requirements)
        *   [Service Documentation](#service-documentation)
    *   [Resource Naming](#resource-naming)
        *   [Node Naming](#node-naming)
        *   [Namespace Naming](#namespace-naming)
        *   [Service Naming](#service-naming)
        *   [Pod Security Labels](#pod-security-labels)
    *   [Infrastructure as Code](#infrastructure-as-code)
    *   [`RBAC` and Security](#rbac-and-security)
    *   [Networking and Access Control](#networking-and-access-control)
*   [Core Services](#core-services)
    *   [Private Container Registry](#private-container-registry)
    *   [CI/CD Pipeline](#cicd-pipeline)
    *   [Security Monitoring](#security-monitoring)
    *   [`CVE` Monitoring System](#cve-monitoring-system)
*   [Security Features](#security-features)
    *   [Network Policies](#network-policies)
    *   [Pod Security Standards](#pod-security-standards)
    *   [Attack Simulation Framework](#attack-simulation-framework)
*   [Operations](#operations)
    *   [Deployment Strategy](#deployment-strategy)
    *   [Rollback Procedures](#rollback-procedures)
    *   [Security Gates](#security-gates)

## Goal

Provide a secure, scalable and maintainable `Kubernetes` cluster that follows defense-in-depth principles and enables rapid, secure application development and deployment. The cluster must support development workflows while maintaining production-grade security posture.

The infrastructure must be reproducible, auditable, and follow the principle of least privilege. Every component should have clear security boundaries, proper monitoring, and the ability to simulate real-world attack scenarios for continuous security validation.

Security and operational requirements drive architectural decisions, not convenience. If a feature introduces unnecessary attack surface, it is excluded by default and can be enabled through explicit configuration when needed.

## Documentation

Markdown code must follow [Google's Markdown style guide](https://google.github.io/styleguide/docguide/style.html). All infrastructure components, services, and procedures must be documented with clear examples and security considerations.

Documentation must not duplicate upstream `Kubernetes` documentation but should reference it appropriately. Security configurations and custom implementations require detailed in-house documentation with threat models and mitigation strategies.

## Architecture

### Infrastructure Layout

*   **Control Plane**: `k8s-master-01` - Single master node with `etcd`, API server, scheduler, and controller manager
*   **Worker Nodes**:
    *   `k8s-worker-01` - General workloads and system services
    *   `k8s-worker-02` - Self-hosted GitHub Actions runners and Harbor registry
    *   `k8s-worker-03` - Security tools, monitoring, and application workloads
*   **Network**: `Tailscale` mesh network with secure node-to-node communication
*   **Storage**: Local persistent volumes with backup strategy
*   **CI/CD Requirements**:
    *   Self-hosted GitHub Actions runners for enhanced security and control
    *   Harbor registry with high availability configuration
    *   Persistent storage for Harbor database, `Redis` cache, and image layers
    *   Network policies for runner and registry isolation
    *   Backup strategy for Harbor data, configuration, and image layers
    *   `S3`-compatible storage for Harbor backup and disaster recovery

### Network Architecture

*   **`Tailscale` Tailnet**: Private mesh network for administrative access
*   **Cluster Network**: `CNI` plugin (Calico) with network policy enforcement
*   **Service Mesh**: `Istio` for east-west traffic encryption and observability
*   **Ingress**: `nginx` Ingress Controller with `TLS` termination
*   **DNS**: `CoreDNS` with security hardening and query logging

### Security Boundaries

*   **Administrative Access**: Only through `Tailscale` tail net
*   **Inter-node Communication**: Encrypted via `Tailscale` and `Kubernetes` `TLS`
*   **Pod-to-Pod**: Network policies with default deny-all
*   **External Access**: Ingress controller with `WAF` and rate limiting
*   **Registry Access**: Private registry with pull secrets and `RBAC`

## Format

### README Requirements

Every service deployment must include a `README.md` with:

*   Table of Contents
*   Service Description and Purpose
*   Security Model and Threat Assessment
*   Installation and Configuration
*   `RBAC` Requirements (if custom roles needed)
*   Network Policy Requirements
*   Monitoring and Alerting Configuration
*   Troubleshooting Guide
*   Security Considerations and Hardening

#### Service Documentation

*   Must document all `RBAC` permissions and justify necessity
*   Must document network policy requirements and traffic flows
*   Must include security scanning results and remediation steps
*   Must document backup and disaster recovery procedures

### Resource Naming

#### Node Naming

*   **Master Node**: `k8s-master-NN` (e.g., `k8s-master-01`)
*   **Worker Nodes**: `k8s-worker-NN` (e.g., `k8s-worker-01`)
*   **Taints and Labels**: Environment and role-based (`node-role.kubernetes.io/worker`)

#### Namespace Naming

*   **System Namespaces**: `kube-system`, `kube-public`, `kube-node-lease`
*   **Security Namespaces**: `security-*` (e.g., `security-monitoring`, `security-scanning`)
*   **Application Namespaces**: `app-*` (e.g., `app-frontend`, `app-backend`)
*   **CI/CD Namespaces**: `cicd-*` (e.g., `cicd-runners`, `cicd-registry`)
*   **Infrastructure Namespaces**: `infra-*` (e.g., `infra-networking`, `infra-storage`)

#### Service Naming

*   **Internal Services**: `svc-NAME` (e.g., `svc-database`, `svc-api`)
*   **External Services**: `ext-NAME` (e.g., `ext-loadbalancer`)
*   **Security Services**: `sec-NAME` (e.g., `sec-scanner`, `sec-monitor`)
*   **Administrative Services**: `admin-NAME` (e.g., `admin-dashboard`)

#### Pod Security Labels

Security context labels based on trust and privilege level:

*   **Critical**: `security.k8s.io/level=critical` - System components, security tools
*   **High**: `security.k8s.io/level=high` - Infrastructure services, CI/CD
*   **Medium**: `security.k8s.io/level=medium` - Application services
*   **Low**: `security.k8s.io/level=low` - Development and testing workloads
*   **Untrusted**: `security.k8s.io/level=untrusted` - External or experimental workloads

### Infrastructure as Code

#### Repository Strategy

**Mono repo Approach**: Single repository with logical separation through folder structure to enable atomic changes, simplified management, and easier coordination between infrastructure and applications.

**Repository Organization**: The repository follows a clear hierarchical structure with separate directories for infrastructure code (`Terraform`, `Ansible`, `Helm` charts), applications (container definitions and `Kubernetes` manifests), `GitOps` configurations (`ArgoCD` applications and overlays), documentation, and automation scripts. This organization enables path-based CI/CD triggers while maintaining clear separation of concerns.

#### Mono repo Advantages for Personal Projects

**Atomic Changes**:
- Infrastructure and application changes in single commit
- No cross-repo dependency management complexity
- Easier rollback of complete feature sets

**Simplified Management**:
- Single repository to backup and maintain
- One set of CI/CD pipelines to manage
- Unified issue tracking and documentation

**Better Coordination**:
- Changes to infrastructure immediately reflected in applications
- No version skew between infrastructure and app definitions
- Easier to track dependencies and impacts

**Single Source of Truth**:
- All project knowledge in one place
- Easier to on-board (yourself after breaks!)
- Complete project history in one timeline

#### `Terraform` Organization

**State Management**:
*   Remote state backend with encryption at rest
*   State locking with `DynamoDB` or equivalent
*   Separate state files per environment and component
*   State file backup and disaster recovery procedures

**Module Structure**:
*   **Compute Module** (`terraform/modules/compute/`):
    *   `VM` provisioning with security baselines
    *   Instance hardening configurations
    *   `Tailscale` agent installation and configuration
    *   Node labeling and tainting for `Kubernetes`
*   **Networking Module** (`terraform/modules/networking/`):
    *   `VPC`/subnet configuration with security groups
    *   `Tailscale` subnet routing and `ACL` configuration
    *   Load balancer and ingress configurations
    *   Network security policies and firewall rules
*   **Security Module** (`terraform/modules/security/`):
    *   Certificate management and `PKI` setup
    *   Secret management infrastructure
    *   Identity and access management
    *   Security monitoring infrastructure
*   **Storage Module** (`terraform/modules/storage/`):
    *   Persistent volume provisioning
    *   Backup infrastructure and policies
    *   Encryption at rest configuration
    *   Storage class definitions

**File Naming Conventions**:
*   `main.tf` - Primary resource definitions
*   `variables.tf` - Input variable declarations with validation
*   `outputs.tf` - Output value definitions
*   `versions.tf` - Provider version constraints
*   `security.tf` - Security-specific resources and policies
*   `data.tf` - Data source definitions
*   `locals.tf` - Local value definitions

#### `Ansible` Organization

**Inventory Management**:
*   Dynamic inventory with cloud provider APIs
*   Environment-specific inventory files
*   Host grouping by role and security zone
*   Variable precedence documentation

**Role Structure**:
*   **Base Hardening** (`roles/base-hardening/`):
    *   OS security baseline (`CIS` benchmarks)
    *   Package management and updates
    *   User and SSH configuration
    *   Audit logging and monitoring agents
*   `Kubernetes` **Installation** (`roles/kubernetes/`):
    *   `kubeadm` cluster bootstrap
    *   `CNI` plugin installation and configuration
    *   `kubelet` security configuration
    *   Certificate rotation setup
*   **Security Tools** (`roles/security-tools/`):
    *   `Falco` installation and rules
    *   Security scanner deployment
    *   Compliance checking tools
    *   Incident response tools
*   **Monitoring** (`roles/monitoring/`):
    *   Prometheus and exporters
    *   Log forwarding configuration
    *   Alert manager setup
    *   Dashboard deployment

**Playbook Organization**:
*   `site.yml` - Main orchestration playbook
*   `cluster-bootstrap.yml` - Initial cluster setup
*   `security-hardening.yml` - Security baseline application
*   `application-deployment.yml` - Application-specific setup
*   `maintenance.yml` - Routine maintenance tasks

**Variable Management**:
*   `group_vars/all.yml` - Global configuration
*   `group_vars/security.yml` - Security-specific variables
*   `group_vars/production.yml` - Environment-specific variables
*   `host_vars/` - Host-specific overrides
*   `Ansible Vault` for sensitive data with key rotation procedures

#### Helm Chart Management

**Chart Organization**:
*   **Infrastructure Charts** (`helm-charts/infrastructure/`):
    *   `ingress-nginx` with security hardening
    *   `cert-manager` with `ACME` and internal `CA`
    *   `external-secrets-operator`
    *   `prometheus-operator` stack
*   **Application Charts** (`helm-charts/applications/`):
    *   `GitHub` with `PostgreSQL` and `Redis`
    *   `ArgoCD` with `RBAC` configuration
    *   Container registry (`Harbor`)
    *   Custom application deployments
*   **Security Charts** (`helm-charts/security/`):
    *   `Falco` with custom rules
    *   Policy engines (`OPA Gatekeeper`)
    *   Vulnerability scanners
    *   Security monitoring stack

**Values File Structure**:
*   `values.yaml` - Default values with secure defaults
*   `values-production.yaml` - Production overrides
*   `values-security.yaml` - Security-specific configurations
*   `secrets.yaml` - Encrypted sensitive values (using helm-secrets)

#### `GitOps` Workflow

**Mono repo `GitOps` Pattern with GitHub Actions**:

**Development Workflow**:
The development process follows a clear sequence where application code changes in the applications directory are paired with corresponding `Kubernetes` manifest updates in the `gitops` directory. Both changes are committed atomically, triggering GitHub Actions workflows based on file path changes. `ArgoCD` monitors the `gitops` directory and automatically syncs detected changes to the cluster.

**Infrastructure Workflow**:
Infrastructure modifications follow a similar pattern where `Terraform` and `Ansible` code updates in the infrastructure directory are accompanied by corresponding `GitOps` manifest changes when needed. The infrastructure workflow validates and applies changes through automated pipelines, while `ArgoCD` handles the synchronization of updated cluster configurations.

**Pipeline Organization**:
GitHub Actions workflows are organized with path-based triggers to optimize execution efficiency. Infrastructure changes trigger `Terraform` planning and security scanning workflows. Application changes initiate build, scan, and container management processes. `GitOps` changes activate manifest validation and policy compliance checks.

**ArgoCD Configuration**:
The cluster uses an App-of-Apps pattern where a root `ArgoCD` application monitors the production cluster directory. Separate `ArgoCD` applications handle infrastructure, applications, and security components. Sync policies are configured for automatic synchronization of non-critical changes while requiring manual approval for infrastructure modifications. Pruning is enabled for applications but disabled for infrastructure components for safety.

**Change Coordination**:
The mono repo approach enables atomic updates where infrastructure and application changes occur in single commits. Dependencies and their consumers can be updated together in the same pull request. Integration tests validate entire stack changes, and rollbacks are simplified through single commit reverts that affect complete features.

#### Security and Validation

**Infrastructure Security Scanning**:
*   `Terraform` security scanning with `Checkov`/`Terrascan`
*   `Ansible` playbook security validation
*   `Helm` chart security scanning with `Kubesec`
*   `Kubernetes` manifest validation with `conftest`

**Secrets Management**:
*   No secrets in version control (pre-commit hooks)
*   `external-secrets-operator` for secret injection from external sources
*   GitHub Actions secrets for `CI/CD` pipeline authentication
*   Sealed Secrets for `GitOps` secret management
*   `Harbor` robot accounts with project-scoped and time-limited access
*   Secret rotation automation and auditing
*   `Harbor webhook` secrets for registry integration
*   `cosign` keyless signing for container images

**State Security**:
*   `Terraform` state encryption and access logging
*   State file integrity checking
*   Backup encryption and retention policies
*   Access control with time-limited credentials

**Validation Pipeline**:
*   Infrastructure code review requirements
*   Automated security policy validation
*   Drift detection and reconciliation
*   Compliance reporting and audit trails

#### Version Control Strategy

**Mono repo Branching Strategy**:

**Primary Branches**:
- **`main`**: Production-ready code, auto-deployed to cluster
- **`develop`**: Integration branch for testing complete features
- **`feature/*`**: Individual feature development branches
- **`hotfix/*`**: Emergency fixes with expedited deployment

**Development Flow**:
1. **Feature Development**: Create `feature/new-monitoring-stack` branch
2. **Atomic Changes**: Update both application code and infrastructure in same branch
3. **Testing**: Validate infrastructure + application changes together
4. **Integration**: Merge to `develop` for end-to-end testing
5. **Production**: Merge to `main` triggers full deployment pipeline

**Tagging Strategy**:
- **Infrastructure Releases**: `infra-v1.2.3` for major infrastructure changes
- **Application Releases**: `app-gitlab-v2.1.0` for specific application versions
- **Full Stack Releases**: `v1.0.0` for coordinated infrastructure + application releases

**Change Management**:
- **Path-Based Pipelines**: Different CI/CD jobs based on what changed
- **Dependency Tracking**: Changes cascade through dependent components
- **Atomic Rollbacks**: Single commit revert for complete feature rollbacks
- **Change Documentation**: Automated changelog generation across all components

**Release Coordination**:
1. **Feature Complete**: All related changes (infra + apps) in single feature branch
2. **Integration Testing**: Validate complete stack in develop branch
3. **Production Release**: Tagged release with comprehensive release notes
4. **Rollback Planning**: Document rollback dependencies and procedures

**Security and Validation**:
- **Pre-commit Hooks**: Security scanning, linting, secret detection
- **Branch Protection**: Required reviews for main branch
- **Pipeline Gates**: Security scans must pass before merge
- **Audit Trail**: Complete change history in single repository timeline

**Backup Strategy**:
- **Repository Mirrors**: Automated backup to external Git provider
- **Release Archives**: Tagged releases archived with artifacts
- **Documentation Backup**: Docs exported to external storage
- **Recovery Procedures**: Complete repository restoration procedures

### `RBAC` and Security

*   **Service Accounts**: Principle of least privilege, namespace-scoped
*   **Cluster Roles**: Minimal necessary permissions, audited regularly
*   **Role Bindings**: Explicit, time-limited where possible
*   **Pod Security**: Admission controllers enforcing security standards

### Networking and Access Control

*   **Default Deny**: All network policies start with default deny-all
*   **Ingress Rules**: Explicit allow rules for required traffic
*   **Egress Rules**: Restricted external access, explicit allow for necessary services
*   **Service Mesh**: `mTLS` for all service-to-service communication

## Core Services

### Private Container Registry

*   **Primary**: Self-hosted Harbor registry for complete control and no vendor lock-in
*   **Namespace**: `cicd-registry`
*   **Implementation**: Harbor with comprehensive security features
*   **Security**:
    *   `Harbor` `RBAC` with project-based access control
    *   Automated vulnerability scanning with `Trivy` integration
    *   Image signing with `cosign` and `Notary v2`
    *   Content trust and image attestation
    *   Malware scanning integration
    *   Cleanup policies for old/vulnerable images
    *   `LDAP`/`OIDC` integration for authentication
    *   Audit logging for all registry operations
*   **High Availability**:
    *   `Redis` for caching and session storage
    *   `PostgreSQL` for metadata storage
    *   Shared storage backend for image layers
*   **Backup**:
    *   Automated backup of Harbor database and configuration
    *   Image layer backup to external storage (`S3`-compatible)
    *   Disaster recovery procedures and testing
*   **Access**: Harbor robot accounts and project-specific access tokens managed via external-secrets operator
*   **Integration**: GitHub Actions authentication via Harbor robot accounts

### CI/CD Pipeline

*   **Platform**: GitHub with GitHub Actions workflows
*   **Container Registry**: Self-hosted Harbor registry (`harbor.k8s.local`)
*   **Components**:
    *   GitHub Actions runners (self-hosted on dedicated worker node)
    *   `Harbor` registry with `Trivy` vulnerability scanning
    *   Integrated security scanning with `GitHub Advanced Security`
    *   `dependabot` for automated dependency updates
*   **Security Gates**:
    *   Static code analysis (`CodeQL`, third-party `SAST`)
    *   Container image scanning (`Trivy` integrated with Harbor)
    *   Infrastructure as Code scanning (`Checkov`, `tfsec`)
    *   Security policy validation (Open Policy Agent)
    *   License compliance scanning
    *   Secrets scanning with GitHub secret scanning
    *   Image signing with `cosign`/`sigstore`
*   **Deployment**: `GitOps` with `ArgoCD` triggered by `GitHub webhooks`
*   **Rollback**: Automated rollback on security policy violations or failed deployments
*   **Registry Integration**: `Harbor webhooks` for vulnerability alerts and policy enforcement

### Security Monitoring

*   **Namespace**: `security-monitoring`
*   **Implementation**:
    *   `Falco` for runtime security monitoring
    *   `Prometheus` and `Grafana` for metrics
    *   `ELK` stack for log aggregation and analysis
    *   `Jaeger` for distributed tracing
*   **Alerting**: `PagerDuty` integration for critical security events

### `CVE` Monitoring System

*   **Namespace**: `security-cve`
*   **Implementation**:
    *   `CVE` database synchronization (`MITRE`, `NVD`)
    *   Automated vulnerability assessment
    *   Risk scoring and prioritization
    *   `Harbor` registry vulnerability scanning integration
    *   `GitHub dependabot` alerts integration for code dependencies
    *   `GitHub Security Advisories` monitoring
*   **Harbor Integration**:
    *   `Trivy` scanner integration for comprehensive vulnerability scanning
    *   Automated policy enforcement based on vulnerability severity
    *   Quarantine policies for high-severity vulnerabilities
    *   `Harbor webhook` integration for real-time vulnerability alerts
*   **GitHub Integration**:
    *   `dependabot` security updates for automated code dependency patching
    *   GitHub Advanced Security for code scanning
    *   Security tab integration for centralized vulnerability management
    *   GitHub Advisory Database integration
*   **Notifications**: Slack/email alerts for high-severity `CVE`s affecting deployed images in `Harbor`
*   **Automation**: GitHub Actions workflows for automated security patching and testing

## Security Features

### Network Policies

*   **Default Deny-All**: Applied to all namespaces
*   **Granular Allow Rules**: Based on service requirements
*   **Namespace Isolation**: Cross-namespace communication requires explicit policy
*   **External Access**: Controlled through ingress policies

### Pod Security Standards

*   **Restricted**: Default for application workloads
*   **Baseline**: For infrastructure services requiring elevated permissions
*   **Privileged**: Only for system components with explicit justification
*   **Admission Controllers**: Enforcing security standards at admission time
*   **GitHub Actions Security**:
    *   Self-hosted runners run with restricted security context
    *   Runner isolation with dedicated namespaces and network policies
    *   Regular runner image updates and vulnerability patching
    *   GitHub App authentication for enhanced security
    *   Encrypted secrets management with GitHub Actions secrets
    *   Audit logging for all runner activities and job executions
*   **Harbor Registry Security**:
    *   Harbor components run with minimal privileges
    *   Database encryption at rest and in transit
    *   Regular Harbor security updates and `CVE` monitoring
    *   Robot account security with time-limited tokens
    *   Two-factor authentication for admin accounts
    *   Comprehensive audit logging for all registry operations

### Attack Simulation Framework

*   **Namespace**: `security-redteam`
*   **Implementation**:
    *   `Kubernetes Goat` for vulnerable application testing
    *   `Kube-hunter` for cluster security assessment
    *   `Kube-bench` for `CIS Kubernetes Benchmark` validation
    *   Custom attack scenarios for specific threat models
*   **Automation**: Scheduled security assessments with reporting
*   **Blue Team**: Automated detection and response validation

## Operations

### Deployment Strategy

*   **Blue-Green Deployments**: For critical services
*   **Canary Deployments**: For application services with gradual rollout
*   **Rolling Updates**: For non-critical services
*   **Security Validation**: All deployments require security gate approval

### Rollback Procedures

*   **Automated Rollback**: Triggered by failed health checks or security violations
*   **Manual Rollback**: `GitOps`-based revert to previous known-good state
*   **Database Rollback**: Coordinated with application rollback procedures
*   **Security Incident**: Emergency rollback procedures for security breaches

### Security Gates

*   **Pre-deployment**:
    *   Container image vulnerability scanning
    *   `Kubernetes` manifest security validation
    *   Network policy validation
    *   `RBAC` permission review
*   **Post-deployment**:
    *   Runtime security monitoring
    *   Performance impact assessment
    *   Security baseline validation
    *   Compliance verification

**Continuous Security**:
*   Daily vulnerability scans of running containers
*   Weekly security policy compliance checks
*   Monthly penetration testing exercises
*   Quarterly security architecture reviews
