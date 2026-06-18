# Agent Skills

A collection of reusable skills for AI coding agents.

## Skills

### [credentials-request-audit](./credentials-request-audit/SKILL.md)

Audits cloud provider permissions in OpenShift CredentialsRequest manifests. Traces code paths in provider controller source, filters for OpenShift's operational context, and validates findings with runtime evidence. Supports AWS, GCP, Azure, and other providers. Designed to minimize IAM permissions while avoiding false removals that could break customer workloads.

### [openshell](./openshell/SKILL.md) *(work in progress — not yet tested)*

Manages the full [OpenShell](https://github.com/NVIDIA/OpenShell) sandbox lifecycle — initial setup, gateway management, provider configuration, sandbox creation/connection/deletion, and policy iteration. Works across macOS/Linux, Podman/Docker, and any LLM provider (Vertex AI, Anthropic API, OpenAI, etc.). This skill is still under development and has not been validated end-to-end.
