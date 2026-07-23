# Provision RHEL Virtual Machines (IaC)

This repository contains the Infrastructure as Code (IaC) playbooks required to provision and manage Red Hat Enterprise Linux (RHEL) Virtual Machines on OpenShift Virtualization. It is specifically designed to automate the deployment and lifecycle management of virtualized workloads into an **OpenShift environment** utilizing OVN-Localnet networking and integrated storage provisioning.

## Overview

The automation manages the following key areas:

* **Dynamic Manifest Generation:** Constructs `VirtualMachine` objects with explicit CPU and memory sizing.
* **Multi-Interface Networking:** Attaches VMs to both the standard pod network and a secondary bridge network via Multus.
* **Storage Automation:** Utilizes `DataVolumeTemplates` to clone existing RHEL images from the cluster's image registry.
* **In-Place Resize:** Patches existing VMs with updated CPU and memory without reprovisioning.

---

## 1. VM Provisioning Playbook (`provision_rhel_vm_full.yml`)

**Purpose:** Deploys a new RHEL Virtual Machine instance with automated SSH key injection and secondary network attachment.
**Target Environment:** OpenShift Virtualization (KubeVirt).

**Key Features:**

* **Explicit CPU and Memory Sizing:** Configures CPU cores and memory directly in the domain spec rather than using `VirtualMachineClusterInstancetype`, giving full control over resource allocation per VM.
* **Automated Credential Injection:** Uses `cloudInitNoCloud` to securely propagate SSH public keys from a Kubernetes Secret into the VM.
* **Advanced Networking:** Configures dual interfaces — a `masquerade` interface for the default pod network and a `bridge` interface connected to a specified Network Attachment Definition (NAD).
* **Image Versioning:** Dynamically selects the source image (RHEL 8, 9, or 10) based on user input, referencing standard cluster `DataSources`.
* **Storage Integration:** Provisions a 30Gi root disk using a specified StorageClass.

**AAP Survey Variables:**

| Variable | Type | Required | Default | Description |
|---|---|---|---|---|
| `survey_vm_name` | Text | Yes | — | Name assigned to the VM and its associated volume |
| `survey_os_choice` | Multiple Choice | Yes | — | RHEL version to deploy: `rhel8`, `rhel9`, or `rhel10` |
| `survey_vm_cpu` | Integer | No | `1` | Number of CPU cores |
| `survey_vm_memory` | Multiple Choice | No | `4Gi` | Memory allocation (e.g., `2Gi`, `4Gi`, `8Gi`, `16Gi`) |
| `survey_url` | Text | Yes | — | API host URL for the target OpenShift cluster |
| `survey_token` | Password | Yes | — | API token used for authentication |

**Example execution:**

```bash
ansible-playbook playbooks/provision_rhel_vm_full.yml \
  -e "survey_vm_name=my-rhel9-vm" \
  -e "survey_os_choice=rhel9" \
  -e "survey_vm_cpu=2" \
  -e "survey_vm_memory=4Gi" \
  -e "survey_url=https://api.cluster.example.com:6443" \
  -e "survey_token=<token>"
```

---

## 2. Bulk VM Provisioning Playbook (`provision_multiple_vms.yml`)

**Purpose:** Deploys multiple RHEL Virtual Machines in a single run using the same networking and storage configuration as `provision_rhel_vm_full.yml`.
**Target Environment:** OpenShift Virtualization (KubeVirt).

**Key Features:**

* **Batch Provisioning:** Loops over a user-defined list of VM definitions, provisioning each sequentially.
* **Per-VM Customization:** Each VM in the list can specify its own OS image, CPU, and memory independently.
* **Shared Infrastructure:** All VMs in a run share the same namespace, storage class, SSH secret, and NAD.

**VM list fields:**

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | Yes | — | Name assigned to the VM and its associated volume |
| `os` | Yes | — | RHEL version to deploy: `rhel8`, `rhel9`, or `rhel10` |
| `cpu` | No | `1` | Number of CPU cores |
| `memory` | No | `4Gi` | Memory allocation (e.g., `2Gi`, `4Gi`, `8Gi`, `16Gi`) |

### Running from the CLI

Create a `vms.yml` file with the list of VMs to provision:

```yaml
# vms.yml
vms:
  - name: web-01
    os: rhel9
    cpu: 2
    memory: 8Gi
  - name: web-02
    os: rhel9
    cpu: 2
    memory: 8Gi
  - name: db-01
    os: rhel9
    cpu: 4
    memory: 16Gi
```

Then run:

```bash
ansible-playbook playbooks/provision_multiple_vms.yml \
  -e "survey_url=https://api.cluster.example.com:6443" \
  -e "survey_token=<token>" \
  -e @vms.yml
```

### Running from AAP (Recommended)

1. Create a Job Template pointing to `provision_multiple_vms.yml`.
2. In the Job Template, add the `vms` list to the **Extra Variables** field:

```yaml
vms:
  - name: web-01
    os: rhel9
    cpu: 2
    memory: 8Gi
  - name: web-02
    os: rhel9
    cpu: 2
    memory: 8Gi
```

3. Enable **Prompt on Launch** for the Extra Variables field so operators can edit the VM list each time the job is run.
4. Add `survey_url` and `survey_token` via a survey or credential injection.

> **AAP tip:** Use JSON format in the Extra Variables field to avoid YAML parsing errors. JSON is unambiguous and AAP accepts it without issues:
> ```json
> {
>   "vms": [
>     {"name": "web-01", "os": "rhel9", "cpu": 2, "memory": "8Gi"},
>     {"name": "web-02", "os": "rhel9", "cpu": 2, "memory": "8Gi"}
>   ]
> }
> ```

> The `vms` list is required. The playbook will fail with a clear error if it is not provided.

---

## 3. VM Resize Playbook (`resize_vm.yml`)

**Purpose:** Modifies the CPU and/or memory of an existing Virtual Machine without reprovisioning it.
**Target Environment:** OpenShift Virtualization (KubeVirt).

**Key Features:**

* **Partial Updates:** CPU and memory are patched independently — supply only the field(s) you want to change.
* **Pre-flight Validation:** Confirms the target VM exists before attempting any changes, and enforces minimum resource floors (1 CPU core, 2Gi memory).
* **Safe Patching:** Uses `state: patched` to update only the domain CPU/memory fields, leaving all other VM configuration (networking, storage, credentials) untouched.
* **Resize Summary:** Emits a post-task message confirming the updated values and reminding that changes take effect on the next VM restart.

**AAP Survey Variables:**

| Variable | Type | Required | Default | Constraints | Description |
|---|---|---|---|---|---|
| `survey_vm_name` | Text | Yes | — | — | Name of the existing VM to resize |
| `survey_url` | Text | Yes | — | — | API host URL for the target OpenShift cluster |
| `survey_token` | Password | Yes | — | — | API token used for authentication |
| `survey_target_namespace` | Text | No | `default` | — | Namespace containing the VM |
| `survey_vm_cpu` | Integer | No | — | Min: `1` | New CPU core count (omit to leave unchanged) |
| `survey_vm_memory` | Multiple Choice | No | — | Min: `2Gi` | New memory allocation (omit to leave unchanged) |

> At least one of `survey_vm_cpu` or `survey_vm_memory` must be provided.

**Example execution:**

```bash
ansible-playbook playbooks/resize_vm.yml \
  -e "survey_vm_name=my-rhel9-vm" \
  -e "survey_vm_cpu=4" \
  -e "survey_vm_memory=8Gi" \
  -e "survey_url=https://api.cluster.example.com:6443" \
  -e "survey_token=<token>"
```

---

## Workflow / Order of Operations

### Provisioning a single VM

1. **Select OS Image:** Ensure the required RHEL image exists as a `DataSource` in the `openshift-virtualization-os-images` namespace.
2. **Verify Network Attachment:** Confirm the `baremetal-network` NAD is already defined in the target namespace.
3. **Execute Provisioning:** Run `provision_rhel_vm_full.yml` to create the VM manifest and trigger the DataVolume clone.
4. **Wait for Deployment:** The `runStrategy: Always` ensures the VM boots automatically once disk cloning is complete.

### Provisioning multiple VMs

1. **Select OS Images:** Ensure all required RHEL images exist as `DataSources` in the `openshift-virtualization-os-images` namespace.
2. **Verify Network Attachment:** Confirm the `baremetal-network` NAD is already defined in the target namespace.
3. **Define VM List:** Prepare the `vms` list in a local `vms.yml` file or in the AAP Job Template Extra Variables field.
4. **Execute Provisioning:** Run `provision_multiple_vms.yml` — VMs are provisioned sequentially in the order listed.
5. **Wait for Deployment:** Each VM boots automatically once its DataVolume clone completes.

### Resizing an existing VM

1. **Run `resize_vm.yml`** with the desired CPU and/or memory values.
2. **Restart the VM** for changes to take effect — KubeVirt applies CPU and memory changes on the next boot cycle.

---

## Known Architectural Notes

* **Explicit Sizing over Instancetypes:** These playbooks set CPU and memory directly in the domain spec (`domain.cpu.cores`, `domain.memory.guest`) rather than referencing `VirtualMachineClusterInstancetype`. This avoids instancetype conflicts when patching existing VMs and provides per-VM resource control.
* **Integer Type Enforcement:** KubeVirt's `cores` field is a `uint32`. The playbooks use `| int` combined with a `from_yaml` block scalar to ensure the value is serialized as a native integer rather than a string in the API payload.
* **SSH Key Propagation:** A `cloudinitdisk` volume is required to bridge the gap between Kubernetes Secrets and the VM's internal `noCloud` metadata service.
* **SSL Verification:** `verify_ssl: false` is set for environments using internal service addresses with self-signed certificates.

## Manual Steps After Provisioning

1. Verify the secondary interface is up and assigned an IP from the OVN-Localnet pool.
