# GCP Marketplace Deployment Stack

This repository contains a production-ready pipeline for baking and deploying a Google Cloud Platform (GCP) Marketplace Virtual Machine image. It leverages **HashiCorp Packer** to build a secure, scrubbed Google Compute Engine (GCE) image and **Terraform** to provision and test that image.

---

## 📋 Prerequisites

Before you begin, ensure you have the following installed and configured on your local machine:

* **Google Cloud SDK (`gcloud`)**: Authenticated to your target GCP project (`gcloud auth application-default login`).
* **Packer**: Version 1.1.4 or higher.
* **Terraform**: Version 5.0 or higher.
* **GCP Project**: A dedicated project with the Compute Engine API enabled and billing attached.

---

## 🛠️ Step 1: Configuration (What to Change)

This stack is driven by your `scaffold.yaml` and the automated setup scripts. Before building, you need to point the pipeline at your actual application.

### 1. Update `scaffold.yaml`
Open your `scaffold.yaml` file and locate the `depends_on` block. Update the URL and branch to point to the Git repository of the application you want to bake into the image.

```yaml
depends_on:
  - url: "[https://github.com/your-org/your-app-repo.git](https://github.com/your-org/your-app-repo.git)"
    branch: "v1.0.0" # Use a stable release tag!
```

### 2. Customize `setup-vm.sh`
The `setup-vm.sh` script automatically installs standard system dependencies and compiles the libraries defined in your configuration. However, for a complete Marketplace product, you may need to add custom application logic at the bottom of this file.

Common additions include:
* Creating a dedicated non-root service user.
* Setting up a `systemd` service to ensure your application starts automatically on boot.
* Downloading proprietary configuration files or license managers.

*Note: The end of `setup-vm.sh` automatically scrubs the bash history, SSH keys, and `apt` caches. This is a strict requirement for GCP Marketplace images. Do not remove the scrubbing block.*

---

## 🧱 Step 2: Baking the Image

Once your configuration is set, you will use Packer to create the machine image. The included `build.sh` script automates this process.

Run the following command from the root of this directory:

```bash
./build.sh build
```

**What happens behind the scenes:**
1. Packer spins up a temporary Ubuntu VM in your GCP project.
2. It transfers and executes `setup-vm.sh` to install your application.
3. It shuts down the VM, takes a snapshot of the disk, and saves it as a Custom Image.
4. The image is tagged with your `project_slug-family` so it is easy to reference later.

---

## 🧪 Step 3: Local Testing

Before submitting to Google, you must verify that your baked image boots correctly and that your application runs as expected. You can use the included Terraform configuration to spin up a test instance.

Initialize Terraform:
```bash
terraform init
```

Provision the test infrastructure:
```bash
terraform apply
```

Terraform will automatically find the newest image in your image family and attach it to a new VM. Once applied, SSH into the VM, verify your application is running, and test your endpoints. When you are finished, destroy the test resources:

```bash
terraform destroy
```

---

## 🚀 Step 4: Submitting to GCP Marketplace

Publishing an image to the GCP Marketplace is a formal business process. Here is the roadmap to get your baked image listed.

### 1. Join Google Cloud Partner Advantage
You cannot publish to the Marketplace as an individual. Your company must apply and be accepted into the **Google Cloud Partner Advantage** program as a "Build" partner.

### 2. Access the Producer Portal
Once accepted, navigate to the **Producer Portal** inside the Google Cloud Console. This is your command center for managing Marketplace listings.

### 3. Create a VM Product
In the Producer Portal, create a new "Virtual Machine" product. You will be asked to provide:
* Marketing materials (Icons, descriptions, feature lists).
* Support contact information and SLAs.
* Pricing models (e.g., Free, BYOL, or Usage-based).

### 4. Self-Certify the Image
Google requires all VM images to pass a specific set of security and compliance checks. Your Packer script (`setup-vm.sh`) already handles the most common requirements:
* No hardcoded passwords or SSH keys.
* Scrubbed command history and temporary files.
* Standardized OS baselines.

Google will provide an automated test suite (often run via a container) that you must run against your running VM. You will submit the passing logs to the Producer Portal.

### 5. Submit for Review
Point your Producer Portal listing to the image URI you generated with Packer. Submit the listing for Google's engineering and marketing review. This process typically takes a few weeks and may involve some back-and-forth communication. Once approved, your application will be live for all GCP users!
