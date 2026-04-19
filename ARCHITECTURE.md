# AI Assistant Guidelines & Architecture Notes

## Project Overview
This repository contains a Helm chart designed to deploy the **OpenClaw AI Gateway** and its optional mission control sidecar, **ClawSuite**, natively onto **Red Hat OpenShift**. 

The chart is built specifically for enterprise environments, heavily leaning on OpenShift-native resources, strict security constraints, and on-cluster builds.

## Key Architectural Principles

1. **OpenShift Native First:**
   - **Authentication:** We use the OpenShift OAuth Proxy (`image-registry.openshift-image-registry.svc:5000/openshift/oauth-proxy`) rather than bespoke authentication. Access is gated via Kubernetes Subject Access Review (SAR) strings.
   - **Ingress:** Always use OpenShift `Route` objects (`route.yaml`, `clawsuite-route.yaml`) utilizing edge or reencrypt TLS termination. Do NOT use vanilla Kubernetes `Ingress` objects.
   - **Builds:** For the ClawSuite sidecar, the chart uses an in-cluster `BuildConfig` and `ImageStream` instead of relying solely on pre-built external images. This ensures security compliance and native auditability.

2. **Security & Least Privilege (Strict Constraints):**
   - **No Root:** All containers must run as non-root users (UID 1000/1001) using `runAsNonRoot: true`. Capabilities must be dropped, and privilege escalation is prohibited.
   - **Network Isolation:** The namespace is locked down with a strict `NetworkPolicy` (`networkpolicy.yaml`). It denies ingress from outside the namespace (except from OpenShift routers) and explicitly controls egress (e.g., WAN and CoreDNS).
   - **RBAC:** The `ServiceAccount` is extremely minimal. It only grants the permissions necessary for the OAuth proxy to function (TokenReviews, SubjectAccessReviews) and nothing more.

3. **Data Persistence:**
   - Configurations natively support persistent storage mapped into the gateway containers via `pvc.yaml` and `workspace-pvc.yaml`. Dynamic volume provisioning is managed via `storageClass`.

## Codebase Structure & Conventions

- `Chart.yaml`: Core Helm metadata, dependencies, and application version configurations.
- `values.yaml`: Default configuration. It clearly separates configuration into distinct scopes: `openclaw` (gateway settings), `clawsuite` (dashboard/build settings), `oauthProxy` (auth constraints), and global features.
- `templates/`: Contains all Kubernetes/OpenShift resource manifests.
  - `deployment.yaml`: Defines the single Pod layout, dynamically injecting `openclaw`, `oauth-proxy`, and `clawsuite` (if enabled) as a unified multi-container workload sharing localhost networking.
  - `_helpers.tpl`: Contains standard Helm template functions for labels, names, and checksums.
- `files/`: Contains raw files packaged and utilized by Helm.
  - `Dockerfile.clawsuite`: Used by the `BuildConfig` to build the ClawSuite sidecar on-cluster. **Important:** It includes custom runtime patches via `git apply` to modify upstream dependencies (e.g., fixing TanStack versions, relaxing CSPs for OpenShift, and injecting an OAuth trust middleware).

## Guidelines for LLM Assistants

- **Modifying ClawSuite:** When adding or altering functionality related to ClawSuite, check `files/Dockerfile.clawsuite`. We modify upstream behavior through injected git patches during the image build rather than maintaining a hard fork.
- **Adding Resources:** Do not introduce objects that clash with Red Hat Enterprise security standards. Never add permissive RBAC roles, elevated privileges, or `NodePort`/`LoadBalancer` services.
- **Handling Secrets:** Never hardcode credentials. Use the predefined `.Values.secrets` pattern to inject custom secrets (e.g., `OPENAI_API_KEY`) safely through Kubernetes `Secret` resources and environment variable references (`valueFrom.secretKeyRef`).
- **Template Syntax:** Follow standard Helm templating best practices. Use `include` for helpers, `nindent` for proper yaml formatting, and ensure resource names are derived from `include "openclaw.fullname" .`.
