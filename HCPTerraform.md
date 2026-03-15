# HCP Terraform (Formerly Terraform Cloud)

### **SECTION 1: HCP TERRAFORM OVERVIEW**

*SaaS platform for team-based Terraform workflows.*

### **1. What is HCP Terraform?**

- **The Rule:** HCP Terraform (formerly Terraform Cloud) is HashiCorp's managed SaaS platform for running Terraform in a team environment. It provides remote state, locking, governance, and VCS integration out of the box.
- **Name Change:** The exam may refer to it as **"Terraform Cloud"** or **"HCP Terraform"** — they are the same product. Know both names.
- **Terraform Enterprise:** Self-hosted version of HCP Terraform. Same features, deployed on your own infrastructure.

**Why Use HCP Terraform Over Open-Source Terraform?**

- **Remote State Storage:** State stored securely in HCP Terraform (not in S3, not on your laptop).
- **State Locking:** Built-in. No need for DynamoDB lock table.
- **Team Collaboration:** Multiple engineers can plan/apply with RBAC (role-based access control).
- **Governance:** Policy as Code (Sentinel/OPA) enforces compliance before apply.
- **VCS Integration:** Connect to GitHub/GitLab — PR triggers plan, merge triggers apply.
- **Audit Trail:** Full run history, who approved what, when.

---

### **2. Organizations and Tiers**

- **Organizations:** Top-level grouping in HCP Terraform. Typically one per company. Contains workspaces, teams, policies, and registry modules.

**Tier Comparison:**

| **Feature** | **Free** | **Plus** | **Enterprise (Self-Hosted)** |
| --- | --- | --- | --- |
| **Remote State** | Yes | Yes | Yes |
| **VCS Integration** | Yes | Yes | Yes |
| **Team Management** | Limited | Full RBAC | Full RBAC |
| **Sentinel/OPA Policies** | No | Yes | Yes |
| **SSO/SAML** | No | Yes | Yes |
| **Audit Logging** | No | No | Yes |
| **Self-Hosted** | No | No | Yes |
| **Air-Gapped Support** | No | No | Yes |

*Exam Trigger:* "Air-gapped environment, need Terraform collaboration" → **Terraform Enterprise** (self-hosted).

*Exam Trigger:* "Policy enforcement, team RBAC on SaaS" → **HCP Terraform Plus tier**.

---

### **SECTION 2: WORKSPACES (HCP TERRAFORM)**

*Not the same as CLI workspaces. This trips up everyone.*

### **1. HCP Terraform Workspaces**

- **The Rule:** In HCP Terraform, each workspace is a **complete working unit**: one root module + one state file + its own variables + run history + optional VCS connection.
- **Projects:** Group related workspaces together (e.g., all workspaces for the "payments" service). Projects are an organizational concept — they do not affect execution.
- **Pattern:** One workspace per environment per component.
    - `vpc-dev`, `vpc-staging`, `vpc-prod`
    - `app-dev`, `app-staging`, `app-prod`

**Workspace Settings:**

- **Terraform Version:** Pin which Terraform version runs in this workspace.
- **Execution Mode:** Remote, Local, or Agent (covered in Section 3).
- **Auto-Apply:** If enabled, apply runs automatically after successful plan (no manual approval).
- **Working Directory:** Subdirectory in the repo where Terraform config lives.
- **VCS Connection:** Which repo/branch triggers runs.

---

### **2. CLI Workspaces vs. HCP Terraform Workspaces**

**The Rule:** These are **completely different concepts** that share the same name. The exam tests this.

| **Feature** | **CLI Workspaces (`terraform workspace`)** | **HCP Terraform Workspaces** |
| --- | --- | --- |
| **Purpose** | Multiple state files for same config | Full working unit (state + variables + VCS + runs) |
| **State** | Multiple states in same backend | One state file per workspace |
| **Variables** | Shared (same `terraform.tfvars`) | Each workspace has its own variables |
| **VCS Connection** | None | Each workspace can connect to a repo |
| **Run History** | None | Full audit trail of every plan/apply |
| **Access Control** | None | Team-based permissions (Read/Plan/Write/Admin) |
| **Use Case** | Quick dev/test separation | Production team collaboration |

*Exam Trap:* "Workspaces in HCP Terraform" ≠ `terraform workspace select`. CLI workspaces are just named state files. HCP Terraform workspaces are full collaboration units with variables, permissions, VCS, and run history.

---

### **SECTION 3: EXECUTION MODES**

*Where does `terraform plan` and `terraform apply` actually run?*

### **1. Execution Mode Comparison**

| **Mode** | **Where Runs Execute** | **State Storage** | **Use Case** |
| --- | --- | --- | --- |
| **Remote** (default) | HCP Terraform managed runners | HCP Terraform | Standard team workflow. No local credentials needed. |
| **Local** | Your local machine | HCP Terraform | Debugging, migration from open-source. State still remote. |
| **Agent** | Self-hosted agent in your network | HCP Terraform | Private infrastructure with no public internet access. |

**Key Details:**

- **Remote Execution (Default):** Terraform runs entirely on HCP Terraform's infrastructure. Your local machine only sends config and receives output. Cloud credentials are stored as workspace variables — never on developer laptops.
- **Local Execution:** `terraform plan` and `terraform apply` run on your machine, but state is read from and written to HCP Terraform. Useful when migrating from open-source or debugging provider issues.
- **Agent Execution:** You install **HCP Terraform Agents** on machines inside your private network. HCP Terraform sends work to agents, agents execute runs and report back. State still stored in HCP Terraform.

*Exam Trigger:* "Team needs to provision resources in a private network with no internet access" → **Agent execution mode**.

*Exam Trigger:* "Developer wants to debug locally but still use remote state" → **Local execution mode**.

---

### **SECTION 4: WORKFLOWS**

*Three ways to trigger runs. The exam tests all three.*

### **1. VCS-Driven Workflow (EXAM FAVORITE)**

**The Rule:** Connect a workspace to a VCS repo (GitHub, GitLab, Bitbucket). Git events trigger Terraform runs.

**How It Works:**

1. Developer opens a **Pull Request** → HCP Terraform runs a **speculative plan** (plan only, no apply). Result appears as a PR check.
2. Team reviews the plan output in the PR.
3. Developer **merges to default branch** → HCP Terraform triggers a **full run** (plan + apply).
4. If **auto-apply** is enabled, apply happens automatically after successful plan. If not, someone must click "Confirm & Apply."

**Key Facts:**

- **Speculative Plans:** Read-only. Cannot be applied. Shown as GitHub/GitLab status check on the PR.
- **Auto-Apply:** Optional workspace setting. Skip manual confirmation after merge.
- **Most common workflow** for production teams.

*Exam Trigger:* "Pull request shows Terraform plan as a status check" → **VCS-driven workflow with speculative plan**.

---

### **2. CLI-Driven Workflow**

**The Rule:** Run `terraform plan` and `terraform apply` from your local CLI. Execution happens on HCP Terraform runners (remote execution mode). State stored in HCP Terraform.

**How It Works:**

1. Developer runs `terraform plan` locally.
2. CLI uploads config to HCP Terraform.
3. HCP Terraform executes the plan on its runners.
4. Output streams back to the developer's terminal.
5. Developer runs `terraform apply` — same process.

**Use Case:** Development, testing, one-off changes, or teams migrating from open-source Terraform.

---

### **3. API-Driven Workflow**

**The Rule:** External systems (Jenkins, GitHub Actions, CircleCI) trigger runs via the **HCP Terraform API**.

**How It Works:**

1. CI/CD pipeline calls HCP Terraform API to create a run.
2. Uploads configuration version.
3. HCP Terraform executes plan/apply.
4. CI/CD pipeline polls for run status.

**Key Facts:**

- **Most flexible** workflow — full programmatic control.
- **Most complex** to set up.
- Used when an existing CI/CD pipeline must orchestrate Terraform runs.

*Exam Trigger:* "Jenkins must trigger Terraform runs and control the workflow" → **API-driven workflow**.

---

### **SECTION 5: RUN WORKFLOW PIPELINE**

*The stages every run passes through.*

### **1. Run Stages**

**The Rule:** Every run in HCP Terraform passes through a pipeline. Each stage can pass or fail. **Failure at any stage stops the pipeline.**

```
Plan → Cost Estimation → Policy Check → Apply
```

**Stage Details:**

1. **Plan:** Generate the execution plan (`terraform plan`). Shows what will be created/changed/destroyed.
2. **Cost Estimation** (optional): Estimates the monthly cost change of the planned infrastructure. Shows "+$50/month" or "-$20/month." Available for AWS, Azure, GCP resources.
3. **Policy Check:** Sentinel or OPA policies evaluate the plan. Policies can block the apply if infrastructure violates rules (e.g., "no public S3 buckets").
4. **Apply:** Execute the changes (`terraform apply`). May require **manual approval** (unless auto-apply is enabled).

*Exam Trap:* Cost Estimation happens **after** plan but **before** policy check. Policy check happens **before** apply. If a policy fails with hard-mandatory enforcement, the apply is **blocked**.

---

### **SECTION 6: GOVERNANCE (POLICY AS CODE)**

*Enforce rules before infrastructure is created.*

### **1. Sentinel**

- **The Rule:** HashiCorp's proprietary policy-as-code language. Written in the **Sentinel language** (not HCL, not Rego).
- Evaluate the Terraform plan and block applies that violate rules.

**Example Policies:**

- "All S3 buckets must have encryption enabled."
- "No security group may allow ingress from 0.0.0.0/0."
- "Instance types must be from an approved list."

**Enforcement Levels:**

| **Level** | **Behavior** | **Override?** |
| --- | --- | --- |
| **advisory** | Warn but allow the run to continue | N/A (always passes) |
| **soft-mandatory** | Fail the run, but an admin can override | Yes (admin override) |
| **hard-mandatory** | Fail the run. No override possible. | **No** |

*Exam Trigger:* "Policy must block non-compliant infrastructure with no exceptions" → **hard-mandatory** Sentinel policy.

*Exam Trigger:* "Policy should warn but not block" → **advisory** enforcement level.

---

### **2. OPA (Open Policy Agent)**

- **The Rule:** Open-source alternative to Sentinel. Uses the **Rego** language.
- Same concept: evaluate plans, enforce compliance.
- Supported in HCP Terraform alongside Sentinel.

---

### **3. Policy Sets and Run Tasks**

- **Policy Sets:** Collections of policies applied to one or more workspaces. Can scope policies to specific workspaces or apply globally across the organization.
- **Run Tasks:** Integrate **third-party tools** into the run pipeline. Examples: security scanners (Snyk, Checkov), compliance checkers, cost management tools. Run Tasks execute at specific points in the pipeline (pre-plan, post-plan, pre-apply).

*Exam Trigger:* "Integrate a third-party security scanner into the Terraform run pipeline" → **Run Tasks**.

---

### **SECTION 7: COLLABORATION FEATURES**

*Team management, variables, and the private registry.*

### **1. Variable Sets**

- **The Rule:** Reusable groups of variables applied across **multiple workspaces**. Avoid duplicating the same variables in every workspace.
- **Use Case:** Share AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) across all AWS workspaces using one Variable Set.
- **Scope:** Can apply to all workspaces in the org or specific workspaces/projects.

---

### **2. Variables**

- **Two Types:**
    - **Terraform Variables:** Correspond to `variable` blocks in your config (e.g., `instance_type = "t3.micro"`).
    - **Environment Variables:** Set shell environment variables for the run (e.g., `AWS_ACCESS_KEY_ID`).
- **Sensitive:** Mark a variable as sensitive → value is write-only. Cannot be read back in UI or API.
- **Scope:** Workspace-level variables or Variable Set variables. Workspace variables override Variable Set values.

---

### **3. Team Permissions**

| **Permission Level** | **What They Can Do** |
| --- | --- |
| **Read** | View state, view runs, view variables |
| **Plan** | Queue plans (cannot approve applies) |
| **Write** | Approve and apply runs |
| **Admin** | Full control: manage workspace settings, variables, team access |

---

### **4. Private Registry**

- **The Rule:** Host **private modules and providers** within your organization. Same interface as the public Terraform Registry.
- **Module Versioning:** Version private modules (e.g., `v1.0.0`, `v1.1.0`). Teams consume specific versions.
- **Access Control:** Control which teams can use which modules.
- **Source Format:**
```hcl
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.2.0"
}
```

*Exam Trigger:* "Share reusable Terraform modules across teams with version control" → **Private Registry** in HCP Terraform.

---

### **5. Other Collaboration Features**

- **SSO/SAML:** Single sign-on integration. Available on Plus and Enterprise tiers.
- **Audit Logging:** Full audit trail of all actions. Enterprise tier only.
- **Notifications:** Configure Slack, email, or webhook notifications for run status changes (plan started, apply succeeded, policy failed, etc.).

---

### **SECTION 8: CONNECTING TO HCP TERRAFORM**

*How to configure your Terraform CLI to use HCP Terraform.*

### **1. Authentication**

- **The Rule:** Use `terraform login` to authenticate your local CLI with HCP Terraform.
- Opens a browser, you log in, a token is stored locally in `~/.terraform.d/credentials.tfrc.json`.

```bash
terraform login
```

---

### **2. The `cloud` Block**

**The Rule:** Configure the `cloud` block inside the `terraform` block to connect a configuration to HCP Terraform.

```hcl
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "my-workspace"
    }
  }
}
```

**Key Facts:**

- The `cloud` block **replaces** the older `backend "remote" {}` configuration. Both work, but `cloud` is the current recommended approach.
- After adding the `cloud` block, run `terraform init` to initialize the connection and **migrate state** from any existing backend to HCP Terraform.
- You can use `tags` instead of `name` to match multiple workspaces:

```hcl
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      tags = ["networking", "production"]
    }
  }
}
```

*Exam Trap:* The `cloud` block and `backend` block are **mutually exclusive**. You cannot use both in the same configuration. If you see both, that is an error.

*Exam Trap:* `terraform init` after adding the `cloud` block will offer to **migrate** existing local state to HCP Terraform. You do not need to manually copy the state file.

---

### **Exam Summary Cheat Sheet (Memorize This)**

1. **HCP Terraform = Terraform Cloud** → Same product, renamed. Exam uses both names.
2. **Terraform Enterprise** → Self-hosted version. For air-gapped/compliance environments.
3. **HCP Terraform workspaces ≠ CLI workspaces** → HCP workspaces = state + variables + VCS + runs + permissions. CLI workspaces = just named state files.
4. **Default execution mode?** → Remote. Runs on HCP Terraform runners.
5. **Private network, no internet?** → Agent execution mode.
6. **Debug locally, state still remote?** → Local execution mode.
7. **PR triggers plan, merge triggers apply?** → VCS-driven workflow.
8. **Jenkins/CI must control Terraform?** → API-driven workflow.
9. **Speculative plan?** → Read-only plan triggered by a PR. Cannot be applied. Shown as PR status check.
10. **Block non-compliant infra, no exceptions?** → Sentinel hard-mandatory policy.
11. **Warn but allow?** → Sentinel advisory policy.
12. **Share variables across workspaces?** → Variable Sets.
13. **Share modules across teams?** → Private Registry.
14. **Integrate third-party scanner into runs?** → Run Tasks.
15. **`cloud` block vs `backend "remote"`?** → `cloud` block is current. They are mutually exclusive.
16. **`terraform login`** → Authenticates CLI with HCP Terraform. Stores token locally.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "PR Status Check" (VCS-Driven Workflow)**

**The Situation:** A DevOps team uses GitHub for version control. They want every pull request to automatically show what Terraform changes would occur, so the team can review before merging. After merge to main, the changes should apply automatically without manual approval.

**The Options:**

A. Use the CLI-driven workflow with `terraform plan` in a GitHub Action.

B. Use the VCS-driven workflow with auto-apply enabled.

C. Use the API-driven workflow with Jenkins.

D. Run `terraform plan` locally and paste the output into the PR.

**The Logic:**

- **Trap:** Option A uses CLI-driven workflow — this works but does not natively show plan output as a GitHub PR check. It requires extra scripting.
- **Trap:** Option D is manual, error-prone, and has no automation.
- **The Fix:** **Option B**. The **VCS-driven workflow** natively triggers a **speculative plan** on PR, which appears as a GitHub status check. When the PR merges to the default branch, HCP Terraform triggers a run. With **auto-apply** enabled, the apply proceeds automatically without manual confirmation.

---

### **Scenario 2: The "Private Data Center" (Agent Execution Mode)**

**The Situation:** A company manages infrastructure in an on-premises data center with **no public internet access**. They want to use HCP Terraform for collaboration, remote state, and governance, but Terraform must be able to reach the private infrastructure to provision resources.

**The Options:**

A. Use remote execution mode (default).

B. Use local execution mode.

C. Use agent execution mode with self-hosted agents inside the private network.

D. Use Terraform Enterprise instead.

**The Logic:**

- **Trap:** Option A fails because HCP Terraform's managed runners cannot reach the private data center (no internet access).
- **Trap:** Option B runs on the developer's machine — possible but loses the benefits of centralized execution and requires each developer to have network access.
- **Trap:** Option D works but is overkill — the question says they want to use HCP Terraform (SaaS), not self-host.
- **The Fix:** **Option C**. Install **HCP Terraform Agents** inside the private network. Agents establish an outbound connection to HCP Terraform, receive work, execute plans/applies against the private infrastructure, and report results back. State is still stored in HCP Terraform.

---

### **Scenario 3: The "Compliance Blocker" (Sentinel Hard-Mandatory)**

**The Situation:** A financial services company requires that **all** S3 buckets created via Terraform must have server-side encryption enabled. No exceptions, no overrides — even an admin should not be able to bypass this rule. They use HCP Terraform Plus tier.

**The Options:**

A. Write a Sentinel policy with **advisory** enforcement.

B. Write a Sentinel policy with **soft-mandatory** enforcement.

C. Write a Sentinel policy with **hard-mandatory** enforcement.

D. Use a Run Task with a third-party compliance tool.

**The Logic:**

- **Trap:** Option A (advisory) only warns — it does not block the apply.
- **Trap:** Option B (soft-mandatory) blocks by default but **admins can override** — the requirement says no exceptions.
- **Trap:** Option D integrates a scanner but Run Tasks may not enforce a hard block depending on configuration.
- **The Fix:** **Option C**. A **hard-mandatory** Sentinel policy evaluates every plan. If an S3 bucket lacks encryption, the policy fails and the apply is **blocked**. No one can override it. This is the only enforcement level that guarantees no exceptions.

---

### **Scenario 4: The "Shared Credentials" (Variable Sets)**

**The Situation:** A team manages 20 HCP Terraform workspaces, all deploying AWS infrastructure. Every workspace needs the same `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables. Currently, they manually set these variables in each workspace. They want to manage credentials in one place and automatically apply them across all workspaces.

**The Options:**

A. Store credentials in a `terraform.tfvars` file in the repo.

B. Create a Variable Set with the AWS credentials and apply it to all workspaces.

C. Hardcode credentials in the provider block.

D. Use a Terraform module to pass credentials.

**The Logic:**

- **Trap:** Option A puts secrets in version control — a security risk and does not solve the duplication problem across 20 workspaces.
- **Trap:** Option C hardcodes secrets in code — worst practice.
- **Trap:** Option D does not work. Provider credentials are not passed via modules this way.
- **The Fix:** **Option B**. Create a **Variable Set** with `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` marked as **sensitive environment variables**. Apply the Variable Set to all 20 workspaces (or the entire organization). Update credentials in one place, and all workspaces inherit the change.

---

### **Scenario 5: The "Cloud Block Migration" (Connecting to HCP Terraform)**

**The Situation:** A team currently uses open-source Terraform with an S3 backend for state storage and a DynamoDB table for locking. They want to migrate to HCP Terraform for collaboration and governance. They need to preserve their existing state during migration.

**The Options:**

A. Delete the S3 state file, add the `cloud` block, and run `terraform init`.

B. Replace the `backend "s3"` block with the `cloud` block and run `terraform init` — Terraform will offer to migrate the state.

C. Manually download the state from S3 and upload it via the HCP Terraform UI.

D. Keep the `backend "s3"` block and add the `cloud` block simultaneously.

**The Logic:**

- **Trap:** Option A deletes state — catastrophic. Terraform would try to recreate all infrastructure.
- **Trap:** Option C is manual and error-prone. Also does not update the configuration to use HCP Terraform going forward.
- **Trap:** Option D is invalid. The `cloud` block and `backend` block are **mutually exclusive** — Terraform will throw an error.
- **The Fix:** **Option B**. Replace `backend "s3" {}` with the `cloud {}` block and run `terraform init`. Terraform detects the backend change and **offers to migrate the existing state** from S3 to HCP Terraform. State is preserved. No manual copying needed. After migration, remove the DynamoDB lock table — HCP Terraform handles locking natively.
