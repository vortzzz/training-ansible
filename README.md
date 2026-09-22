# training-ansible — Step by Step

A hands-on guide to configure an Azure VM with Ansible, where **Terraform builds
the infrastructure and Ansible installs the software inside it**.

When you finish you will have:

- An Azure VM (Ubuntu 22.04) created with Terraform
- Docker CE installed remotely with Ansible
- A Super Mario Bros container running on `http://<PUBLIC_IP>:8787`

> Estimated time: ~20 minutes (most of it is Azure provisioning).

---

## Prerequisites

| Tool | Check with | Install |
|------|-----------|---------|
| Azure CLI | `az version` | [docs](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| Terraform | `terraform version` | [docs](https://developer.hashicorp.com/terraform/install) |
| Ansible | `ansible --version` | `sudo apt install -y ansible` |
| sshpass | `sshpass -V` | `sudo apt install -y sshpass` |

### Step 0 — Install the local tools

```bash
sudo apt update
sudo apt install -y ansible sshpass
```

Confirm:

```bash
$ ansible --version
ansible [core 2.16.3]

$ sshpass -V
sshpass 1.09
```

> **Why `sshpass`?** Your VM authenticates with a **password**, not an SSH key
> (`disable_password_authentication = false` in `main.tf`). Ansible's default
> `ssh` connection plugin calls the real OpenSSH client, and OpenSSH refuses to
> take a password non-interactively. `sshpass` is the small helper that feeds it
> to OpenSSH. Without it you get:
>
> ```
> to use the 'ssh' connection type with passwords ... you must install the sshpass program
> ```
>
> **Why not `pip install ansible`?** Not needed. The Ubuntu package already
> bundles all community collections, including `community.docker` — which is what
> the `docker_container` module lives in.

### Step 1 — Log in to Azure

```bash
az login
az account show
```

A browser window opens. Use a student/trial subscription if you have one.

---

## Part A — Create the infrastructure with Terraform

Run everything in the **parent folder** (`VMTerraform/`), not in this one.

### Step 2 — Set your credentials

```bash
cd VMTerraform
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

```hcl
admin_username = "ValeriaTerraformVM"
admin_password = "Password1234567!"
```

The password must satisfy Azure's rules: min. 12 characters, upper + lower case,
a number and a symbol.

> **This file is required.** `admin_password` has no default, so `terraform plan`
> fails with `No value for variable admin_password` if it is missing.
> It is gitignored — never commit it.

### Step 3 — Initialize and apply

```bash
terraform init      # downloads the azurerm + local providers (first time only)
terraform plan      # preview — no changes are made yet
terraform apply     # type 'yes'
```

Terraform creates: resource group → vnet → subnet → public IP → NSG → NIC → VM.

Grab the public IP:

```bash
terraform output public_ip
# 20.62.117.94
```

### Step 4 — Make Terraform generate the Ansible inventory

Add this to `main.tf`. It is what bridges the two tools:

```hcl
resource "local_file" "ansible_inventory" {
  filename        = "${path.module}/training-ansible/inventory/hosts.ini"
  file_permission = "0600"

  content = <<-EOT
    [azure_vm]
    ${azurerm_public_ip.pip.ip_address} ansible_user=${var.admin_username} ansible_ssh_pass=${var.admin_password}
  EOT
}
```

And declare the provider it needs, in `providers.tf`:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}
```

Apply again:

```bash
terraform apply
```

Verify the file was written:

```bash
cat training-ansible/inventory/hosts.ini
```

```ini
[azure_vm]
20.62.117.94 ansible_user=ValeriaTerraformVM ansible_ssh_pass=Password1234567!
```

> **Why generate it instead of typing the IP?** Because the IP changes every time
> you destroy and recreate the VM, and a stale inventory silently points Ansible
> at the wrong machine. Now `terraform apply` is the single source of truth.
>
> **Why `file_permission = "0600"`?** The file holds a password — only your user
> may read it. For the same reason it is **gitignored**; a safe template lives in
> `inventory/hosts.ini.example`.

### Step 5 — Open the port for the container

Still in `main.tf`, add a second `security_rule` inside
`azurerm_network_security_group.nsg`:

```hcl
  security_rule {
    name                       = "DockerContainer"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "8787"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
```

```bash
terraform apply
az network nsg rule list --nsg-name vmterraform-nsg --resource-group vmterraform-rg \
  --query "[].{name:name,prio:priority,port:destinationPortRange}" -o table
```

```
Name             Prio    Port
---------------  ------  ------
DockerContainer  1002    8787
SSH              1001    22
```

> **Why this step?** The NSG is Azure's firewall and it denies everything that is
> not explicitly allowed. Docker can map `8787 → 8080` *inside* the VM and the
> container still looks perfectly healthy from within — but nobody outside can
> reach it. This is the most confusing failure in the whole guide, so do not skip
> it. Priority `1002` because priorities must be unique and `1001` is already SSH.

---

## Part B — Build the Ansible project

### Step 6 — Create the folder structure

```bash
mkdir -p training-ansible/{inventory,playbooks,roles/docker_install/tasks,roles/docker_container/tasks}
cd training-ansible
```

```text
training-ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini                   # generated by terraform
│   └── hosts.ini.example
├── playbooks/
│   ├── install_docker.yml
│   └── run_container.yml
└── roles/
    ├── docker_install/tasks/main.yml
    └── docker_container/tasks/main.yml
```

### Step 7 — `ansible.cfg`

```ini
[defaults]
host_key_checking = False
roles_path = ./roles
inventory = ./inventory/hosts.ini
```

> `host_key_checking = False` skips the "REMOTE HOST IDENTIFICATION HAS CHANGED"
> prompt when you recreate the VM but reuse the same IP.

### Step 8 — `playbooks/install_docker.yml`

```yaml
---
- hosts: azure_vm
  become: yes
  roles:
    - docker_install
```

### Step 9 — `roles/docker_install/tasks/main.yml`

```yaml
- name: Install Docker dependencies
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - software-properties-common
    state: present
    update_cache: yes

- name: Add the official Docker GPG key
  ansible.builtin.apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add the Docker repository
  ansible.builtin.apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu jammy stable
    state: present

- name: Install Docker CE
  apt:
    name: docker-ce
    state: present
    update_cache: yes

# The docker_container module talks to Docker through the Python SDK,
# which must be installed on the TARGET, not on your machine.
- name: Install the Docker SDK for Python
  apt:
    name: python3-docker
    state: present

# So the normal user can run docker without sudo
- name: Add the user to the docker group
  user:
    name: "{{ ansible_user_id }}"
    groups: docker
    append: yes
```

> **Three details that matter:**
>
> | Detail | Why |
> |--------|-----|
> | `jammy` in the repo URL | The Docker apt repo is per Ubuntu release. Your VM is 22.04 = `jammy`. Using `bionic` (18.04) breaks dependency resolution. |
> | `gnupg` in the list | Required by `apt_key`, and listed in Docker's official install docs. |
> | **`python3-docker`** | Without it `run_container.yml` fails with `Failed to import the required Python library (Docker SDK for Python)`. The SDK goes on the **server**, not on your laptop. |

### Step 10 — `playbooks/run_container.yml`

```yaml
---
- hosts: azure_vm
  become: yes
  roles:
    - docker_container
```

### Step 11 — `roles/docker_container/tasks/main.yml`

```yaml
- name: Run Mario Bros
  docker_container:
    name: supermario-container
    image: "pengbai/docker-supermario:latest"
    state: started
    ports:
      - "8787:8080"
```

### Step 12 — Gitignore the credentials

```bash
echo "inventory/hosts.ini" > .gitignore
git rm --cached inventory/hosts.ini   # only if it was already tracked
```

Create a safe template:

```bash
cat > inventory/hosts.ini.example <<'EOF'
[azure_vm]
<PUBLIC_IP> ansible_user=<USER> ansible_ssh_pass=<PASSWORD>
EOF
```

---

## Part C — Run it

### Step 13 — Test connectivity first

**Always run from inside `training-ansible/`** — `ansible.cfg` uses relative
paths.

```bash
ansible azure_vm -m ping
```

```
20.62.117.94 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

Do not continue until you see `pong`. This isolates SSH/credential problems from
playbook problems.

### Step 14 — Install Docker

```bash
ansible-playbook playbooks/install_docker.yml
```

```
PLAY RECAP *********************************************************************
20.62.117.94 : ok=7  changed=6  unreachable=0  failed=0  skipped=0
```

`failed=0` is what you want. `changed=6` means it actually installed things.

### Step 15 — Start the container

```bash
ansible-playbook playbooks/run_container.yml
```

```
PLAY RECAP *********************************************************************
20.62.117.94 : ok=2  changed=1  unreachable=0  failed=0  skipped=0
```

---

## Part D — Verify

### Step 16 — Confirm the container is running

```bash
ansible azure_vm -m shell -a "docker ps" -b
```

```
CONTAINER ID   IMAGE                              STATUS        PORTS                   NAMES
854e1a09cff2   pengbai/docker-supermario:latest   Up 1 minute   0.0.0.0:8787->8080/tcp  supermario-container
```

> The `-b` flag (become/root) is required: your user was added to the `docker`
> group, but a group change only takes effect in a **new login session**.

### Step 17 — Confirm it is reachable from the internet

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 20 "http://$(terraform output -raw public_ip):8787/"
```

```
HTTP 200
```

> **Test from your laptop, not from inside the VM.** A check from inside would
> pass even with a broken NSG rule. This single `curl` proves three things at
> once: the NSG rule, the Docker port mapping and the container itself.

### Step 18 — Play

Open in your browser:

```text
http://<PUBLIC_IP>:8787
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `you must install the sshpass program` | `sudo apt install -y sshpass` |
| `No inventory was parsed` / `Could not match supplied host pattern: azure_vm` | You are not inside `training-ansible/`. |
| `No value for variable admin_password` | `terraform.tfvars` is missing — see Step 2. |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | `ssh-keygen -f ~/.ssh/known_hosts -R '<PUBLIC_IP>'` |
| `permission denied while trying to connect to the docker API` | Add `-b` to the command, or log in to the VM again. |
| `ModuleNotFoundError: No module named 'docker'` | Re-run `install_docker.yml` — `python3-docker` is missing on the target. |
| `paramiko is not installed` | You passed `-c paramiko`. Drop the flag, or install `python3-paramiko`. |
| Container runs but the browser won't load | Check Step 5 — the NSG rule for 8787. |

---

## Cleanup

On a student account the credit is limited. From the parent folder:

```bash
cd ..
terraform destroy     # type 'yes'
```

This removes the VM, network, NSG, public IP and resource group — and with them
the container and Docker. `terraform init` does **not** need to be run again
afterwards.

To bring it back: `terraform plan` → `terraform apply`.

---

## Summary of what was configured and why

| Change | Reason |
|--------|--------|
| `local_file` resource + `local` provider | Terraform writes the inventory, so Ansible always targets the right IP. |
| NSG rule for port `8787` | Azure denies everything not explicitly allowed. |
| Repository `bionic` → `jammy` | The Docker apt repo must match Ubuntu 22.04. |
| Added `gnupg` | Required by `apt_key`. |
| Added `python3-docker` | The `docker_container` module needs the Python SDK **on the target**. |
| Added user to `docker` group | Run `docker` without `sudo`. |
| `hosts.ini` gitignored | It contains a real password. |
| Installed `ansible` + `sshpass` | To run the playbooks against a password-authenticated VM. |

---

*PLATAFORMAS 2 — Azure VM + Terraform + Ansible.*
