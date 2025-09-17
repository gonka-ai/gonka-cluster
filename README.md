# Gonka Two-Cluster Deployment Scripts

This repository contains Ansible scripts for deploying two independent Gonka Network clusters with optimized model distribution.

## 🚀 Quick Start

### Prerequisites

**Note:** The `inferenced` CLI binary should be placed in the same directory as the Ansible deployment scripts. The deployment will run CLI commands from the current working directory.

### Local Environment Setup

1. **Clone the Repository:**
```bash
# Replace 'gonka' with your actual GitHub organization/username
git clone https://github.com/gonka/gonka-cluster.git
cd gonka-cluster
```

2. **Download and Setup Inferenced CLI:**

**For macOS ARM64 (M1/M2):**
```bash
curl -L -o inferenced.zip https://github.com/gonka-ai/gonka/releases/download/release%2Fv0.2.0/inferenced-darwin-arm64.zip
unzip inferenced.zip
chmod +x inferenced
./inferenced --help
```

**⚠️ MacOS Users:** If you see a security warning, go to System Settings → Privacy & Security, scroll down to find the warning about "inferenced" being blocked, and click "Allow Anyway" to enable execution.

3. **Set Environment Variable:**
```bash
export GONKA_KEYRING_PASSWORD="your-secure-password-here"
```

4. **Generate Account Keys** (run locally with the environment variable set):
```bash
./inferenced keys add gonka-account-key-cluster1 --keyring-backend file --home ./gonka-keys-cluster1
./inferenced keys add gonka-account-key-cluster2 --keyring-backend file --home ./gonka-keys-cluster2
```

5. **Update Inventory** - Replace placeholder IPs in `inventory.ini` with your actual server IPs

6. **Install Ansible** on your deployment machine:

**For macOS (with Homebrew):**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install ansible
ansible-galaxy collection install community.docker ansible.posix
```

**For macOS (with pip):**
```bash
python3 -m pip install --upgrade pip
pip3 install ansible
ansible-galaxy collection install community.docker ansible.posix
```

**Verify Installation:**
```bash
ansible --version
ansible-galaxy --version
```

### Deploy Test Cluster

**Important:** Keep the `GONKA_KEYRING_PASSWORD` environment variable set during the entire deployment process.

```bash
# Ensure environment variable is set
echo $GONKA_KEYRING_PASSWORD

# Run complete deployment
ansible-playbook -i inventory.ini playbooks/deploy.yml

# Or run specific phases for testing
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase0
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase1
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase2
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase3
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase4
ansible-playbook -i inventory.ini playbooks/deploy.yml --tags phase5

# Validate the test deployment
ansible-playbook -i inventory.ini playbooks/validate.yml
```

## 📋 Cluster Architecture

### Each Cluster (6 servers total):
- **1 Network Node**: Runs Qwen/Qwen3-32B-FP8 (empty args)
- **5 ML Nodes**:
  - ML-01 to ML-04: Qwen/Qwen3-235B-A22B-Instruct-2507-FP8 (tensor-parallel-size: 4)
  - ML-05: Qwen/Qwen3-32B-FP8 (empty args)

### Total Deployment:
- **2 Clusters** × **6 servers** = **12 servers**
- **All models cached** via HuggingFace CLI
- **Independent operation** with separate account keys

## 📁 Directory Structure

```
gonka-cluster/
├── ansible.cfg                 # Ansible configuration
├── inventory.ini              # Server inventory (UPDATE IPs!)
├── vars/
│   └── main.yml              # Global variables
├── templates/
│   ├── config.env.j2         # Environment config template
│   └── node-config.json.j2   # Node config template (generates node-config-generated.json)
├── roles/                    # Ansible roles
│   ├── key-setup/           # Phase 0: Account key extraction
│   ├── firewall/            # Firewall configuration
│   ├── dependencies/        # System dependencies
│   ├── git/                 # Repository cloning
│   ├── huggingface/         # Model downloads
│   ├── docker/              # Docker image management
│   ├── config/              # Configuration generation
│   ├── network-init/        # Initial blockchain services
│   ├── keygen/              # ML operational key generation
│   ├── registration/        # Host registration
│   ├── permissions/         # Permission granting
│   ├── network-full/        # Full network activation
│   ├── ml-deploy/           # ML node deployment
│   ├── ml-register/         # ML node registration
│   ├── cluster-check/       # Health validation
│   └── performance/         # Performance monitoring
└── playbooks/               # Deployment playbooks
    ├── deploy.yml          # Main deployment
    ├── validate.yml        # Health checks
    └── cleanup.yml         # Emergency cleanup
```

## ⚙️ Configuration

### Update Server IPs

Edit `inventory.ini` and replace placeholder IPs with your actual server IPs:

```ini
# Replace YOUR_CLUSTER1_NETWORK_IP with actual IP
[cluster1_network_node]
cluster1-network ansible_host=YOUR_CLUSTER1_NETWORK_IP

# Replace all YOUR_*_IP placeholders with actual IPs
[cluster1_ml_nodes]
cluster1-ml-01 ansible_host=YOUR_CLUSTER1_ML01_IP
cluster1-ml-02 ansible_host=YOUR_CLUSTER1_ML02_IP
# ... etc
```

### SSH Configuration

Ensure you have SSH key access to all servers:

**Option 1: Copy SSH key to servers:**
```bash
ssh-copy-id -i ~/.ssh/gonka-deploy-key ubuntu@YOUR_SERVER_IP
```

**Option 2: Update ansible.cfg with your key path:**
```bash
private_key_file = ~/.ssh/your-key-file
```

### Docker-Firewall Integration
The deployment automatically configures Docker to respect UFW firewall rules:
- **Docker Group Setup**: Creates docker group and adds ubuntu user for sudo-less Docker usage
- **Firewall Integration**: Disables Docker's default iptables management to prevent rule conflicts
- **Performance Optimization**: Disables userland-proxy for direct iptables routing (better for ML workloads)
- **Inter-Container Communication**: Allows containers to communicate with each other (required for distributed ML)
- **Inferenced Execution**: All blockchain operations run inside Docker containers (API containers)
- **Security Enforcement**: Ensures UFW rules apply to all Docker container ports
- **Network Isolation**: Maintains proper security isolation between clusters and services

**Important**: This configuration ensures that your carefully crafted firewall rules (restricting ML node ports to network node access only) actually work as intended, preventing Docker from bypassing UFW.

## 🎯 Deployment Phases

### Phase 0: Account Key Setup (Local Machine)
- Extract public keys from locally pre-generated account keys
- Use local inferenced binary (macOS version)
- Validate key format and prepare for remote deployment
- No remote server access required
- No sudo/become privileges required (runs as current user)

### Phase 1: Infrastructure Setup
- Configure firewall rules
- Install system dependencies
- Validate system requirements

### Phase 2: Repository & Models
- Clone Gonka deployment repository
- Download all required models via HuggingFace
- Pull Docker images

### Phase 3: Configuration & Keys
- Generate config.env for all servers
- Generate node-config-generated.json for network nodes
- Create ML operational keys (remote)
- Register hosts with blockchain (remote)
- Grant ML operation permissions (local machine, no sudo required)

### Phase 4: Network Activation
- Launch complete network node services
- Verify API and RPC endpoints

### Phase 5: ML Node Deployment
- Deploy ML inference services
- **DAPI connectivity validation**: Check ML node can access network node DAPI API (using actual IP)
- **Pre-registration validation**: Check ML node accessibility from network node
- Register ML nodes with network node
- Validate model loading

### Phase 6: Validation
- Health checks across all services
- Performance monitoring
- Cluster integration verification

## 🔍 Validation & Monitoring

### Health Checks
```bash
# Validate all services
ansible-playbook -i inventory.ini playbooks/validate.yml

# Check specific cluster
ansible-playbook -i inventory.ini playbooks/validate.yml --limit cluster1

# Check ML nodes only
ansible-playbook -i inventory.ini playbooks/validate.yml --limit cluster1_ml_nodes
```

### Performance Monitoring
```bash
# Check GPU utilization
ansible cluster1_ml_nodes -m command -a "nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits"

# Check Docker status
ansible cluster1 -m command -a "docker ps --format table"

# Check model cache
ansible cluster1_ml_nodes -m command -a "du -sh /mnt/shared"
```

## 🔐 Security & Access

### Admin API Access (Port 9200)

The **Admin API is NOT exposed publicly** for security reasons. To access it:

```bash
# SSH tunnel to access Admin API locally
ssh -L 9200:localhost:9200 ubuntu@YOUR_NETWORK_NODE_IP

# Then access locally on your machine
curl http://localhost:9200/admin/v1/nodes
```

### Port Security

- **✅ Public Ports**: 22 (SSH), 80/443 (HTTP/HTTPS), 8000 (API), 26657 (RPC)
- **🔒 Restricted Ports**:
  - 5000 (Inference), 8080 (Management) - ML nodes only accessible from network node
  - 9100 (DAPI ML Server), 9300 (gRPC ML Server) - Network node only accessible from ML nodes
- **❌ Private Ports**: 9200 (Admin API) - SSH tunneling required
- **🔒 Firewall**: UFW configured with minimal required ports + IP-based restrictions

## 🚨 Troubleshooting

### Common Issues

1. **SSH Connection Failed**
   ```bash
   # Test SSH connectivity
   ansible all -m ping -i inventory.ini

   # Debug SSH issues
   ansible all -m command -a "whoami" -i inventory.ini -vvv
   ```

2. **Model Download Failed**
   ```bash
   # Check disk space
   ansible cluster1 -m command -a "df -h /mnt/shared"

   # Retry model downloads
   ansible-playbook -i inventory.ini playbooks/deploy.yml --tags huggingface
   ```

3. **Docker Service Issues**
   ```bash
   # Check Docker status
   ansible cluster1 -m command -a "docker ps"

   # Restart services
   ansible cluster1 -m command -a "docker-compose restart"
   ```

### Emergency Cleanup
```bash
# Stop all services and clean up
ansible-playbook -i inventory.ini playbooks/cleanup.yml

# Clean specific cluster
ansible-playbook -i inventory.ini playbooks/cleanup.yml --limit cluster1
```

## 📊 Model Distribution

### Qwen/Qwen3-32B-FP8 (Empty Args)
- **Network Nodes**: Both cluster1 and cluster2
- **ML Node 5**: In each cluster
- **Total**: 4 servers across both clusters

### Qwen/Qwen3-235B-A22B-Instruct-2507-FP8 (Tensor Parallel: 4)
- **ML Nodes 1-4**: In each cluster
- **Args**: `--max-model-len 240000 --tensor-parallel-size 4`
- **Total**: 8 servers across both clusters

## 🔐 Security Notes

- **Environment Variable**: `GONKA_KEYRING_PASSWORD` is used for all key operations
- **Secure Storage**: Never store passwords in files or version control
- **Session Only**: Set environment variable only for deployment duration
- **Separate Account Keys**: Each cluster uses its own account key
- **Isolated Keyrings**: `./gonka-keys-cluster1`, `./gonka-keys-cluster2`
- **Firewall Rules**: Only required ports are open
- **SSH Keys**: Key-based authentication only
- **Backup Keys**: ML operational key mnemonics are backed up

### Environment Variable Management
```bash
# Set password for deployment
export GONKA_KEYRING_PASSWORD="your-password"

# Run deployment
ansible-playbook -i inventory.ini playbooks/deploy.yml

# Clear password when done
unset GONKA_KEYRING_PASSWORD
```
