# Agent Skills

A collection of reusable skills for AI coding agents.

## Skills

### [credentials-request-audit](./credentials-request-audit/SKILL.md)

Audits cloud provider permissions in OpenShift CredentialsRequest manifests. Traces code paths in provider controller source, filters for OpenShift's operational context, and validates findings with runtime evidence. Supports AWS, GCP, Azure, and other providers. Designed to minimize IAM permissions while avoiding false removals that could break customer workloads.
