# Enterprise Engineering Guide: Hardened GKE Spoke, Namespace Isolation, and Secretless Azure Identity Federation

This architecture and deployment guide details the end-to-end implementation of an enterprise-grade, production-hardened Google Kubernetes Engine (GKE) Standard Spoke Cluster. It includes rigid local namespace isolation boundaries, developer restriction policies, and an open OIDC federated trust plane connecting GKE directly to Microsoft Entra ID (Azure AD) for secretless IAM cross-cloud validation.

---

## 1. Enterprise Decision Matrix (Best Practices Alignment)

Production-grade environments require structural isolation, minimal attack surfaces, and strict programmatic restrictions. The architectural selections below enforce compliance with zero-trust design rules.

| Infrastructure Component | 🟢 Selected Enterprise Option | 🔴 Rejected Alternative | 🔎 Strategic Architectural Justification |
| :--- | :--- | :--- | :--- |
| **Network Core Controller** | **Network Connectivity Center (NCC) Hub** | Legacy VPC Network Peering Mesh | VPC Peering exhibits a strict limit of 25 network connections and introduces mesh peering loops. NCC handles automated routing tables linearly across hundreds of spoke subnets. |
| **GKE Network Layer** | **VPC-Native (Alias IP Ranges)** | Routes-Based IP Allocation | VPC-Native registers Pod IPs natively into the Google Cloud VPC. It eliminates complex custom static routing matrices and is required to run advanced Dataplane V2 architectures. |
| **Kubernetes Data Plane** | **GKE Dataplane V2 (Cilium eBPF)** | Legacy Kube-Proxy (iptables) | Kube-proxy evaluates sequential firewall rules line-by-line, causing packet drops at enterprise scale. Dataplane V2 injects eBPF code directly into the Linux kernel for real-time routing. |
| **Cluster Node Access** | **Private Worker Nodes + Cloud NAT** | Public Node Pool Infrastructures | Public node clusters assign external IPs to compute hosts, leaving them exposed to port scanners. Private nodes hide instances entirely, routing outbound via a secure, stateful NAT gateway. |
| **Node Identity Profile** | **Minimal Custom Service Account (SA)** | Default Compute Engine SA | The default Compute Engine service account carries editor rights capable of deleting project assets. Custom SAs limit nodes strictly to telemetry and logging scopes. |
| **Developer Access Model** | **Namespace Scoped Roles & Bindings** | ClusterRoleBindings | ClusterRoleBindings grant administrative authority across the entire compute footprint. Namespace-scoped tokens contain human error and prevent developers from touching system components. |
| **Cross-Cloud Identity** | **OIDC Workload Identity Federation** | Storing Static Azure Principal Keys | Hardcoding long-lived passwords or client secrets inside Kubernetes creates an immediate credential-leak vector. Federated trusts exchange ephemeral tokens dynamically. |

---

## 2. Infrastructure Automation Blueprint (`main.tf`)

Save the template block below exactly as your infrastructure deployment manifest (`main.tf`). This configuration establishes your production spoke networks, custom subnet secondary ranges, outbound Cloud NAT engines, custom IAM node roles, and the fully hardened GKE infrastructure cluster.

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.10.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# --- Infrastructure Orchestration Variables ---
variable "project_id" {
  type        = string
  description = "The target Google Cloud Project ID hosting your production GKE spoke workloads"
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "Target geographical infrastructure deployment region"
}

variable "zone" {
  type        = string
  default     = "us-central1-a"
  description = "Compute Engine node hardware deployment zone"
}

# --- Hardened Spoke Network Topologies ---
resource "google_compute_network" "spoke_vpc" {
  name                    = "gke-spoke-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "gke_subnet" {
  name          = "gke-spoke-sub"
  ip_cidr_range = "10.30.0.0/22"
  region        = var.region
  network       = google_compute_network.spoke_vpc.id

  secondary_ip_range {
    range_name    = "pods-range"
    ip_cidr_range = "172.16.0.0/14"
  }

  secondary_ip_range {
    range_name    = "services-range"
    ip_cidr_range = "172.20.0.0/20"
  }
}

# --- Secure Outbound Egress Gateways (Cloud NAT) ---
resource "google_compute_router" "nat_router" {
  name    = "spoke-nat-router"
  network = google_compute_network.spoke_vpc.id
  region  = var.region
}

resource "google_compute_router_nat" "nat_gateway" {
  name                               = "spoke-nat-gateway"
  router                             = google_compute_router.nat_router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# --- Hardened Node Account IAM Definitions ---
resource "google_service_account" "gke_sa" {
  account_id   = "gke-node-sa"
  display_name = "Hardened Least Privilege GKE Node Pool Service Account"
}

resource "google_project_iam_binding" "log_writer" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  members = ["serviceAccount:${google_service_account.gke_sa.email}"]
}

resource "google_project_iam_binding" "metric_writer" {
  project = var.project_id
  role    = "roles/monitoring.metricWriter"
  members = ["serviceAccount:${google_service_account.gke_sa.email}"]
}

# --- Hardened Enterprise GKE Cluster Blueprint ---
resource "google_container_cluster" "prod_cluster" {
  name     = "prod-spoke-gke-01"
  location = var.zone

  network    = google_compute_network.spoke_vpc.id
  subnetwork = google_compute_subnetwork.gke_subnet.id

  # Deletes standard generic default node profiles completely
  remove_default_node_pool = true
  initial_node_count       = 1

  # GKE Dataplane V2 Enforcement (Cilium eBPF network routing stack)
  datapath_provider = "ADVANCED_DATAPATH"

  # Enforce host-level Shielded Nodes
  enable_shielded_nodes = true
  networking_mode       = "VPC_NATIVE"

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods-range"
    services_secondary_range_name = "services-range"
  }

  # Control Plane Isolation Matrix
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = true
    master_ipv4_cidr_block  = "10.30.252.0/28"
  }

  # Limits management access strictly to designated transit core segments
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.10.0.0/24"
      display_name = "central-transit-hub-network"
    }
  }

  # Native Google Workload Identity Configuration
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
}

# --- Dedicated Secure Production Node Pool Subsystem ---
resource "google_container_node_pool" "hardened_nodes" {
  name       = "prod-core-pool"
  location   = var.zone
  cluster    = google_container_cluster.prod_cluster.name
  node_count = 2

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    machine_type = "e2-standard-4"
    image_type   = "COS_CONTAINERD" # Container-Optimized OS enforces read-only storage root flags

    service_account = google_service_account.gke_sa.email
    oauth_scopes    = ["https://googleapis.com"]

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    metadata = {
      disable-legacy-endpoints = "true"
    }
  }
}

# --- Link the Spoke VPC into the Central Network Connectivity Center (NCC) Hub ---
resource "google_network_connectivity_spoke" "gke_spoke_attachment" {
  name        = "gke-production-spoke"
  location    = "global"
  hub         = "projects/YOUR_HUB_TRANSIT_PROJECT_ID/global/hubs/global-ncc-hub"
  description = "Production GKE Spoke integration into the global enterprise transit fabric"

  linked_vpc_network {
    uri = google_compute_network.spoke_vpc.id
  }
}
```

---

## 3. Imperative Step-by-Step CLI Execution Sequence

If you are executing deployment stages manually or preparing pipelines, execute the following commands in exact order inside your Cloud Shell session.

```bash
# 3.1 Session Variable Mapping Configuration
export PROJECT_ID="YOUR_GCP_PRODUCTION_PROJECT_ID"
export REGION="us-central1"
export ZONE="us-central1-a"

# 3.2 Initialize Core GCP API Gateways
gcloud services enable ://googleapis.com ://googleapis.com --project=$PROJECT_ID

# 3.3 Link Terminal Credentials context to the deployed Cluster
gcloud container clusters get-credentials prod-spoke-gke-01 --zone=$ZONE --project=$PROJECT_ID

# 3.4 Extract Public GKE Cluster OIDC Issuer Endpoint for Azure Trust Setup
gcloud container clusters describe prod-spoke-gke-01 \
    --zone=$ZONE \
    --project=$PROJECT_ID \
    --format="value(identityServiceConfig.serviceProviderConfig)"
```
*Note the resulting OIDC string (e.g., `https://://googleapis.com/v1/projects/...`). This value is required for Step 5.*

---

## 4. Kubernetes Runtime Containment Configuration

Save the configuration text block below as `01-developer-containment.yaml` and execute its application via `kubectl apply -f 01-developer-containment.yaml`. This script generates an isolated application namespace, limits engineering roles strictly to that coordinate space, blocks systemic views, and deploys physical computing capacity throttles to safeguard `kube-system` resources.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod-apps
---
