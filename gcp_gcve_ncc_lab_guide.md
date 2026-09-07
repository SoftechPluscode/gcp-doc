# Comprehensive Lab Guide: End-to-End GCVE and GCE Integration via NCC Hub

This comprehensive guide outlines the exact, production-grade architectural configurations, Terraform templates, and imperative `gcloud` CLI commands required to build and test an end-to-end network data path. 

This layout connects a native Google Compute Engine (GCE) workload environment to a Google Cloud VMware Engine (GCVE) single-node evaluation private cloud, using Network Connectivity Center (NCC) as the global transit framework.

---

## 1. Network Topology & IP Allocation Grid

To prevent routing overlapping failures across the cloud fabric, each segmented zone utilizes distinct, non-overlapping IP allocation blocks:

| Environment Domain | Infrastructure Purpose | Private Network IP CIDR Block | Test Endpoint Allocation |
| :--- | :--- | :--- | :--- |
| **`central-hub-vpc`** | Central transit core hooked directly to NCC control plane. | `10.10.0.0/24` | Transit Plane (No instances required) |
| **`gce-workload-vpc`** | Houses native compute resources and application workloads. | `10.20.0.0/24` | **`gce-linux-vm`** (IP: `10.20.0.50`) |
| **`gcve-mgmt-range`** | Reserved for backend bare-metal engine appliances (ESXi, vCenter). | `10.190.0.0/24` | System Control Infrastructure Only |
| **`nsx-workload-segment`** | Software-defined network overlay built inside VMware NSX-T. | `10.200.10.0/24` | **`gcve-test-vm`** (IP: `10.200.10.50`) |

---

## 2. Infrastructure Automation: Terraform Configuration (`main.tf`)

Save the block below as `main.tf`. This template completely builds out your target baseline global VPC topology, provisions firewall profiles, structures the global NCC Hub engine, and instantiates the lightweight cost-efficient testing micro-instance.

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# --- Variables ---
variable "project_id" {
  type        = string
  description = "The target Google Cloud Project ID for deployment"
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "Primary GCP region for localized compute resources"
}

variable "zone" {
  type        = string
  default     = "us-central1-a"
  description = "The target computing availability zone"
}

# --- VPC Architecture Deployment ---
resource "google_compute_network" "central_hub" {
  name                    = "central-hub-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "hub_subnet" {
  name          = "central-hub-sub"
  ip_cidr_range = "10.10.0.0/24"
  region        = var.region
  network       = google_compute_network.central_hub.id
}

resource "google_compute_network" "gce_workload" {
  name                    = "gce-workload-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gce_subnet" {
  name          = "gce-workload-sub"
  ip_cidr_range = "10.20.0.0/24"
  region        = var.region
  network       = google_compute_network.gce_workload.id
}

# --- Network Connectivity Center (NCC) Orchestration ---
resource "google_network_connectivity_hub" "ncc_hub" {
  name        = "global-ncc-hub"
  description = "Global core transit hub linking workload and engine segments"
}

resource "google_network_connectivity_spoke" "hub_spoke" {
  name     = "central-hub-spoke"
  location = "global"
  hub      = google_network_connectivity_hub.ncc_hub.id
  linked_vpc_network {
    uri = google_compute_network.central_hub.id
  }
}

resource "google_network_connectivity_spoke" "gce_spoke" {
  name     = "gce-workload-spoke"
  location = "global"
  hub      = google_network_connectivity_hub.ncc_hub.id
  linked_vpc_network {
    uri = google_compute_network.gce_workload.id
  }
}

# --- Lab Diagnostics Security Firewall Matrices ---
resource "google_compute_firewall" "allow_icmp_hub" {
  name    = "allow-icmp-hub"
  network = google_compute_network.central_hub.name

  allow {
    protocol = "icmp"
  }
  source_ranges = ["0.0.0.0/0"]
}

resource "google_compute_firewall" "allow_icmp_ssh_gce" {
  name    = "allow-icmp-gce"
  network = google_compute_network.gce_workload.name

  allow {
    protocol = "icmp"
  }
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  source_ranges = ["0.0.0.0/0"]
}

# --- Native Verification Host Instance ---
resource "google_compute_instance" "gce_vm" {
  name         = "gce-linux-vm"
  machine_type = "e2-micro"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.gce_subnet.id
    network_ip = "10.20.0.50"
    # Omit access_config block to preserve private-only security containment
  }
}
```

---

## 3. Command Line Implementation Sequence (`gcloud`)

Because certain lifecycle controllers for the VMware Engine Evaluation tier are highly optimized via the CLI stack, execute the initialization procedures below sequentially inside your terminal window to bring up the bare-metal environment.

### Step 3.1: Initialize Ecosystem Scope Variables
Configure your session workspace variables explicitly to minimize input string errors:
```bash
export PROJECT_ID="YOUR_ACTUAL_GCP_PROJECT_ID"
export REGION="us-central1"
export ZONE="us-central1-a"
```

### Step 3.2: Initialize Google Cloud APIs
```bash
gcloud services enable ://googleapis.com --project=$PROJECT_ID
```

### Step 3.3: Instantiate the Core GCVE Enterprise Network Fabric
```bash
gcloud vmwareengine networks create gcve-global-network \
    --type=STANDARD \
    --location=global \
    --project=$PROJECT_ID
```

### Step 3.4: Deploy the Single-Node Bare-Metal Private Cloud Cluster
Execute the baseline provisioning command to pull a single node. This environment mounts local storage disks via VMware vSAN automatically utilizing an operational `FTT=0` data structure profile.
```bash
gcloud vmwareengine private-clouds create lab-sddc-01 \
    --location=$ZONE \
    --cluster-id="lab-cluster-01" \
    --management-cidr="10.190.0.0/24" \
    --node-type="ve1-standard-72" \
    --node-count=1 \
    --type=evaluation \
    --project=$PROJECT_ID
```
> **⏳ Deployment Latency Alert:** Building a dedicated physical multi-gigabit bare-metal host configuration and deploying VMware Cloud Foundation (VCF) takes roughly **30 to 45 minutes** to return a ready status flag.

### Step 3.5: Interconnect GCVE to the Central VPC Transit Core
Create a low-latency direct VPC Peering interconnect layer to bridge your GCVE cluster directly to the `central-hub-vpc` framework built via Terraform.
```bash
gcloud vmwareengine network-peerings create gcve-to-hub-peering \
    --vmwareengine-network=gcve-global-network \
    --peer-vpc="projects/$PROJECT_ID/global/networks/central-hub-vpc" \
    --export-custom-routes=true \
    --import-custom-routes=true \
    --location=global \
    --project=$PROJECT_ID
```

---

## 4. VMware SDDC Platform Customizations (Web UI Steps)

Once the core infrastructure layers report a healthy status, you must interface directly with the VMware control applications layer.

### Step 4.1: Retrieve System Credentials
Query the administrative entry passwords dynamically generated by Google Cloud:
```bash
gcloud vmwareengine private-clouds show lab-sddc-01 --location=$ZONE --project=$PROJECT_ID
```
*Note the administrative URLs and login accounts for both vCenter Server and NSX Manager.*

### Step 4.2: Build the NSX-T Software Defined Segment
1. Launch the **NSX Manager URL** in an authorized web browser window and sign in using your admin tokens.
2. Select the **Networking** menu located on the global top panel bar.
3. Select **Segments** on the left column interface, then select **Add Segment**.
4. Define the Segment properties:
   * **Segment Name:** `nsx-workload-segment`
   * **Connected Gateway:** Select the pre-configured system **`Tier-1 Gateway`**.
   * **Subnets/Gateway IP Mapping:** Set the value to **`10.200.10.1/24`**.
5. Select **Save** to distribute the network settings across the host layer.

### Step 4.3: Deploy Your Virtual Machine Inside vCenter
1. Authenticate to your **vCenter Server UI** console window.
2. Right-click your compute host cluster element tree object and click **New Virtual Machine**.
3. Proceed through the template specifications configuration wizard (Sizing, Operating System).
4. When customizing virtual hardware interfaces, pull down the **Network Adapter** configuration menu and explicitly select your newly minted software-defined layer: **`nsx-workload-segment`**.
5. Complete the setup wizard, boot the target operating system up, and configure its system configuration profile with a static IP setting of **`10.200.10.50`** (using a `/24` or `255.255.255.0` netmask with a default gateway mapping targeting `10.200.10.1`).

---

## 5. End-to-End Diagnostic Verification Procedures

Because your VPC networks are coupled to the global **NCC Hub** control engine, the software plane dynamically learns and advertises the subnets out to all associated nodes across the network mesh. 

### Step 5.1: Verify Native GCE Ingress Connectivity Path
Open a secure shell connection into the native Compute Engine machine and ping the guest device running on the VMware side to confirm proper packet transmission over the NCC boundary layer:
```bash
gcloud compute ssh gce-linux-vm --zone=$ZONE --project=$PROJECT_ID
```
