# Infrastructure as Code Concepts

### **SECTION 1: WHAT IS INFRASTRUCTURE AS CODE (IaC)?**

*Managing infrastructure through code instead of manual processes.*

### **1. IaC Overview**

- **The Rule:** Infrastructure as Code means defining, provisioning, and managing infrastructure using **machine-readable configuration files** instead of manual CLI commands or clickops.
- **Core Idea:** Treat infrastructure the same way you treat application code -- write it, version it, review it, test it.

**Benefits of IaC:**

- **Versioning:** Store infrastructure definitions in version control (Git). Track every change, blame, and rollback.
- **Automation:** Provision entire environments with a single command. No manual steps.
- **Consistency:** Same config = same result. Eliminates "it works on my machine" for infrastructure.
- **Reusability:** Write modules once, reuse across projects, teams, and environments (dev/staging/prod).
- **Collaboration:** Teams review infrastructure changes via pull requests, just like application code.

**When IaC Matters:**

- **Drift Prevention:** Manual changes cause configuration drift. IaC detects and corrects drift automatically.
- **Auditability:** Every change is a commit. You have a full history of who changed what and when.
- **Reproducibility:** Destroy and recreate identical environments on demand (disaster recovery, testing).

*Exam Trigger:* "Ensure infrastructure changes are tracked and auditable" -> IaC with version control.

---

### **2. Declarative vs. Imperative Approaches**

**The Rule:** Terraform is **declarative**. You describe the **desired end state**, not the steps to get there. Terraform figures out the how.

**Declarative (What):**

- You define: "I want 3 EC2 instances behind a load balancer."
- The tool determines what actions to take (create, update, destroy) to reach that state.
- **Examples:** Terraform (HCL), CloudFormation (JSON/YAML).

**Imperative (How):**

- You define: "First create a VPC, then create a subnet, then launch an instance..."
- You specify the exact sequence of steps.
- **Examples:** Bash scripts, AWS CLI scripts, Ansible playbooks (procedural).

```hcl
# Declarative (Terraform) - WHAT you want
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  count         = 3
}

# If you later change count = 3 to count = 5,
# Terraform creates only 2 more (not 5 new ones).
```

*Exam Trigger:* "Describe the desired state of infrastructure" -> Declarative approach -> Terraform.

*Exam Trap:* Terraform is **not** imperative. It does **not** require you to specify step-by-step instructions. If a question describes writing ordered commands to build infrastructure, that is **imperative**, not Terraform's model.

---

### **3. Idempotent Behavior**

**The Rule:** Applying the same Terraform configuration multiple times produces the **exact same result** with **no unintended side effects**.

**How It Works:**

- First `terraform apply`: Creates 3 instances.
- Second `terraform apply` (no config change): **No changes.** Terraform detects current state matches desired state.
- Third `terraform apply` (change instance type): **Only updates** the instance type. Nothing else touched.

**Why It Matters:**

- Safe to re-run. No duplicated resources.
- Terraform compares **desired state** (config) vs. **current state** (state file) and only makes the **delta**.

*Exam Trigger:* "Running the same configuration multiple times without side effects" -> Idempotent.

---

### **4. Mutable vs. Immutable Infrastructure**

**The Rule:** Terraform encourages **immutable** infrastructure. Instead of patching a running server, you **replace** it with a new one built from the updated configuration.

| **Approach** | **Mutable** | **Immutable** |
| --- | --- | --- |
| **How Updates Work** | Modify existing resources in place (SSH in, install patches) | Destroy old resource, create new one from updated config |
| **Drift Risk** | **High** (Manual changes accumulate) | **Low** (Every deploy is a clean build) |
| **Consistency** | Degrades over time (snowflake servers) | Guaranteed (identical builds every time) |
| **Rollback** | Difficult (undo manual changes?) | Easy (redeploy previous version) |
| **Tools** | Ansible, Chef, Puppet (config mgmt) | Terraform, Packer (image-based deploys) |

**Example:**

- **Mutable:** SSH into server, run `apt-get update && apt-get upgrade`.
- **Immutable:** Build a new AMI with Packer, update the `ami` in Terraform config, run `terraform apply`. Old instance destroyed, new one created.

*Exam Trigger:* "Replace infrastructure instead of modifying it" -> Immutable infrastructure.

*Exam Trap:* Terraform **can** update resources in place (e.g., changing a tag), but its philosophy and best practice is immutable. If a resource change requires replacement, Terraform will destroy and recreate it automatically.

---

### **SECTION 2: TERRAFORM'S ROLE AND ADVANTAGES**

*Why Terraform specifically, and how it compares.*

### **5. Terraform in the Infrastructure Lifecycle**

**The Rule:** Terraform manages the full infrastructure lifecycle: **Provision -> Update -> Destroy**.

**Lifecycle Stages:**

- **Day 0 (Provision):** Write config, run `terraform apply`, infrastructure is created from scratch.
- **Day 1+ (Manage):** Modify config, run `terraform apply`, Terraform calculates and applies only the changes needed.
- **End of Life (Destroy):** Run `terraform destroy`, all managed resources are removed cleanly.

**How Terraform Tracks State:**

- **State File (`terraform.tfstate`):** Terraform records what it created. This is how it knows what exists and what needs to change.
- **Plan Before Apply:** `terraform plan` shows what will change **before** anything happens. Safe review step.

*Exam Trigger:* "Infrastructure lifecycle management from provisioning to decommissioning" -> Terraform.

---

### **6. Multi-Cloud Advantage**

**The Rule:** Terraform uses **providers** to interact with any cloud or service. One tool, one language (HCL), multiple platforms.

**Key Points:**

- **Provider Agnostic:** AWS, Azure, GCP, Kubernetes, Datadog, GitHub -- all managed with the same workflow.
- **Single Workflow:** `terraform init` -> `terraform plan` -> `terraform apply` works regardless of provider.
- **Multi-Cloud Deployments:** Define AWS and Azure resources in the same configuration.
- **No Vendor Lock-In:** Switch or mix cloud providers without learning a new tool.

```hcl
# Multi-cloud in a single config
provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "azurerm_virtual_machine" "web" {
  # Azure VM config here
}
```

*Exam Trigger:* "Single tool to manage resources across multiple cloud providers" -> Terraform.

*Exam Trap:* CloudFormation **cannot** manage Azure or GCP resources. If the question mentions multi-cloud, CloudFormation is **wrong**.

---

### **7. Terraform vs. Other Tools**

**The Rule:** Know what category each tool falls into and what makes Terraform different.

| **Tool** | **Type** | **Approach** | **Scope** | **Language** | **Key Limitation** |
| --- | --- | --- | --- | --- | --- |
| **Terraform** | IaC (Provisioning) | Declarative | Multi-cloud | HCL | State file management required |
| **CloudFormation** | IaC (Provisioning) | Declarative | **AWS only** | JSON/YAML | No multi-cloud support |
| **Pulumi** | IaC (Provisioning) | Declarative (code-based) | Multi-cloud | Python, TypeScript, Go, etc. | Requires programming language knowledge |
| **Ansible** | Config Management | **Procedural/Imperative** | Multi-cloud | YAML (Playbooks) | Not designed for provisioning lifecycle; no state tracking |
| **Chef** | Config Management | **Procedural/Imperative** | Multi-cloud | Ruby (DSL) | Config management, **not** IaC provisioning |
| **Puppet** | Config Management | Declarative (for config) | Multi-cloud | Puppet DSL | Config management, **not** IaC provisioning |

**Key Distinctions:**

- **Provisioning vs. Configuration Management:**
    - **Terraform:** Provisions infrastructure (creates VPCs, VMs, databases).
    - **Ansible/Chef/Puppet:** Configures software on existing infrastructure (installs packages, manages files).
- **Terraform vs. CloudFormation:** Both declarative IaC, but CloudFormation is **AWS-only**. Terraform is multi-cloud.
- **Terraform vs. Pulumi:** Both multi-cloud IaC, but Terraform uses HCL while Pulumi uses general-purpose programming languages.
- **Terraform vs. Ansible:** Terraform is declarative with state tracking. Ansible is procedural without built-in state management.

*Exam Trigger:* "Provision cloud infrastructure using a declarative, multi-cloud tool" -> Terraform.

*Exam Trap:* Ansible **can** create cloud resources, but it is primarily a **configuration management** tool. It does **not** maintain a state file and is **not** idempotent by design the way Terraform is.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **IaC benefits?** -> Versioning, automation, consistency, reusability, collaboration.
2. **Terraform approach?** -> Declarative (desired state, not step-by-step).
3. **Same config applied twice?** -> Idempotent (no changes on second run).
4. **Replace servers instead of patching?** -> Immutable infrastructure.
5. **One tool for AWS + Azure + GCP?** -> Terraform (multi-cloud via providers).
6. **AWS-only IaC tool?** -> CloudFormation.
7. **IaC with Python/TypeScript?** -> Pulumi.
8. **Install packages on existing servers?** -> Configuration management (Ansible, Chef, Puppet), not Terraform.
9. **Track infrastructure changes with Git?** -> IaC + version control.
10. **Prevent configuration drift?** -> IaC (Terraform detects and corrects drift).
11. **Infrastructure lifecycle (provision, update, destroy)?** -> Terraform.
12. **Terraform plans before applying?** -> `terraform plan` shows changes before execution.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "Multi-Cloud Requirement" (Terraform vs. CloudFormation)**

**The Situation:** A company runs workloads on both AWS and Azure. They want a single IaC tool to manage infrastructure across both cloud providers using one consistent workflow. The team currently uses AWS CloudFormation for their AWS resources.

**The Options:**

A. Continue using AWS CloudFormation and add Azure Resource Manager templates for Azure.

B. Use Terraform to manage both AWS and Azure resources.

C. Use Ansible to provision and manage infrastructure on both clouds.

D. Use AWS CDK with cross-cloud plugins.

**The Logic:**

- **Trap:** CloudFormation (Option A) is **AWS-only**. You would need two separate tools and two different languages. Does not meet the "single tool" requirement.
- **Trap:** Ansible (Option C) can interact with both clouds but is a **configuration management** tool, not purpose-built for infrastructure provisioning. It lacks state tracking and a plan-before-apply workflow.
- **The Fix:** **Option B**. **Terraform** supports both AWS and Azure through providers. Single language (HCL), single workflow (`init/plan/apply`), multi-cloud. Meets every requirement.

---

### **Scenario 2: The "Declarative vs. Imperative" (Understanding Terraform's Model)**

**The Situation:** A team currently manages infrastructure using a long Bash script that runs AWS CLI commands in sequence: first creates a VPC, then subnets, then security groups, then EC2 instances. If a step fails midway, they must manually clean up and re-run. They want a more reliable approach.

**The Options:**

A. Add error handling and retry logic to the Bash script.

B. Rewrite the script in Python using the AWS SDK (Boto3).

C. Use Terraform to define the desired infrastructure state declaratively.

D. Use Ansible playbooks to run the same AWS CLI commands.

**The Logic:**

- **Trap:** Adding retry logic (Option A) still uses an **imperative** approach. Failures still require manual intervention.
- **Trap:** Python/Boto3 (Option B) is still imperative scripting, just in a different language.
- **The Fix:** **Option C**. **Terraform** is **declarative**. You define the desired end state. Terraform handles ordering, dependencies, and partial failure recovery. If a run fails midway, re-running `terraform apply` picks up where it left off. No manual cleanup needed.

---

### **Scenario 3: The "Configuration Drift" (Why IaC Matters)**

**The Situation:** A company provisions servers manually through the AWS Console. Over time, different team members make ad-hoc changes -- one adds a security group rule, another changes an instance type, another modifies a route table. Now no one knows the actual state of production, and deploying a new environment takes weeks of manual work.

**The Options:**

A. Document all manual changes in a shared spreadsheet.

B. Implement Infrastructure as Code with Terraform and store configs in Git.

C. Take regular AMI snapshots of all servers.

D. Restrict console access to one team member.

**The Logic:**

- **Trap:** A spreadsheet (Option A) becomes outdated immediately. Manual documentation does not prevent drift.
- **Trap:** AMI snapshots (Option C) capture server state but not network, security, or other infrastructure configuration.
- **The Fix:** **Option B**. **IaC with Terraform** eliminates drift because the config files **are** the source of truth. Any manual change is detected on the next `terraform plan`. Storing in Git provides auditability, versioning, and collaboration via pull requests. New environments are reproducible with a single `terraform apply`.

---

### **Scenario 4: The "Immutable vs. Mutable" (Replacing Instead of Patching)**

**The Situation:** A company runs a fleet of web servers. A critical security patch is released. The current process is to SSH into each server and run update commands. This takes hours, some servers fail to update, and the fleet ends up in an inconsistent state.

**The Options:**

A. Write an Ansible playbook to automate the SSH-and-patch process across all servers.

B. Build a new AMI with the patch using Packer, update the Terraform config with the new AMI, and run `terraform apply` to replace all servers.

C. Manually patch each server and document the changes.

D. Ignore the patch and rely on security groups for protection.

**The Logic:**

- **Trap:** Ansible (Option A) automates the patching but still follows a **mutable** approach. Servers may still end up inconsistent if some updates fail.
- **Trap:** Manual patching (Option C) is the current broken process.
- **The Fix:** **Option B**. This is the **immutable** approach. Build a new golden image, update the Terraform config, and replace all servers. Every server is identical, fully patched, and consistent. Rollback is easy -- just point Terraform back to the previous AMI.

---

### **Scenario 5: The "Right Tool for the Job" (Terraform vs. Config Management)**

**The Situation:** A team needs to: (1) provision an AWS VPC, subnets, and EC2 instances, and (2) install and configure Nginx on those instances after they launch. A junior engineer suggests using Ansible for everything.

**The Options:**

A. Use Ansible for both provisioning the AWS infrastructure and configuring Nginx.

B. Use Terraform to provision the infrastructure and Ansible to configure Nginx on the instances.

C. Use Terraform for both provisioning the infrastructure and configuring Nginx.

D. Use CloudFormation to provision the infrastructure and Chef to configure Nginx.

**The Logic:**

- **Trap:** Ansible for everything (Option A) **can** provision AWS resources, but it lacks state management. If you run the playbook twice, it may try to create duplicate resources. Ansible is not designed for infrastructure lifecycle management.
- **Trap:** Terraform for Nginx config (Option C) is possible with provisioners, but provisioners are a **last resort** in Terraform. Terraform is not a configuration management tool.
- **The Fix:** **Option B**. Use the **right tool for each job**. Terraform excels at **provisioning** infrastructure (declarative, state-tracked, idempotent). Ansible excels at **configuring** software on existing servers. Together they cover the full stack. This is the recommended pattern in the Terraform ecosystem.
