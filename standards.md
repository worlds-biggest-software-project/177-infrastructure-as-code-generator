# Standards & API Reference

> Project: Infrastructure as Code Generator · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Relevant because AI-generated IaC must produce configurations that comply with an organisation's ISMS. Generated resources (IAM policies, network ACLs, encryption settings) should be auditable against ISO 27001 controls. Many enterprise buyers require ISO 27001 certification from SaaS vendors hosting generation infrastructure.

**ISO/IEC 27017:2015 — Cloud Services Security Controls**
- URL: https://www.iso.org/standard/43757.html
- Extends ISO 27001 with cloud-specific controls directly relevant to the resources being provisioned (virtual machines, storage buckets, managed databases). AI generators should default to ISO 27017-aligned configurations (shared responsibility model, data residency controls).

**ISO/IEC 38505-1:2017 — Data Governance**
- URL: https://www.iso.org/standard/56639.html
- Applies when generated IaC provisions data stores; generators should be aware of data governance requirements (retention, classification, access controls) that affect how storage resources are configured.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Baseline HTTP standard applicable to IaC generators exposing REST APIs for code generation, plan retrieval, and deployment triggering. All API endpoints should conform to standard HTTP method semantics.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Relevant to API response design for navigation between related resources (e.g., linking generated code artefacts to their associated plan, state, and policy check results).

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Essential for authenticating generator API calls to cloud providers (AWS, Azure, GCP) and for securing the generator's own API. All provider credential flows use OAuth 2.0 or derivative standards.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Used for API authentication tokens in generator platforms and for cloud provider temporary credential exchange (AWS STS tokens, Azure AD tokens, GCP service account tokens).

**RFC 9110 — HTTP Semantics (HTTP/2 and HTTP/3)**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- The current authoritative HTTP standard; supersedes RFC 7231 for modern API implementations. Relevant for streaming plan output and long-running generation operations.

---

### Data Model & API Specifications

**HCL (HashiCorp Configuration Language) Specification**
- URL: https://github.com/hashicorp/hcl/blob/main/spec.md
- The de-facto standard syntax for Terraform and OpenTofu. Any IaC generator targeting the dominant market must produce valid HCL. The spec defines the structural and expression syntax, native functions, and type system. HCL is the minimum output format required for market viability (76% practitioner share, CNCF 2024).

**OpenTofu Provider Protocol**
- URL: https://opentofu.org/docs/internals/provider-protocol/
- Defines the gRPC-based protocol between the OpenTofu core and provider plugins. Relevant for generators that introspect provider schemas at generation time to produce valid resource configurations without hallucination.

**Terraform Plugin Framework (SDK)**
- URL: https://developer.hashicorp.com/terraform/plugin/framework
- The current Go SDK for building Terraform/OpenTofu providers. Relevant when a generator needs to create or extend providers, and for understanding provider schema structures used to validate generated code.

**OpenAPI 3.x Specification (OAS)**
- URL: https://spec.openapis.org/oas/v3.1.0
- Used by HashiCorp's `terraform-plugin-codegen-openapi` to generate Terraform provider code from service API specs. An IaC generator can leverage OAS to ingest a service's API definition and automatically produce a corresponding Terraform/Pulumi provider or resource configuration.

**OpenAPI Provider Spec Generator (HashiCorp)**
- URL: https://developer.hashicorp.com/terraform/plugin/code-generation/openapi-generator
- CLI tool that transforms an OpenAPI 3.x specification into a Provider Code Specification for automated Terraform provider generation. Directly relevant for generators that need to support custom cloud services or new provider resources.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used to define and validate the structure of CloudFormation templates, Helm chart `values.yaml` files, and Kubernetes manifests. AI generators targeting these formats should validate output against published JSON schemas before returning results.

**AWS CloudFormation Resource Specification / CloudFormation Registry**
- URL: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-resource-specification.html
- Defines the schema for every CloudFormation resource type. Generators producing CloudFormation YAML/JSON output should validate against this registry to ensure attribute names and allowed values are correct.

**Kubernetes API Specification (OpenAPI)**
- URL: https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- The Kubernetes API is fully described by an OpenAPI 2.0/3.0 specification. Generators producing Kubernetes manifests or Helm charts must align with this spec for resource definitions, API versions, and field validation.

---

### Security & Authentication Standards

**Open Policy Agent (OPA) / Rego**
- URL: https://www.openpolicyagent.org/docs/latest/policy-language/
- The dominant policy-as-code framework for IaC governance. Generators should either produce Rego policies alongside IaC code or integrate with OPA to validate generated configurations against organisational policies before delivery. Used by Spacelift, env0, Scalr, and Brainboard.

**NIST SP 800-53 Rev 5 — Security and Privacy Controls for Federal Information Systems**
- URL: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
- The reference control framework for US federal compliance (FedRAMP). Enterprise IaC generators should map generated resource configurations to SP 800-53 controls and surface compliance gaps in generation output.

**NIST SP 800-190 — Application Container Security Guide**
- URL: https://csrc.nist.gov/pubs/sp/800/190/final
- Applicable when generators produce Kubernetes manifests, Helm charts, or container-related Terraform resources (ECS, EKS, AKS). Generators should embed SP 800-190-aligned defaults (image scanning, registry access controls, orchestrator RBAC).

**OSCAL (Open Security Controls Assessment Language)**
- URL: https://pages.nist.gov/OSCAL/
- NIST's machine-readable framework for expressing security controls and compliance evidence. An advanced IaC generator could emit OSCAL assessment artefacts alongside generated code, providing auditors with traceable links from infrastructure resources to compliance controls. Emerging but underserved by current tools.

**CIS Benchmarks — AWS/Azure/GCP Foundations**
- URL: https://www.cisecurity.org/cis-benchmarks
- Centre for Internet Security hardening guides are the most widely adopted cloud security baselines. Generated IaC should default to CIS Benchmark-compliant configurations (e.g., S3 block public access, RDS encryption at rest, GCP uniform bucket-level access) without requiring user prompting.

**OAuth 2.0 / OpenID Connect**
- URL: https://openid.net/connect/
- All major cloud providers (AWS Cognito, Azure AD, GCP Identity) use OIDC for workload identity federation. Generated IaC for IAM resources should follow OIDC-based patterns (not long-lived access keys) as the secure default.

**OWASP Top 10 — Infrastructure as Code Security**
- URL: https://owasp.org/www-project-top-ten/
- While primarily a web application framework, OWASP's principles (injection, broken access control, security misconfiguration) directly apply to IaC. The IaC-specific interpretation covers overly permissive IAM, unencrypted storage, and publicly exposed resources.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Version 2025-11-25**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- The current canonical MCP specification (donated to Linux Foundation / Agentic AI Foundation in December 2025). An IaC generator exposing an MCP server allows any MCP-compatible AI assistant (Claude Code, Cursor, GitHub Copilot, Windsurf) to drive infrastructure generation, plan preview, and deployment initiation without a dedicated UI. Pulumi, Spacelift Intent, and env0 have all shipped MCP servers.

**MCP Transport: Streamable HTTP (2025-11-25)**
- URL: https://modelcontextprotocol.io/docs/concepts/transports
- The production-grade transport layer for MCP servers. Enables long-running, streaming generation operations (plan output, progress events) over HTTP/2 without WebSocket complexity. Recommended over stdio transport for hosted generator deployments.

**MCP GitHub Repository (Reference Servers)**
- URL: https://github.com/modelcontextprotocol/servers
- Contains reference MCP server implementations for GitHub, Slack, PostgreSQL, Docker, and Kubernetes. Reviewing these implementations provides patterns for exposing IaC generation tools, state queries, and deployment triggers as MCP resources and tools.

---

## Similar Products — Developer Documentation & APIs

### Pulumi

- **Description:** Multi-language IaC platform with AI generation (Pulumi AI) and agentic infrastructure management (Pulumi Neo). Supports AWS, Azure, GCP, Kubernetes, and 4,800+ providers.
- **API Documentation:** https://www.pulumi.com/docs/reference/cloud-rest-api/
- **Deployments REST API:** https://www.pulumi.com/docs/pulumi-cloud/deployments/api/
- **SDKs/Libraries:** Node.js, Python, Go, .NET, Java — https://www.pulumi.com/docs/reference/
- **MCP Server:** https://www.pulumi.com/docs/ai/mcp-server/ (hosted: `https://mcp.ai.pulumi.com/mcp`)
- **Developer Guide:** https://www.pulumi.com/docs/
- **Standards:** REST/JSON, OAuth 2.0 (MCP auth), gRPC (provider protocol)
- **Authentication:** OAuth 2.0 (Pulumi Cloud tokens), provider credential pass-through

---

### HashiCorp Terraform / OpenTofu

- **Description:** Dominant declarative IaC runtime using HCL. OpenTofu is the MPL 2.0-licensed community fork under Linux Foundation governance.
- **API Documentation (Terraform Cloud):** https://developer.hashicorp.com/terraform/cloud-docs/api-docs
- **OpenTofu Documentation:** https://opentofu.org/docs/
- **Provider SDK:** https://developer.hashicorp.com/terraform/plugin/framework
- **OpenAPI Provider Generator:** https://developer.hashicorp.com/terraform/plugin/code-generation/openapi-generator
- **SDKs/Libraries:** Go (plugin SDK), CDK for Terraform (TypeScript, Python) — https://developer.hashicorp.com/terraform/cdktf
- **Developer Guide:** https://developer.hashicorp.com/terraform/tutorials
- **Standards:** HCL spec, gRPC (provider protocol), REST/JSON (Terraform Cloud API)
- **Authentication:** API tokens (Terraform Cloud), provider-specific IAM credentials

---

### Pulumi AIaC / Firefly AIaC (Open Source CLI)

- **Description:** Open-source CLI tool that generates IaC (Terraform, CloudFormation, Pulumi, Helm, Dockerfiles) from natural language prompts via OpenAI.
- **Repository:** https://github.com/gofireflyio/aiac
- **API Documentation:** https://aiac.dev (CLI documentation)
- **SDKs/Libraries:** Go; importable as a Go library for embedding in other tools
- **Developer Guide:** https://github.com/gofireflyio/aiac#readme
- **Standards:** OpenAI Chat Completions API (JSON), CLI flags follow POSIX conventions
- **Authentication:** OpenAI API key (OPENAI_API_KEY environment variable)

---

### Spacelift + Spacelift Intent

- **Description:** IaC orchestration platform with OPA-based policy-as-code, multi-IaC support, and Intent — an open-source MCP server for natural language infrastructure provisioning.
- **API Documentation:** https://docs.spacelift.io/integrations/api (GraphQL API)
- **Intent Repository:** https://github.com/spacelift-io/spacelift-intent
- **Intent Documentation:** https://docs.spacelift.io/concepts/intent
- **SDKs/Libraries:** GraphQL client (any language); Spacelift Terraform provider for managing Spacelift resources as code
- **Developer Guide:** https://docs.spacelift.io/
- **Standards:** GraphQL, MCP (Intent), OPA/Rego (policy), REST/JSON (webhooks)
- **Authentication:** API tokens (JWT), SAML 2.0 / OIDC for SSO

---

### AWS CloudFormation

- **Description:** AWS-native IaC service using YAML/JSON template syntax. Manages AWS resource lifecycle with rollback support.
- **API Documentation:** https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/
- **SDKs/Libraries:** AWS SDK for Python (Boto3), JavaScript, Java, Go, .NET, Rust — https://docs.aws.amazon.com/code-library/latest/ug/cloudformation_code_examples.html
- **Developer Guide:** https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/
- **Standards:** YAML/JSON (template format), JSON Schema (resource type registry), REST/JSON (API)
- **Authentication:** AWS IAM (SigV4 request signing), temporary credentials via STS

---

### AWS CDK (Cloud Development Kit)

- **Description:** Code-first IaC for AWS using general-purpose programming languages. Synthesises CloudFormation templates under the hood.
- **API Documentation:** https://docs.aws.amazon.com/cdk/api/v2/
- **Toolkit Library:** https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib-readme.html
- **SDKs/Libraries:** TypeScript/JavaScript, Python, Java, Go, C# — https://github.com/aws/aws-cdk
- **Construct Hub:** https://constructs.dev (third-party construct discovery)
- **Developer Guide:** https://docs.aws.amazon.com/cdk/v2/guide/home.html
- **Standards:** CloudFormation (output), OpenAPI (CDK API docs), AWS SDK credentials
- **Authentication:** AWS IAM credentials (same as CloudFormation)

---

### Checkov (Bridgecrew / Palo Alto Networks)

- **Description:** Open-source static analysis tool for IaC security and compliance. Scans Terraform, CloudFormation, Kubernetes, Helm, ARM templates, and more against 1,000+ built-in policies mapped to CIS, SOC 2, PCI-DSS, and HIPAA.
- **API Documentation:** https://www.checkov.io/1.Welcome/1.What%20is%20Checkov.html (CLI-first; REST API available via Prisma Cloud integration)
- **Repository:** https://github.com/bridgecrewio/checkov
- **SDKs/Libraries:** Python package (`pip install checkov`); importable as Python library
- **Developer Guide:** https://www.checkov.io/2.Basics/2.Installing%20Checkov.html
- **Standards:** OPA/Rego (custom policies), YAML/Python (built-in policies), SARIF (scan output for CI integration), JUnit XML (test result format)
- **Authentication:** API key for Prisma Cloud integration; standalone CLI requires no auth

---

### env0

- **Description:** Cloud governance and IaC automation platform. Supports Terraform, OpenTofu, Pulumi, CloudFormation, Helm, Kubernetes. Includes AI PR summaries, Cloud Analyst, and MCP server for IDE integration.
- **API Documentation:** https://docs.env0.com/reference/api-reference
- **MCP Server:** https://docs.env0.com/docs/mcp-server (IDE infrastructure management)
- **SDKs/Libraries:** Terraform provider for env0 management; REST API (JSON)
- **Developer Guide:** https://docs.env0.com/
- **Standards:** REST/JSON (API), OPA/Rego (policy-as-code), MCP (IDE integration), OpenID Connect (SSO)
- **Authentication:** API keys, OIDC tokens, SAML 2.0 for enterprise SSO

---

### Brainboard

- **Description:** Visual IaC designer that converts cloud architecture diagrams to Terraform/OpenTofu code. Includes embedded CI/CD, drift detection, security scanning, and cost estimation.
- **API Documentation:** https://brainboard.co/api-documentation (limited public documentation; primarily UI-driven)
- **Developer Guide:** https://docs.brainboard.co/
- **Standards:** REST/JSON (API), Terraform HCL (output), GitHub/GitLab webhooks (GitOps)
- **Authentication:** API tokens, GitHub/GitLab OAuth for VCS integration

---

## Notes

**Emerging gap — OSCAL integration:** None of the surveyed tools currently emit OSCAL-formatted compliance artefacts alongside generated IaC. As FedRAMP 20x and NIST AI RMF adoption grows in the US federal and regulated sectors, this represents a differentiation opportunity. The OSCAL Compass project (https://pages.nist.gov/OSCAL/) provides tooling to map OSCAL assessment plans to OPA/Rego policies, which could bridge IaC generation and compliance evidence generation.

**MCP standardisation:** The MCP specification's November 2025 release (v2025-11-25) is the current stable version. The 2026 roadmap from the Agentic AI Foundation (https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) includes richer authentication (OAuth 2.0 with scoped tool permissions), batched tool calls, and formal resource subscription semantics — all relevant for a production IaC generator MCP server.

**OpenTofu vs. Terraform licensing:** Generators should default to producing OpenTofu-compatible HCL output and reference the OpenTofu provider registry (https://registry.opentofu.org) rather than the HashiCorp Terraform Registry to remain unambiguously within open-source licencing. The two registries are currently largely identical, but divergence is expected over time.

**Provider schema introspection:** The Terraform Plugin Framework exposes provider schemas via `terraform providers schema -json`, and OpenTofu provides equivalent output. An AI IaC generator that loads provider schemas at runtime can validate generated attribute names and values against the actual provider API, eliminating a major source of hallucination in current tools.
