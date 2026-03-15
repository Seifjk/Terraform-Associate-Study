# Terraform Modules

### **SECTION 1: MODULE BASICS**

*Reusable infrastructure building blocks.*

### **1. What Is a Module?**

- **The Rule:** A module is **any directory containing `.tf` files**. Every Terraform configuration is a module.
- **Root Module:** The working directory where you run `terraform plan/apply`. It **always** exists.
- **Child Module:** A module called from another module using a `module` block.

**Module Scope:**

- Each module has its **own scope**. Resources inside a module are encapsulated.
- You **cannot** directly reference a resource inside a child module from the root module.
- You must use **outputs** to expose values from a child module.

---

### **2. Module Block Syntax**

**The Rule:** You call a child module using a `module` block with a `source` argument.

```hcl
module "vpc" {
  source  = "./modules/vpc"

  vpc_cidr = "10.0.0.0/16"
  environment = "production"
}
```

**Key Arguments:**

- **`source`** (Required): Where to find the module code.
- **`version`** (Optional): Version constraint (only for registry modules).
- **All other arguments:** Passed as **input variables** to the module.

*Exam Trigger:* "Reuse infrastructure code across projects" → Use Terraform Modules.

---

### **SECTION 2: MODULE SOURCES**

*Where Terraform fetches module code from.*

### **3. Source Types**

**The Rule:** The `source` argument tells Terraform where to download or reference the module. Different source types have different behaviors.

**Local Path:**

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

- Referenced from the filesystem. No download needed.
- **No version constraint** supported (or needed) — it uses whatever is on disk.

**Terraform Registry:**

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "~> 0.1"
}
```

- Format: `<NAMESPACE>/<NAME>/<PROVIDER>`
- **Version constraint is REQUIRED** for production use.

**GitHub:**

```hcl
module "vpc" {
  source = "github.com/hashicorp/example"
}

module "subnet" {
  source = "github.com/hashicorp/example//modules/subnet"
}
```

- Uses HTTPS by default. Use `git::` prefix for SSH.
- **Double-slash (`//`)** separates the repo from a **subdirectory** within the repo.

**Other Sources:**

```hcl
# Bitbucket
module "vpc" {
  source = "bitbucket.org/hashicorp/terraform-consul-aws"
}

# S3 Bucket
module "vpc" {
  source = "s3::https://s3-eu-west-1.amazonaws.com/bucket/module.zip"
}

# GCS Bucket
module "vpc" {
  source = "gcs::https://www.googleapis.com/storage/v1/bucket/module.zip"
}

# Generic HTTP URL
module "vpc" {
  source = "https://example.com/vpc-module.zip"
}
```

**Module Source Comparison:**

| **Source Type** | **Example** | **Version Constraint Support** |
| --- | --- | --- |
| **Local Path** | `./modules/vpc` | **NO** (uses files on disk) |
| **Terraform Registry** | `hashicorp/consul/aws` | **YES** (use `version` argument) |
| **GitHub** | `github.com/org/repo` | **NO** (use `ref` query param for tags) |
| **Bitbucket** | `bitbucket.org/org/repo` | **NO** (use `ref` query param for tags) |
| **S3** | `s3::https://...` | **NO** |
| **GCS** | `gcs::https://...` | **NO** |
| **HTTP** | `https://example.com/module.zip` | **NO** |

*Exam Trap:* The `version` argument **only works with registry modules**. For GitHub/S3/HTTP sources, it is **ignored** — use Git tags or specific URLs instead.

---

### **SECTION 3: MODULE VERSIONING**

*Pinning modules to specific versions.*

### **4. Version Constraints**

**The Rule:** The `version` argument only works with **Terraform Registry** modules. It controls which version Terraform downloads.

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "~> 3.0"
}
```

**Constraint Operators:**

| **Operator** | **Example** | **Meaning** |
| --- | --- | --- |
| `=` | `= 3.0.0` | Exact version only |
| `~>` | `~> 3.0` | Pessimistic — allows `3.x` but NOT `4.0` |
| `~>` | `~> 3.0.1` | Pessimistic — allows `3.0.x` but NOT `3.1.0` |
| `>=` | `>= 3.0` | Minimum version (3.0 or higher) |
| `<` | `< 4.0` | Maximum version (below 4.0) |
| Combined | `>= 3.0, < 4.0` | Range — at least 3.0 but below 4.0 |

**Pessimistic Constraint (`~>`) Deep Dive:**

- `~> 3.0` → allows `3.0, 3.1, 3.5, 3.99` but NOT `4.0` (constrains the **major** version).
- `~> 3.0.1` → allows `3.0.1, 3.0.2, 3.0.99` but NOT `3.1.0` (constrains the **minor** version).

*Exam Trigger:* "Allow patch updates but block breaking changes" → `~> 3.0.1` (pessimistic constraint).

---

### **5. Downloading Modules**

**The Rule:** `terraform init` downloads modules and providers. Modules are stored in `.terraform/modules/`.

- **`terraform init`**: Downloads modules **and** providers. Must run after adding/changing a module source.
- **`terraform get`**: Downloads modules **only** (no providers). Lighter-weight command.
- **`terraform init -upgrade`**: Re-downloads modules and updates to the **latest allowed version** within constraints.

*Exam Trap:* If you change a module's `source` or `version`, you **must** run `terraform init` again before `plan` or `apply`.

---

### **SECTION 4: MODULE INPUTS AND OUTPUTS**

*How modules communicate with each other.*

### **6. Module Inputs (Variables)**

**The Rule:** Module inputs are **variables declared inside the module** and passed as **arguments in the module block**.

**Inside the module (`modules/vpc/variables.tf`):**

```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

**Calling the module (root module):**

```hcl
module "vpc" {
  source   = "./modules/vpc"

  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
}
```

- Variables **without defaults** are **required** — you must pass them.
- Variables **with defaults** are **optional** — they use the default if not passed.

---

### **7. Module Outputs**

**The Rule:** Outputs declared inside a module are accessed as `module.<MODULE_NAME>.<OUTPUT_NAME>`.

**Inside the module (`modules/vpc/outputs.tf`):**

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The ID of the VPC"
}
```

**Accessing from root module:**

```hcl
resource "aws_subnet" "public" {
  vpc_id = module.vpc.vpc_id
  cidr_block = "10.0.1.0/24"
}
```

**Key Facts:**

- You **cannot** access `aws_vpc.main.id` directly from outside the module. It must be exposed via an `output` block.
- Outputs must be **explicitly declared**. Nothing is exposed by default.
- Module outputs are the **only way** to pass data out of a module.

*Exam Trap:* "Access a resource attribute from a child module" → You **must** declare an `output` in the child module. Direct resource references across module boundaries do **not** work.

---

### **8. Passing Outputs Between Modules**

**The Rule:** One module's output can be another module's input. This is how modules communicate.

```hcl
module "vpc" {
  source   = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}

module "web_server" {
  source = "./modules/web_server"

  vpc_id    = module.vpc.vpc_id
  subnet_id = module.vpc.public_subnet_id
}
```

- Module A outputs `vpc_id` → Module B receives it as an input variable.
- Terraform automatically determines the **dependency order** (Module B depends on Module A).

---

### **SECTION 5: TERRAFORM REGISTRY**

*The marketplace for Terraform modules.*

### **9. Public Registry**

**The Rule:** The **Terraform Registry** at `registry.terraform.io` hosts publicly available modules and providers.

**Module Naming Convention:**

- Format: `<NAMESPACE>/<NAME>/<PROVIDER>`
- Example: `hashicorp/consul/aws`
    - **Namespace:** `hashicorp` (the publisher)
    - **Name:** `consul` (the module name)
    - **Provider:** `aws` (the target provider)

**Verified Modules:**

- Marked with a **blue checkmark** badge.
- Published by **HashiCorp partners** who have been verified.
- Actively maintained and reviewed.

**Module Documentation:**

- **Auto-generated** from the module's `variables.tf` (Inputs), `outputs.tf` (Outputs), and README.
- Shows required vs. optional inputs, default values, and descriptions.

*Exam Trigger:* "Find pre-built, community-maintained infrastructure code" → Terraform Public Registry.

---

### **10. Private Registry**

**The Rule:** **HCP Terraform** (formerly Terraform Cloud) includes a **Private Registry** for hosting internal modules.

**Key Facts:**

- Same interface as the public registry.
- Host **private modules** only your organization can access.
- Module naming: `<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>`
- Supports versioning, documentation, and access controls.

```hcl
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "~> 1.0"
}
```

*Exam Trigger:* "Share modules across teams in an organization" → Private Registry in HCP Terraform.

---

### **SECTION 6: MODULE BEST PRACTICES**

*Writing and consuming modules the right way.*

### **11. Best Practices**

**The Rule:** Modules should be **self-contained, focused, and versioned**.

**Key Practices:**

- **Use published registry modules** when possible — don't reinvent the wheel.
- **Pin versions in production** — never leave version unconstrained.
- **One purpose per module** — a VPC module should create VPCs, not also create EC2 instances.
- **Modules should be self-contained** — include all resources, variables, and outputs they need.
- **Minimize required inputs** — use sensible defaults for optional values.

**Bad Practice:**

```hcl
# No version constraint — will grab latest on next init
module "vpc" {
  source = "hashicorp/vpc/aws"
}
```

**Good Practice:**

```hcl
# Pinned with pessimistic constraint
module "vpc" {
  source  = "hashicorp/vpc/aws"
  version = "~> 5.0"
}
```

*Exam Trap:* A module without a version constraint on a registry source will download the **latest version** on `terraform init`. This can introduce **breaking changes** in production.

---

### **12. Nested Modules**

**The Rule:** Modules can call other modules. A child module can itself be a parent to other modules.

```hcl
# Root module calls "network" module
module "network" {
  source = "./modules/network"
}

# Inside ./modules/network/main.tf, it calls a "subnet" module
module "subnet" {
  source = "./modules/subnet"
  vpc_id = aws_vpc.main.id
}
```

**Key Facts:**

- Keep nesting **shallow** (1-2 levels deep). Deep nesting makes debugging difficult.
- Each nested module has its **own scope** — outputs must be passed up through each level.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **What is a module?** → Any directory with `.tf` files.
2. **Root module?** → The working directory where you run Terraform. Always exists.
3. **Child module?** → Called from root via `module` block with `source`.
4. **Version constraint on local path?** → **NOT supported**. Only registry modules support `version`.
5. **Version constraint operator `~> 3.0`?** → Allows `3.x` but NOT `4.0` (pessimistic constraint).
6. **Version constraint operator `~> 3.0.1`?** → Allows `3.0.x` but NOT `3.1.0`.
7. **Access child module resource?** → Must use `output` block. Reference as `module.<NAME>.<OUTPUT>`.
8. **Download modules?** → `terraform init` (modules + providers) or `terraform get` (modules only).
9. **Update module version?** → `terraform init -upgrade`.
10. **Registry module naming?** → `<NAMESPACE>/<NAME>/<PROVIDER>`.
11. **Verified modules?** → Blue checkmark, HashiCorp partner verified.
12. **Share private modules?** → Private Registry in HCP Terraform.
13. **GitHub subdirectory source?** → Double-slash: `github.com/org/repo//modules/vpc`.
14. **Module best practice?** → Pin versions, one purpose per module, self-contained.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "Module Source Version" (Registry Versioning)**

**The Situation:** A team uses a VPC module from the Terraform Public Registry in production. After running `terraform init`, a new major version of the module is downloaded that introduces breaking changes, causing `terraform plan` to show unexpected resource destruction.

**The Options:**

A. Switch to a local path module to avoid version changes.

B. Add a `version = "~> 3.0"` constraint to the module block to pin the major version.

C. Use `terraform get` instead of `terraform init` to prevent version updates.

D. Lock the module version in `.terraform.lock.hcl`.

**The Logic:**

- **Trap:** Local path (Option A) avoids versioning but loses the benefit of registry module updates and community maintenance.
- **Trap:** `terraform get` (Option C) also downloads modules and does not prevent version changes.
- **Trap:** `.terraform.lock.hcl` (Option D) locks **provider** versions, not module versions.
- **The Fix:** **Option B**. Adding `version = "~> 3.0"` ensures only patch and minor updates within the `3.x` range are allowed. Major version `4.0` is blocked, preventing breaking changes.

---

### **Scenario 2: The "Can't Access Child Resource" (Module Outputs)**

**The Situation:** A developer creates a child module that provisions an AWS VPC. In the root module, they try to reference `aws_vpc.main.id` from the child module to create a subnet, but Terraform throws an error saying the resource is not found.

**The Options:**

A. Move the VPC resource into the root module.

B. Add an `output` block in the child module to expose `vpc_id`, then reference it as `module.vpc.vpc_id`.

C. Use `terraform import` to bring the VPC into the root module state.

D. Use a `data` source to look up the VPC by tag.

**The Logic:**

- **Trap:** Moving the resource to root (Option A) defeats the purpose of modularization.
- **Trap:** `terraform import` (Option C) does not solve cross-module reference issues.
- **Trap:** A `data` source (Option D) works but adds unnecessary complexity and a dependency on tags existing.
- **The Fix:** **Option B**. Modules have their **own scope**. You must declare an `output` block inside the child module to expose values. Then reference it as `module.vpc.vpc_id` in the root module.

---

### **Scenario 3: The "GitHub Subdirectory Module" (Source Syntax)**

**The Situation:** A team stores multiple Terraform modules in a single GitHub repository under a `modules/` directory. They want to reference only the `modules/networking` subdirectory as a module source. They try `source = "github.com/org/infra-repo/modules/networking"` but `terraform init` fails.

**The Options:**

A. Use `source = "github.com/org/infra-repo//modules/networking"` with a double-slash before the subdirectory path.

B. Clone the repository locally and use a local path source.

C. Create a separate GitHub repository for the networking module.

D. Use `source = "github.com/org/infra-repo"` and set a `path` argument in the module block.

**The Logic:**

- **Trap:** Cloning locally (Option B) works but loses the ability to reference remote updates directly.
- **Trap:** Separate repo (Option C) adds overhead when the team wants a monorepo structure.
- **Trap:** There is no `path` argument (Option D) in the module block.
- **The Fix:** **Option A**. The **double-slash (`//`)** is the Terraform convention for specifying a subdirectory within a repository. The correct syntax is `github.com/org/infra-repo//modules/networking`.

---

### **Scenario 4: The "Module Init Required" (Terraform Workflow)**

**The Situation:** A developer adds a new module block referencing a Terraform Registry module to their configuration. They immediately run `terraform plan` but receive an error: `Module not installed. Run "terraform init" to install all modules.`

**The Options:**

A. Run `terraform get` to download the module, then run `terraform plan`.

B. Run `terraform init` to download the module and initialize the backend, then run `terraform plan`.

C. Run `terraform apply` directly — it will auto-download modules.

D. Manually download the module and place it in the `.terraform/modules/` directory.

**The Logic:**

- **Trap:** `terraform get` (Option A) downloads modules but does not initialize providers that the new module may require.
- **Trap:** `terraform apply` (Option C) does **not** auto-download modules. Init must run first.
- **Trap:** Manually placing files (Option D) bypasses Terraform's module management and will cause issues.
- **The Fix:** **Option B**. `terraform init` is the **correct and complete** command. It downloads modules, installs providers, and initializes the backend. Always run `terraform init` after adding or changing module sources.

---

### **Scenario 5: The "Private Module Sharing" (Private Registry)**

**The Situation:** A company has 10 engineering teams that each manage their own Terraform configurations. The platform team builds a standardized VPC module that must be shared across all teams. The module contains proprietary network architecture and must not be publicly accessible. Teams need versioning and documentation.

**The Options:**

A. Publish the module to the Terraform Public Registry with restricted access.

B. Copy the module source code into each team's repository.

C. Host the module in a Private Registry in HCP Terraform with version constraints.

D. Store the module in an S3 bucket and reference it with an S3 source.

**The Logic:**

- **Trap:** The Public Registry (Option A) does **not** support private/restricted modules. Everything is public.
- **Trap:** Copying source code (Option B) creates version drift — no central source of truth, no versioning.
- **Trap:** S3 bucket (Option D) works for distribution but lacks versioning, documentation, and a discovery interface.
- **The Fix:** **Option C**. The **Private Registry in HCP Terraform** provides centralized hosting, semantic versioning, auto-generated documentation, and organization-level access control. Teams reference it like any registry module: `source = "app.terraform.io/my-org/vpc/aws"` with `version = "~> 1.0"`.
