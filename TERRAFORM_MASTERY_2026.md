# Terraform Mastery Concepts 2026

## Core Fundamentals
- Infrastructure as Code (IaC) Principles
- Declarative vs Imperative Approach
- Idempotency
- Immutable Infrastructure
- Terraform Architecture
- Terraform Core
- Terraform Providers
- Terraform CLI
- Terraform Workflow (Write, Plan, Apply)
- Terraform Registry

## HashiCorp Configuration Language (HCL)
- HCL Syntax
- Blocks
- Arguments
- Expressions
- Comments
- File Structure (.tf, .tfvars)
- JSON Configuration Alternative
- Override Files
- terraform.tfvars
- Auto-Loaded Variable Files

## Providers
- Provider Configuration
- Provider Versioning
- Version Constraints (~>, >=, <=, !=)
- Provider Aliases
- Multiple Provider Instances
- Provider Requirements Block
- Provider Authentication
- Provider Caching
- Required Providers Block
- Implicit vs Explicit Provider Configuration

### Major Providers
- AWS Provider
- Azure Provider (azurerm)
- Google Cloud Provider
- Kubernetes Provider
- Helm Provider
- Docker Provider
- GitHub Provider
- Vault Provider
- Datadog Provider
- New Relic Provider
- PagerDuty Provider
- Cloudflare Provider
- Okta Provider
- Random Provider
- Null Provider
- Local Provider
- TLS Provider
- Time Provider
- External Provider
- Archive Provider
- HTTP Provider

## Resources
- Resource Blocks
- Resource Types
- Resource Names
- Resource Arguments
- Resource Attributes
- Meta-Arguments
- depends_on
- count
- for_each
- provider
- lifecycle
- Lifecycle Rules
- create_before_destroy
- prevent_destroy
- ignore_changes
- replace_triggered_by
- precondition
- postcondition
- Resource Addressing
- Resource Dependencies
- Implicit Dependencies
- Explicit Dependencies
- Resource Timeouts
- Custom Timeouts

## Data Sources
- Data Source Blocks
- Data Source Configuration
- Data vs Resource
- Remote State Data Source
- External Data Source
- Local Data Sources
- Data Source Dependencies
- Data Source Filtering
- Querying Existing Infrastructure

## Variables
- Input Variables
- Variable Blocks
- Variable Types
- Primitive Types (string, number, bool)
- Complex Types (list, set, map, object, tuple)
- Type Constraints
- any Type
- Variable Defaults
- Variable Descriptions
- Variable Validation
- Custom Validation Rules
- Sensitive Variables
- Nullable Variables
- Variable Precedence
- Environment Variables (TF_VAR_)
- Command Line Variables (-var)
- Variable Definition Files (.tfvars)
- Auto-Loaded Variables
- Variable Files Precedence

## Outputs
- Output Blocks
- Output Values
- Output Descriptions
- Sensitive Outputs
- Output Dependencies
- depends_on for Outputs
- Accessing Child Module Outputs
- Output Preconditions
- Querying Outputs (terraform output)

## Locals
- Local Values
- Local Blocks
- Locals vs Variables
- Complex Local Expressions
- Reducing Repetition with Locals
- Computed Locals

## Expressions
- Types & Values
- String Literals
- Heredoc Strings
- String Templates
- References
- Resource References
- Module References
- Variable References
- Local References
- Path References (path.module, path.root, path.cwd)
- Arithmetic Operators
- Equality Operators
- Comparison Operators
- Logical Operators
- Conditional Expressions (Ternary)
- Splat Expressions ([*])
- Legacy Splat (.*)
- Full Splat ([*])
- Dynamic Blocks
- Dynamic Block Content
- Iterator in Dynamic Blocks
- Nested Dynamic Blocks

## Built-in Functions

### String Functions
- chomp
- format
- formatlist
- indent
- join
- lower
- upper
- regex
- regexall
- replace
- split
- strrev
- substr
- title
- trim
- trimprefix
- trimsuffix
- trimspace

### Numeric Functions
- abs
- ceil
- floor
- log
- max
- min
- parseint
- pow
- signum

### Collection Functions
- alltrue
- anytrue
- chunklist
- coalesce
- coalescelist
- compact
- concat
- contains
- distinct
- element
- flatten
- index
- keys
- length
- list (deprecated, use tolist)
- lookup
- map (deprecated, use tomap)
- matchkeys
- merge
- one
- range
- reverse
- setintersection
- setproduct
- setsubtract
- setunion
- slice
- sort
- sum
- transpose
- values
- zipmap

### Encoding Functions
- base64decode
- base64encode
- base64gzip
- csvdecode
- jsondecode
- jsonencode
- textdecodebase64
- textencodebase64
- urlencode
- yamldecode
- yamlencode

### Filesystem Functions
- abspath
- dirname
- pathexpand
- basename
- file
- fileexists
- fileset
- filebase64
- templatefile

### Date & Time Functions
- formatdate
- plantimestamp
- timeadd
- timecmp
- timestamp

### Hash & Crypto Functions
- base64sha256
- base64sha512
- bcrypt
- filebase64sha256
- filebase64sha512
- filemd5
- filesha1
- filesha256
- filesha512
- md5
- rsadecrypt
- sha1
- sha256
- sha512
- uuid
- uuidv5

### IP Network Functions
- cidrhost
- cidrnetmask
- cidrsubnet
- cidrsubnets

### Type Conversion Functions
- can
- defaults (deprecated)
- nonsensitive
- sensitive
- tobool
- tolist
- tomap
- tonumber
- toset
- tostring
- try
- type

## State Management
- Terraform State
- State File (terraform.tfstate)
- State File Structure
- State Locking
- State Versioning
- Local State
- Remote State
- State Backends
- State Storage
- State Encryption
- State Backup
- terraform.tfstate.backup
- State Drift
- Drift Detection
- State Refresh
- terraform refresh (deprecated)
- -refresh flag

### State Commands
- terraform state list
- terraform state show
- terraform state mv
- terraform state rm
- terraform state pull
- terraform state push
- terraform state replace-provider
- terraform force-unlock

### State Operations
- State Import
- terraform import
- Import Blocks (Terraform 1.5+)
- Bulk Import
- Import Configuration Generation
- State Migration
- State Splitting
- State Merging

## Backends
- Backend Configuration
- Backend Types
- Local Backend
- Remote Backends
- S3 Backend
- Azure Storage Backend
- GCS Backend
- Consul Backend
- etcd Backend
- HTTP Backend
- Kubernetes Backend
- PostgreSQL Backend
- cos Backend (Tencent Cloud)
- oss Backend (Alibaba Cloud)
- Terraform Cloud Backend
- Backend Authentication
- Backend State Locking
- DynamoDB for Locking (AWS)
- Backend Encryption
- Backend Configuration in CI/CD
- Partial Backend Configuration
- Backend Migration

## Modules
- Module Basics
- Root Module
- Child Modules
- Module Blocks
- Module Sources
- Local Paths
- Terraform Registry
- GitHub
- Bitbucket
- Generic Git
- Generic Mercurial
- HTTP URLs
- S3 Buckets
- GCS Buckets
- Module Versioning
- Module Inputs (Variables)
- Module Outputs
- Module Dependencies
- Module Composition
- Nested Modules
- Module Best Practices
- Module Documentation
- Module Testing
- Publishing Modules
- Private Module Registry
- Module Refactoring
- moved Blocks

### Module Patterns
- Wrapper Modules
- Facade Modules
- Utility Modules
- Service Modules
- Infrastructure Modules
- Composition Modules
- Monorepo vs Polyrepo Modules

## Workspaces
- Workspace Concepts
- Default Workspace
- Named Workspaces
- terraform workspace Commands
- workspace new
- workspace select
- workspace list
- workspace show
- workspace delete
- terraform.workspace
- Workspace State Isolation
- Workspace vs Directories
- Workspace Naming Conventions
- Workspace Limitations

## Terraform Cloud & Enterprise
- Terraform Cloud Overview
- Organizations
- Workspaces (Cloud)
- Workspace Variables
- Workspace VCS Connection
- Runs & Run Triggers
- Speculative Plans
- Cost Estimation
- Sentinel Policies
- Policy Sets
- Policy Enforcement Levels
- OPA Integration
- Run Tasks
- Private Registry
- Agents (Self-Hosted)
- Agent Pools
- Teams & Permissions
- SSO Integration
- Audit Logging
- API Tokens
- Remote Execution
- Local Execution
- CLI-Driven Workflows
- VCS-Driven Workflows
- API-Driven Workflows
- Notifications
- Webhooks

## Terraform CLI Commands
- terraform init
- terraform validate
- terraform plan
- terraform apply
- terraform destroy
- terraform fmt
- terraform show
- terraform output
- terraform graph
- terraform providers
- terraform version
- terraform login
- terraform logout
- terraform get
- terraform console
- terraform test
- terraform metadata

### Plan & Apply Options
- -auto-approve
- -target
- -var
- -var-file
- -out
- -input
- -lock
- -lock-timeout
- -parallelism
- -refresh
- -replace
- -destroy
- Plan File Output
- JSON Output (-json)
- Compact Warnings

## Provisioners
- Provisioner Types
- local-exec
- remote-exec
- file Provisioner
- Connection Blocks
- SSH Connections
- WinRM Connections
- Provisioner Failure Behavior
- on_failure
- When to Use Provisioners
- Provisioners vs User Data
- Provisioners vs Configuration Management
- null_resource for Provisioners
- terraform_data Resource

## Testing & Validation
- terraform validate
- terraform fmt -check
- terraform plan (Preview)
- Custom Validation Rules
- Preconditions
- Postconditions
- Check Blocks (Terraform 1.5+)
- terraform test (Native Testing)
- Test Files (.tftest.hcl)
- Test Variables
- Test Assertions
- Mock Providers
- Terratest
- Kitchen-Terraform
- Checkov
- tfsec
- Terrascan
- Snyk IaC
- KICS
- OPA Conftest
- Sentinel Testing
- Integration Testing
- Unit Testing
- Contract Testing
- Compliance Testing

## Security Best Practices
- State File Security
- State Encryption
- Remote State Access Control
- Sensitive Variables
- Sensitive Outputs
- No Secrets in Code
- Secret Injection
- Vault Integration
- AWS Secrets Manager Integration
- Azure Key Vault Integration
- GCP Secret Manager Integration
- Environment Variables for Secrets
- .gitignore for Terraform
- Provider Credential Management
- IAM Roles for Providers
- Workload Identity
- OIDC Authentication
- Security Scanning in CI/CD
- Policy as Code
- Supply Chain Security
- Provider Verification
- Module Verification

## CI/CD Integration
- Terraform in CI/CD Pipelines
- GitHub Actions
- GitLab CI/CD
- Azure DevOps Pipelines
- Jenkins Pipelines
- CircleCI
- Bitbucket Pipelines
- AWS CodePipeline
- Google Cloud Build
- Atlantis
- Spacelift
- env0
- Scalr
- Harness
- Plan Artifacts
- Apply Automation
- Manual Approval Gates
- Drift Detection in CI/CD
- Scheduled Plans
- Pull Request Automation
- GitOps for Terraform
- Branch Protection
- Environment Promotion

## Advanced Patterns

### Code Organization
- Directory Structure
- Environment Separation
- Layer Separation
- Component Separation
- Monorepo vs Polyrepo
- Module Composition
- Root Module Design
- Shared Module Library

### Multi-Environment
- Environment-Specific Variables
- Workspace per Environment
- Directory per Environment
- Branch per Environment
- Terraform Cloud Workspaces
- Environment Promotion Patterns

### Multi-Region
- Multi-Region Deployments
- Provider Aliases for Regions
- Regional Resource Naming
- Regional State Separation
- Cross-Region Dependencies
- Global vs Regional Resources

### Multi-Account / Multi-Subscription
- AWS Multi-Account
- Azure Multi-Subscription
- GCP Multi-Project
- Provider Aliases per Account
- Assume Role Patterns
- Cross-Account Resources
- Hub-and-Spoke Architecture
- Landing Zone Patterns

### Multi-Cloud
- Multi-Cloud Strategies
- Cloud-Agnostic Modules
- Provider Abstraction
- Common Interfaces

## Terragrunt
- Terragrunt Overview
- DRY Configuration
- terragrunt.hcl
- Include Blocks
- Dependency Blocks
- Dependencies Between Modules
- generate Blocks
- terraform Block
- remote_state Block
- Inputs
- Locals in Terragrunt
- Before/After Hooks
- Terragrunt Functions
- run_cmd
- get_env
- path_relative_to_include
- find_in_parent_folders
- Terragrunt CLI Commands
- Terragrunt with CI/CD
- Terragrunt vs Native Terraform

## Terraform CDK (CDKTF)
- CDKTF Overview
- Supported Languages
- TypeScript
- Python
- Java
- C#
- Go
- CDKTF CLI
- cdktf init
- cdktf synth
- cdktf deploy
- cdktf destroy
- Constructs
- Stacks
- Providers in CDKTF
- Modules in CDKTF
- Testing CDKTF
- CDKTF vs HCL

## Import & Migration
- terraform import Command
- Import Blocks (1.5+)
- Import Configuration Generation
- -generate-config-out
- Bulk Import Strategies
- Importing Existing Infrastructure
- State-Only Import
- Configuration + State Import
- Import Limitations
- Post-Import Cleanup
- Migrating from CloudFormation
- Migrating from ARM Templates
- Migrating from Pulumi
- Migrating Between Backends
- Migrating Between State Files
- Refactoring with moved Blocks

## Performance & Optimization
- Parallelism (-parallelism)
- Resource Targeting (-target)
- Refresh Optimization (-refresh=false)
- State Size Management
- Module Caching
- Provider Caching
- Plugin Caching
- TF_PLUGIN_CACHE_DIR
- Large State File Strategies
- State Splitting
- Incremental Changes
- Plan Performance
- Apply Performance

## Debugging & Troubleshooting
- TF_LOG Environment Variable
- Log Levels (TRACE, DEBUG, INFO, WARN, ERROR)
- TF_LOG_PATH
- TF_LOG_CORE
- TF_LOG_PROVIDER
- terraform console
- terraform graph
- Dependency Graph Visualization
- State Inspection
- Plan Output Analysis
- Error Messages
- Common Errors
- Provider Errors
- State Lock Errors
- Dependency Errors
- Circular Dependency Resolution
- Resource Recreation Issues
- Tainted Resources
- terraform untaint (deprecated)
- -replace Flag

## Collaboration
- Remote State for Teams
- State Locking
- Terraform Cloud for Teams
- Workspaces for Team Isolation
- RBAC (Role-Based Access Control)
- VCS Integration
- Pull Request Workflows
- Code Review for Terraform
- Atlantis for PR Automation
- Documentation
- README Files
- terraform-docs
- Architecture Decision Records
- Runbooks

## Policy as Code
- Sentinel (HashiCorp)
- Sentinel Language
- Sentinel Policies
- Sentinel Mocks
- Policy Sets
- Enforcement Levels
- Advisory
- Soft Mandatory
- Hard Mandatory
- OPA (Open Policy Agent)
- Rego Language
- Conftest
- OPA with Terraform Cloud
- Policy Testing
- Policy Libraries

## Cost Management
- Infracost
- Cost Estimation
- Terraform Cloud Cost Estimation
- Cost Policies
- Budget Alerts
- Cost Optimization
- Resource Right-Sizing
- Reserved Capacity Planning
- Tagging for Cost Allocation

## Documentation
- terraform-docs
- Output Formats
- Markdown
- JSON
- YAML
- README Generation
- Module Documentation
- Input Documentation
- Output Documentation
- Requirement Documentation
- Provider Documentation
- Resource Documentation
- Diagram Generation

## Terraform 1.x Features

### Terraform 1.5+
- Import Blocks
- Check Blocks
- Configuration Generation
- Lifecycle replace_triggered_by

### Terraform 1.6+
- terraform test (GA)
- Test Mocking
- Provider Iteration

### Terraform 1.7+
- Removed Blocks
- Config-Driven Import Improvements
- Test Improvements

### Terraform 1.8+
- Provider-Defined Functions
- Enhanced Validation
- Backend Changes

### Terraform 1.9+ / 2026 Features
- Ephemeral Values
- Deferred Actions
- Provider Functions
- Enhanced Stacks
- Enhanced Testing
- Performance Improvements

## Best Practices Summary
- Use Remote State
- Enable State Locking
- Version Pin Providers
- Version Pin Modules
- Use Modules for Reusability
- Keep Modules Small & Focused
- Use Consistent Naming Conventions
- Tag All Resources
- Use Variables for Configurability
- Validate Inputs
- Use Outputs for Sharing
- Separate Environments
- Use Workspaces Appropriately
- Implement Policy as Code
- Automate with CI/CD
- Security Scan Configurations
- Document Everything
- Review Plans Before Apply
- Never Apply Without Review in Production
- Test Infrastructure Code
- Use Least Privilege for Providers
- Encrypt State at Rest
- Limit State Access
- Regular State Backup
- Monitor for Drift

## Ecosystem & Tools
- Terraform Registry
- Terraform Cloud
- Terraform Enterprise
- Terragrunt
- CDKTF
- Atlantis
- Spacelift
- env0
- Scalr
- terraform-docs
- tflint
- tfsec
- Checkov
- Terrascan
- Infracost
- Terratest
- Kitchen-Terraform
- Rover (Visualization)
- Blast Radius (Visualization)
- Pike (IAM Generator)
- tf-summarize
- tfswitch
- tfenv
- asdf-terraform
- pre-commit-terraform
