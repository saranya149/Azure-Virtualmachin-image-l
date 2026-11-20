# Azure VM Image Lab

Create a custom image from a configured Azure Linux VM and use it to deploy identical VMs quickly.

---

## 1. Requirement

- Active Azure subscription
- Azure CLI installed (or use Azure Cloud Shell)
- Basic Linux knowledge (SSH, apt, systemctl)
- Permissions to create Resource Groups, VMs, and Images
- SSH key pair or password for VM login

---

## 2. Use Case

- Create a **golden image** with the OS, packages, and settings already configured
- Deploy multiple identical VMs without repeating manual setup
- Standardize environments for Dev, Test, and Production
- Speed up scaling (VM Scale Sets, load-balanced web servers)

---

## 3. Real-Time Scenario

A company runs a web application on 10 servers. Instead of installing Nginx, security patches, monitoring agents, and app dependencies on each VM manually:

1. The admin configures **one** VM perfectly.
2. Captures it as a custom image.
3. Deploys new VMs from that image in minutes during traffic spikes or when replacing a failed server.

---

## 4. Advantages

- Fast and consistent VM deployment
- Saves time; no repeated manual configuration
- Reduces human error
- Easy to scale out and recover from failures
- Image can be versioned and shared (Azure Compute Gallery)
- Works well with automation (Terraform, Azure DevOps)

---

## 5. Disadvantages

- Image becomes outdated; needs regular rebuilds for patches
- Source VM is unusable after generalization
- Storage cost for stored images
- Images are region-bound unless replicated via Compute Gallery
- Secrets or user data must not be baked into the image
- Managing many image versions adds overhead

---

## 6. How to Work (Step-by-Step)

### Step 1: Login and create a Resource Group

```bash
az login
az group create --name rg-image-lab --location centralindia
```

### Step 2: Create the source VM

```bash
az vm create \
  --resource-group rg-image-lab \
  --name vm-source \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard
```

### Step 3: Connect and customize the VM

```bash
ssh azureuser@<PUBLIC_IP>

sudo apt update && sudo apt install -y nginx
sudo systemctl enable nginx
echo "Hello from Custom Image" | sudo tee /var/www/html/index.html
```

### Step 4: Deprovision (clean) the VM

```bash
sudo waagent -deprovision+user -force
exit
```

### Step 5: Deallocate and generalize

```bash
az vm deallocate --resource-group rg-image-lab --name vm-source
az vm generalize --resource-group rg-image-lab --name vm-source
```

### Step 6: Create the image

```bash
az image create \
  --resource-group rg-image-lab \
  --name img-nginx-ubuntu \
  --source vm-source \
  --hyper-v-generation V1
```

> Use `--hyper-v-generation V2` if the source VM is Gen2.

### Step 7: Create a new VM from the image

```bash
az vm create \
  --resource-group rg-image-lab \
  --name vm-from-image \
  --image img-nginx-ubuntu \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

az vm open-port --resource-group rg-image-lab --name vm-from-image --port 80
```

### Step 8: Verify

Open `http://<NEW_VM_PUBLIC_IP>` in the browser. You should see **"Hello from Custom Image"**, confirming that the new VM was built from your image.

### Step 9: Clean up (avoid charges)

```bash
az group delete --name rg-image-lab --yes --no-wait
```

---

## 7. Key Learnings

- Difference between **deallocate**, **generalize**, and **capture**
- Why `waagent -deprovision` is required before imaging
- How custom images speed up deployments
- Managed Image vs. Azure Compute Gallery

---

## 8. Author

**Saranya T** | [GitHub](https://github.com/saranya149)
