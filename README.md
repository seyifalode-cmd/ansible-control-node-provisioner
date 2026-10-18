# Ansible Control Node Provisioner

Oluwaseyi Michael Falode | Cloud & DevOps Engineering | Toronto, ON

Terraform-driven provisioning of an AWS EC2 instance that bootstraps itself into a fully configured Ansible control node — including SSH key generation — using remote-exec and file provisioners.

---

## Project at a Glance

| | |
|---|---|
| **Tools Used** | Terraform, Ansible, AWS EC2, AWS SSM Parameter Store |
| **Platform** | Amazon Web Services (us-east-1) |
| **Languages** | HCL (Terraform), YAML (Ansible) |
| **What It Does** | Provisions an EC2 instance, installs Ansible, and generates an SSH key pair for managing downstream nodes |

---

## The Problem This Project Solves

Standing up an Ansible control node by hand is a repeatable but error-prone process. Engineers must launch an EC2 instance, SSH in, install Ansible and its dependencies, configure EPEL, and then generate an SSH key pair — steps that differ across environments and are easy to forget or misconfigure. Manual provisioning also leaves no auditable record of how the environment was built.

This project eliminates that toil. A single `terraform apply` launches the EC2 instance, installs Ansible via `amazon-linux-extras`, enables the EPEL repository (needed for certain Ansible modules), and runs a playbook that generates a 4096-bit RSA SSH key pair under the `ec2-user` account. The public key is captured and displayed as Terraform output, ready to be injected into managed node authorized_keys or a separate provisioner.

The result is a fully reproducible, code-driven control node that can be rebuilt in minutes from a clean AWS account. This is the foundation layer of a multi-tier Ansible automation environment — the control node whose IP and key are later consumed by the managed node provisioner.

---

## Architecture

```
Local Machine
     |
     | terraform apply
     v
+---------------------------+
|   Terraform (HCL)         |
|   - AWS Provider          |
|   - aws_key_pair          |
|   - aws_security_group    |
|   - aws_instance          |
+---------------------------+
     |
     | Provisions
     v
+------------------------------------+
|  EC2 Instance (Amazon Linux 2)     |
|  t3.micro  |  us-east-1            |
|  Security Group: SSH(22), HTTP(80) |
|                                    |
|  remote-exec (inline):             |
|    1. yum update                   |
|    2. install ansible2 (EPEL)      |
|    3. enable EPEL repo             |
|                                    |
|  file provisioner:                 |
|    install_control_node.yaml -->   |
|    /home/ec2-user/                 |
|                                    |
|  ansible-playbook (localhost):     |
|    - openssh_keypair (4096-bit)    |
|    - cat ~/.ssh/id_rsa.pub         |
|    - debug: display public key     |
+------------------------------------+
     |
     | Terraform output
     v
  Public IP of the control node
  Public SSH key (printed to stdout)
```

---

## Repository Structure

```
ansible-control-node-provisioner/
├── main.tf                     # EC2 instance, security group, key pair, provisioners
├── variables.tf                # SSH key paths and example variable definitions
├── outputs.tf                  # Exports the control node's public IP
├── install_control_node.yaml   # Ansible playbook: generates SSH key pair on the node
├── host.ini                    # Sample inventory file for downstream managed nodes
└── .gitignore
```

---

## How It Works

**Step 1 — Terraform resolves the AMI.**
Instead of hardcoding an AMI ID, the configuration fetches the latest Amazon Linux 2 HVM AMI from AWS SSM Parameter Store. This keeps the code region-agnostic and always current.

**Step 2 — EC2 is launched with an SSH key pair.**
Terraform uploads the local public key (`~/.ssh/id_ed25519.pub` by default) as an AWS key pair resource and associates it with the instance, enabling the subsequent SSH-based provisioners.

**Step 3 — The playbook is copied to the instance.**
The `file` provisioner transfers `install_control_node.yaml` from the local workspace to the home directory of `ec2-user` on the new EC2 instance.

**Step 4 — Ansible is installed and the playbook is run.**
The `remote-exec` provisioner runs inline commands: updates the OS, installs Ansible 2 via `amazon-linux-extras`, enables the EPEL repository, and executes the playbook against `localhost`.

**Step 5 — SSH key pair is generated.**
The playbook uses the `openssh_keypair` module to create a 4096-bit RSA key at `~/.ssh/id_rsa`. The public key is echoed and registered, then displayed via the `debug` module. This public key is what gets added to the `authorized_keys` of the managed nodes provisioned by the companion project.

---

## Walkthrough

```bash
# 1. Initialize Terraform and download the AWS provider
terraform init

# 2. Preview what will be created
terraform plan

# 3. Provision the control node (SSH key paths can be overridden)
terraform apply \
  -var="ssh_key_private=~/.ssh/id_ed25519" \
  -var="ssh_key_public=~/.ssh/id_ed25519.pub"

# 4. Retrieve the public IP from Terraform output
terraform output Ansible-ControlNode-PublicIP

# 5. SSH into the newly provisioned control node
ssh -i ~/.ssh/id_ed25519 ec2-user@<public-ip>

# 6. Verify Ansible is installed and the SSH key exists
ansible --version
cat ~/.ssh/id_rsa.pub

# 7. Tear down the environment when finished
terraform destroy
```

---

## How to Reproduce

**Prerequisites**

- Terraform >= 1.0.0
- An AWS account with credentials configured (`aws configure` or environment variables)
- An existing SSH key pair on your local machine

```bash
# Clone the repository
git clone https://github.com/seyifalode-cmd/ansible-control-node-provisioner.git
cd ansible-control-node-provisioner

# Initialize Terraform
terraform init

# Apply — replace key paths with your actual key locations
terraform apply \
  -var="ssh_key_private=${HOME}/.ssh/id_ed25519" \
  -var="ssh_key_public=${HOME}/.ssh/id_ed25519.pub" \
  -auto-approve

# Note the output
terraform output Ansible-ControlNode-PublicIP
```

After `apply` completes, the control node is live. The public key printed during provisioning can be passed to the `ansible-managed-nodes-provisioner` project to authorize the control node's access to managed hosts.

---

## Related Projects

This repository is part of a multi-project Ansible automation series:

- `ansible-sandbox-ec2` — Minimal single-node Ansible sandbox
- `ansible-control-node-provisioner` — **This project** — Full control node with SSH key generation
- `ansible-managed-nodes-provisioner` — Four managed nodes in a custom VPC
- `ansible-control-node-setup` — Playbooks for configuring managed nodes (httpd + MariaDB)
- `mariadb-ansible-setup` — Self-contained MariaDB provisioning via Terraform + Ansible

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
