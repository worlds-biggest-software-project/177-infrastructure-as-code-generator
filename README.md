# Infrastructure as Code Generator

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source generator that turns natural language descriptions into production-quality Terraform, OpenTofu, and Pulumi code.

This project aims to lower the IaC authoring skill barrier by translating plain-language intent into idiomatic, multi-cloud infrastructure code. It targets platform engineers, DevOps leads, SREs, and developers who need to provision cloud environments without deep HCL or Pulumi expertise, while preserving the plan/apply review workflow that operators already trust.

---

## Why Infrastructure as Code Generator?

- HCL verbosity and state management complexity remain the dominant adoption barriers to Terraform, which still commands roughly 76% practitioner share (CNCF 2024).
- HashiCorp's Business Source Licence change drove the OpenTofu fork; an open-source generator targeting MPL 2.0 OpenTofu output sidesteps BSL restrictions for downstream users.
- Existing AI-native generators (Pulumi Neo, Firefly AIaC, StackGen) are either tied to proprietary SaaS, custom-priced, or limited to single-format output.
- General-purpose assistants like GitHub Copilot and Amazon Q Developer lack cloud state awareness and frequently hallucinate provider attributes for less common resources.
- Most tools generate functional but insecure code; security-by-default and integrated cost estimation are gaps across the incumbent landscape.

---

## Key Features

### Natural Language Generation

- Prompt-driven generation of modular Terraform/OpenTofu HCL (not monolithic `main.tf` files)
- Multi-cloud first-class support for AWS, Azure, and GCP
- Iterative conversational refinement within a session ("make this highly available across three AZs and add a read replica")
- Optional Pulumi (TypeScript/Python) output for teams preferring general-purpose languages

### Security and Policy

- Built-in security scanning on every generated output via Checkov or OPA rules
- Security-by-default generation embedding CIS benchmark controls, encryption at rest, and least-privilege IAM
- Policy-as-code enforcement compatible with OPA/Rego

### Developer Workflow

- CLI with explicit plan-preview diff before any apply action
- Git/VCS integration for PR-based review workflows
- MCP server interface so MCP-compatible AI assistants (Claude Code, Cursor, Copilot) can drive generation
- Module registry lookup: recommend existing modules before generating from scratch

### Cloud Awareness

- Read existing provider state before generating to avoid resource conflicts and drift
- Automated cost estimation projected before `terraform apply`
- Drift detection signalled back into the conversational loop

---

## AI-Native Advantage

Unlike general-purpose code assistants, this project reads existing cloud state via provider APIs before generation, preventing the resource conflicts and hallucinated attributes common in stateless prompt-based tools. Security controls, least-privilege IAM, and encryption defaults are embedded into generated code rather than added as a later scan. Conversational refinement and module recommendation reduce the prompt engineering burden currently required to produce production-quality IaC.

---

## Tech Stack & Deployment

The MVP targets OpenTofu-compatible HCL (MPL 2.0) as the primary output to avoid Terraform BSL constraints, with Pulumi output planned for v1.1. Deployment is CLI-first for power users and CI/CD pipelines, with an MCP server interface for IDE-embedded AI assistants. Standards alignment includes HCL, OPA/Rego for policy, GitOps via Flux/ArgoCD-compatible repository layouts, and CIS Benchmarks for security baselines. An air-gapped/local inference mode for enterprises without cloud connectivity is on the backlog.

---

## Market Context

The IaC market was estimated at $2.2B in 2025 and is projected to reach $12.9B by 2032 at a 28.6% CAGR (Precedence Research); a separate estimate from GM Insights puts 2025 at $1.3B growing to $9.4B by 2034 at 24.4% CAGR. IBM's $6.4B acquisition of HashiCorp in 2024 reflects strategic value in the category, while Pulumi raised a $41M Series C in October 2023. Primary buyers are platform engineering teams, cloud infrastructure engineers, DevOps leads, and SREs at cloud-native organisations; incumbent pricing ranges from free OSS (OpenTofu) to $20/user/month (Terraform Cloud) and per-deployment-minute fees (Pulumi at $0.01/minute), with most AI-native generators custom-priced.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. Source files note OpenTofu (MPL 2.0), AIaC CLI (MIT), and Pulumi engine (Apache 2.0) as compatible open-source references; the project should target a permissive licence aligned with OpenTofu output.
