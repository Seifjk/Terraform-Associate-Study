# Terraform Configuration (Objective 4)

### **SECTION 1: VARIABLES, OUTPUTS, LOCALS**

*Parameterize and expose values in your configurations.*

### **1. Input Variables**

- **The Rule:** Input variables are declared with `variable` blocks. They parameterize your configuration so it can be reused across environments.

**Variable Declaration Arguments:**

- **type** — Constrains the allowed value type (string, number, bool, list, map, object, etc.).
- **default** — Default value if none is provided. If omitted, the variable is **required**.
- **description** — Documents the variable's purpose. Shown in `terraform plan` prompts.
- **sensitive** — When `true`, Terraform redacts the value from CLI output (`(sensitive value)`). Does **NOT** encrypt state.
- **nullable** — When `false`, Terraform rejects `null` as a value. Default is `true`.
- **validation** — Custom validation rules (covered in Section 4).

```hcl
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type for the web server"
  sensitive   = false
  nullable    = false

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}
```

**Referencing Variables:** Use `var.<name>` anywhere in configuration.

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

---

### **2. Variable Types**

**Primitive Types:**

| **Type** | **Example** | **Description** |
| --- | --- | --- |
| `string` | `"hello"` | Text value |
| `number` | `42` | Numeric value (integer or float) |
| `bool` | `true` | Boolean (`true` or `false`) |

**Complex Types (Collection):**

| **Type** | **Example** | **Description** |
| --- | --- | --- |
| `list(type)` | `["a", "b", "c"]` | Ordered sequence, accessed by index |
| `map(type)` | `{ key1 = "val1" }` | Key-value pairs, accessed by key |
| `set(type)` | `toset(["a", "b"])` | Unordered unique values, no index access |

**Complex Types (Structural):**

| **Type** | **Example** | **Description** |
| --- | --- | --- |
| `object({...})` | `object({ name = string, age = number })` | Fixed structure with named attributes and specific types |
| `tuple([...])` | `tuple([string, number, bool])` | Fixed-length sequence where each element has a specific type |

**Special Type:**

- **`any`** — Terraform infers the type from the provided value. Use when you want maximum flexibility.

```hcl
variable "settings" {
  type = object({
    name        = string
    replicas    = number
    enable_logs = bool
    tags        = map(string)
  })
}
```

*Exam Trap:* `list` and `set` look similar but behave differently. `list` preserves order and allows duplicates. `set` is unordered and unique. `for_each` requires a **map or set**, not a list.

---

### **3. Variable Precedence (LOWEST to HIGHEST)**

- **The Rule:** When the same variable is set in multiple places, the **highest-precedence** source wins. The exam tests this order verbatim.

| **Priority** | **Source** | **Notes** |
| --- | --- | --- |
| 1 (Lowest) | Default value in `variable` block | Only used if nothing else sets the variable |
| 2 | `terraform.tfvars` | Auto-loaded if present in working directory |
| 3 | `*.auto.tfvars` | Auto-loaded in **alphabetical order** |
| 4 | `-var-file=filename` flag | Explicitly specified on CLI |
| 5 (Highest) | `-var 'key=value'` flag **or** `TF_VAR_name` environment variable | Last value encountered wins between these two |

**Exam Trigger:** "Developer sets a variable in terraform.tfvars AND passes -var on CLI" -- the `-var` CLI flag wins (higher precedence).

**Exam Trap:** `terraform.tfvars` and `*.auto.tfvars` are **auto-loaded** (no flags needed). Any other `.tfvars` file requires `-var-file` to be loaded.

```bash
# All of these set the same variable — highest precedence wins
export TF_VAR_region="us-west-2"                   # Priority 5
terraform apply \
  -var 'region=eu-west-1' \                         # Priority 5 (last wins)
  -var-file="prod.tfvars"                           # Priority 4
# terraform.tfvars exists with region = "us-east-1" # Priority 2
# variable default is "ap-southeast-1"              # Priority 1
```

---

### **4. Output Values**

- **The Rule:** Output values expose information about your infrastructure after `apply`. Used to pass data between modules or display to operators.

**Output Declaration Arguments:**

- **value** — (Required) The expression to output.
- **description** — Documents the output's purpose.
- **sensitive** — Redacts value from CLI output. Still stored in state.
- **depends_on** — Explicitly declare dependencies (rare, for ordering).

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "The public IP of the web server"
  sensitive   = false
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

**Key Facts:**

- Outputs are shown after `terraform apply`.
- Access child module outputs: `module.<module_name>.<output_name>`.
- `terraform output` command displays all outputs. Use `terraform output -json` for programmatic access.
- `sensitive = true` redacts from CLI but does **NOT** encrypt state.

---

### **5. Local Values**

- **The Rule:** Local values assign a name to an expression so you can reuse it **within a module** without repeating yourself.

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
  }

  name_prefix = "${var.project_name}-${var.environment}"
}

resource "aws_instance" "web" {
  tags = local.common_tags
}
```

**Key Facts:**

- Declared in a `locals {}` block (plural).
- Referenced as `local.<name>` (singular).
- Cannot be overridden from outside the module (unlike variables).
- Use for computed values, repeated expressions, or simplifying complex logic.

*Exam Trap:* Declaration is `locals {}` (plural), reference is `local.name` (singular). Do not confuse them.

---

### **SECTION 2: EXPRESSIONS AND FUNCTIONS**

*Transform, filter, and compute values dynamically.*

### **1. For Expressions**

- **The Rule:** `for` expressions transform collections by iterating over each element.

```hcl
# List comprehension — transform a list
variable "names" {
  default = ["alice", "bob", "charlie"]
}

output "upper_names" {
  value = [for name in var.names : upper(name)]
  # Result: ["ALICE", "BOB", "CHARLIE"]
}

# Map comprehension — transform into a map
output "name_lengths" {
  value = { for name in var.names : name => length(name) }
  # Result: { "alice" = 5, "bob" = 3, "charlie" = 7 }
}

# Filtering with if
output "long_names" {
  value = [for name in var.names : upper(name) if length(name) > 3]
  # Result: ["ALICE", "CHARLIE"]
}
```

---

### **2. Splat Expressions**

- **The Rule:** Shorthand for extracting an attribute from all elements in a list.

```hcl
# These two are equivalent:
output "all_ids_splat" {
  value = aws_instance.web[*].id
}

output "all_ids_for" {
  value = [for instance in aws_instance.web : instance.id]
}
```

**Key Facts:**

- `[*]` is the **splat** operator.
- Works on lists and resources with `count`.
- Does **not** work with `for_each` resources (use `values(aws_instance.web)[*].id` instead).

---

### **3. Conditional Expressions**

- **The Rule:** Ternary syntax: `condition ? true_value : false_value`.

```hcl
resource "aws_instance" "web" {
  instance_type = var.environment == "production" ? "m5.large" : "t3.micro"
}

# Conditional resource creation with count
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0
  # ...
}
```

---

### **4. Built-in Functions**

- **The Rule:** Terraform includes many built-in functions. Terraform does **NOT** support user-defined functions.

**String Functions:**

| **Function** | **Example** | **Result** |
| --- | --- | --- |
| `format` | `format("Hello, %s!", "world")` | `"Hello, world!"` |
| `join` | `join(", ", ["a", "b", "c"])` | `"a, b, c"` |
| `split` | `split(",", "a,b,c")` | `["a", "b", "c"]` |
| `upper` | `upper("hello")` | `"HELLO"` |
| `lower` | `lower("HELLO")` | `"hello"` |
| `trimspace` | `trimspace("  hi  ")` | `"hi"` |
| `replace` | `replace("hello", "l", "L")` | `"heLLo"` |
| `regex` | `regex("[a-z]+", "123abc")` | `"abc"` |

**Collection Functions:**

| **Function** | **Example** | **Result** |
| --- | --- | --- |
| `length` | `length(["a", "b", "c"])` | `3` |
| `lookup` | `lookup({a="1", b="2"}, "a", "default")` | `"1"` |
| `element` | `element(["a", "b", "c"], 1)` | `"b"` |
| `merge` | `merge({a="1"}, {b="2"})` | `{a="1", b="2"}` |
| `concat` | `concat(["a"], ["b", "c"])` | `["a", "b", "c"]` |
| `flatten` | `flatten([["a", "b"], ["c"]])` | `["a", "b", "c"]` |
| `keys` | `keys({a="1", b="2"})` | `["a", "b"]` |
| `values` | `values({a="1", b="2"})` | `["1", "2"]` |
| `zipmap` | `zipmap(["a", "b"], ["1", "2"])` | `{a="1", b="2"}` |

**Filesystem Functions:**

| **Function** | **Example** | **Description** |
| --- | --- | --- |
| `file` | `file("${path.module}/script.sh")` | Reads file content as string |
| `templatefile` | `templatefile("tmpl.tftpl", { name = "web" })` | Renders template with variables |
| `filebase64` | `filebase64("${path.module}/image.png")` | Reads file and returns base64-encoded string |

**Type Conversion Functions:**

| **Function** | **Description** |
| --- | --- |
| `toset` | Convert to set (removes duplicates, loses order) |
| `tolist` | Convert to list |
| `tomap` | Convert to map |
| `tonumber` | Convert string to number |
| `tostring` | Convert number/bool to string |

**Encoding Functions:**

| **Function** | **Description** |
| --- | --- |
| `jsonencode` | Encode value as JSON string |
| `jsondecode` | Decode JSON string to Terraform value |
| `yamlencode` | Encode value as YAML string |
| `yamldecode` | Decode YAML string to Terraform value |
| `base64encode` | Encode string to base64 |
| `base64decode` | Decode base64 to string |

**Other Useful Functions:**

| **Function** | **Example** | **Description** |
| --- | --- | --- |
| `coalesce` | `coalesce("", "fallback")` | Returns first non-empty value |
| `try` | `try(var.obj.attr, "default")` | Returns first expression that doesn't error |
| `can` | `can(regex("^t3", var.type))` | Returns `true` if expression evaluates without error |

```hcl
# Practical example combining functions
locals {
  # Merge default tags with user-provided tags
  all_tags = merge(
    { ManagedBy = "terraform" },
    var.extra_tags
  )

  # Read and render a user-data script
  user_data = templatefile("${path.module}/userdata.tftpl", {
    db_host = aws_db_instance.main.address
    db_port = aws_db_instance.main.port
  })
}
```

*Exam Trap:* Terraform does **NOT** support user-defined functions. If a question asks how to create a custom function, the answer is: you cannot. Use `locals` to encapsulate reusable expressions.

---

### **SECTION 3: RESOURCE META-ARGUMENTS**

*Control how resources are created, ordered, and managed.*

### **1. count**

- **The Rule:** Creates **N identical copies** of a resource. Access the current index with `count.index`.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"  # web-0, web-1, web-2
  }
}

# Reference a specific instance
output "first_instance_id" {
  value = aws_instance.web[0].id
}

# Reference all instances
output "all_instance_ids" {
  value = aws_instance.web[*].id
}
```

**Key Facts:**

- Resources are identified by **index**: `aws_instance.web[0]`, `aws_instance.web[1]`, etc.
- Removing an item from the middle causes all subsequent items to shift indexes and be **destroyed/recreated**.
- Use `count = var.create_resource ? 1 : 0` for conditional resource creation.

---

### **2. for_each**

- **The Rule:** Creates one instance per item in a **map or set**. Access current item with `each.key` and `each.value`.

```hcl
variable "instances" {
  default = {
    web  = "t3.micro"
    api  = "t3.small"
    db   = "t3.medium"
  }
}

resource "aws_instance" "server" {
  for_each      = var.instances
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value

  tags = {
    Name = each.key  # "web", "api", "db"
  }
}

# Reference a specific instance
output "web_instance_id" {
  value = aws_instance.server["web"].id
}
```

**Using for_each with a list (must convert to set):**

```hcl
variable "subnet_ids" {
  type    = list(string)
  default = ["subnet-abc", "subnet-def", "subnet-ghi"]
}

resource "aws_instance" "web" {
  for_each      = toset(var.subnet_ids)
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = each.value
}
```

*Exam Trap:* `for_each` does **NOT** accept a list directly. You must use `toset()` to convert a list to a set first.

---

### **3. count vs for_each Comparison**

| **Feature** | **count** | **for_each** |
| --- | --- | --- |
| **Input** | A whole number | A map or set |
| **Identifier** | Numeric index (`[0]`, `[1]`) | Key-based (`["web"]`, `["api"]`) |
| **Current Item** | `count.index` | `each.key`, `each.value` |
| **Remove Middle Item** | All subsequent resources shift and are **recreated** | Only the removed resource is destroyed |
| **Best For** | Identical copies, conditional creation | Distinct resources with unique keys |
| **Supports Lists?** | Yes (natively) | No (must use `toset()`) |

**Exam Trigger:** "Resources are being unexpectedly destroyed when removing items from the middle of a list" -- switch from `count` to `for_each`.

---

### **4. depends_on**

- **The Rule:** Explicitly declare a dependency when Terraform **cannot automatically infer** the relationship.

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Terraform can't see this implicit dependency
  depends_on = [aws_s3_bucket.data]
}
```

**Key Facts:**

- Terraform automatically detects dependencies from resource references (e.g., `aws_instance.web.id`).
- Only use `depends_on` when the dependency is **not visible** in the configuration (e.g., application-level dependency).
- Accepts a list of resources or modules.
- Available on resources, data sources, modules, and outputs.

---

### **5. provider**

- **The Rule:** Select a non-default provider configuration when you have multiple providers of the same type.

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west_server" {
  provider      = aws.west
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
}
```

---

### **6. lifecycle**

- **The Rule:** Customize the default resource lifecycle behavior.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true   # Create replacement before destroying original
    prevent_destroy       = true   # Terraform will error if you try to destroy this
    ignore_changes        = [tags] # Don't trigger update if tags change outside Terraform

    replace_triggered_by = [       # Force replacement when these change
      aws_security_group.web.id
    ]
  }
}
```

| **Argument** | **Behavior** |
| --- | --- |
| `create_before_destroy` | Creates new resource **before** destroying old one (avoids downtime) |
| `prevent_destroy` | Terraform errors on any plan that would destroy this resource |
| `ignore_changes` | Ignores changes to specified attributes (use `all` to ignore everything) |
| `replace_triggered_by` | Forces replacement when referenced resources/attributes change |

*Exam Trigger:* "Zero-downtime replacement" -- use `create_before_destroy = true`.

*Exam Trap:* `prevent_destroy` does **not** prevent `terraform destroy`. It only prevents destruction caused by configuration changes. Running `terraform destroy` will still remove the resource.

---

### **SECTION 4: CUSTOM CONDITIONS (NEW IN 004)**

*Validate inputs, enforce invariants, and assert expectations.*

### **1. Variable Validation Blocks**

- **The Rule:** Validate variable values during `terraform plan`. The `condition` must return `true` or Terraform errors with the `error_message`.

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed."
  }
}
```

**Key Facts:**

- Checked during **plan** (before any resources are created).
- Can only reference the variable being validated (cannot reference other variables or resources).
- Multiple `validation` blocks are allowed per variable.

---

### **2. Precondition Blocks**

- **The Rule:** Validate assumptions **before** a resource or data source is created or updated. If the condition fails, Terraform halts with an error.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.environment == "production" ? var.instance_type != "t3.micro" : true
      error_message = "Production instances must not use t3.micro."
    }
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"]
  }

  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
  }
}
```

**Key Facts:**

- Placed inside a `lifecycle` block on resources or data sources.
- Can reference variables, locals, data sources, and other resources.
- Evaluated **before** the resource is applied.

---

### **3. Postcondition Blocks**

- **The Rule:** Validate guarantees **after** a resource or data source is created or updated. Uses `self` to reference the resource's own attributes.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP address."
    }
  }
}
```

**Key Facts:**

- Evaluated **after** the resource is applied.
- Uses `self.*` to reference the resource's own computed attributes.
- Catches cases where the cloud provider returns unexpected values.

---

### **4. Check Blocks with Assert**

- **The Rule:** Continuous validation that produces **warnings**, not errors. Does **not** block `terraform apply`.

```hcl
check "website_health" {
  data "http" "web" {
    url = "https://${aws_lb.main.dns_name}/health"
  }

  assert {
    condition     = data.http.web.status_code == 200
    error_message = "Website is not returning 200 OK."
  }
}
```

**Key Facts:**

- Top-level block (not inside a resource).
- Contains a scoped `data` source and one or more `assert` blocks.
- Produces **warnings** on failure, never errors.
- Does **not** block apply or cause rollback.
- Re-evaluated on every plan/apply (continuous validation).
- Use case: verify external health checks, DNS resolution, certificate expiry.

---

### **5. Conditions Comparison Table**

| **Feature** | **Variable Validation** | **Precondition** | **Postcondition** | **Check Block** |
| --- | --- | --- | --- | --- |
| **Where** | `variable` block | `lifecycle` on resource/data | `lifecycle` on resource/data | Top-level `check` block |
| **When Evaluated** | During plan | Before apply | After apply | Every plan/apply |
| **On Failure** | **Error** (blocks plan) | **Error** (blocks apply) | **Error** (blocks apply) | **Warning** (does not block) |
| **Can Reference** | Only its own variable | Variables, locals, resources | `self.*` attributes | Scoped data source |
| **Use Case** | Validate input values | Validate assumptions | Validate outcomes | Continuous monitoring |

**Exam Trigger:** "Validate that an AMI exists before creating an instance" -- precondition.

**Exam Trigger:** "Ensure instance got a public IP after creation" -- postcondition (uses `self.*`).

**Exam Trigger:** "Monitor website health without blocking deploys" -- check block with assert.

**Exam Trap:** Check blocks produce **warnings**, not errors. They never block `terraform apply`.

---

### **SECTION 5: EPHEMERAL VALUES (NEW IN 004)**

*Values that exist only during plan/apply and are NEVER written to state.*

### **1. Ephemeral Variables**

- **The Rule:** Variables marked `ephemeral = true` are never persisted in plan files or state. They exist only in memory during the Terraform run.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
  ephemeral = true
}
```

---

### **2. Ephemeral Outputs**

- **The Rule:** Outputs marked `ephemeral = true` are not stored in state and cannot be referenced by other configurations via `terraform_remote_state`.

```hcl
output "temp_token" {
  value     = provider_resource.token.value
  ephemeral = true
}
```

---

### **3. Ephemeral Resources**

- **The Rule:** Resources declared with `ephemeral` resource type exist only during the Terraform operation. They are never recorded in state.

```hcl
ephemeral "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "my-db-password"
}

resource "aws_db_instance" "main" {
  password = ephemeral.aws_secretsmanager_secret_version.db_password.secret_string
}
```

---

### **4. Write-Only Arguments**

- **The Rule:** Certain resource arguments can be marked as write-only by the provider. These values are sent to the API but **never stored in state** and never appear in plan output.

```hcl
resource "aws_db_instance" "main" {
  engine         = "mysql"
  instance_class = "db.t3.micro"

  # write-only: sent to AWS API but not stored in state
  password_wo         = var.db_password
  password_wo_version = 1
}
```

**Key Facts:**

- Write-only arguments are defined by the **provider**, not by the user.
- The value is transmitted to the provider during apply but never recorded in state.
- Use a version number argument to trigger updates when the secret changes (since Terraform cannot detect changes to a value it does not store).

---

### **5. Ephemeral Values Summary**

| **Feature** | **Stored in State?** | **Stored in Plan File?** | **Use Case** |
| --- | --- | --- | --- |
| `sensitive = true` | **YES** (in state) | **YES** (in plan) | Redact from CLI output only |
| `ephemeral = true` (variable) | **NO** | **NO** | Secrets passed to providers |
| `ephemeral = true` (output) | **NO** | **NO** | Temporary values between modules |
| Ephemeral resource | **NO** | **NO** | Fetch secrets at runtime only |
| Write-only argument | **NO** | **NO** | Provider sends value but never stores it |

**Exam Trigger:** "Prevent secrets from being stored in Terraform state" -- use ephemeral values or write-only arguments.

**Exam Trap:** `sensitive = true` only redacts from CLI output. The value is **still stored in state**. Only `ephemeral = true` prevents state persistence.

---

### **SECTION 6: PROVISIONERS (LAST RESORT)**

*Execute scripts on local or remote machines. Use only when no better alternative exists.*

### **1. Provisioner Types**

- **The Rule:** HashiCorp recommends provisioners as a **last resort**. Prefer provider-native resources (e.g., `aws_instance` `user_data`) whenever possible.

**local-exec:** Runs a command on the machine running Terraform.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }
}
```

**remote-exec:** Runs commands on the remote resource via SSH or WinRM.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }
}
```

**file:** Copies files or directories from local machine to remote resource.

```hcl
resource "aws_instance" "web" {
  # ... connection block required ...

  provisioner "file" {
    source      = "app/config.yml"
    destination = "/etc/app/config.yml"
  }
}
```

---

### **2. Creation-Time vs Destroy-Time Provisioners**

- **The Rule:** By default, provisioners run at **creation time**. Use `when = destroy` to run at destroy time.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  # Runs when resource is created
  provisioner "local-exec" {
    command = "echo 'Instance created: ${self.id}'"
  }

  # Runs when resource is destroyed
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance destroyed: ${self.id}'"
  }
}
```

---

### **3. on_failure Behavior**

| **Setting** | **Behavior** |
| --- | --- |
| `on_failure = fail` | (Default) Terraform marks resource as **tainted** and halts |
| `on_failure = continue` | Terraform logs the error but continues execution |

```hcl
provisioner "local-exec" {
  command    = "some-command-that-might-fail"
  on_failure = continue
}
```

---

### **4. Connection Block**

- **The Rule:** `remote-exec` and `file` provisioners require a `connection` block to reach the remote resource.

```hcl
connection {
  type        = "ssh"        # or "winrm"
  user        = "ubuntu"
  private_key = file("~/.ssh/id_rsa")
  host        = self.public_ip
  timeout     = "5m"
}
```

**Exam Trigger:** "Run a script on a remote server after creation" -- `remote-exec` provisioner with a `connection` block.

**Exam Trap:** HashiCorp always considers provisioners a **last resort**. If an exam question offers a provider-native alternative (e.g., `user_data` for EC2), choose the native approach.

---

### **SECTION 7: DYNAMIC BLOCKS**

*Generate repeated nested blocks from collections.*

### **1. Dynamic Block Syntax**

- **The Rule:** Use `dynamic` blocks to programmatically generate repeated nested blocks (like `ingress` rules) from a variable.

```hcl
variable "ingress_ports" {
  default = [80, 443, 8080]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**How It Works:**

- `dynamic "<block_label>"` — The label of the nested block to generate (e.g., `"ingress"`).
- `for_each` — The collection to iterate over.
- `content {}` — The body of each generated block.
- Inside `content`, use `<block_label>.key` and `<block_label>.value` to access the current item.

```hcl
# Dynamic block with a map
variable "ingress_rules" {
  default = {
    http  = { port = 80,  cidr = "0.0.0.0/0" }
    https = { port = 443, cidr = "0.0.0.0/0" }
    ssh   = { port = 22,  cidr = "10.0.0.0/8" }
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

**Custom Iterator Name:**

```hcl
dynamic "ingress" {
  for_each = var.ingress_ports
  iterator = port       # Use "port" instead of "ingress"
  content {
    from_port = port.value
    to_port   = port.value
    protocol  = "tcp"
  }
}
```

*Exam Trap:* HashiCorp recommends using dynamic blocks **sparingly**. Overuse makes configurations harder to read and maintain. If only a few blocks are needed, write them out explicitly.

---

### Exam Summary Cheat Sheet (Memorize This)

1. **Variable precedence (lowest to highest)?** -- Default < terraform.tfvars < *.auto.tfvars < -var-file < -var / TF_VAR_ (last wins).
2. **terraform.tfvars vs other .tfvars files?** -- `terraform.tfvars` and `*.auto.tfvars` are auto-loaded. All others require `-var-file`.
3. **for_each won't accept a list?** -- Use `toset()` to convert list to set first.
4. **Resources destroyed when removing middle item?** -- Switch from `count` (index-based) to `for_each` (key-based).
5. **Validate variable input?** -- `validation` block inside `variable` declaration.
6. **Validate before resource creation?** -- `precondition` in `lifecycle` block.
7. **Validate after resource creation?** -- `postcondition` in `lifecycle` block (uses `self.*`).
8. **Continuous validation without blocking apply?** -- `check` block with `assert` (produces warnings only).
9. **Prevent secrets from being stored in state?** -- Ephemeral values (`ephemeral = true`) or write-only arguments.
10. **sensitive = true stores value in state?** -- **YES**. It only redacts CLI output. Use ephemeral for state exclusion.
11. **Run script on remote server?** -- `remote-exec` provisioner with `connection` block (last resort).
12. **Generate repeated nested blocks?** -- `dynamic` block with `for_each` and `content`.
13. **No user-defined functions?** -- Correct. Terraform only supports built-in functions. Use `locals` for reusable expressions.
14. **Zero-downtime replacement?** -- `lifecycle { create_before_destroy = true }`.
15. **Conditional resource creation?** -- `count = var.enabled ? 1 : 0`.

---

# **REAL EXAM SCENARIOS**

### **Scenario 1: The "Variable Precedence Battle" (Variable Precedence)**

**The Situation:** A team defines `region = "us-east-1"` as the default in their variable block. They also have `region = "us-west-2"` in `terraform.tfvars`. A CI/CD pipeline passes `-var 'region=eu-west-1'` on the command line. Which region does Terraform use?

**The Options:**

A. `us-east-1` because the default value in the variable block is always used.

B. `us-west-2` because `terraform.tfvars` overrides everything.

C. `eu-west-1` because the `-var` CLI flag has the highest precedence.

D. Terraform throws an error because the variable is set in multiple places.

**The Logic:**

- **Trap:** Option A is wrong. Default values have the **lowest** precedence and are overridden by any other source.
- **Trap:** Option B is wrong. `terraform.tfvars` is auto-loaded but has lower precedence than CLI flags.
- **The Fix:** **Option C**. The precedence order is: Default (1) < terraform.tfvars (2) < -var CLI flag (5). The `-var` flag wins. Terraform does not error on multiple definitions -- it uses the highest-precedence value.

---

### **Scenario 2: The "Disappearing Servers" (count vs for_each)**

**The Situation:** A team manages 5 servers using `count = length(var.server_names)` where `var.server_names = ["web", "api", "db", "cache", "worker"]`. They remove `"api"` from the list (now 4 items). After running `terraform plan`, Terraform wants to destroy **3 servers** and recreate them instead of just removing 1.

**The Options:**

A. Use `terraform taint` on the api server before removing it from the list.

B. Switch from `count` to `for_each` with `toset(var.server_names)`.

C. Add `lifecycle { prevent_destroy = true }` to the resource.

D. Use `terraform state rm` to manually remove the api server from state.

**The Logic:**

- **Trap:** Option A taints a resource for recreation, which does not help here -- the issue is index shifting.
- **Trap:** Option C prevents destruction entirely, which blocks the desired removal of the api server.
- **The Fix:** **Option B**. With `count`, resources are identified by index (`[0]`, `[1]`, `[2]`...). Removing index `[1]` shifts all subsequent indexes, causing `[2]` to become `[1]`, etc. -- Terraform sees different configurations at each index and destroys/recreates. With `for_each`, resources are identified by **key** (`["web"]`, `["api"]`, `["db"]`...). Removing `"api"` only affects that one resource.

---

### **Scenario 3: The "Precondition Guard" (Custom Conditions -- NEW 004)**

**The Situation:** A platform team wants to enforce that production workloads never use `t3.micro` instances. Developers occasionally set `instance_type = "t3.micro"` for production deployments. The team wants Terraform to **block the plan** before any resources are created.

**The Options:**

A. Add a `check` block with an `assert` that validates the instance type.

B. Add a `postcondition` in the instance's `lifecycle` block.

C. Add a `precondition` in the instance's `lifecycle` block that checks `var.environment` and `var.instance_type`.

D. Add a `validation` block to the `instance_type` variable.

**The Logic:**

- **Trap:** Option A is wrong. `check` blocks produce **warnings**, not errors. They would not block the deployment.
- **Trap:** Option B is wrong. `postcondition` runs **after** the resource is created -- the undersized instance would already exist.
- **Trap:** Option D is tempting, but `validation` blocks can only reference **their own variable**. They cannot cross-reference `var.environment` with `var.instance_type`.
- **The Fix:** **Option C**. A `precondition` runs **before** apply and can reference multiple variables and resources. It produces an **error** that blocks execution. Example: `condition = var.environment == "production" ? var.instance_type != "t3.micro" : true`.

---

### **Scenario 4: The "Leaking Secrets" (Ephemeral Values -- NEW 004)**

**The Situation:** A security audit reveals that database passwords are stored in plain text in Terraform state files (even though `sensitive = true` is set on the variable). The security team requires that passwords **never appear in state** under any circumstances. The team uses AWS Secrets Manager to store the password.

**The Options:**

A. Encrypt the state file with a KMS key using backend encryption settings.

B. Mark the variable as `sensitive = true` and store state in S3 with SSE.

C. Use an ephemeral resource to fetch the secret from Secrets Manager, ensuring the value is never written to state.

D. Use `terraform state rm` to remove the password from state after each apply.

**The Logic:**

- **Trap:** Option A encrypts state at rest, but the password is still **inside** the state file (encrypted). An operator with state access can still decrypt and see it.
- **Trap:** Option B is the same problem. `sensitive = true` redacts CLI output but the value is still written to state.
- **Trap:** Option D is a manual process that is error-prone and would break on the next apply.
- **The Fix:** **Option C**. Ephemeral resources exist **only during the Terraform run**. The secret is fetched from Secrets Manager at apply time, passed to the provider, and never recorded in state or plan files. This is the only option that truly eliminates secrets from state.

---

### **Scenario 5: The "Dynamic Security Group" (Dynamic Blocks + for_each)**

**The Situation:** A team manages a security group with 20 ingress rules. Each rule has a different port, protocol, and CIDR block stored in a variable of type `map(object({...}))`. Writing 20 separate `ingress` blocks is verbose and error-prone. They want to generate the rules dynamically from the variable.

**The Options:**

A. Use `count` on the `aws_security_group` resource to create 20 security groups, one per rule.

B. Use a `dynamic "ingress"` block with `for_each` to generate all 20 rules inside a single security group.

C. Use `for_each` on the `aws_security_group` resource to create 20 security groups.

D. Write a custom Terraform function to generate the ingress blocks.

**The Logic:**

- **Trap:** Option A creates 20 **separate security groups**, not 20 rules in one group.
- **Trap:** Option C has the same problem -- 20 security groups, each with one rule.
- **Trap:** Option D is impossible. Terraform does **not** support user-defined functions.
- **The Fix:** **Option B**. A `dynamic "ingress"` block iterates over the map variable with `for_each`, generating one `ingress` block per map entry inside a **single** security group. Each generated block uses `ingress.value.port`, `ingress.value.protocol`, etc. from the `content` block.
