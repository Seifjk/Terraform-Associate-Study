# Terraform Associate (004) Study Guide

**Exam:** HashiCorp Certified: Terraform Associate (004)
**Format:** ~57 questions, 60 min, ~70% to pass
**Question Types:** Multiple choice, multiple answer, true/false
**Cost:** $70.50 USD
**Valid:** 2 years
**Terraform Version:** 1.12
**Proctored:** Online via Certiverse

---

## Your Advantage

You already use Terraform daily in DevOps. This exam tests **concepts and terminology**, not hands-on skills. Most of it you already know — you just need to learn the exact wording HashiCorp uses and fill gaps around HCP Terraform (Terraform Cloud) and some edge-case behaviors.

**Estimated prep time:** 10-14 days at 1 hour/day. This is not SAA-level effort.

---

## The 8 Exam Objectives

| # | Objective | Weight* | Your Likely Gap |
|---|-----------|---------|-----------------|
| 1 | IaC Concepts | Low | None — you live this |
| 2 | Terraform Fundamentals | Medium | Provider lock files, state concepts |
| 3 | Core Workflow | Medium | Exact CLI flag behaviors |
| 4 | Terraform Configuration | High | `depends_on`, lifecycle rules, custom conditions, ephemeral values |
| 5 | Modules | Medium | Module versioning, registry semantics |
| 6 | State Management | High | `moved`/`removed`/`import` blocks (new in 004) |
| 7 | Maintain Infrastructure | Medium | Drift detection, taint replacement |
| 8 | HCP Terraform | Medium-High | Workspaces, VCS-driven workflow, policies |

*HashiCorp doesn't publish official weights. Estimates based on question distribution in practice exams.

---

## What's New in 004 (vs 003)

These are the topics that catch people who studied old material:

1. **`depends_on` + `lifecycle { create_before_destroy }`** — know exactly when to use each
2. **Custom configuration conditions** — `precondition`/`postcondition` blocks, `check` blocks with `assert`
3. **Ephemeral values & write-only arguments** — new concept, prevents sensitive values from being stored in state
4. **HCP Terraform workspaces & projects** — organizing workspaces into projects, execution modes (CLI-driven vs VCS-driven)
5. **`moved`, `removed`, `import` blocks** — declarative state management (replacing CLI commands)

---

## Study Plan (14 Days)

**Start:** After SAA exam (late April 2026)
**Pace:** 1 hour/day
**Exam:** Book for ~2 weeks after you start

---

### Day 1 — Objective 1: IaC Concepts

**Time: 30 min study + 30 min practice**

This is your easiest day. Skim to confirm you know the terminology:

- Benefits of IaC: versioning, automation, consistency, reusability, collaboration
- Declarative vs Imperative: Terraform is **declarative** (you define desired state, not steps)
- Idempotent: applying the same config multiple times = same result
- Mutable vs Immutable infrastructure: Terraform encourages **immutable** (replace, don't patch)
- Multi-cloud: Terraform works with any provider (AWS, Azure, GCP, K8s, etc.)
- Terraform vs other tools: CloudFormation (AWS-only), Ansible (procedural/config mgmt), Pulumi (code-based)

**Practice:** HashiCorp sample questions for Objective 1.

---

### Day 2 — Objective 2: Terraform Fundamentals (Part 1)

**Providers & Resources (60 min)**

- **Providers:** Plugins that interact with APIs (aws, azurerm, google, kubernetes)
  - `required_providers` block in `terraform {}` — declares provider + version constraint
  - **Dependency lock file** (`.terraform.lock.hcl`): locks provider versions. Commit to VCS. Updated by `terraform init -upgrade`.
  - Provider aliases: use multiple configurations of the same provider (e.g., two AWS regions)
- **Resources:** The core building block. `resource "aws_instance" "web" {}`
  - Each resource belongs to a provider
  - Resource addressing: `aws_instance.web`, `module.vpc.aws_subnet.public[0]`
- **Data Sources:** Read existing infrastructure. `data "aws_ami" "latest" {}`
  - Read-only — never creates/modifies resources

**Write from memory:** What goes in `.terraform.lock.hcl`? When is it updated? Should you commit it?

---

### Day 3 — Objective 2: Terraform Fundamentals (Part 2)

**State & Backends (60 min)**

- **State file** (`terraform.tfstate`): Maps config to real-world resources. JSON file.
  - Contains resource IDs, attributes, metadata
  - **Sensitive data IS stored in state** (even if marked `sensitive`) — secure your backend
- **Backends:** Where state is stored
  - **Local:** Default. State on disk. No locking (unless using a local lock file).
  - **Remote (S3 + DynamoDB):** State in S3, locking via DynamoDB. Standard AWS pattern.
  - **HCP Terraform:** Built-in remote state, locking, encryption
- **State locking:** Prevents concurrent modifications. DynamoDB for S3 backend. Automatic in HCP Terraform.
- **`terraform refresh`:** Reconcile state with real infrastructure (now implicit in `plan`/`apply`)

**Exam traps:**
- State is **not** encrypted at rest by default (local backend). S3 backend + encryption = secure.
- `terraform.tfstate` should **never** be committed to version control.
- State locking prevents **corruption**, not unauthorized access.

---

### Day 4 — Objective 3: Core Workflow

**CLI Commands Deep Dive (60 min)**

The workflow: **Write → Init → Plan → Apply → Destroy**

| Command | What It Does | Key Flags |
|---------|-------------|-----------|
| `terraform init` | Downloads providers, initializes backend, installs modules | `-upgrade` (update providers), `-migrate-state` (change backend) |
| `terraform fmt` | Formats `.tf` files to canonical style | `-check` (check only, no write), `-recursive` |
| `terraform validate` | Validates config syntax (no API calls) | Runs after `init` |
| `terraform plan` | Preview changes, no modifications | `-out=plan.tfplan` (save plan), `-destroy` (preview destroy) |
| `terraform apply` | Execute changes | `-auto-approve` (skip confirmation), can pass saved plan file |
| `terraform destroy` | Destroy all managed resources | `-target=resource` (destroy specific resource) |
| `terraform output` | Display output values | `-json` (JSON format) |
| `terraform show` | Show current state or plan | |
| `terraform state list` | List resources in state | |
| `terraform state show` | Show details of specific resource | |
| `terraform state mv` | Move/rename resource in state | |
| `terraform state rm` | Remove resource from state (doesn't destroy it) | |
| `terraform taint` | Mark resource for recreation (DEPRECATED → use `-replace`) | |
| `terraform import` | Import existing resource into state (CLI method) | |

**Key concepts:**
- `plan` + `apply` with saved plan = safe CI/CD pattern (plan in PR, apply on merge)
- `terraform plan -replace=aws_instance.web` replaces `taint` (newer, preferred)
- `-target` is for **exceptional use only** — not for normal workflow

**Write from memory:** Full workflow order. What does `init` download? Difference between `validate` and `plan`.

---

### Day 5 — Objective 4: Configuration (Part 1)

**Variables, Outputs, Types (60 min)**

- **Input Variables:**
  ```hcl
  variable "instance_type" {
    type        = string
    default     = "t3.micro"
    description = "EC2 instance type"
    sensitive   = true       # Hides from CLI output
    nullable    = false      # Cannot be null
    validation {
      condition     = contains(["t3.micro", "t3.small"], var.instance_type)
      error_message = "Must be t3.micro or t3.small."
    }
  }
  ```
- **Variable precedence** (lowest to highest):
  1. Default value in declaration
  2. `terraform.tfvars` / `*.auto.tfvars`
  3. `-var-file=filename`
  4. `-var 'key=value'`
  5. `TF_VAR_name` environment variable
  6. **Highest:** `-var` on CLI (overrides everything)

  *Exam loves testing this order.*

- **Output Values:** `output "ip" { value = aws_instance.web.public_ip }`
  - `sensitive = true` hides from CLI but still in state
- **Local Values:** `locals { name = "${var.prefix}-web" }` — computed values, not inputs
- **Complex Types:** `list(string)`, `map(string)`, `object({...})`, `tuple([...])`, `set(string)`

---

### Day 6 — Objective 4: Configuration (Part 2)

**Dependencies, Lifecycle, Conditions (60 min)**

- **Implicit dependencies:** Terraform auto-detects via references (`resource.name.attr`)
- **Explicit dependencies:** `depends_on = [aws_iam_role.example]` — use when Terraform can't infer the dependency
- **Lifecycle rules:**
  ```hcl
  lifecycle {
    create_before_destroy = true    # Create new before destroying old
    prevent_destroy       = true    # Block terraform destroy
    ignore_changes        = [tags]  # Don't detect drift on these
    replace_triggered_by  = [...]   # Force replacement when dependency changes
  }
  ```
- **Custom conditions (NEW in 004):**
  ```hcl
  resource "aws_instance" "web" {
    # ...
    precondition {
      condition     = var.instance_type != "t3.nano"
      error_message = "t3.nano is too small for production."
    }
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
  ```
  - `precondition` = validate **before** apply
  - `postcondition` = validate **after** apply
- **`check` blocks with `assert`:** Continuous validation (checked every plan/apply, but don't block)
- **Ephemeral values (NEW in 004):** Values that exist only during plan/apply, never stored in state. Write-only arguments prevent sensitive data from persisting.

- **Provisioners:** `local-exec`, `remote-exec`, `file`. HashiCorp considers these a **last resort**. Prefer provider-native resources.
- **`for_each` vs `count`:**
  - `count` = create N identical resources (index-based)
  - `for_each` = create resources from a map/set (key-based, more stable)
  - *Exam trap:* `for_each` doesn't work with lists — convert with `toset()`

---

### Day 7 — Objective 5: Modules

**Modules Deep Dive (60 min)**

- **What is a module?** Any directory with `.tf` files. Root module = your working directory.
- **Child modules:** Called from root via `module "name" { source = "..." }`
- **Module sources:**
  - Local path: `source = "./modules/vpc"`
  - Terraform Registry: `source = "hashicorp/consul/aws"` (version required)
  - GitHub: `source = "github.com/org/repo"`
  - S3, GCS, HTTP URLs
- **Module versioning:** `version = "~> 3.0"` (only works with registry modules)
  - `= 3.0.0` exact
  - `~> 3.0` allows 3.x but not 4.0
  - `>= 3.0, < 4.0` range
- **Module inputs:** Variables declared in module, passed as arguments
- **Module outputs:** `output` blocks in module, accessed as `module.name.output_name`
- **`terraform get`:** Downloads modules. `terraform init` also does this.

**Best practices:**
- Modules should be self-contained (own providers, variables, outputs)
- Use published registry modules when possible
- Pin module versions in production

**Write from memory:** 4 module sources. How versioning constraints work. Difference between `terraform get` and `terraform init`.

---

### Day 8 — Objective 6: State Management

**State Operations (60 min)**

- **`terraform state` commands:** `list`, `show`, `mv`, `rm`, `pull`, `push`
  - `state mv` = rename or move resource (e.g., refactoring without destroy/recreate)
  - `state rm` = remove from state (resource continues to exist, Terraform just stops managing it)

- **Declarative state management (NEW in 004):**
  ```hcl
  # moved block — refactor without destroy
  moved {
    from = aws_instance.old_name
    to   = aws_instance.new_name
  }

  # removed block — stop managing without destroying
  removed {
    from = aws_instance.legacy
    lifecycle {
      destroy = false
    }
  }

  # import block — bring existing resource under management
  import {
    to = aws_instance.web
    id = "i-1234567890abcdef0"
  }
  ```
  - `moved` replaces `terraform state mv` (declarative, in config, reviewable in PR)
  - `removed` replaces `terraform state rm`
  - `import` block replaces `terraform import` CLI command
  - **Exam will test these new blocks heavily** — they're a key 004 addition

- **State file security:**
  - Use remote backend with encryption (S3 + KMS, HCP Terraform)
  - Enable state locking (DynamoDB for S3)
  - Restrict access via IAM/RBAC
  - Never commit `terraform.tfstate` to git

---

### Day 9 — Objective 7: Maintain Infrastructure

**Day-to-Day Operations (60 min)**

- **Drift detection:** `terraform plan` shows differences between state and real infrastructure
  - If someone changes a resource manually → plan shows diff → apply corrects it
- **Resource replacement:**
  - `terraform apply -replace=aws_instance.web` (preferred over deprecated `taint`)
  - When: corrupted resource, force fresh provisioning
- **`terraform refresh`:** Update state to match real infrastructure. Now implicit in `plan`/`apply`. Standalone use is discouraged.
- **Debugging:**
  - `TF_LOG=DEBUG terraform plan` — enable verbose logging
  - Log levels: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`
  - `TF_LOG_PATH=terraform.log` — write logs to file
- **Workspaces (CLI):**
  - `terraform workspace new dev` / `terraform workspace select dev`
  - Each workspace has its own state file
  - Use `terraform.workspace` in config to differentiate (e.g., instance size per env)
  - *Not the same as HCP Terraform workspaces*
- **Sensitive data:**
  - `sensitive = true` on variables/outputs hides from CLI output
  - Still stored in state file — secure your backend
  - Ephemeral values (004) = never written to state at all

---

### Day 10 — Objective 8: HCP Terraform (Part 1)

**HCP Terraform Basics (60 min)**

This is probably your biggest gap since DevOps teams often use open-source Terraform with S3 backend rather than HCP Terraform (formerly Terraform Cloud).

- **HCP Terraform** = SaaS platform for team Terraform use (formerly Terraform Cloud)
- **Terraform Enterprise** = Self-hosted version of HCP Terraform

- **Workspaces (HCP):** Not the same as CLI workspaces!
  - Each workspace = one root module + one state file + variables + run history
  - **Projects:** Group related workspaces (e.g., all workspaces for "payments" service)

- **Execution Modes:**
  | Mode | Where Runs Execute | Use Case |
  |------|-------------------|----------|
  | **Remote (default)** | HCP Terraform runners | Standard team workflow |
  | **Local** | Your machine, state stored remote | Debugging, migration |
  | **Agent** | Self-hosted agents in your network | Private infrastructure |

- **VCS-driven workflow:**
  - Connect workspace to GitHub/GitLab repo
  - PR triggers speculative `plan` (shown in PR checks)
  - Merge to main triggers `apply`
  - *Exam loves this workflow*

- **CLI-driven workflow:**
  - Run `terraform plan`/`apply` from local CLI
  - Execution happens on HCP Terraform runners (remote execution)
  - State stored in HCP Terraform

---

### Day 11 — Objective 8: HCP Terraform (Part 2)

**Governance & Collaboration (60 min)**

- **Sentinel / OPA policies:**
  - Policy-as-code: enforce rules before apply (e.g., "no public S3 buckets")
  - Sentinel = HashiCorp's policy language
  - OPA = Open Policy Agent (open-source alternative)
  - Policies run between plan and apply (policy check step)

- **Run workflow in HCP Terraform:**
  1. Plan
  2. Cost Estimation (optional)
  3. Policy Check (Sentinel/OPA)
  4. Apply (requires approval if configured)

- **Team access & permissions:**
  - Organizations → Teams → Workspace permissions
  - Permissions: Read, Plan, Write, Admin
  - SSO/SAML integration for enterprise

- **Private Registry:**
  - Host private modules and providers within your organization
  - Same interface as public Terraform Registry
  - Version modules, control access

- **Variable Sets:**
  - Reusable groups of variables applied across multiple workspaces
  - Example: AWS credentials shared across all AWS workspaces

---

### Day 12 — Full Review

**All 8 Objectives — Rapid Fire (60 min)**

Go through this checklist. Cover the answer, say it out loud:

1. Terraform is declarative or imperative? → **Declarative**
2. Where are provider versions locked? → **`.terraform.lock.hcl`**
3. Should you commit `.terraform.lock.hcl`? → **Yes**
4. Should you commit `terraform.tfstate`? → **Never**
5. Variable precedence highest to lowest? → **CLI `-var` > env `TF_VAR_` > `-var-file` > `*.auto.tfvars` > `terraform.tfvars` > default**
6. `count` vs `for_each`? → **`count` = index-based, `for_each` = key-based (more stable)**
7. `depends_on` when? → **When Terraform can't infer implicit dependency**
8. `create_before_destroy` does what? → **Creates replacement before destroying original**
9. `precondition` vs `postcondition`? → **Pre = before apply, Post = after apply**
10. `moved` block does what? → **Rename/refactor resource without destroy (replaces `state mv`)**
11. `removed` block does what? → **Stop managing resource without destroying (replaces `state rm`)**
12. `import` block does what? → **Bring existing resource into state (replaces CLI `import`)**
13. Module version constraint `~> 3.0`? → **Allows 3.x, not 4.0**
14. CLI workspace vs HCP workspace? → **CLI = same config, different state. HCP = full workspace with variables, runs, VCS**
15. VCS-driven workflow? → **PR = speculative plan, merge = apply**
16. Sentinel/OPA runs when? → **Between plan and apply**
17. `terraform plan -replace=X`? → **Force recreate specific resource (replaces taint)**
18. Ephemeral values? → **Never stored in state (new in 004)**
19. `terraform validate` vs `plan`? → **Validate = syntax check, no API. Plan = full API check + diff.**
20. `sensitive = true` on variable? → **Hides from CLI output, still in state file**

If you miss more than 5 → review those objectives before mock exams.

---

### Day 13 — Mock Exam #1

**Take a full practice exam (60 min)**

- Use: [Terraform Associate 004 Practice Exams (Bryan Krausen)](https://www.udemy.com/course/terraform-associate-004-practice-exams/) or [6 Practice Tests 2026](https://www.udemy.com/course/terraform-associate-004-certification-6-practice-tests-2026/)
- Simulate real conditions: 57 questions, 60 minutes, no notes
- **Target: 75%+** (exam passing is ~70%, aim higher)

**After:** Review every wrong answer. Note which objective it falls under. That's your study focus for tomorrow.

---

### Day 14 — Mock Exam #2 + Final Review

**Morning (30 min):** Review weak topics from Mock #1
**Take Mock #2 (60 min):** Different practice exam from the same set

| Score | Action |
|-------|--------|
| 80%+ | Book exam for this week. You're ready. |
| 70-79% | Book exam for next week. Review Objectives 4, 6, 8 (where most people fail). |
| <70% | Take 2-3 more days. Focus on HCP Terraform and new 004 topics. |

---

## Resources

**Official (Free):**
- [Exam Content List (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-review-004) — maps every objective to a doc page
- [Learning Path (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-study-004) — HashiCorp's recommended study order
- [Sample Questions (004)](https://developer.hashicorp.com/terraform/tutorials/certification-004/associate-questions-004) — official practice questions

**Practice Exams (Paid — buy at least one):**
- [Bryan Krausen — 004 Practice Exams](https://www.udemy.com/course/terraform-associate-004-practice-exams/) — ~350 questions, highly recommended
- [6 Practice Tests 2026](https://www.udemy.com/course/terraform-associate-004-certification-6-practice-tests-2026/) — 6 full timed exams

**Video Course (Optional):**
- If you want a video walkthrough, any 004-specific Udemy course works. But with your DevOps background, the docs + practice exams are probably enough.

---

## Key Differences From SAA

| | SAA-C03 | Terraform Associate 004 |
|---|---------|------------------------|
| **Duration** | 130 min | 60 min |
| **Questions** | 65 | ~57 |
| **Difficulty** | Medium-Hard | Easy-Medium |
| **Prep time** | 3-4 weeks | 1-2 weeks |
| **Your advantage** | Some AWS knowledge | You use Terraform daily |
| **Biggest gap** | Cost/DR/Migration | HCP Terraform features |
| **Hands-on?** | No | No (but helps) |

---

## Quick Reference: Exam Traps

1. **`terraform.tfstate` in git?** → NEVER. Use remote backend.
2. **`.terraform.lock.hcl` in git?** → YES. Always commit.
3. **`sensitive = true` protects state?** → NO. Still in state. Only hides CLI output.
4. **Provisioners recommended?** → NO. Last resort. Use provider-native resources.
5. **`count` vs `for_each` for maps?** → `for_each`. `count` is index-based (fragile).
6. **`terraform refresh` standalone?** → Discouraged. Use `plan`/`apply` instead.
7. **CLI workspaces = HCP workspaces?** → NO. Completely different concepts.
8. **`taint` command?** → DEPRECATED. Use `terraform apply -replace=resource`.
9. **`terraform import` CLI?** → Still works but `import` block is preferred (004).
10. **Module version constraints only work with?** → Terraform Registry modules.
