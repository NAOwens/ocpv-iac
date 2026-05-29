# Provision RHEL Virtual Machines (IaC)

This repository contains the Infrastructure as Code (IaC) playbook required to provision Red Hat Enterprise Linux (RHEL) Virtual Machines (VMs) on OpenShift Virtualization. It is specifically designed to automate the deployment of virtualized workloads into an **OpenShift environment** utilizing OVN-Localnet networking and integrated storage provisioning.

## Overview

This toolset consists of a single, comprehensive playbook designed to handle the end-to-end lifecycle of a VM deployment. By leveraging the `kubernetes.core.k8s` module, the playbook translates simple variable inputs into a complex KubeVirt manifest.

The automation manages the following key areas:

* **Dynamic Manifest Generation:** Constructs a `VirtualMachine` object with precise instance types and OS preferences.
* **Multi-Interface Networking:** Attaches the VM to both the standard pod network and a secondary bridge network via Multus.
* **Storage Automation:** Utilizes `DataVolumeTemplates` to clone existing RHEL images from the cluster's image registry.

---

## 1. VM Provisioning Playbook (`provision_rhel_vm_full.yml`)

**Purpose:** Deploys a RHEL Virtual Machine instance with automated SSH key injection and secondary network attachments.
**Target Environment:** OpenShift Virtualization (KubeVirt).

**Key Features:**

* **Automated Credential Injection:** Uses `cloudInitNoCloud` to securely propagate SSH public keys from a Kubernetes Secret into the VM.
* **Advanced Networking:** Configures dual interfaces: a `masquerade` interface for the default pod network and a `bridge` interface connected to a specified Network Attachment Definition (NAD).
* **Image Versioning:** Dynamically selects the source image (RHEL 8, 9, or 10) based on user input, referencing standard cluster `DataSources`.
* **Storage Integration:** Provisions a 30Gi root disk using a specified StorageClass, ensuring the volume is created and ready before the VM starts.

**Required Variables / Survey Inputs:**

* `survey_vm_name`: The name assigned to the VM and its associated volume.
* `survey_os_choice`: The RHEL version to deploy (e.g., `rhel8`, `rhel9`, `rhel10`).
* `survey_url`: The API host URL for the target OpenShift cluster.
* `survey_token`: The API token used for authentication.

---

## Workflow / Order of Operations

When deploying a new Virtual Machine, execute the playbook as follows:

1. **Select OS Image:** Ensure the required RHEL image exists as a `DataSource` in the `openshift-virtualization-os-images` namespace.
2. **Verify Network Attachment:** Confirm the `baremetal-network` (or specified `nad_name`) is already defined in the target namespace.
3. **Execute Provisioning:** Run `provision_rhel_vm_full.yml` to create the VM manifest and trigger the DataVolume clone.
4. **Wait for Deployment:** The `runStrategy: Always` ensures the VM boots automatically once the disk cloning is complete.

## Known Architectural Quirks Managed by this Codebase

* **SSH Key Propagation:** The playbook explicitly defines a `cloudinitdisk` volume. This is required to bridge the gap between Kubernetes Secrets and the VM's internal `noCloud` metadata service.
* **SSL Verification:** For environments using internal service addresses with self-signed certificates, `verify_ssl` is set to `false` to prevent connection failures during deployment.
* **Instance Typing:** Uses `VirtualMachineClusterInstancetype` to standardize hardware resources (CPU/RAM) across the cluster.

## Manual steps needed on the new VM after running deploy

1. Verify that the secondary interface is correctly up and assigned an IP from the OVN-Localnet pool.
