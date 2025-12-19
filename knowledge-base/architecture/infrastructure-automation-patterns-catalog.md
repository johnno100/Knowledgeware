# Infrastructure Automation Architectural Patterns Catalog

**Domain**: Infrastructure as Code, DevOps, Network Automation  
**Date**: December 2024  
**Source**: Nautilus Infrastructure Automation Platform Design

## Overview

This catalog documents architectural patterns, design decisions, and integration strategies discovered during the design and implementation of the Nautilus infrastructure automation platform. These patterns are applicable to any organization implementing infrastructure as code workflows with NetBox, GitLab, and IaC tools.

---

## Pattern 1: Dual Source of Truth

### Context
Organizations need to manage both planned infrastructure (what should exist) and deployed infrastructure (what does exist) while maintaining consistency and auditability.

### Problem
Using a single system for both planning and state management creates:
- Complexity in distinguishing planned vs. deployed resources
- Risk of accidental deployments
- Difficulty in rollback and change management
- Confusion about the "current state" of infrastructure

### Solution
Maintain two complementary sources of truth:

```
┌─────────────────────────────────────────┐
│  NetBox: "What SHOULD Exist"           │
│  - Planned infrastructure               │
│  - Network topology                     │
│  - IP address management                │
│  - Device inventory                     │
└─────────────────────────────────────────┘
              ↓
        (generates)
              ↓
┌─────────────────────────────────────────┐
│  IaC Code: "How to Create It"          │
│  - Terraform configurations             │
│  - Ansible playbooks                    │
└─────────────────────────────────────────┘
              ↓
        (provisions)
              ↓
┌─────────────────────────────────────────┐
│  Terraform State: "What DOES Exist"    │
│  - Deployed resources                   │
│  - Resource dependencies                │
│  - Current configuration                │
└─────────────────────────────────────────┘
```

### Implementation

**NetBox Configuration**:
- Use `status` field to distinguish lifecycle stages (planned, active, decommissioned)
- Tag objects with work package identifiers
- Store metadata in custom fields

**GitLab Configuration**:
- Store Terraform state in GitLab backend
- Version control all IaC code
- Use CI/CD pipelines for deployment

**Synchronization**:
- After successful deployment, update NetBox status: `planned` → `active`
- Periodically audit deployed resources against NetBox inventory
- Implement drift detection to identify discrepancies

### Benefits
- ✅ Clear separation between planning and execution
- ✅ Safe experimentation with planned infrastructure
- ✅ Auditability of changes
- ✅ Rollback capabilities via Git
- ✅ Each system optimized for its purpose

### Trade-offs
- ⚠️ Requires synchronization between systems
- ⚠️ Potential for drift if not automated
- ⚠️ Additional integration complexity

### Status
✅ **Implemented** in Nautilus Phase 1

### Related Patterns
- Work Package Lifecycle Pattern
- Tag-Based Filtering Pattern

---

## Pattern 2: Work Package Lifecycle

### Context
Infrastructure changes need to be planned, reviewed, approved, and deployed in a structured manner that aligns with agile development practices.

### Problem
Ad-hoc infrastructure changes lead to:
- Lack of visibility into what's being deployed
- Difficulty tracking dependencies
- No clear approval process
- Challenges in rollback
- Poor communication with stakeholders

### Solution
Treat infrastructure changes as "work packages" with a defined lifecycle:

```
┌──────────────────────────────────────────────────────┐
│  1. PLANNING                                         │
│  - Create work package (wp:project-name)             │
│  - Create feature branch                             │
│  - Tag NetBox objects with work package              │
│  - Set status: planned                               │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  2. DEVELOPMENT                                      │
│  - Generate IaC code from NetBox                     │
│  - Customize and test                                │
│  - Commit to feature branch                          │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  3. REVIEW                                           │
│  - Create merge request                              │
│  - Peer review                                       │
│  - Automated testing                                 │
│  - Approval gates                                    │
└──────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│  4. DEPLOYMENT                                       │
│  - Merge to main                                     │
│  - Execute Terraform/Ansible                         │
│  - Update NetBox status: planned → active            │
│  - Notify stakeholders                               │
└──────────────────────────────────────────────────────┘
```

### Implementation

**Work Package Structure**:
```
Work Package: wp:q4-network-upgrade
├── NetBox Tag: wp:q4-network-upgrade
├── Git Branch: feature/q4-network-upgrade
├── Documentation: docs/work-packages/q4-network-upgrade.md
└── IaC Code:
    ├── terraform/q4-network-upgrade/
    └── ansible/q4-network-upgrade/
```

**CLI Commands**:
```bash
# Initialize work package
nautilus wp init --name "Q4 Network Upgrade"

# Create planned infrastructure
nautilus netbox create-vm --name web01 --role webserver

# Generate IaC code
nautilus iac generate

# Deploy infrastructure
nautilus iac deploy

# Update status
nautilus netbox update-status --tag wp:q4-network-upgrade --status active
```

### Benefits
- ✅ Structured change management
- ✅ Clear audit trail
- ✅ Aligns with sprint planning
- ✅ Enables parallel work on multiple projects
- ✅ Facilitates rollback
- ✅ Improves stakeholder communication

### Trade-offs
- ⚠️ Requires discipline to follow process
- ⚠️ Additional overhead for small changes
- ⚠️ Needs automation to be efficient

### Status
✅ **Implemented** (Phases 1-2)  
🟨 **Partially Implemented** (Phases 3-4 automation pending)

### Related Patterns
- GitLab Branch Mapping Pattern
- Tag-Based Filtering Pattern

---

## Pattern 3: Tag-Based Filtering and Sequencing

### Context
Large infrastructure deployments need to be broken into manageable chunks that can be deployed incrementally, with proper dependency management and parallel execution where possible.

### Problem
Deploying all planned infrastructure at once:
- Increases risk of failures
- Makes troubleshooting difficult
- Doesn't account for dependencies
- Can't leverage parallel execution
- Overwhelms change approval processes

### Solution
Use NetBox tags and custom fields to organize infrastructure into deployable units:

**Tag Hierarchy**:
```
wp:q4-datacenter-expansion          # Work package
  ├── sprint:2024-q4-sprint-1       # Sprint assignment
  ├── priority:high                 # Deployment priority
  ├── wave:network-foundation       # Deployment wave
  └── depends-on:wp:power-upgrade   # Dependencies
```

**Custom Fields for Sequencing**:
```python
deployment_sequence: 1              # Order within work package
deployment_wave: "wave-1"           # Parallel deployment group
estimated_duration: 30              # Minutes
rollback_complexity: "low"          # Risk assessment
```

### Implementation

**Query Pattern**:
```python
# Get all infrastructure for a work package
objects = netbox.api.virtualization.virtual_machines.filter(
    status="planned",
    tag="wp:q4-datacenter-expansion"
)

# Get specific deployment wave
wave_1_objects = netbox.api.virtualization.virtual_machines.filter(
    status="planned",
    tag=["wp:q4-datacenter-expansion", "wave:network-foundation"]
)

# Get objects in sequence order
for obj in sorted(objects, key=lambda x: x.custom_fields.get('deployment_sequence', 999)):
    deploy(obj)
```

**Deployment Workflow**:
```bash
# Deploy by wave
nautilus iac deploy --tag wp:q4-datacenter-expansion --wave wave-1
nautilus iac deploy --tag wp:q4-datacenter-expansion --wave wave-2

# Deploy by priority
nautilus iac deploy --tag wp:q4-datacenter-expansion --priority high

# Deploy specific sequence
nautilus iac deploy --tag wp:q4-datacenter-expansion --sequence 1-5
```

### Benefits
- ✅ Incremental deployment reduces risk
- ✅ Parallel execution where possible
- ✅ Clear dependency management
- ✅ Flexible deployment strategies
- ✅ Easy to pause and resume deployments

### Trade-offs
- ⚠️ Requires upfront planning
- ⚠️ Tag namespace can become complex
- ⚠️ Dependency validation needed

### Status
📋 **Specified** (Design complete, implementation pending)

### Related Patterns
- Work Package Lifecycle Pattern
- Dynamic Inventory Integration Pattern

---

## Pattern 4: GitLab Branch Mapping

### Context
Infrastructure changes need clear traceability between planning documents, IaC code, and deployed resources.

### Problem
Without clear conventions:
- Difficult to find code for a specific project
- No automatic context detection
- Manual work package tracking
- Confusion about which branch to use

### Solution
Establish a strict mapping between work packages, Git branches, and file structure:

**Convention**:
```
Work Package Tag: wp:project-name
       ↓
Feature Branch: feature/project-name
       ↓
Documentation: docs/work-packages/project-name.md
       ↓
IaC Code: terraform/project-name/
          ansible/project-name/
```

### Implementation

**Automatic Detection**:
```python
def get_current_work_package():
    """Detect work package from current Git branch."""
    branch = subprocess.check_output(
        ['git', 'rev-parse', '--abbrev-ref', 'HEAD'],
        text=True
    ).strip()
    
    if branch.startswith('feature/'):
        project_name = branch.replace('feature/', '')
        return f"wp:{project_name}"
    
    return None

# CLI automatically detects context
$ git checkout feature/q4-network-upgrade
$ nautilus netbox create-vm --name web01
# Automatically tags with wp:q4-network-upgrade
```

**Branch Protection**:
```yaml
# .gitlab-ci.yml
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^feature\//
      variables:
        WORK_PACKAGE: ${CI_COMMIT_BRANCH#feature/}
        WORK_PACKAGE_TAG: "wp:${WORK_PACKAGE}"
```

### Benefits
- ✅ Automatic context detection
- ✅ Clear file organization
- ✅ Easy to find related code
- ✅ Supports parallel development
- ✅ Simplifies CLI usage

### Trade-offs
- ⚠️ Requires strict naming conventions
- ⚠️ Branch names must be carefully chosen
- ⚠️ Refactoring branches can be complex

### Status
✅ **Implemented** in Nautilus CLI

### Related Patterns
- Work Package Lifecycle Pattern
- Tag-Based Filtering Pattern

---

## Pattern 5: Dynamic Inventory Integration

### Context
IaC tools need to query NetBox for infrastructure data rather than hardcoding values, enabling dynamic and data-driven infrastructure provisioning.

### Problem
Hardcoded infrastructure definitions:
- Require manual updates for changes
- Prone to errors and drift
- Difficult to maintain consistency
- Don't leverage NetBox as source of truth

### Solution
Integrate NetBox as a dynamic data source for both Terraform and Ansible:

**Terraform Data Source Pattern**:
```hcl
# Query NetBox for planned VMs
data "netbox_virtual_machines" "planned_vms" {
  filter {
    name  = "status"
    value = "planned"
  }
  filter {
    name  = "tag"
    value = var.work_package_tag
  }
}

# Transform NetBox data for Terraform
locals {
  planned_vms = [
    for vm in data.netbox_virtual_machines.planned_vms.virtual_machines : {
      name    = vm.name
      vcpus   = vm.vcpus
      memory  = vm.memory
      role    = vm.role.name
      cluster = vm.cluster.name
    }
  ]
}

# Create resources dynamically
resource "vsphere_virtual_machine" "vms" {
  for_each = { for vm in local.planned_vms : vm.name => vm }
  
  name     = each.value.name
  num_cpus = each.value.vcpus
  memory   = each.value.memory
  # ... additional configuration
}
```

**Ansible Dynamic Inventory Pattern**:
```yaml
# netbox_inventory.yml
plugin: netbox.netbox.nb_inventory
api_endpoint: "{{ netbox_url }}"
token: "{{ netbox_token }}"
validate_certs: true

# Filter by work package
query_filters:
  - status: planned
  - tag: "{{ work_package_tag }}"

# Group by role
group_by:
  - device_roles
  - tags

# Compose host variables from NetBox data
compose:
  ansible_host: primary_ip4.address
  netbox_role: device_role.name
  netbox_site: site.name
  work_package: "{{ tags | select('match', '^wp:') | first }}"
```

**Playbook Usage**:
```yaml
---
- name: Configure infrastructure
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Display NetBox metadata
      debug:
        msg: |
          Host: {{ inventory_hostname }}
          Role: {{ netbox_role }}
          Work Package: {{ work_package }}
    
    - name: Install packages based on role
      include_role:
        name: "{{ netbox_role }}"
```

### Implementation

**Bidirectional Sync**:
```python
# After Terraform deployment, update NetBox
def update_netbox_after_deployment(terraform_state, work_package_tag):
    """Update NetBox with deployed resource information."""
    for resource in terraform_state['resources']:
        if resource['type'] == 'vsphere_virtual_machine':
            vm_name = resource['instances'][0]['attributes']['name']
            vm_ip = resource['instances'][0]['attributes']['default_ip_address']
            
            # Update NetBox VM
            netbox_vm = netbox.api.virtualization.virtual_machines.get(name=vm_name)
            netbox_vm.status = 'active'
            netbox_vm.primary_ip4 = create_or_get_ip(vm_ip)
            netbox_vm.save()
```

### Benefits
- ✅ Single source of truth (NetBox)
- ✅ Automatic updates when NetBox changes
- ✅ Reduced manual configuration
- ✅ Consistent data across tools
- ✅ Enables advanced filtering and grouping

### Trade-offs
- ⚠️ Requires NetBox API availability
- ⚠️ Network latency for queries
- ⚠️ Complexity in error handling

### Status
✅ **Templates Created**  
🟨 **Runtime Integration Pending** (Phase 2)

### Related Patterns
- Dual Source of Truth Pattern
- Tag-Based Filtering Pattern

---

## Pattern 6: Template-Based IaC Generation

### Context
Organizations need to generate consistent, maintainable IaC code across multiple cloud providers and infrastructure types.

### Problem
Manual IaC code creation:
- Time-consuming and error-prone
- Inconsistent formatting and structure
- Difficult to enforce standards
- Hard to maintain across teams

### Solution
Use Jinja2 templates to generate IaC code from NetBox data:

**Template Structure**:
```
templates/
├── terraform/
│   ├── main.tf.j2
│   ├── variables.tf.j2
│   ├── outputs.tf.j2
│   └── providers/
│       ├── vsphere.tf.j2
│       ├── aws.tf.j2
│       └── azure.tf.j2
└── ansible/
    ├── playbook.yml.j2
    ├── inventory.yml.j2
    └── roles/
        ├── webserver.yml.j2
        └── database.yml.j2
```

**Terraform Template Example**:
```jinja2
# main.tf - Generated by Nautilus
# Work Package: {{ work_package_name }}
# Generated: {{ generation_timestamp }}

terraform {
  required_version = ">= 1.0"
  
  backend "http" {
    address = "{{ gitlab_url }}/api/v4/projects/{{ project_id }}/terraform/state/{{ work_package_tag }}"
    lock_address = "{{ gitlab_url }}/api/v4/projects/{{ project_id }}/terraform/state/{{ work_package_tag }}/lock"
    unlock_address = "{{ gitlab_url }}/api/v4/projects/{{ project_id }}/terraform/state/{{ work_package_tag }}/lock"
  }
}

# Query NetBox for planned infrastructure
data "netbox_virtual_machines" "planned_vms" {
  filter {
    name  = "status"
    value = "planned"
  }
  filter {
    name  = "tag"
    value = "{{ work_package_tag }}"
  }
}

{% if provider == "vsphere" %}
# vSphere Provider Configuration
provider "vsphere" {
  vsphere_server = var.vsphere_server
  user           = var.vsphere_user
  password       = var.vsphere_password
}

# Create VMs
{% for vm in virtual_machines %}
resource "vsphere_virtual_machine" "{{ vm.name | replace('-', '_') }}" {
  name             = "{{ vm.name }}"
  resource_pool_id = data.vsphere_resource_pool.pool.id
  datastore_id     = data.vsphere_datastore.datastore.id
  
  num_cpus = {{ vm.vcpus }}
  memory   = {{ vm.memory }}
  
  network_interface {
    network_id = data.vsphere_network.network.id
  }
  
  disk {
    label = "disk0"
    size  = {{ vm.disk_size }}
  }
}
{% endfor %}
{% endif %}
```

**Generation Process**:
```python
def generate_terraform(work_package_tag, provider='vsphere'):
    """Generate Terraform code from NetBox data."""
    # Query NetBox
    vms = netbox.api.virtualization.virtual_machines.filter(
        status='planned',
        tag=work_package_tag
    )
    
    # Prepare template context
    context = {
        'work_package_name': work_package_tag.replace('wp:', ''),
        'work_package_tag': work_package_tag,
        'generation_timestamp': datetime.now().isoformat(),
        'provider': provider,
        'virtual_machines': [
            {
                'name': vm.name,
                'vcpus': vm.vcpus,
                'memory': vm.memory,
                'disk_size': vm.disk or 50,
                'role': vm.role.name if vm.role else 'generic'
            }
            for vm in vms
        ],
        'gitlab_url': config.gitlab_url,
        'project_id': config.gitlab_project_id
    }
    
    # Render template
    template = jinja_env.get_template('terraform/main.tf.j2')
    return template.render(**context)
```

### Benefits
- ✅ Consistent code generation
- ✅ Easy to customize templates
- ✅ Non-developers can modify templates
- ✅ Multi-cloud support
- ✅ Enforces organizational standards

### Trade-offs
- ⚠️ Template syntax can be complex
- ⚠️ Requires template testing
- ⚠️ Less type safety than programmatic generation

### Status
✅ **Templates Created**  
📋 **Generation Logic Pending** (Phase 2)

### Related Patterns
- Dynamic Inventory Integration Pattern
- Work Package Lifecycle Pattern

---

## Pattern 7: AI Agent Constitution

### Context
AI agents need clear guidance on how to interact with infrastructure automation tools safely and effectively.

### Problem
Without explicit guidance, AI agents:
- May execute dangerous operations
- Lack context awareness
- Provide inconsistent responses
- Don't follow organizational standards
- Can't map user intent to correct actions

### Solution
Create a comprehensive "constitution" document (NAUTILUS.md) that defines:

**Core Components**:
```markdown
# NAUTILUS.md - AI Agent Constitution

## 1. Core Mission
Define the agent's primary purpose and responsibilities

## 2. Guiding Principles
- Safety First: Always validate before executing
- Auditability: Maintain complete change logs
- Source of Truth: NetBox is authoritative
- Clarity: Explain actions before executing

## 3. Tool Usage Guidelines
Map user intent to specific CLI commands

## 4. Context Awareness
- Detect current Git branch
- Infer work package from context
- Understand infrastructure lifecycle

## 5. Communication Patterns
- How to explain technical concepts
- When to ask for confirmation
- Error reporting standards

## 6. Security Protocols
- Credential handling
- Change approval requirements
- Rollback procedures

## 7. Limitations & Boundaries
- What the agent can and cannot do
- When to escalate to humans
- Risk assessment criteria
```

**Intent Mapping Example**:
```markdown
### User Intent: "Plan a new web server"

**Agent Interpretation**:
1. Check if on a feature branch
   - If not: Suggest creating work package first
   - If yes: Extract work package from branch

2. Execute command:
   ```bash
   nautilus netbox create-vm \
     --name web01 \
     --role webserver \
     --cluster prod-cluster
   ```

3. Provide context:
   - Explain that VM is in "planned" status
   - Suggest next steps (generate IaC, review, deploy)
   - Mention related documentation

4. Offer additional actions:
   - "Would you like me to create additional web servers?"
   - "Should I generate the Terraform code now?"
```

**Safety Checks**:
```markdown
### Before Destructive Operations

1. **Validate Current State**
   - Query NetBox for affected resources
   - Check Terraform state
   - Verify no dependencies

2. **Explain Impact**
   - List resources to be modified/deleted
   - Estimate downtime
   - Identify risks

3. **Confirm with User**
   - Present summary
   - Ask for explicit confirmation
   - Offer dry-run option

4. **Execute with Logging**
   - Use --dry-run first
   - Log all actions
   - Report results clearly
```

### Implementation

**File Structure**:
```
/project-root/
├── NAUTILUS.md              # Main constitution
├── docs/
│   ├── ai-agent-guide.md    # Detailed usage guide
│   └── examples/
│       ├── planning.md      # Planning scenarios
│       ├── deployment.md    # Deployment scenarios
│       └── troubleshooting.md
```

**Integration with AI Tools**:
```python
# Load constitution into AI context
def get_ai_context():
    """Prepare context for AI agent."""
    with open('NAUTILUS.md', 'r') as f:
        constitution = f.read()
    
    current_branch = get_current_branch()
    work_package = get_work_package_from_branch(current_branch)
    
    context = f"""
{constitution}

## Current Context
- Git Branch: {current_branch}
- Work Package: {work_package}
- User: {get_current_user()}
- Timestamp: {datetime.now().isoformat()}
"""
    return context
```

### Benefits
- ✅ Consistent AI agent behavior
- ✅ Safety guardrails built-in
- ✅ Context-aware responses
- ✅ Clear intent-to-action mapping
- ✅ Organizational standards enforced
- ✅ Reduces errors and misunderstandings

### Trade-offs
- ⚠️ Requires maintenance as tools evolve
- ⚠️ AI agents may still misinterpret edge cases
- ⚠️ Verbose documentation needed

### Status
✅ **Fully Documented** (NAUTILUS.md created)  
💡 **AI Integration Pending** (MCP Server needed)

### Related Patterns
- Work Package Lifecycle Pattern
- GitLab Branch Mapping Pattern

---

## Pattern 8: Secrets Management with OpenBao/Vault

### Context
Infrastructure automation requires secure handling of credentials for multiple systems (NetBox, GitLab, cloud providers).

### Problem
Insecure credential management:
- Hardcoded credentials in code
- Credentials in version control
- No credential rotation
- Difficult audit trail
- Compliance violations

### Solution
Integrate OpenBao/Vault for centralized secrets management:

**Architecture**:
```
┌─────────────────────────────────────────┐
│  OpenBao/Vault                          │
│  ┌───────────────────────────────────┐  │
│  │  Secret Engines                   │  │
│  │  ├── secret/nautilus/netbox_token │  │
│  │  ├── secret/nautilus/gitlab_token │  │
│  │  └── secret/nautilus/vsphere_creds│  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
              ↓
        (authenticate)
              ↓
┌─────────────────────────────────────────┐
│  Nautilus CLI                           │
│  - AppRole authentication               │
│  - Token caching                        │
│  - Automatic credential retrieval       │
└─────────────────────────────────────────┘
```

### Implementation

**Configuration**:
```python
# config.py
class ConfigManager:
    def __init__(self):
        self.vault_client = self._get_vault_client()
    
    def _get_vault_client(self):
        """Initialize Vault client with appropriate auth."""
        vault_url = os.getenv('VAULT_ADDR')
        
        # Try AppRole first (production)
        role_id = os.getenv('VAULT_ROLE_ID')
        secret_id = os.getenv('VAULT_SECRET_ID')
        
        if role_id and secret_id:
            client = hvac.Client(url=vault_url)
            client.auth.approle.login(
                role_id=role_id,
                secret_id=secret_id
            )
            return client
        
        # Fall back to token (development)
        token = os.getenv('VAULT_TOKEN')
        if token:
            return hvac.Client(url=vault_url, token=token)
        
        raise ValueError("No Vault authentication method available")
    
    def get_secret(self, path):
        """Retrieve secret from Vault."""
        secret = self.vault_client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point='secret'
        )
        return secret['data']['data']
    
    def get_netbox_token(self):
        """Get NetBox API token from Vault."""
        return self.get_secret('nautilus/netbox_token')['token']
```

**Usage in Handlers**:
```python
# netbox_handler.py
class NetBoxHandler:
    def __init__(self):
        config = get_config_manager()
        self.api = pynetbox.api(
            url=config.get_netbox_url(),
            token=config.get_netbox_token()  # From Vault
        )
```

**Credential Rotation**:
```bash
# Rotate NetBox token
vault kv put secret/nautilus/netbox_token token="new_token_value"

# Nautilus automatically uses new token on next run
nautilus status
```

### Benefits
- ✅ Centralized credential management
- ✅ No credentials in code or version control
- ✅ Audit trail of credential access
- ✅ Easy credential rotation
- ✅ Multiple authentication methods
- ✅ Compliance-ready

### Trade-offs
- ⚠️ Additional infrastructure (Vault server)
- ⚠️ Complexity in setup
- ⚠️ Requires network access to Vault

### Status
✅ **Fully Implemented** with comprehensive testing

### Related Patterns
- None (foundational security pattern)

---

## Cross-Cutting Concerns

### Observability

**Logging Strategy**:
```python
import logging

# Structured logging
logger = logging.getLogger('nautilus')
logger.info(
    'Creating VM',
    extra={
        'work_package': work_package_tag,
        'vm_name': vm_name,
        'user': current_user,
        'action': 'create_vm'
    }
)
```

**Audit Trail**:
- All CLI commands logged
- NetBox changes tracked
- Git commits provide code audit trail
- Vault access logged

### Error Handling

**Graceful Degradation**:
```python
try:
    netbox_vm = create_vm(name, role, cluster)
except NetBoxAPIError as e:
    logger.error(f"Failed to create VM in NetBox: {e}")
    # Don't fail entire operation
    # Allow manual NetBox update later
except Exception as e:
    logger.critical(f"Unexpected error: {e}")
    raise
```

### Testing Strategy

**Test Pyramid**:
```
         ┌─────────────────┐
         │  E2E Tests      │  ← Full workflow tests
         └─────────────────┘
       ┌───────────────────────┐
       │  Integration Tests    │  ← API interactions
       └───────────────────────┘
   ┌─────────────────────────────────┐
   │     Unit Tests                  │  ← Individual functions
   └─────────────────────────────────┘
```

---

## Conclusion

These architectural patterns provide a comprehensive framework for building infrastructure automation platforms that are:
- **Safe**: Multiple validation and approval gates
- **Auditable**: Complete traceability from planning to deployment
- **Scalable**: Supports large, complex infrastructure
- **Maintainable**: Clear conventions and documentation
- **AI-Ready**: Designed for AI agent orchestration

The patterns are proven in the Nautilus implementation and can be adapted to other infrastructure automation contexts.

---

**Document Version**: 1.0  
**Last Updated**: December 2024  
**Maintained By**: Infrastructure Automation Team
