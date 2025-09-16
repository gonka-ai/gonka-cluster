# Gonka Network Cluster Deployment Plan

## Overview

This document outlines the automated deployment script for setting up **two independent Gonka Network clusters** consisting of:
- **Each cluster**: 1 Network Node + 5 ML Nodes = **6 servers per cluster**
- **Network Nodes**: Full nodes with chain, API services, and Qwen3-32B inference
- **ML Nodes**: Inference-only nodes (4× Qwen3-235B + 1× Qwen3-32B per cluster)
- **Total: 12 servers** with 8xH100 GPUs each across both clusters

### Model Distribution per Cluster
- **Network Node**: Qwen/Qwen3-32B-FP8 (empty args)
- **ML Node 1-4**: Qwen/Qwen3-235B-A22B-Instruct-2507-FP8 (tensor-parallel-size: 4)
- **ML Node 5**: Qwen/Qwen3-32B-FP8 (empty args)

## Cluster Architecture

```
Cluster 1:
├── Network Node (cluster1-network)
│   ├── Chain Node (Tendermint)
│   ├── API Node (REST API)
│   ├── ML Inference (Qwen3-32B-FP8)
│   └── Admin API (port 9200)
└── ML Nodes (cluster1-ml-01 to cluster1-ml-05)
    ├── ML-01 to ML-04: Qwen3-235B-A22B-Instruct-2507-FP8 (tensor-parallel-size: 4)
    ├── ML-05: Qwen3-32B-FP8 (empty args)
    ├── Inference Service (port 5000)
    ├── Health Check (port 5000/health)
    ├── GPU Monitoring (nvidia-smi)
    └── Connected to Network Node via Admin API

Cluster 2: (Identical structure)
├── Network Node (cluster2-network)
│   ├── Chain Node (Tendermint)
│   ├── API Node (REST API)
│   ├── ML Inference (Qwen3-32B-FP8)
│   └── Admin API (port 9200)
└── ML Nodes (cluster2-ml-01 to cluster2-ml-05)
    ├── ML-01 to ML-04: Qwen3-235B-A22B-Instruct-2507-FP8 (tensor-parallel-size: 4)
    ├── ML-05: Qwen3-32B-FP8 (empty args)
    ├── Inference Service (port 5000)
    ├── Health Check (port 5000/health)
    ├── GPU Monitoring (nvidia-smi)
    └── Connected to Network Node via Admin API
```

## Prerequisites

### Pre-Deployment Setup: Account Key Generation

**Before running the main deployment, you must generate account keys for each cluster:**

```bash
# Generate account key for Cluster 1
./inferenced keys add gonka-account-key-cluster1 --keyring-backend file --home ./gonka-keys-cluster1

# Generate account key for Cluster 2
./inferenced keys add gonka-account-key-cluster2 --keyring-backend file --home ./gonka-keys-cluster2

# Verify keys were created
./inferenced keys list --keyring-backend file --home ./gonka-keys-cluster1
./inferenced keys list --keyring-backend file --home ./gonka-keys-cluster2
```

**Store the public keys for each cluster - you'll need them during deployment.**

### Hardware Requirements
- **12 servers** running Ubuntu 22.04.5 (6 per cluster)
- **8x H100 GPUs** per server (640GB total VRAM per server)
- **16-core CPU** minimum per server (AMD64 architecture)
- **64GB+ RAM** per server (1.5x GPU VRAM ratio recommended)
- **1TB NVMe SSD** per server (NVMe for optimal I/O)
- **100Mbps+ network** (1Gbps preferred, static IPs required)

### Cluster Architecture
- **Two independent clusters** (cluster1, cluster2)
- **Each cluster**: 1 network node + 5 ML nodes = 6 servers total
- **Network nodes**: Run chain node, API node, and ML inference (Qwen3-32B)
- **ML nodes**: Dedicated inference servers (4× Qwen3-235B + 1× Qwen3-32B per cluster)

### System Requirements
- **SSH Key Authentication**: Pre-configured key-based SSH access
- **Sudo Access**: Passwordless sudo for deployment user
- **DNS Resolution**: All servers can resolve each other by hostname/IP
- **NTP Synchronization**: All servers have synchronized time
- **Firewall**: UFW installed (ports 22, 80, 443 currently open)

### Software Requirements (Pre-installed)
- Docker 28.4.0+
- NVIDIA Container Toolkit 1.17.8+
- NVIDIA GPU drivers with CUDA 12.6-12.9
- Python 3.10.12+
- SSH access with key-based authentication

### Ansible Requirements
- **Ansible 2.15+** installed on deployment machine
- **Python 3.10+** on deployment machine
- **SSH access** to all servers with key-based authentication
- **ansible.posix** and **community.docker** collections

### Control Node Setup
- Ubuntu/Debian-based deployment machine
- Git for repository cloning
- Python packages: `paramiko`, `pyyaml`, `jinja2`

## Deployment Orchestration

### Ansible-Based Deployment
This deployment uses **Ansible** for orchestration, providing:
- **Parallel execution** across all 8 servers simultaneously
- **Built-in retry logic** with exponential backoff
- **Idempotent operations** (safe to run multiple times)
- **Comprehensive error handling** and rollback capabilities
- **SSH connection management** with automatic key handling
- **Inventory-based configuration** for easy server management

### Ansible Components
- **Inventory file**: Dynamic inventory of all 8 servers
- **Playbooks**: Main deployment orchestration
- **Roles**: Modular components for each deployment phase
- **Variables**: Configurable parameters for different environments

### Network Requirements
- **Public IP addresses** for all 8 servers (provided as input list)
- **Currently open ports**: 22 (SSH), 80, 443
- **Required Gonka ports** to be opened: 5000, 26657, 8000
- Internet connectivity for model downloads
- Internal network connectivity between servers

### Firewall Configuration
- **Automatic port opening** for required Gonka services
- **UFW configuration** on Ubuntu servers
- **Port validation** after firewall updates
- **Security group updates** if applicable

### Port Mapping Architecture
- **External Load Balancer/Proxy** maps ports 5050→ML:5000, 8080→ML:8080
- **Network Node** does NOT expose 5050/8080 directly
- **ML Nodes** receive requests via internal routing (not network node proxy)
- **Network Node** focuses on blockchain operations and API services

## Deployment Script Structure

### Phase 0: Account Key Setup

**Phase 0 Overview:**
- **0.1 Key Extraction**: Extract public keys from pre-generated account keys for each cluster
- **0.2 Key Validation**: Verify keys are accessible and properly formatted
- **0.3 Key Distribution**: Make keys available to cluster-specific operations

**This phase must run before any other deployment steps.**

#### 0.1 Account Key Extraction (key-setup role)
- **Local execution**: Run on deployment control node
- **Cluster-specific keys**: Separate keys for cluster1 and cluster2
- **Public key extraction**: Get gonka1 addresses for configuration
- **Secure handling**: Keys stored securely for deployment use

**Account Key Extraction Implementation:**
```yaml
# Extract public key for Cluster 1
- name: Extract cluster1 account public key
  shell: >
    printf '%s\n' "$KEYRING_PASSWORD" |
    ./inferenced keys show gonka-account-key-cluster1
    --keyring-backend file
    --home ./gonka-keys-cluster1
  register: cluster1_key_info
  when: inventory_hostname == groups['cluster1_network_node'][0]

# Extract public key for Cluster 2
- name: Extract cluster2 account public key
  shell: >
    printf '%s\n' "$KEYRING_PASSWORD" |
    ./inferenced keys show gonka-account-key-cluster2
    --keyring-backend file
    --home ./gonka-keys-cluster2
  register: cluster2_key_info
  when: inventory_hostname == groups['cluster2_network_node'][0]

# Store cluster-specific pubkey key values
- name: Set cluster1 account pubkey
  set_fact:
    cluster1_account_pubkey: "{{ cluster1_key_info.stdout | regex_findall('\"key\":\"([^\"]+)\"') | first }}"
  when: inventory_hostname == groups['cluster1_network_node'][0]

- name: Set cluster2 account pubkey
  set_fact:
    cluster2_account_pubkey: "{{ cluster2_key_info.stdout | regex_findall('\"key\":\"([^\"]+)\"') | first }}"
  when: inventory_hostname == groups['cluster2_network_node'][0]

# Validate pubkey key values are properly extracted
- name: Validate cluster1 pubkey format
  assert:
    that: cluster1_account_pubkey is defined and cluster1_account_pubkey | length > 10
    fail_msg: "Invalid cluster1 account pubkey format"
  when: inventory_hostname == groups['cluster1_network_node'][0]

- name: Validate cluster2 pubkey format
  assert:
    that: cluster2_account_pubkey is defined and cluster2_account_pubkey | length > 10
    fail_msg: "Invalid cluster2 account pubkey format"
  when: inventory_hostname == groups['cluster2_network_node'][0]
```

### Phase 1: Environment Setup & Pre-Validation

#### 1.1 SSH Connection Management (Ansible inventory setup)
- **Ansible inventory**: Dynamic inventory file with all 8 server IPs
- **SSH connection validation**: Parallel connectivity tests
- **Host key management**: Automatic SSH key acceptance
- **Connection retry logic**: Built-in Ansible retry mechanisms

**Implementation Details:**
```bash
# Ansible inventory file (inventory.ini) - Two Clusters
# Cluster 1 (6 servers: 1 network + 5 ML)
[cluster1_network_node]
cluster1-network ansible_host=192.168.1.100

[cluster1_ml_nodes]
cluster1-ml-01 ansible_host=192.168.1.101  # Qwen3-235B
cluster1-ml-02 ansible_host=192.168.1.102  # Qwen3-235B
cluster1-ml-03 ansible_host=192.168.1.103  # Qwen3-235B
cluster1-ml-04 ansible_host=192.168.1.104  # Qwen3-235B
cluster1-ml-05 ansible_host=192.168.1.105  # Qwen3-32B

[cluster1:children]
cluster1_network_node
cluster1_ml_nodes

# Cluster 2 (6 servers: 1 network + 5 ML)
[cluster2_network_node]
cluster2-network ansible_host=192.168.1.106

[cluster2_ml_nodes]
cluster2-ml-01 ansible_host=192.168.1.107  # Qwen3-235B
cluster2-ml-02 ansible_host=192.168.1.108  # Qwen3-235B
cluster2-ml-03 ansible_host=192.168.1.109  # Qwen3-235B
cluster2-ml-04 ansible_host=192.168.1.110  # Qwen3-235B
cluster2-ml-05 ansible_host=192.168.1.111  # Qwen3-32B

[cluster2:children]
cluster2_network_node
cluster2_ml_nodes

# All clusters for common tasks
[gonka_clusters:children]
cluster1
cluster2

# Ansible configuration (ansible.cfg)
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
private_key_file = ~/.ssh/gonka-deploy-key
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = memory
callback_whitelist = profile_tasks
stdout_callback = yaml

[inventory]
enable_plugins = ini

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
```

#### 1.2 System Validation (facts gathering)
- **Ansible facts**: Automatic system information collection
- **OS validation**: Verify Ubuntu 22.04.5 across all servers
- **Docker checks**: Service status and version validation
- **GPU validation**: NVIDIA GPU count and driver verification
- **Resource checks**: Disk space, memory, CPU core validation
- **Network tests**: Port availability and connectivity verification

**Validation Commands:**
```yaml
# System validation playbook tasks
- name: Gather system facts
  setup:

- name: Validate Ubuntu version
  assert:
    that:
      - ansible_distribution == "Ubuntu"
      - ansible_distribution_version == "22.04"
    fail_msg: "Server must run Ubuntu 22.04.5"

- name: Check Docker service
  service:
    name: docker
    state: started
  register: docker_status

- name: Validate Docker version
  command: docker --version
  register: docker_version
  failed_when: "'28.4.0' not in docker_version.stdout"

- name: Check NVIDIA GPU
  command: nvidia-smi --query-gpu=name,memory.total --format=csv,noheader,nounits
  register: gpu_info
  failed_when: gpu_info.stdout_lines | length < 8

- name: Validate disk space
  stat:
    path: /
  register: root_fs
  failed_when: (root_fs.stat.size_available / 1024 / 1024 / 1024) < 500

- name: Test inter-server connectivity
  command: ping -c 3 {{ item }}
  loop: "{{ groups['gonka_clusters'] }}"
  ignore_errors: true
```

#### 1.3 Firewall Configuration (ufw role)
- **UFW module**: Ansible's ufw module for firewall management
- **Conditional port opening**: Different ports for different server roles
  - **Network node**: Opens ports 5000, 26657, 8000 (blockchain + API services)
- **Admin API (9200)**: NOT exposed publicly - access via SSH tunneling only
  - **ML nodes**: Opens ports 5000, 8080 (inference + management) **BUT ONLY FROM NETWORK NODE IP**
- **Security**: ML node ports are restricted to network node access only for enhanced security
- **External mapping**: Ports 5050/8080 mapped externally to ML node 5000/8080
- **Group-based configuration**: Uses Ansible host groups for role-specific rules
- **Rule validation**: Verify firewall rules are applied correctly
- **Connectivity tests**: Test inter-server communication

#### 1.4 Dependency Installation (package role)
- **Package management**: Ansible apt module for system packages
- **Hugging Face setup**: CLI installation and HF_HOME environment configuration
- **Directory creation**: Deployment directory structure setup
- **Environment configuration**: System variables and paths including HF_HOME

**Package Installation Implementation:**
```yaml
# Install system dependencies
- name: Update package cache
  apt:
    update_cache: yes
  become: true

- name: Install required packages
  apt:
    name:
      - python3-pip
      - git
      - curl
      - wget
      - build-essential
    state: present
  become: true

# Install Hugging Face CLI
- name: Install Hugging Face CLI
  pip:
    name: huggingface_hub
    state: present

# Set up HF_HOME environment variable for all servers
- name: Create Hugging Face cache directory
  file:
    path: /mnt/shared
    state: directory
    mode: '0755'
  become: true

- name: Set HF_HOME environment variable globally
  lineinfile:
    path: /etc/environment
    line: "HF_HOME=/mnt/shared"
  become: true

- name: Set HF_HOME in user profile
  lineinfile:
    path: /home/ubuntu/.bashrc
    line: "export HF_HOME=/mnt/shared"
    create: true

- name: Set HF_HOME in root profile
  lineinfile:
    path: /root/.bashrc
    line: "export HF_HOME=/mnt/shared"
    create: true

# Verify HF_HOME is set correctly
- name: Verify HF_HOME environment variable
  shell: echo "HF_HOME=$HF_HOME"
  register: hf_home_check
  failed_when: "'/mnt/shared' not in hf_home_check.stdout"
```

### Phase 2: Repository & Model Preparation

#### 2.1 Repository Deployment (git role)
- **Git module**: Parallel repository cloning on all servers
- **Version control**: Tag/branch checkout with verification
- **Integrity checks**: Repository checksum validation
- **Directory structure**: Standardized deployment workspace setup

**Repository Details:**
```yaml
# Repository cloning tasks
- name: Clone Gonka deployment repository
  git:
    repo: https://github.com/gonka-network/gonka-deploy.git
    dest: /opt/gonka-deploy
    version: main  # or specific tag like v1.0.0
    depth: 1
  become: true

- name: Set correct permissions
  file:
    path: /opt/gonka-deploy
    owner: ubuntu
    group: ubuntu
    recurse: true
  become: true

- name: Verify repository integrity
  command: git verify-commit HEAD
  args:
    chdir: /opt/gonka-deploy
  ignore_errors: true
```

#### 2.2 Model Weight Management (huggingface role)
- **Async tasks**: Parallel model downloads with Ansible async (HF_HOME configured globally)
- **Hugging Face CLI**: Uses pre-installed CLI with globally configured HF_HOME
- **Progress monitoring**: Download status tracking and failure handling
- **Checksum validation**: Model integrity verification
- **Cache optimization**: Efficient storage utilization with globally set HF_HOME

**Model Download Details:**
```yaml
# Model download variables (download all models for caching)
model_list:
  small:
    - name: "Qwen/Qwen2.5-7B-Instruct"
      size: "~15GB"
      description: "Base Qwen2.5 7B model for basic inference"
    - name: "RedHatAI/Qwen2.5-7B-Instruct-quantized.w8a16"
      size: "~4GB"
      description: "Quantized Qwen2.5 7B model (optimized for efficiency)"
  medium:
    - name: "Qwen/Qwen3-32B-FP8"
      size: "~16GB"
      description: "Qwen3 32B model with FP8 quantization"
    - name: "Qwen/QwQ-32B"
      size: "~20GB"
      description: "QwQ 32B reasoning model"
  large:
    - name: "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8"
      size: "~120GB"
      description: "Large Qwen3 235B model with FP8 quantization"

# Cluster model assignment variables
cluster_models:
  cluster1:
    network_node_model: "Qwen/Qwen3-32B-FP8"  # Network node runs Qwen3-32B
    ml_node_models:
      - model: "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8"
        count: 4  # 4 ML nodes run Qwen3-235B
      - model: "Qwen/Qwen3-32B-FP8"
        count: 1  # 1 ML node runs Qwen3-32B
  cluster2:
    network_node_model: "Qwen/Qwen3-32B-FP8"  # Network node runs Qwen3-32B
    ml_node_models:
      - model: "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8"
        count: 4  # 4 ML nodes run Qwen3-235B
      - model: "Qwen/Qwen3-32B-FP8"
        count: 1  # 1 ML node runs Qwen3-32B

# Model configuration for vLLM arguments
model_configs:
  "Qwen/Qwen3-32B-FP8":
    args: []
  "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8":
    args: ["--max-model-len", "240000", "--tensor-parallel-size", "4"]

# Download tasks (HF_HOME should be available from environment setup)
- name: Download required models
  shell: >
    export HF_HOME=/mnt/shared &&
    huggingface-cli download {{ item.name }}
    --local-dir /mnt/shared/{{ item.name | replace('/', '_') }}
    --local-dir-use-symlinks False
  loop: "{{ model_list.small + model_list.medium + model_list.large }}"
  async: 7200  # 2 hour timeout for large models
  poll: 60     # Check every minute
  register: download_result

- name: Verify model downloads
  stat:
    path: "/mnt/shared/{{ item.name | replace('/', '_') }}/config.json"
  loop: "{{ model_list.small + model_list.medium + model_list.large }}"
  register: model_verification

- name: Report download status
  debug:
    msg: "Model {{ item.item.name }}: {{ 'SUCCESS' if item.stat.exists else 'FAILED' }}"
  loop: "{{ model_verification.results }}"
```

#### 2.3 Docker Environment Preparation (docker role)
- **Docker compose pull**: Pull all images from compose files on ALL servers
- **NVIDIA runtime**: GPU compatibility verification on all servers
- **Compose validation**: Verify configuration files on all servers
- **Image management**: Cleanup and optimization tasks

**Docker Images via Docker Compose:**
```yaml
# Use docker-compose pull to automatically pull all required images
- name: Pull all Docker images from compose files
  command: docker compose -f docker-compose.yml -f docker-compose.mlnode.yml pull
  args:
    chdir: /opt/gonka-deploy
  async: 3600  # 1 hour timeout for large images
  poll: 60     # Check every minute
  register: pull_result

- name: Verify NVIDIA runtime
  command: docker info
  register: docker_info
  failed_when: "'nvidia' not in docker_info.stdout"

- name: Clean up unused Docker images
  command: docker image prune -f
  ignore_errors: true

- name: Show pulled images
  command: docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
  register: image_list

- name: Verify compose files are valid
  command: docker compose -f docker-compose.yml -f docker-compose.mlnode.yml config
  args:
    chdir: /opt/gonka-deploy
  register: compose_validation
```

### Phase 2.1: Post-Repository Validation

#### 2.1.1 Repository Integrity Check (repo-validation role)
- **Configuration files validation**: Verify docker-compose.yml, docker-compose.mlnode.yml present
- **Repository structure check**: Confirm deploy/join directory exists
- **HF cache validation**: Ensure Hugging Face cache directory created
- **Docker images verification**: Check that required images were pulled

**Repository Validation Implementation:**
```yaml
# Check configuration files after repo clone
- name: Validate deployment configuration files are present
  stat:
    path: "{{ gonka_deploy_dir }}/{{ item }}"
  loop:
    - docker-compose.yml
    - docker-compose.mlnode.yml
    - .gitignore
  register: config_files
  failed_when: config_files.results | selectattr('stat.exists', 'equalto', false) | list | length > 0
```

### Phase 3: Key Management & Configuration

**Phase 3 Overview:**
- **3.1 Account Key Generation**: Generate cold wallet locally, extract public key
- **3.2 Configuration Generation**: Create config.env on all servers using account public key
- **Prerequisite**: Phase 3.1 must complete before Phase 3.2 (public key needed for config)
- **Scope**: 3.1 runs locally, 3.2 runs on all servers in cluster

#### 3.1 Account Key Generation (keygen-local role)
- **Local execution**: Generate cold wallet (account key) on deployment machine
- **Public key extraction**: Get gonka1 address for configuration
- **Secure storage**: Backup mnemonic phrase securely
- **Required before config**: Public key needed for config.env generation
- **One-time operation**: Account key generated once for the cluster

**Account Key Generation Implementation:**
```yaml
# Generate account key locally (cold wallet)
- name: Generate account key locally
  command: >
    ./inferenced keys add gonka-account-key
    --keyring-backend file
    --home ./gonka-keys
  register: account_key_generation
  failed_when: "'already exists' in account_key_generation.stderr and 'created' not in account_key_generation.stdout"
  when: inventory_hostname == groups['network_node'][0]  # Run only once on network node server

# Extract public key and address
- name: Extract account public key
  shell: >
    printf '%s\n' "$KEYRING_PASSWORD" |
    ./inferenced keys show gonka-account-key
    --keyring-backend file
    --home ./gonka-keys
  register: public_key_info
  when: inventory_hostname == groups['network_node'][0]

# Store pubkey key value for config generation
- name: Set fact for account pubkey
  set_fact:
    account_pubkey: "{{ public_key_info.stdout | regex_findall('\"key\":\"([^\"]+)\"') | first }}"
  when: inventory_hostname == groups['network_node'][0]
```

#### 3.2 Configuration File Generation (config role)
- **Template module**: Jinja2 templates for `config.env` generation
- **All servers**: Generate config.env on both network node and ML nodes
- **Host-specific variables**: Dynamic configuration based on server roles
- **Public URL configuration**: Automatic endpoint URL generation
- **Account key integration**: Uses public key from Phase 3.1
- **Required before Docker**: Config.env needed for container execution

**Config.env Generation (All Servers):**
```yaml
# Create config.env for all servers (network node + ML nodes)
- name: Create config.env template for all servers
  template:
    src: config.env.j2
    dest: /opt/gonka-deploy/config.env
  vars:
    network_node_ip: "{{ groups[cluster_name + '_network_node'][0] }}"
    account_pubkey: "{{ hostvars[groups[cluster_name + '_network_node'][0]][cluster_name + '_account_pubkey'] | default('PLACEHOLDER_PUBKEY') }}"
    cluster_name: "{{ group_names | select('match', 'cluster[0-9]+') | first }}"
  when: "'gonka_clusters' in group_names"

# Create node-config-generated.json for network nodes (Qwen3-32B configuration)
- name: Create node-config-generated.json for network nodes
  template:
    src: node-config.json.j2
    dest: /opt/gonka-deploy/node-config-generated.json
  vars:
    cluster_name: "{{ group_names | select('match', 'cluster[0-9]+') | first }}"
    network_node_model: "{{ cluster_models[cluster_name].network_node_model }}"
  when: "'network_node' in group_names"
```

**Node Configuration Templates:**

**node-config-generated.json.j2 Template:**
```json
[
  {
    "id": "network-node",
    "host": "inference",
    "inference_port": 5000,
    "poc_port": 8080,
    "max_concurrent": 500,
    "models": {
      "{{ network_node_model }}": {
        "args": {{ model_configs[network_node_model].args | to_json }}
      }
    }
  }
]
```

**Configuration File Templates (Based on Gonka Documentation):**

**config.env.j2 Template (Used on All Servers):**
```bash
# Key Configuration (from Gonka docs)
export KEY_NAME={{ key_name | default('gonka-ml-key') }}
export KEYRING_PASSWORD={{ keyring_password }}

# API and Port Configuration (from Gonka docs)
export API_PORT=8000
export PORT=8080
export INFERENCE_PORT=5050
export KEYRING_BACKEND=file

# Network Configuration (from Gonka docs)
export PUBLIC_URL=http://{{ network_node_ip }}:8000
export P2P_EXTERNAL_ADDRESS=tcp://{{ network_node_ip }}:5000

# Account PubKey Configuration (from Gonka docs - base64 key value)
export ACCOUNT_PUBKEY={{ account_pubkey | default('PLACEHOLDER_PUBKEY') }}

# Model Cache Configuration (from Gonka docs)
export HF_HOME=/mnt/shared

# Node Configuration (from Gonka docs - only for network node)
{% if 'network_node' in group_names %}
export NODE_CONFIG=./node-config-generated.json
{% endif %}

# Seed Node URLs (from Gonka docs - keep as is)
export SEED_API_URL=http://node2.gonka.ai:8000
export SEED_NODE_RPC_URL=http://node2.gonka.ai:26657
export SEED_NODE_P2P_URL=tcp://node2.gonka.ai:5000

# DAPI Configuration (from Gonka docs - keep as is)
export DAPI_API__POC_CALLBACK_URL=http://api:9100
export DAPI_CHAIN_NODE__URL=http://node:26657
export DAPI_CHAIN_NODE__P2P_URL=http://node:26656

# RPC Server URLs (from Gonka docs - keep as is)
export RPC_SERVER_URL_1=http://node1.gonka.ai:26657
export RPC_SERVER_URL_2=http://node2.gonka.ai:26657

# Additional Configuration for Our Deployment
export COMPOSE_PROJECT_NAME=gonka-network
export TM_HOME=/opt/gonka-deploy/.tendermint
```

**How config.env.j2 Works:**
1. **Template Variables**: `{{ network_node_ip }}`, `{{ account_pubkey }}` (base64 key value), `{{ keyring_password }}` get replaced by Ansible
2. **Matches Gonka Docs**: Follows exact format from [Gonka documentation](https://gonka.ai/host/quickstart/#server-edit-your-network-node-configuration)
3. **Dynamic Configuration**: IP address and public key are dynamically inserted
4. **Ansible Template**: Uses Jinja2 templating engine for variable substitution

**⚠️ CRITICAL: Config.env Must Be Sourced Before Docker Container Operations**
- **Required by Gonka**: Docker commands that RUN containers must source config.env first
- **Environment Variables**: `$KEYRING_PASSWORD`, `$ACCOUNT_PUBKEY`, `$PUBLIC_URL`, etc. must be loaded
- **Command Format**: `source /opt/gonka-deploy/config.env && docker compose up/run/exec ...`
- **NOT Required For**: `docker compose pull`, `docker compose config` (image operations)
- **Failure Without Sourcing**: Running containers cannot access required environment variables

#### 3.2 Initial Services Launch (network-init role)
- **Docker compose up**: Start tmkms and node services (blockchain connectivity)
- **Blockchain synchronization**: Wait for Tendermint RPC to be available
- **Service health verification**: Ensure blockchain services are running
- **Prerequisite for key generation**: Services must be running before key creation
- **Config.env required**: Environment variables needed for service startup

**Initial Services Startup Commands:**
```yaml
# Start initial services (tmkms and node only) - needed for key generation
- name: Start initial services (tmkms and node) - SOURCE config.env FIRST
  shell: source /opt/gonka-deploy/config.env && docker compose up tmkms node -d --no-deps
  args:
    chdir: /opt/gonka-deploy
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Wait for blockchain sync before key generation
- name: Wait for Tendermint RPC to be available
  uri:
    url: "http://localhost:26657/status"
    method: GET
  register: rpc_check
  until: rpc_check.status == 200
  retries: 30
  delay: 10
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"
```

#### 3.3 Cryptographic Key Setup (keygen role)
- **Docker compose run**: Generate ML Operational Key inside API container (per Gonka docs)
- **File keyring backend**: Required for programmatic access
- **Secure key storage**: Keys stored in persistent Docker volume
- **Mnemonic backup**: Critical for account recovery
- **Services dependency**: Requires running tmkms + node services

**Key Generation Commands (Following Gonka Documentation):**
```yaml
# Generate ML Operational Key inside API container (per Gonka docs)
- name: Create ML Operational Key inside API container - SOURCE config.env FIRST
  shell: >
    source /opt/gonka-deploy/config.env &&
    docker compose run --rm --no-deps -T api /bin/sh -c '
    printf "%s\n%s\n" "$KEYRING_PASSWORD" "$KEYRING_PASSWORD" |
    inferenced keys add "$KEY_NAME" --keyring-backend file
    '
  args:
    chdir: /opt/gonka-deploy
  register: ml_key_creation
  delegate_to: "{{ groups['network_node'][0] }}"
  no_log: true

# Secure backup of mnemonic
- name: Backup ML key mnemonic securely
  copy:
    content: "{{ ml_key_creation.stdout }}"
    dest: "/opt/gonka-deploy/ml-key-backup-{{ ansible_date_time.iso8601 }}.txt"
    mode: '0600'
  delegate_to: "{{ groups['network_node'][0] }}"
  when: ml_key_creation.stdout is defined
```

**Docker Compose Files:**
- **docker-compose.yml**: Main compose file (cloned from repository)
- **docker-compose.mlnode.yml**: ML node specific configuration (cloned from repository)
- **No custom files needed** - using existing repository files directly

#### 3.4 Host Registration with Network (registration role) - **EXECUTES: inferenced register-new-participant**
- **Docker compose run**: Execute registration command inside API container
- **Inferenced CLI**: Use the official Gonka registration command
- **Environment Variables**: Uses `$DAPI_API__PUBLIC_URL`, `$ACCOUNT_PUBKEY`, `$DAPI_CHAIN_NODE__SEED_API_URL`
- **Blockchain Transaction**: Submits registration to Gonka network
- **Executed After**: ML Operational Key creation (needs API container running)

**Registration Command (Following Gonka Documentation):**
```yaml
# Execute registration inside API container (per Gonka docs)
- name: Register host with Gonka Network - SOURCE config.env FIRST
  shell: >
    source /opt/gonka-deploy/config.env &&
    docker compose run --rm --no-deps -T api /bin/sh -c '
    printf "%s\n" "$KEYRING_PASSWORD" |
    inferenced register-new-participant \
      $DAPI_API__PUBLIC_URL \
      $ACCOUNT_PUBKEY \
      --node-address $DAPI_CHAIN_NODE__SEED_API_URL
    '
  args:
    chdir: /opt/gonka-deploy
  register: host_registration
  delegate_to: "{{ groups['network_node'][0] }}"
  failed_when: "'error' in host_registration.stderr"

# Extract ML operational address from registration output
# Typical output format from register-new-participant:
# "...Found participant with pubkey: ... (balance: 0)
# Participant is now available at http://...:8000/v1/participants/gonka1xxxxx..."
# We extract the gonka1 address which is the ML operational key address
- name: Extract ML operational address from registration
  set_fact:
    ml_operational_address: "{{ host_registration.stdout | regex_findall('gonka1[a-z0-9]{38}') | first }}"
  when: host_registration.stdout is defined
  delegate_to: "{{ groups['network_node'][0] }}"

# Alternative extraction methods if regex doesn't work:
# - Look for "address:" or "participant:" in the output
# - Parse JSON response if the command returns structured data
# - Manual extraction from debug output if needed

# Debug: Show extracted address
- name: Display extracted ML operational address
  debug:
    msg: "ML Operational Address: {{ ml_operational_address }}"
  when: ml_operational_address is defined
  delegate_to: "{{ groups['network_node'][0] }}"

# Wait for registration confirmation
- name: Wait for blockchain registration confirmation
  uri:
    url: "{{ public_url }}/v1/participants/{{ account_address }}"
    method: GET
  register: registration_check
  until: registration_check.status == 200
  retries: 30
  delay: 10
  delegate_to: "{{ groups['network_node'][0] }}"
```

#### 3.5 Permission Granting (permissions role) - **MOVED: Local Machine Execution**
- **Local execution**: Permission granting happens from local machine with cluster-specific Account Key
- **Cluster-specific keys**: Uses `gonka-account-key-cluster1` or `gonka-account-key-cluster2`
- **Inferenced CLI**: Use local CLI installation for transaction submission
- **Blockchain transaction**: Submit permission grant to the network
- **Transaction confirmation**: Wait for inclusion in block
- **Executed after registration**: Requires host to be registered first

**Permission Granting Commands:**
```yaml
# First, fetch the ML operational address from the network node
- name: Get ML operational address from network node
  set_fact:
    ml_operational_address: "{{ hostvars[groups['network_node'][0]]['ml_operational_address'] }}"
  when: hostvars[groups['network_node'][0]]['ml_operational_address'] is defined

# Debug: Verify we have the address
- name: Display ML operational address for permission granting
  debug:
    msg: "Granting permissions to ML address: {{ ml_operational_address }}"

# This step runs on the local/control machine
# UNCERTAINTY: Gonka documentation shows 'gonka-account-key' as first parameter,
# but this might need to be the actual account address instead of key name
# Alternative interpretation: grant-ml-ops-permissions <account-address> <ml-operational-address>

# Current implementation follows documentation literally:
# Expected output from inferenced command:
# Transaction sent with hash: FB9BBBB5F8C155D0732B290C443A0D06BC114CDF43E8EE8FB329D646C608062E
# Waiting for transaction to be included in a block...
# Transaction confirmed successfully!
# Block height: 174
- name: Grant ML operation permissions using cluster-specific account key
  shell: >
    printf '%s\n' "$KEYRING_PASSWORD" |
    ./inferenced tx inference grant-ml-ops-permissions
    {{ cluster_name }}-account-key
    {{ ml_operational_address }}
    --from {{ cluster_name }}-account-key
    --keyring-backend file
    --home ./gonka-keys-{{ cluster_name }}
    --gas 2000000
    --node {{ seed_node_rpc_url }}
  vars:
    cluster_name: "{{ group_names | select('match', 'cluster[0-9]+') | first }}"
  args:
    chdir: .  # Current directory where Ansible script is run
  register: permission_grant
  failed_when: "'error' in permission_grant.stderr or 'Transaction confirmed successfully' not in permission_grant.stdout"
  when: ml_operational_address is defined

# ALTERNATIVE if first parameter should be account address:
# {{ cluster_name }}-account-key → {{ cluster_name }}_account_address
# (uncomment and use if the above fails)
#
# NOTE: If account_address is needed, it can be derived from account_pubkey
# using: ./inferenced keys show gonka-account-key --keyring-backend file
# This would return the account address associated with the key

# VERIFICATION: Since inferenced provides confirmation output, just log the result
- name: Log permission transaction confirmation (inferenced already waited)
  debug:
    msg: |
      Permission transaction completed successfully!
      Transaction hash: {{ permission_grant.stdout | regex_findall('hash:\\s*([a-f0-9]+)') | first }}
      Block height: {{ permission_grant.stdout | regex_findall('Block height:\\s*(\\d+)') | first }}
  when: "'Transaction confirmed successfully' in permission_grant.stdout"

# Optional: Additional RPC verification (uncomment if needed)
# - name: Additional RPC verification of transaction
#   uri:
#     url: "http://{{ network_node_ip }}:26657/tx?hash={{ permission_grant.stdout | regex_findall('hash:\\s*([a-f0-9]+)') | first }}"
#     method: GET
#   register: tx_confirmation
#   failed_when: tx_confirmation.json.result.tx_result.code != 0
```

### Phase 4: Initial Services Launch (Network Node Only)

#### 4.1 Full Node Activation (network-full role) - **FINAL: Complete Network Node**
- **Docker_compose module**: Launch all services including persistent API
- **Complete stack**: Start API, database, Redis, and all supporting services
- **Service orchestration**: Proper startup order and health checks
- **Network connectivity**: Verify external accessibility
- **Executed after permissions**: All setup complete before full activation

**Full Node Activation Implementation:**
```yaml
# Launch the complete Docker Compose stack with all services
- name: Launch complete Docker Compose stack - SOURCE config.env FIRST
  shell: source /opt/gonka-deploy/config.env && docker compose -f docker-compose.yml -f docker-compose.mlnode.yml up -d
  args:
    chdir: /opt/gonka-deploy
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Wait for all services to be healthy
- name: Wait for API service to be ready
  uri:
    url: "http://{{ network_node_ip }}:8000/health"
    method: GET
  register: api_health
  until: api_health.status == 200
  retries: 30
  delay: 10
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Wait for services to be ready (based on your actual compose files)

# Verify containers are running
- name: Check container status
  command: docker compose -f docker-compose.yml -f docker-compose.mlnode.yml ps
  args:
    chdir: /opt/gonka-deploy
  register: container_status
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Test API endpoints
- name: Test API endpoints availability
  uri:
    url: "{{ item }}"
    method: GET
  loop:
    - "http://{{ network_node_ip }}:8000/v1/status"
    - "http://{{ network_node_ip }}:8000/v1/participants"
  register: api_tests
  failed_when: item.status != 200
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Log successful activation
- name: Log successful full node activation
  debug:
    msg: |
      🎉 Full node activation completed successfully!
      🌐 API available at: http://{{ network_node_ip }}:8000
      🔗 Blockchain RPC at: http://{{ network_node_ip }}:26657
      📊 Container status: Check above output for service details
  when: "'network_node' in group_names"
```

### Phase 5: ML Nodes Deployment

#### 5.1 ML Node Launch Sequence (ml-deploy role)
- **Docker_compose module**: Parallel ML service deployment
- **Throttle**: Controlled deployment rate to prevent overload
- **Health checks**: Service readiness verification using documented endpoints
- **GPU monitoring**: Memory utilization tracking
- **Async tasks**: Non-blocking service startup monitoring
- **NVIDIA-SMI**: GPU initialization verification

**ML Node Deployment Implementation:**
```yaml
# Deploy ML services with controlled parallelism
- name: Deploy ML services on all nodes (throttled to prevent overload)
  shell: export HF_HOME=/mnt/shared && docker compose -f docker-compose.mlnode.yml up -d
  args:
    chdir: /opt/gonka-deploy
  throttle: 2  # Only deploy 2 ML nodes at a time
  async: 600   # 10 minute timeout
  poll: 30     # Check every 30 seconds
  register: ml_deployment
  when: "'ml_nodes' in group_names"

# Wait for ML services to be ready
- name: Wait for ML inference service to be ready
  uri:
    url: "http://{{ ansible_host }}:5000/health"
    method: GET
  register: ml_health
  until: ml_health.status == 200
  retries: 30
  delay: 10
  when: "'ml_nodes' in group_names"

# Verify GPU access in containers
- name: Verify GPU access in ML containers
  command: docker exec $(docker ps -q -f name=gonka-ml) nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits
  register: gpu_check
  failed_when: gpu_check.stdout | length == 0
  when: "'ml_nodes' in group_names"

# Test basic ML service health (using documented health endpoint)
- name: Test ML service basic health
  uri:
    url: "http://{{ ansible_host }}:5000/health"
    method: GET
  register: ml_service_health
  failed_when: ml_service_health.status != 200
  when: "'ml_nodes' in group_names"

# Log successful ML deployment
- name: Log successful ML node deployment
  debug:
    msg: |
      🚀 ML Node {{ inventory_hostname }} deployed successfully!
      🎯 Inference API: http://{{ ansible_host }}:5000
      🔍 Service health: {{ 'OK' if ml_service_health.status == 200 else 'CHECKING...' }}
      🎮 GPU memory: {{ gpu_check.stdout | default('checking...') }} MB used
  when: "'ml_nodes' in group_names"
```

#### 5.2 Node Registration with Network (ml-register role)
- **URI module**: Admin API registration calls from network node (port 9200)
- **Loop**: Sequential registration of all ML nodes from network node
- **Delegate_to**: Execute registration tasks on network node
- **Admin API**: Uses documented `/admin/v1/nodes` endpoint with proper parameters
- **Model Configuration**: Registers supported models with vLLM arguments

**ML Node Registration Implementation:**
```yaml
# Register all ML nodes with the network node using Admin API (per documentation)
- name: Register ML nodes with network node
  uri:
    url: "http://{{ groups['network_node'][0] }}:9200/admin/v1/nodes"
    method: POST
    body_format: json
    body:
      id: "ml-node-{{ item | regex_replace('\\.', '-') }}"
      host: "{{ item }}"
      inference_port: 5000
      poc_port: 8000
      max_concurrent: 500
      models: "{{ {item.model: model_configs[item.model]} if item.model in model_configs else {} }}"
  loop: "{{ groups['ml_nodes'] }}"
  register: registration_result
  failed_when: item.status != 200
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Verify nodes are registered (check via admin API)
- name: Verify ML nodes are registered
  uri:
    url: "http://{{ groups['network_node'][0] }}:9200/admin/v1/nodes"
    method: GET
  register: registered_nodes
  failed_when: "registered_nodes.json | length < groups['ml_nodes'] | length"
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Verify all nodes appear in active participants list
- name: Verify ML nodes in active participants list
  uri:
    url: "http://{{ groups['network_node'][0] }}:8000/v1/participants/active"
    method: GET
  register: participants_list
  failed_when: "groups['ml_nodes'] | difference(participants_list.json.nodes) | length > 0"
  when: "'network_node' in group_names"
  delegate_to: "{{ groups['network_node'][0] }}"

# Log successful registration
- name: Log successful ML node registrations
  debug:
    msg: |
      ✅ All ML Nodes registered successfully from Network Node!
      📋 Registered nodes: {{ groups['ml_nodes'] | join(', ') }}
      📊 Total registered nodes: {{ registered_nodes.json | length }}
      🔗 Network Node Admin API: {{ groups['network_node'][0] }}:9200
  when: "'network_node' in group_names"
```

**ML Node Configuration:**
- No configuration files needed - controlled entirely by network node after registration
- Compose files provide all necessary container configuration
- HF_HOME environment variable set globally for model caching

## Deployment Parameters

### Ansible Input Parameters
- **Inventory file**: List of all 8 public IP addresses with host groups
- **SSH private key**: Path to SSH private key for authentication
- **Ansible variables**: Network node designation and configuration parameters
- **Extra variables**: Runtime parameters for deployment customization

### Ansible Inventory Structure (Two Clusters)
```ini
# Cluster 1 (6 servers: 1 network + 5 ML)
[cluster1_network_node]
cluster1-network ansible_host=192.168.1.100

[cluster1_ml_nodes]
cluster1-ml-01 ansible_host=192.168.1.101  # Qwen3-235B
cluster1-ml-02 ansible_host=192.168.1.102  # Qwen3-235B
cluster1-ml-03 ansible_host=192.168.1.103  # Qwen3-235B
cluster1-ml-04 ansible_host=192.168.1.104  # Qwen3-235B
cluster1-ml-05 ansible_host=192.168.1.105  # Qwen3-32B

[cluster1:children]
cluster1_network_node
cluster1_ml_nodes

# Cluster 2 (6 servers: 1 network + 5 ML)
[cluster2_network_node]
cluster2-network ansible_host=192.168.1.106

[cluster2_ml_nodes]
cluster2-ml-01 ansible_host=192.168.1.107  # Qwen3-235B
cluster2-ml-02 ansible_host=192.168.1.108  # Qwen3-235B
cluster2-ml-03 ansible_host=192.168.1.109  # Qwen3-235B
cluster2-ml-04 ansible_host=192.168.1.110  # Qwen3-235B
cluster2-ml-05 ansible_host=192.168.1.111  # Qwen3-32B

[cluster2:children]
cluster2_network_node
cluster2_ml_nodes

# All clusters for common tasks
[gonka_clusters:children]
cluster1
cluster2
```

**Host Group Usage:**
- **`cluster1_network_node`**: Network node runs Qwen/Qwen3-32B-FP8 (empty args)
- **`cluster1_ml_nodes`**: 4 servers run Qwen3-235B, 1 server runs Qwen3-32B
- **`cluster2_network_node`**: Network node runs Qwen/Qwen3-32B-FP8 (empty args)
- **`cluster2_ml_nodes`**: 4 servers run Qwen3-235B, 1 server runs Qwen3-32B
- **`gonka_clusters`**: All 12 servers across both clusters for common tasks (model downloads, etc.)

**Model Distribution:**
- **Qwen/Qwen3-32B-FP8**: Empty args `[]`
- **Qwen/Qwen3-235B-A22B-Instruct-2507-FP8**: Args `["--max-model-len", "240000", "--tensor-parallel-size", "4"]`

**Conditional Configurations:**
- **Network nodes**: Get `node-config-generated.json` with Qwen3-32B configuration
- **ML nodes**: Get assigned models based on cluster configuration
- **Resource allocation**: Qwen3-235B nodes need more GPU resources

### Configurable Variables
- Server IP addresses and hostnames
- Model configurations and weights
- GPU allocation per node
- Network endpoints and ports
- Ansible fork limits and timeouts

### Ansible Variables
- `network_node_ip`: Auto-detected from inventory
- `hf_home`: Model cache directory (/opt/huggingface/cache)
- `docker_compose_files`: Compose file selection
- `deployment_mode`: full/ml-only
- `ssh_key_path`: SSH private key location
- `keyring_password`: Auto-generated secure password
- `public_key_info`: Extracted from key generation
- `model_list`: Complete list of required models with sizes
- `docker_images`: List of required Docker images

### Main Ansible Playbook Structure
```yaml
# deploy.yml - Main deployment playbook
---
- name: Phase 1 - Environment Setup & Pre-Validation
  hosts: gonka_clusters
  gather_facts: true
  roles:
    - firewall
    - pre-validation

- name: Phase 1.4 - Dependency Installation
  hosts: gonka_clusters
  roles:
    - dependencies

- name: Phase 2 - Repository & Model Preparation
  hosts: gonka_clusters
  roles:
    - git
    - huggingface
    - docker

- name: Phase 2.1 - Post-Repository Validation
  hosts: gonka_clusters
  roles:
    - repo-validation

- name: Phase 3 - Key Management (Network Node Only)
  hosts: network_node
  roles:
    - keygen
    - config

- name: Phase 4 - Network Node Deployment
  hosts: network_node
  roles:
    - network-init
    - registration
    - permissions
    - network-full

- name: Phase 5 - ML Nodes Deployment
  hosts: ml_nodes
  roles:
    - ml-config
    - ml-deploy
    - ml-register

- name: Phase 6 - Verification & Health Checks
  hosts: gonka_clusters
  roles:
    - validation
    - cluster-check
    - performance

- name: Phase 7 - Monitoring & Maintenance
  hosts: gonka_clusters
  roles:
    - logging
    - monitoring
    - backup
```

### Execution Commands
```bash
# 1. Test connectivity
ansible all -m ping -i inventory.ini

# 2. Run complete deployment
ansible-playbook -i inventory.ini deploy.yml

# 3. Run specific phase
ansible-playbook -i inventory.ini deploy.yml --tags phase2

# 4. Run on specific host group
ansible-playbook -i inventory.ini deploy.yml --limit ml_nodes

# 5. Check deployment status
ansible-playbook -i inventory.ini validate.yml

# 6. Emergency cleanup
ansible-playbook -i inventory.ini cleanup.yml
```

### Directory Structure
```
gonka-cluster-deployment/
├── ansible.cfg
├── inventory.ini
├── deploy.yml
├── validate.yml
├── cleanup.yml
├── roles/
│   ├── firewall/
│   │   ├── tasks/main.yml
│   │   └── vars/main.yml
│   ├── validate/
│   ├── dependencies/
│   ├── git/
│   ├── huggingface/
│   ├── docker/
│   ├── keygen/
│   ├── config/
│   ├── network-init/
│   ├── registration/
│   ├── permissions/
│   ├── network-full/
│   ├── ml-config/
│   ├── ml-deploy/
│   ├── ml-register/
│   ├── validation/
│   ├── cluster-check/
│   ├── performance/
│   ├── logging/
│   ├── monitoring/
│   └── backup/
├── vars/
│   └── main.yml
└── templates/
    └── config.env.j2
```

## Success Criteria

### Deployment Success Indicators
- All 8 servers have services running
- Network node registered and synchronized
- All 7 ML nodes registered with network node
- GPU utilization within expected ranges
- API endpoints responding correctly

### Performance Benchmarks
- Model loading time < 10 minutes
- Inference latency < 5 seconds
- GPU memory utilization > 80%
- Network synchronization < 5 minutes

### Verification Commands
```bash
# Check all containers are running
ansible gonka_clusters -m command -a "docker ps --format table"

# Verify network node status
curl http://{{ network_node_ip }}:26657/status

# Check API endpoints
curl http://{{ network_node_ip }}:8000/v1/participants

# Validate GPU utilization
ansible gonka_clusters -m command -a "nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits"

# Test model availability
ansible ml_nodes -m command -a "ls -la {{ hf_home }}"
```

## Troubleshooting Guide

### Common Issues
- Docker daemon connectivity problems
- GPU driver compatibility issues
- Model download failures
- Network connectivity issues
- Blockchain synchronization delays

### Diagnostic Commands
- Container status: `docker ps`
- GPU status: `nvidia-smi`
- Network connectivity: `curl` tests
- Blockchain status: Tendermint RPC queries

### Phase-Specific Troubleshooting

**Phase 1: Connection Issues**
```bash
# Test SSH connectivity
ansible all -m ping -i inventory.ini

# Debug SSH connection
ansible all -m command -a "whoami" -i inventory.ini -vvv

# Check firewall status
ansible all -m command -a "ufw status" -i inventory.ini
```

**Phase 2: Download Issues**
```bash
# Check disk space
ansible gonka_clusters -m command -a "df -h /opt"

# Monitor download progress
ansible gonka_clusters -m command -a "du -sh {{ hf_home }}"

# Check network connectivity
ansible gonka_clusters -m command -a "curl -I https://huggingface.co"
```

**Phase 3: Key Generation Issues**
```bash
# Check Docker service on network node
ansible network_node -m command -a "docker ps"

# Verify keyring directory
ansible network_node -m command -a "ls -la /opt/gonka-deploy/.inference"

# Check key generation logs
ansible network_node -m command -a "docker logs $(docker ps -q -f name=gonka-api)"
```

**Phase 4: Network Node Issues**
```bash
# Check Tendermint status
curl http://{{ network_node_ip }}:26657/status

# View network node logs
ansible network_node -m command -a "docker-compose logs node"

# Check blockchain sync
ansible network_node -m shell -a "source /opt/gonka-deploy/config.env && docker-compose exec node tendermint show-validator"
```

**Phase 5: ML Node Issues**
```bash
# Check ML node containers
ansible ml_nodes -m command -a "docker ps"

# Verify GPU access
ansible ml_nodes -m command -a "docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi"

# Check model loading
ansible ml_nodes -m command -a "ls -la {{ hf_home }}/*/config.json"
```

### Recovery Procedures

**Failed Model Downloads:**
```bash
# Resume interrupted downloads
ansible-playbook -i inventory.ini deploy.yml --tags huggingface --start-at-task="Download required models"

# Clean and retry
ansible gonka_clusters -m command -a "rm -rf {{ hf_home }}/*"
ansible-playbook -i inventory.ini deploy.yml --tags huggingface
```

**Key Generation Recovery:**
```bash
# Use existing keys if available
ansible-playbook -i inventory.ini deploy.yml --tags keygen --extra-vars "skip_keygen=true"

# Regenerate keys
ansible network_node -m command -a "rm -rf /opt/gonka-deploy/.inference"
ansible-playbook -i inventory.ini deploy.yml --tags keygen
```

**Container Restart:**
```bash
# Restart all services
ansible gonka_clusters -m command -a "docker-compose restart"

# Restart specific service
ansible network_node -m command -a "docker-compose restart api"
```

## Next Steps

After reviewing this comprehensive Ansible deployment plan:

1. **Feedback and Modifications**: Provide any required changes to the implementation
2. **IP List Specification**: Provide the list of 8 public IP addresses for inventory generation
3. **SSH Key Path**: Confirm the SSH private key path for Ansible connections
4. **Network Node Selection**: Specify which IP should be the network node (first IP in list)
5. **Ansible Setup**: Install Ansible and required collections on deployment machine
6. **Project Creation**: Generate the complete Ansible project structure with all playbooks and roles
7. **Testing**: Test deployment on staging environment if available
8. **Production Deployment**: Execute full cluster deployment using the provided commands

### Pre-Deployment Checklist
- [ ] All 8 servers accessible via SSH with key authentication
- [ ] Ansible 2.15+ installed on deployment machine
- [ ] Required Ansible collections installed (`ansible-galaxy collection install community.docker ansible.posix`)
- [ ] SSH private key available and permissions set (600)
- [ ] All servers have at least 1TB free disk space
- [ ] Network connectivity verified between all servers
- [ ] Firewall ports (22, 80, 443) open on all servers
- [ ] GPU drivers and Docker NVIDIA runtime verified

### Post-Deployment Checklist
- [ ] All containers running across 8 servers
- [ ] Network node synchronized with blockchain
- [ ] All ML nodes registered with network node
- [ ] API endpoints responding correctly
- [ ] GPU utilization within expected ranges
- [ ] Model files present on all servers
- [ ] Backup files created and secured

### Ansible Project Structure
```
gonka-cluster-deployment/
├── ansible.cfg
├── inventory.ini
├── playbooks/
│   ├── deploy.yml
│   ├── validate.yml
│   └── cleanup.yml
├── roles/
│   ├── firewall/
│   ├── docker/
│   ├── gonka-network/
│   ├── gonka-ml/
│   └── monitoring/
└── vars/
    └── main.yml
```

---

**Document Version**: 2.0
**Last Updated**: September 16, 2025
**Review Status**: Complete Self-Sufficient Implementation - Ready for Deployment
