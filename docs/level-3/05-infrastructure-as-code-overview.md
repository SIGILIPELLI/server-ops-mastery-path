# 05 · Infrastructure as Code Overview

Every server built by hand — SSH in, run commands from module 1's memory —
is a snowflake: nobody fully remembers how it got into its current state,
and rebuilding it after a disaster (module 03) means hoping the runbook is
complete. Infrastructure as Code (IaC) means describing infrastructure in
files, checked into version control, that a tool applies to make reality
match the description.

## Why IaC, concretely

- **Reproducibility** — the same Terraform/Ansible files produce the same
  infrastructure every time, on demand — this is what actually makes a DR
  plan's "rebuild from scratch" step achievable within an RTO, instead of
  aspirational.
- **Review and audit trail** — infrastructure changes go through the same
  pull-request review as application code; `git blame`/`git log` shows who
  changed what firewall rule and when, and why (the PR description).
  Directly relevant to Level 4's compliance/audit module.
- **Drift detection** — the tool can tell you when reality has diverged
  from the declared state (see module 06), instead of that divergence
  being invisible until it causes an incident.
- **Disposable environments** — spinning up a full staging environment
  identical to prod becomes "run the same code with a different variable
  file," not "SSH around for a day copying config by hand."

## Two categories of tools

**Provisioning tools** (Terraform, Pulumi, CloudFormation) — declare *what
infrastructure should exist*: this VPC, these subnets, this load balancer,
this database instance. They talk to a cloud provider's API and reconcile
actual resources to match the declared state.

**Configuration management tools** (Ansible, Chef, Puppet, Salt) — declare
*what should be true on top of already-existing machines*: these packages
installed, this config file with this content, this service enabled and
running. They typically run over SSH (Ansible) or an agent (Chef/Puppet).

In practice the two are often combined: Terraform provisions the VMs and
networking, Ansible configures what runs on each VM. Kubernetes manifests
and Helm charts play a similar declarative role one layer up, for workloads
running inside a cluster.

## Declarative vs. imperative

A bash script that runs `apt install nginx` is **imperative** — it
describes the *steps* to take, and running it twice may error or behave
differently the second time (`apt install` is idempotent-ish, but plenty of
shell scripts are not — see module 06). IaC tools are **declarative** — you
describe the *desired end state*, and the tool figures out (and shows you)
what changes are needed to get there, safely re-running any number of
times.

```hcl
# Terraform: declare desired state
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.small"
  tags = { Name = "web-01" }
}
```

```yaml
# Ansible: declare desired state on an existing host
- name: nginx is installed and running
  hosts: web
  tasks:
    - name: install nginx
      apt:
        name: nginx
        state: present
    - name: nginx enabled and started
      service:
        name: nginx
        state: started
        enabled: true
```

Run either of these ten times in a row with nothing else changing, and
nothing changes on the tenth run either — that's the idempotency property
module 06 covers in depth.

## Worked example: Terraform for a VM + Ansible for its config

`main.tf` — provision one VM and a firewall rule allowing HTTP/HTTPS/SSH:

```hcl
resource "aws_security_group" "web_sg" {
  name = "web-sg"
  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["203.0.113.0/24"]   # admin network only, not 0.0.0.0/0
  }
  ingress {
    from_port = 443
    to_port   = 443
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t3.small"
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = "deploy-key"
  tags = { Name = "web-01" }
}

output "web_public_ip" {
  value = aws_instance.web.public_ip
}
```

```bash
terraform init
terraform plan     # shows what WOULD change — review before applying, always
terraform apply    # provisions the VM + security group
```

`inventory.ini` + `site.yml` — configure what Terraform just created:

```ini
[web]
web-01 ansible_host=<terraform output web_public_ip>
```

```yaml
- name: configure web-01
  hosts: web
  become: true
  tasks:
    - name: install nginx
      apt: { name: nginx, state: present, update_cache: true }
    - name: deploy site config
      copy:
        src: files/app.conf
        dest: /etc/nginx/sites-available/app.conf
      notify: reload nginx
    - name: enable site
      file:
        src: /etc/nginx/sites-available/app.conf
        dest: /etc/nginx/sites-enabled/app.conf
        state: link
      notify: reload nginx
  handlers:
    - name: reload nginx
      service: { name: nginx, state: reloaded }
```

```bash
ansible-playbook -i inventory.ini site.yml
```

`terraform plan` before every `apply` is the single most important habit in
this file — it's the diff review step that catches "this change will
destroy and recreate the database instance" before it happens instead of
after.

## State: the part that bites people

Terraform tracks what it created in a **state file**. Losing it, or having
two people apply from stale state at the same time, causes Terraform to
lose track of real resources (leading to duplicate resources, or attempts
to recreate things that already exist). In any team setting, state must be
stored remotely (S3 + DynamoDB lock table, Terraform Cloud, etc.), never
just as a local file on one engineer's laptop:

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "web/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

The `dynamodb_table` provides locking — a second `terraform apply` started
while one is already running waits/fails instead of racing against the
first and corrupting state.

## How It Actually Works

**Why Terraform needs a state file at all, when the cloud provider already
knows what exists.** Terraform's `.tf` files describe desired resources by
logical name (`aws_instance.web`), but cloud APIs identify resources by
opaque IDs (`i-0abc123...`) that don't appear anywhere in your code. The
state file is the mapping between the two — without it, `terraform plan`
would have no way to know that `aws_instance.web` in your code *is*
`i-0abc123` in AWS rather than a brand-new resource to create. This is
also why deleting a resource outside Terraform (module 06's drift) doesn't
just get silently ignored: `plan` reads the state file's ID, queries AWS
for it, gets "not found," and reports that as a resource needing to be
recreated — the state file, not the code, is what `plan` diffs live
infrastructure against first.

**Why the DynamoDB lock table prevents state corruption, mechanically.**
Terraform doesn't merge concurrent changes to state — it treats state as a
single mutable blob it downloads, mutates in memory during `apply`, and
uploads back. Two `apply` runs racing against the same state file would
each read the same starting point, make independent changes, and the
second upload would silently clobber the first's changes with no
awareness they ever happened. The DynamoDB table is used purely as a
distributed lock: before touching state, Terraform writes a lock record
with a client ID; a second `apply` attempting to acquire the same lock
sees the existing record and refuses to proceed until it's released —
turning a silent, undetectable race into an explicit "state locked, try
again later" error.

**Why `terraform apply` sometimes wants to destroy and recreate a resource
you only edited slightly.** Not every attribute of a cloud resource is
mutable in place — an EC2 instance's AMI, for instance, can't be changed
on a running instance via API; the only way to get that instance onto a
new AMI is to terminate it and launch a new one. Terraform's provider
schema marks such attributes `ForceNew`, so any diff touching one produces
a destroy-then-create plan rather than an update — which is exactly the
kind of line `terraform plan`'s review step is meant to catch before it
runs against a resource holding a live database, since "recreate" for a
stateful resource means data loss unless it's backed by separate durable
storage.

## Exercise

1. Write a Terraform file that provisions one VM and a security group
   restricting SSH to a specific CIDR (not `0.0.0.0/0`). Run `terraform
   plan`, read the plan output line by line before running `apply`.
2. Write an Ansible playbook that installs and configures nginx on the VM
   Terraform created, using an inventory built from the Terraform output.
3. Run the Ansible playbook twice in a row and confirm the second run
   reports no changes (`ok=N changed=0`) — this is idempotency in practice,
   the subject of the next module.
4. Delete the VM out-of-band (via the cloud console, not Terraform) to
   simulate drift, then run `terraform plan` again and explain what it
   reports and why.
