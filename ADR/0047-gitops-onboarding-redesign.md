# 47. GitOps Onboarding Redesign

Date: 2024-12-21

## Status

Proposed

## Context

Currently, Konflux offers two distinct onboarding paths that create friction and complexity for developers:

### UI Onboarding
- Very easy to use with good validation and feedback
- Handles secrets, build config, environments through web interface
- Provides immediate feedback but doesn't align with GitOps principles

### GitOps Onboarding
- Requires manually authoring YAML across multiple files
- Validation happens during GitLab CI runs, creating tension between Konflux and GitLab CI workflows
- Secrets must still be created via UI, breaking the GitOps flow
- Onboarding is slow and often requires multiple failed PRs to converge due to trial-and-error approach

### Current Pain Points
1. **Fragmented experience**: Developers must context-switch between UI and GitOps approaches
2. **Validation delays**: Schema validation only occurs during CI runs, not locally
3. **Error-prone process**: Manual YAML authoring leads to frequent failures
4. **Secrets inconsistency**: Secrets cannot be managed declaratively through Git
5. **Monorepo complexity**: Current GitOps monorepo for tenant configuration creates CODEOWNERS management overhead
6. **UI improvement limitations**: Despite continuous improvements to the UI, many users still choose GitOps onboarding for its inherent disaster recovery and change management benefits, leaving them unable to benefit from UI enhancements

## Decision

Redesign the onboarding experience to converge UI and GitOps into a single coherent, Git-centric flow through the following architectural changes:

### Alternative Approaches Considered

**Improving the existing UI** was considered but rejected because:
- Despite continuous UI improvements, many users still choose GitOps onboarding for its inherent disaster recovery and change management benefits
- UI enhancements don't address the fundamental issue of maintaining two separate onboarding flows
- Users who prefer GitOps principles cannot benefit from UI improvements, creating a permanent split in the user experience

**Chosen approach** replaces both current flows with a unified Git-centric solution that preserves GitOps benefits while providing the usability improvements users expect.

### Core Shift
- **Replace UI configuration** with a VS Code plugin for onboarding and editing
- **Use local schema validation**, linting, and previews to reduce onboarding errors
- **Use Git as the single source of truth** for applications, components and secrets
- **Preserve the UI for read-only activities** for monitoring, logs, build inspection, and metrics
- **Commit to single onboarding flow**: UI-based onboarding will be completely deprecated with no fallback option to minimize maintenance overhead and deliver optimal user outcomes

### IDE Selection Rationale
VS Code was chosen as the primary IDE target because:
- **Wide adoption**: VS Code is extensively used across development teams
- **Robust plugin ecosystem**: Provides comprehensive APIs for linting, schema validation, and custom tooling
- **Advanced visualization capabilities**: Plugin system supports rich visualizations and can run full, sandboxed React-based applications within webview windows
- **Extensibility**: Enables building sophisticated onboarding wizards and configuration interfaces directly within the familiar IDE environment

### Component Architecture

| Component | Role |
|-----------|------|
| VS Code Plugin | Main interface for onboarding and configuration |
| Git Repos | Source of truth for Konflux config |
| Vault | Preferred secrets backend |
| Kubernetes Secrets | Alternative backend for secrets |
| External Secrets Operator (ESO) | Delivery of secrets to Kubernetes |
| Konflux UI | Monitoring and runtime insight only |
| Fleet Registry | Service discovery and instance metadata for multi-instance deployments |

### VS Code Plugin Design

**Core Features:**
- Forms and wizards to scaffold and edit YAML objects
- Built-in schema-aware YAML editor
- Linting and validation against Konflux schemas
- Integration with local secret store session (Vault or Kubernetes)
- Git-aware diff view and PR-ready commit generation
- Optional: preview pipeline graph / dependency resolution

**Secrets UX:**
- Form-based secret creation (with validation, TTLs, metadata)
- Generated secret definitions point to secret store paths via ESO
- Secrets created in the secret store configured for that instance of Konflux
- Initially supported stores: Vault and Kubernetes Secrets
- Optional: audit log stored in `.konflux/` for local tracking

**GitOps Flow:**
1. Developer opens repo in VS Code
2. Launches Konflux plugin
3. Uses wizard to configure new component/build
4. Adds secrets as needed via plugin form
5. Plugin generates YAML and commits to Git
6. Developer pushes PR
7. CI runs lightweight schema validation (should pass if plugin was used)

### Multi-Repo GitOps Model

**Tenant Onboarding via Repo Registration:**
- Konflux will support multiple GitOps repositories, each representing an independent tenant
- Each registered GitOps repo is treated as a distinct tenant with its own Kubernetes namespace
- Teams register GitOps repo via VS Code plugin, CLI (`konflux register`), or UI
- Konflux provisions a new Kubernetes namespace and runs initial validation
- Decouples team onboarding and eliminates complex CODEOWNERS management

### Secrets Management Integration

**Core Capability:**
- VS Code plugin enables declarative secrets creation directly from the IDE
- Secrets are managed through Git configuration with secure backend integration
- Generated secret definitions use External Secrets Operator (ESO) to reference backend stores

**Supported Backends:**
- **Kubernetes Secrets**: Native Kubernetes secret storage
- **Vault**: HashiCorp Vault integration for enhanced security features
- Future backends can be added as needed

**Security Principles:**
- Plugin only creates secrets, does not read or list existing secrets
- All secret operations are scoped and audited
- Optional: CLI fallback (`konflux secrets create`)

### Fleet Awareness

**Multi-Instance Discovery:**
When a user's organization operates multiple Konflux instances, the VS Code plugin should provide fleet-wide awareness and intelligent instance selection. Fleet metadata is simple metadata with a description of the clusters capabilities and limitations (text), plus an indicator if the cluster has capacity for new tenants namespace. Each Konflux instance should have a complete copy of the fleet metadata.

**Key Capabilities:**
- **Instance Discovery**: Plugin can discover all Konflux instances within the organization's fleet
- **Unified Authentication**: Authentication to one instance enables discovery and basic information access across the fleet
- **Instance Selection Guidance**: Plugin helps users choose the optimal instance for new tenant repo onboarding based on:
  - Security zones and compliance requirements
  - Instance capacity and resource availability
  - Geographic or network proximity
  - Operational policies (e.g., dev/staging/prod instance separation)

**User Experience:**
- Plugin presents a unified view of all available instances
- Provides recommendations for instance selection during tenant onboarding
- Displays instance health, capacity, and policy information
- Enables seamless switching between instances for different projects

**Implementation Considerations:**
- Fleet registry or service discovery mechanism for instance enumeration
- Federated authentication system across instances
- Instance metadata API for capacity and policy information
- Security controls to prevent unauthorized cross-instance access

### CI/Validation Design

- Introduce a Konflux CLI validator (or GitHub Action) for local + CI use
- Validates schema, references, build logic statically
- Same validator embedded in VS Code plugin
- Optional: Plugin lints against live cluster (e.g., available build agents)

### UI Changes

**Preserve:**
- Dashboard
- Build and release logs
- Live environment status
- Observability / metrics

**Remove/Demote:**
- UI forms for configuring components
- UI secrets creation

## Consequences

### Positive Consequences

1. **Unified Developer Experience**: Single Git-centric flow eliminates context switching between UI and GitOps
2. **Faster Onboarding**: Local validation reduces time-to-merge for onboarding PRs
3. **Declarative Secrets Management**: Secrets can be managed through Git with proper security controls
4. **Improved Team Autonomy**: Multi-repo model allows teams to own their GitOps repositories
5. **Better IDE Integration**: Leverages familiar VS Code tooling and workflows
6. **Maintained GitOps Benefits**: Preserves disaster recovery, change history, and automation capabilities
7. **Enterprise Fleet Management**: Fleet awareness enables intelligent instance selection and centralized management for large organizations

### Negative Consequences

1. **Tool Dependency**: Requires VS Code and plugin installation for optimal experience
2. **Learning Curve**: Developers must adapt to new plugin-based workflow
3. **Development Overhead**: Requires building and maintaining VS Code plugin
4. **Secrets Backend Infrastructure**: Requires secure secrets backend infrastructure (Kubernetes Secrets or Vault)
5. **Migration Complexity**: Existing users must migrate from current UI/GitOps hybrid approach

### Risks and Mitigations

1. **Plugin Adoption**: Risk of low adoption if plugin is complex or unreliable
   - Mitigation: Provide CLI fallback and comprehensive documentation
2. **Secret Store Integration**: Vault integration may face authentication/authorization challenges
   - Mitigation: Support multiple secret backends and provide clear fallback options
3. **Schema Validation**: Local validation may diverge from server-side validation
   - Mitigation: Use same validation logic in plugin and CI
4. **No Rollback Option**: Once UI-based onboarding is deprecated, there is no fallback to the previous approach
   - Mitigation: Ensure thorough testing and gradual rollout with comprehensive user training and documentation

### Open Questions

1. **Secrets Backend Priority**: Should there be a preferred backend order, or should instance configuration determine the default?
2. **Fleet Discovery Mechanism**: How should instances discover and register with each other in the fleet?
3. **Cross-Instance Authentication**: What authentication model should be used for fleet-wide access?
4. **Instance Selection Criteria**: What specific metrics and policies should guide automatic instance recommendations?
5. **Fleet Management Scope**: Should fleet awareness be limited to discovery and selection, or include cross-instance management capabilities?

### Migration Path

1. **Phase 1**: Develop VS Code plugin with local validation
2. **Phase 2**: Implement multi-repo GitOps support
3. **Phase 3**: Add secrets management integration (Kubernetes and Vault backends)
4. **Phase 4**: Migrate existing users and fully deprecate UI configuration flows
5. **Phase 5**: Remove deprecated UI configuration components

**Note**: Once the GitOps approach is implemented, the UI-based onboarding will be completely deprecated with no fallback option. This decision prioritizes streamlining to a single onboarding flow to minimize maintenance overhead and deliver the best user outcomes.

## References

- [ADR-0022 Secret Management for User Workloads](./0022-secret-mgmt-for-user-workloads.md)
- [External Secrets Operator](https://external-secrets.io/) 