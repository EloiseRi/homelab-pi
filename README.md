# Kubernetes on Raspberry Pi 4 with Ansible

Automate a lightweight Kubernetes cluster on 3× Raspberry Pi 4 (Ubuntu Server arm64) using Ansible.

---

## 📦 Required Hardware

- Raspberry Pi devices (x3 at this moment)
- microSD cards (16 GB or larger recommended, might switch to SSD in near future)
- Power supply (PoE+ HAT)
- Network connection
- Switch with PoE+ support
- PC to prepare cards and manage the cluster

---

## 1️⃣ Preparing SD Cards with Raspberry Pi Imager

1. Download and install [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Launch Raspberry Pi Imager, select **Ubuntu Server 20.04 LTS 64-bit**.
3. Select the appropriate microSD card.
4. Before clicking “Write,” press `Ctrl+Shift+X` (`Cmd+Shift+X` on Mac) to open advanced options.
5. In advanced options, configure:
    - **Enable SSH** (important!)
    - Set a custom **username/password** (default is `ubuntu` / `ubuntu`).
    - Set the hostname (e.g., `control-plane`).
    - Set locale, timezone, and keyboard layout.
6. Confirm and write the image.
7. Repeat for each card, changing the hostname for each Pi (e.g., `worker-01`, `worker-02`).

---

## 2️⃣ First Boot and Connecting

1. Insert each microSD into a Raspberry Pi, power on and connect to the network.
2. On a Mac or Linux system, you can usually connect via SSH using the Pi’s hostname (e.g., `ssh username@hostname.local`). However, Windows often has trouble resolving `.local` hostnames via mDNS, so the Pi’s hostname may not work out of the box.
    
    If the hostname doesn’t resolve, find the Pi’s IP address on your network (for example, via your router’s device list), and connect using the IP address instead: `ssh username@<ip_address>` 
    

<aside>
💡

### Adding Additional SSH Access from Other Machines

If you want to connect to your Raspberry Pi cluster from other client machines (e.g., another PC), you need to add their SSH public keys to each Pi’s authorized keys.

On each new client machine:

1. Generate an SSH key pair if you don't have one already: `ssh-keygen -t rsa -b 4096` 
2. Copy the public key to all Raspberry Pis:
    
    ```bash
    ssh-copy-id ubuntu@control-plane.local
    ssh-copy-id ubuntu@worker-01.local
    ssh-copy-id ubuntu@worker-02.local
    ```
    
</aside>

---

## 3️⃣ System Update and Upgrade

Once connected to each Raspberry Pi, it is recommended to update and upgrade the system packages to ensure security and stability:

```bash
sudo apt update && sudo apt upgrade -y

```

---

## 📁 Repository Structure

    .
    ├── inventory.yml
    ├── main.yml
    ├── 01_prereqs.yaml
    ├── 02_init_controlplane.yaml
    └── 03_join_workers.yaml

- **inventory.yml**  
  Inventory (host aliases)  
- **main.yml**  
  Root playbook importing the three phases  
- **01_prereqs.yaml**  
  Prepare OS & containerd on all nodes  
- **02_init_controlplane.yaml**  
  Initialize the control-plane and deploy Calico via Ansible `k8s` module
- **03_join_workers.yaml**  
  Join workers idempotently and validate node readiness  

---

## ⚙️ Prerequisites

- **Control machine** (MacBook, Linux, etc.)  
  - Python 3 & Ansible (or `ansible-core`)  
  - **Python Kubernetes client** (`kubernetes` & `openshift`) installed in your venv:  
    ```bash
    python3 -m venv ~/.ansible-venv
    source ~/.ansible-venv/bin/activate
    pip install --upgrade pip
    pip install ansible-core kubernetes openshift
    ```  
- **3× Raspberry Pi 4**  
  - Ubuntu Server 64-bit (arm64)  
  - SSH key-based auth configured (`ssh-copy-id`)  
  - Static IPs on your LAN  

---

## 🗂️ Inventory (`inventory.yml`)

    all:
      hosts:
        control_plane:
          ansible_host: 192.168.1.100
        worker_01:
          ansible_host: 192.168.1.200
        worker_02:
          ansible_host: 192.168.1.201

      children:
        control_planes:
          hosts:
            control_plane: {}
        workers:
          hosts:
            worker_01: {}
            worker_02: {}

---

## ▶️ Usage

    # Activate your Python venv
    source ~/.ansible-venv/bin/activate

    # Run the full automation
    ansible-playbook -i hosts.yml main.yml

    # Or simulate changes only
    ansible-playbook -i hosts.yml main.yml --check --diff

---

## 📝 Playbook Breakdown

### 01_prereqs.yaml

Prepares **all** Pi nodes:

    - disable swap & comment /etc/fstab
    - load overlay & br_netfilter modules + apply sysctl
    - install & configure containerd (SystemdCgroup)
    - add Kubernetes APT repo, install kubelet/kubeadm/kubectl
    - hold K8s packages against upgrades
    - enable & start containerd, restart on config change

### 02_init_controlplane.yaml

**Play 1 (control_plane)**

    - kubeadm init --pod-network-cidr=10.42.0.0/16 \
      --apiserver-advertise-address={{ ansible_default_ipv4.address }}
    - create ~/.kube/config on the Pi (mode 0600)
    - generate join command (kubeadm token create --print-join-command)

**Play 2 (localhost)**

    - fetch /etc/kubernetes/admin.conf to ~/.kube/config locally
    - apply Calico CRDs via k8s
    - apply Tigera Operator via k8s
    - download custom-resources.yaml, patch cidr to 10.42.0.0/16
    - apply custom-resources via k8s

### 03_join_workers.yaml

**Play 1 (workers)**

    - fetch admin.conf from control_plane → /tmp/admin.conf
    - copy /tmp/admin.conf → ~/.kube/config (mode 0600)
    - stat /etc/kubernetes/kubelet.conf
    - kubeadm join (creates: /etc/kubernetes/kubelet.conf)

**Play 2 (control_plane)**

    - kubectl wait --for=condition=Ready nodes --all --timeout=120s
      changed_when: false
      retries: 8
      delay: 15

---

## 🔑 Best Practices

- **Idempotence:** use Ansible modules (`apt`, `file`, `dpkg_selections`, `lineinfile`, `k8s`)
- **Security:**
  - `.kube/` directories set to `0700`
  - kubeconfig files set to `0600`
- **Modularity:** clear separation across three playbooks + `main.yml`
- **Dry-run:** always test with `--check --diff`

---

## 🛠 Troubleshooting

- **Calico controllers CrashLoopBackOff** ⇒ ensure `calico-node` DaemonSet is Ready before controllers
- **AlreadyExists errors** ⇒ prefer `k8s: state=present` or `kubectl apply`
- **Join preflight failures** ⇒ protect join with `args: creates: /etc/kubernetes/kubelet.conf`
- **Invalid kube-config (k8s module)** ⇒ fetch `admin.conf` locally and point `kubeconfig: ~/.kube/config`