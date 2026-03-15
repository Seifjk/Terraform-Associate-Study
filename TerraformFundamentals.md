# Terraform Fundamentals

### **SECTION 1: PROVIDERS AND RESOURCES**

*The building blocks of every Terraform configuration.*

### **1. Providers**

- **The Rule:** Providers are plugins that let Terraform interact with APIs (AWS, Azure, GCP, etc.). Every resource belongs to a provider.
- **Installation:** Providers are downloaded during `terraform init` and stored in `.terraform/providers/`.

**Declaring Providers (required_providers block):**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.0, < 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

- **source:** Registry address in the format `namespace/type` (e.g., `hashicorp/aws`).
- **version:** Constraint on which provider versions are acceptable.

**Provider Version Constraints:**

| **Operator** | **Meaning** | **Example** | **Matches** |
| --- | --- | --- | --- |
| `= 5.0.0` | Exact version only | `= 5.0.0` | 5.0.0 only |
| `>= 5.0` | Minimum version | `>= 5.0` | 5.0, 5.1, 6.0, etc. |
| `~> 5.0` | Pessimistic (rightmost increments) | `~> 5.0` | >= 5.0, < 6.0 |
| `~> 5.0.0` | Pessimistic (patch only) | `~> 5.0.0` | >= 5.0.0, < 5.1.0 |
| `>= 3.0, < 4.0` | Compound constraint | `>= 3.0, < 4.0` | 3.x only |

- *Exam Trigger:* "`~> 5.0`" means any **5.x** version but **never 6.0**. The `~>` operator only allows the **rightmost** version component to increment.
- *Exam Trap:* `~> 5.0` and `~> 5.0.0` are **different**. `~> 5.0` allows 5.0 through 5.99. `~> 5.0.0` allows only 5.0.0 through 5.0.99.

---

### **2. Dependency Lock File (.terraform.lock.hcl)**

- **The Rule:** Created/updated during `terraform init`. Records the **exact** provider versions and **hashes** selected.
- **Purpose:** Ensures all team members and CI/CD pipelines use the **same** provider versions.
- **Location:** Root of the working directory (next to your `.tf` files).

**Key Facts:**

- **Contains:** Provider version, version constraints, and cryptographic hashes (h1:, zh:).
- **Created by:** `terraform init` (first run).
- **Updated by:** `terraform init -upgrade` (to pick up newer versions within constraints).
- **MUST commit to version control (VCS).** This is how teams stay in sync.
- **Does NOT contain:** Module versions (modules have their own lock mechanism).

*Exam Trigger:* "Ensure consistent provider versions across team" --> Commit `.terraform.lock.hcl` to VCS.

*Exam Trap:* The `.terraform/` **directory** should be in `.gitignore` (contains downloaded binaries). The `.terraform.lock.hcl` **file** should be committed.

---

### **3. Provider Aliases (Multiple Configurations)**

**The Rule:** Use `alias` to configure the same provider multiple times with different settings (e.g., two AWS regions).

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "east_server" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  # Uses default provider (us-east-1) - no provider argument needed
}

resource "aws_instance" "west_server" {
  provider      = aws.west
  ami           = "ami-0abcdef9876543210"
  instance_type = "t3.micro"
  # Uses aliased provider (us-west-2)
}
```

**Key Facts:**

- The **default** provider (no alias) is used when `provider` is not specified in a resource.
- Aliased providers **must** be explicitly referenced with `provider = <provider>.<alias>`.
- Common use cases: multi-region deployments, multi-account setups.

*Exam Trigger:* "Deploy resources in two AWS regions from one configuration" --> Provider alias.

---

### **4. Resources**

**The Rule:** Resources are the **core building block** of Terraform. Each resource block describes one or more infrastructure objects.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  tags = {
    Name = "WebServer"
  }
}
```

- **Syntax:** `resource "<provider>_<type>" "<local_name>" { ... }`
- **Provider type prefix:** The first word before the underscore identifies the provider (`aws_instance` --> `aws` provider).
- **Local name:** Used only within the Terraform configuration to reference this resource.

**Resource Addressing:**

| **Address** | **Meaning** |
| --- | --- |
| `aws_instance.web` | Resource type `aws_instance`, name `web` |
| `aws_instance.web[0]` | First instance when using `count` |
| `aws_instance.web["app"]` | Instance keyed `"app"` when using `for_each` |
| `module.vpc.aws_subnet.public[0]` | Resource inside a module named `vpc` |

**Resource Behavior (Lifecycle):**

- `terraform plan` --> Shows what will be created/changed/destroyed.
- `terraform apply` --> Creates/updates real infrastructure and records in state.
- `terraform destroy` --> Removes resources from real infrastructure and state.

---

### **5. Data Sources**

**The Rule:** Data sources **read** existing information from the provider. They **never create, update, or delete** anything.

```hcl
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.latest_amazon_linux.id
  instance_type = "t3.micro"
}
```

- **Syntax:** `data "<provider>_<type>" "<local_name>" { ... }`
- **Reference:** `data.<type>.<name>.<attribute>` (e.g., `data.aws_ami.latest_amazon_linux.id`).
- **Use Cases:** Look up AMI IDs, fetch existing VPCs, read IAM policies, query availability zones.

*Exam Trap:* Data sources are **read-only**. If the question says "fetch existing resource info without managing it" --> Data source. If "create/manage a resource" --> Resource block.

*Exam Trigger:* `data` block = read. `resource` block = create/manage.

---

### **SECTION 2: STATE AND BACKENDS**

*How Terraform tracks what it manages.*

### **1. State File (terraform.tfstate)**

- **The Rule:** The state file is Terraform's **record of reality**. It maps your configuration to real-world resources.
- **Format:** JSON.
- **Default Location:** Local file named `terraform.tfstate` in the working directory.

**What State Contains:**

- **Resource IDs:** The unique identifier of each real resource (e.g., EC2 instance ID `i-0abc123`).
- **Resource Attributes:** All attributes of the resource (IP addresses, ARNs, tags, etc.).
- **Metadata:** Provider configuration, resource dependencies.
- **Dependencies:** The dependency graph between resources.

**Why State Matters:**

- Without state, Terraform cannot know what it previously created.
- Terraform uses state to determine **what changed** between your config and reality.
- State enables `terraform plan` to show accurate diffs.

*Exam Trap:* **Sensitive data IS stored in state in plain text** even when you mark a variable or output as `sensitive = true`. The `sensitive` flag only prevents values from appearing in CLI output and logs. The state file itself **still contains the raw value**. This is why state files must be secured.

---

### **2. Backends**

**The Rule:** A backend determines **where** state is stored and **how** operations (plan/apply) are executed.

**Backend Configuration:**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Backend Types:**

| **Feature** | **Local** | **S3 + DynamoDB** | **HCP Terraform** |
| --- | --- | --- | --- |
| **State Storage** | Local disk | S3 bucket | HCP Terraform (managed) |
| **State Locking** | No | Yes (DynamoDB) | Yes (built-in) |
| **Encryption** | No (unless OS-level) | Yes (SSE-S3/SSE-KMS) | Yes (built-in) |
| **Sharing** | Not practical | Team via shared bucket | Team via workspace |
| **Remote Execution** | No | No | Yes (optional) |
| **Cost** | Free | S3 + DynamoDB charges | Free tier available |
| **Best For** | Learning/dev | Production teams on AWS | Any cloud, full platform |

*Exam Trigger:* "Team collaboration with state locking on AWS" --> S3 backend + DynamoDB table for locking.

*Exam Trigger:* "Managed platform with built-in locking and remote execution" --> HCP Terraform.

---

### **3. State Locking**

**The Rule:** State locking **prevents concurrent modifications** to the same state file. If two people run `terraform apply` simultaneously, locking ensures one waits for the other.

**How Locking Works:**

- Before any state-modifying operation, Terraform attempts to **acquire a lock**.
- If the lock is held by another process, Terraform **waits or fails**.
- After the operation, Terraform **releases the lock**.

**Locking by Backend:**

- **Local:** No locking (single user assumed).
- **S3:** Uses a **DynamoDB table** for locking (table must have a partition key named `LockID`).
- **HCP Terraform:** Built-in locking (no extra setup).

*Exam Trap:* S3 alone does **not** provide state locking. You **must** add a DynamoDB table. Without it, concurrent writes can corrupt state.

*Exam Trigger:* "Prevent simultaneous Terraform runs from corrupting state" --> State locking (DynamoDB for S3 backend).

---

### **4. terraform refresh**

**The Rule:** `terraform refresh` reconciles the state file with real infrastructure. It reads the actual state of all resources and updates the state file to match.

**Key Facts:**

- **Does NOT modify infrastructure.** Only updates the state file.
- **Implicit in plan/apply:** Since Terraform 0.15.4, `terraform plan` and `terraform apply` automatically perform a refresh before computing changes. You rarely need to run `terraform refresh` directly.
- **Deprecated as standalone:** HashiCorp recommends `terraform apply -refresh-only` instead of `terraform refresh` for explicit refresh operations.

*Exam Trigger:* "Someone changed a resource manually, how to update state?" --> `terraform apply -refresh-only` (or `terraform refresh`).

---

### **5. Backend Migration**

**The Rule:** When you change the backend configuration, run `terraform init -migrate-state` to move the existing state to the new backend.

```hcl
# Changing from local to S3 backend:
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Then run:

```bash
terraform init -migrate-state
```

**Key Facts:**

- Terraform detects the backend change and prompts to migrate state.
- The old state is copied to the new backend.
- `-migrate-state` is the flag that triggers state migration during init.
- `-reconfigure` is the alternative that **ignores** existing state (starts fresh with new backend).

*Exam Trap:* `terraform init -migrate-state` **copies** state to the new backend. `terraform init -reconfigure` does **not** migrate -- it reinitializes with an empty state in the new backend.

---

### **6. State File Security**

**The Rule:** The state file contains **sensitive data** (passwords, keys, resource attributes) and must be protected.

**Security Best Practices:**

- **Never commit state to Git.** Add `*.tfstate` and `*.tfstate.backup` to `.gitignore`.
- **Enable encryption at rest:** Use S3 server-side encryption (SSE) or HCP Terraform (encrypted by default).
- **Restrict access:** Use IAM policies to limit who can read/write the S3 bucket.
- **Enable versioning:** Turn on S3 bucket versioning to recover from state corruption.
- **Use remote backends:** Avoid local state for team environments.

*Exam Trigger:* "How to secure Terraform state?" --> Remote backend + encryption + access controls + never commit to VCS.

*Exam Trap:* Even with `sensitive = true` on variables/outputs, the state file **still stores the actual value**. Encryption and access controls on the backend are the only protection.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **Providers declared where?** --> `required_providers` block inside `terraform {}`.
2. **`~>` operator?** --> Pessimistic constraint. Rightmost version component increments only.
3. **Lock file (.terraform.lock.hcl)?** --> Commit to VCS. Updated with `terraform init -upgrade`.
4. **Deploy to two regions?** --> Provider alias.
5. **Read existing infrastructure without managing it?** --> Data source (`data` block).
6. **Data source modifies resources?** --> Never. Data sources are read-only.
7. **State file format?** --> JSON.
8. **Sensitive data in state?** --> Yes, always stored in plain text regardless of `sensitive = true`.
9. **State locking on S3?** --> Requires a DynamoDB table with `LockID` partition key.
10. **Migrate state to new backend?** --> `terraform init -migrate-state`.
11. **Reconcile state with real infrastructure?** --> `terraform apply -refresh-only`.
12. **Secure state file?** --> Remote backend, encryption, access controls, never commit to Git.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "Lock File Mismatch" (Dependency Lock File)**

**The Situation:** A team of five engineers works on the same Terraform project. One engineer runs `terraform init` and gets provider version 5.30.0. Another engineer clones the repo and runs `terraform init` a week later, getting version 5.32.0. Resources behave differently across their environments, causing inconsistent plans.

**The Options:**

A. Pin the provider to an exact version using `= 5.30.0`.

B. Commit the `.terraform.lock.hcl` file to version control.

C. Add the `.terraform/` directory to version control.

D. Use `terraform providers lock` to generate lock files for all platforms.

**The Logic:**

- **Trap:** Exact pinning (Option A) works but is overly restrictive and defeats the purpose of version constraints.
- **Trap:** Committing `.terraform/` (Option C) includes large binary files and is explicitly not recommended.
- **The Fix:** **Option B**. The `.terraform.lock.hcl` file records the **exact** version and hashes selected. When committed to VCS, all team members running `terraform init` will get the same provider version. Option D is useful for cross-platform teams but does not solve the core sharing problem without also committing the file.

---

### **Scenario 2: The "Multi-Region Deployment" (Provider Aliases)**

**The Situation:** A company needs to deploy an S3 bucket in `us-east-1` for their primary application and a DynamoDB table in `eu-west-1` for European compliance requirements. Both resources must be managed in a single Terraform configuration.

**The Options:**

A. Create two separate Terraform configurations, one per region.

B. Use a provider alias to configure a second AWS provider for `eu-west-1`.

C. Use a `for_each` loop with a region variable to deploy to multiple regions.

D. Use the `region` argument directly on the resource block.

**The Logic:**

- **Trap:** Separate configurations (Option A) work but add operational overhead and prevent cross-resource references.
- **Trap:** Resources do not have a `region` argument (Option D) -- the region is set on the provider.
- **The Fix:** **Option B**. Define a default provider for `us-east-1` and an aliased provider (`alias = "europe"`) for `eu-west-1`. Reference the alias on the DynamoDB resource with `provider = aws.europe`. Single config, two regions.

---

### **Scenario 3: The "Corrupted State" (State Locking)**

**The Situation:** Two engineers on the same team run `terraform apply` at the same time against a shared S3 backend. The state file becomes corrupted, and resources are in an inconsistent state. The team needs to prevent this from happening again.

**The Options:**

A. Switch to a local backend so only one person can access the state.

B. Add a DynamoDB table for state locking to the S3 backend configuration.

C. Use S3 bucket versioning to recover from corruption.

D. Implement a manual process where engineers announce when they are running Terraform.

**The Logic:**

- **Trap:** Local backend (Option A) prevents collaboration entirely.
- **Trap:** S3 versioning (Option C) helps recover **after** corruption but does not **prevent** it.
- **The Fix:** **Option B**. Adding a **DynamoDB table** for state locking ensures that only one `terraform apply` can run at a time. The second engineer's command will wait or fail until the lock is released. The DynamoDB table must have a partition key named `LockID`.

---

### **Scenario 4: The "Secret in State" (State Security)**

**The Situation:** A Terraform configuration creates an RDS database with a password passed as a variable marked `sensitive = true`. After running `terraform apply`, a security audit discovers the database password is visible in the `terraform.tfstate` file stored in an S3 bucket with public read access.

**The Options:**

A. The `sensitive = true` flag should have prevented the password from appearing in state. This is a Terraform bug.

B. Use a `local` backend instead of S3 to keep the state file on a secure machine.

C. Restrict S3 bucket access with IAM policies, enable encryption, and enable bucket versioning.

D. Remove the password from the state file using `terraform state rm`.

**The Logic:**

- **Trap:** `sensitive = true` (Option A) only hides values from CLI output. It does **NOT** prevent values from being stored in state. This is expected behavior, not a bug.
- **Trap:** `terraform state rm` (Option D) removes the resource from state entirely, orphaning the real resource.
- **The Fix:** **Option C**. The state file **will always contain sensitive values**. The correct approach is to secure the backend: restrict bucket access via IAM, enable SSE encryption, block public access, and enable versioning for recovery.

---

### **Scenario 5: The "Data Source vs. Resource" (Read-Only Lookup)**

**The Situation:** A team manages their VPC and subnets manually through the AWS Console. A separate Terraform project needs to launch EC2 instances into one of these existing subnets. The team does **not** want Terraform to manage or modify the VPC or subnets.

**The Options:**

A. Import the VPC and subnets into Terraform state using `terraform import`.

B. Define the VPC and subnets as `resource` blocks and reference them.

C. Use `data` blocks to look up the existing VPC and subnets by tags or IDs.

D. Hardcode the subnet ID directly in the EC2 instance resource.

**The Logic:**

- **Trap:** `terraform import` (Option A) brings the VPC/subnets **under Terraform management**. Future `terraform destroy` or changes could modify or delete them.
- **Trap:** Defining them as `resource` blocks (Option B) would cause Terraform to try to **create** them, resulting in errors or duplicates.
- **Trap:** Hardcoding IDs (Option D) works but is brittle and not portable across environments.
- **The Fix:** **Option C**. **Data sources** (`data` blocks) read existing infrastructure **without managing it**. Terraform queries the AWS API to look up the VPC/subnet by tags or filters, and the EC2 resource references `data.aws_subnet.existing.id`. The VPC/subnets remain untouched by Terraform.
