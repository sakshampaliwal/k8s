# kubeadm Setup

# Kubernetes Cluster Setup Using Kubeadm on Ubuntu

This guide provides a step-by-step explanation for setting up a Kubernetes cluster using `kubeadm` on Ubuntu. Each command is explained to help you understand its purpose in the setup process.

---

## **1. System Update and Preparations**

### Update System Packages

```bash
sudo apt update && sudo apt upgrade -y

```

**Why:** Ensures that all installed packages are up-to-date, which minimizes compatibility issues.

### Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

```

**Why:** Kubernetes requires swap to be disabled for proper performance of the kubelet.

### Set Hostname

### For Master Node:

```bash
sudo hostnamectl set-hostname master-node

```

### For Worker Node:

```bash
sudo hostnamectl set-hostname worker01

```

**Why:** Assigns unique hostnames to the master and worker nodes for identification.

### Update Hosts File

```bash
sudo vi /etc/hosts

```

Add entries mapping IP addresses to the hostnames of all nodes in the cluster.
**Why:** Ensures nodes can resolve each other's hostnames.

---

## **2. Load Kernel Modules and Configure Networking**

### Enable Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

```

**Why:** Enables required kernel modules to support container networking.

`sudo modprobe overlay:` 
 is used to load the **overlay filesystem** kernel module into the Linux kernel. 
modprobe is a Linux utility that allows you to add or remove modules from the kernel. The `modprobe` command automatically handles module dependencies, loading them if necessary.

In Kubernetes, containers (called **pods**) need to talk to each other. If these pods are running on different physical machines (servers), they still need to communicate over the network.

Without the **overlay** network, the pods on different machines wouldn’t know how to talk to each other directly. The **overlay** network allows Kubernetes to create a **virtual network** that connects all the pods, regardless of which physical machine they are running on.

Think of it like setting up a **virtual bridge** that connects the pods across different computers, making them act as if they are on the same machine. This is where the **overlay** network module comes in: it creates this virtual bridge that Kubernetes uses to allow communication between the pods.
To verify overlay module is loaded:  `lsmod | grep overlay`

### What is "bridge networking"?

In many container setups (like Docker), the containers are connected to a **bridge** network. Think of a **bridge** like a virtual switch that connects all containers on the same machine.

- Containers can talk to each other over this bridge network.
- The **host machine** (your server) is also part of the bridge, so containers can communicate with the host and with each other.

### Why is **`br_netfilter`** needed?

When you have containers on a bridge network, the network traffic going **in and out of containers** needs to be **filtered** and controlled.

- **`br_netfilter`** ensures that **Kubernetes network policies** (rules for how containers should talk to each other) work properly, especially when traffic flows between containers and the host system.
- It allows Kubernetes to properly **filter** traffic based on these policies, ensuring security and correct communication.

Imagine a **virtual bridge** where all containers connect. Now, you want to make sure that some containers can talk to each other, but others cannot. The **`br_netfilter`** module helps Kubernetes enforce these rules.

### Configure Sysctl for Kubernetes

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

```

**Why:** Configures networking parameters for Kubernetes, enabling packet forwarding and bridge networking.

### What do these settings do?

- **`net.bridge.bridge-nf-call-iptables = 1`**:
    - This setting tells the Linux kernel to pass network traffic from containers through the **iptables** firewall.
    - It's important for Kubernetes because it enables **network policies** to filter traffic between containers based on the rules you set in iptables.
- **`net.bridge.bridge-nf-call-ip6tables = 1`**:
    - Similar to the previous one, but for **IPv6** traffic.
    - It ensures that IPv6 traffic from containers is also passed through ip6tables (which is the firewall for IPv6).
    - Just like with IPv4 (the older version of IP addresses), Kubernetes needs to control and filter **IPv6 traffic** between pods. Enabling this setting ensures that ip6tables (the IPv6 equivalent of iptables) can be used to manage traffic and enforce network security policies. Without this setting, Kubernetes wouldn't be able to apply rules to IPv6 traffic, which might cause issues in clusters that use IPv6.
- **`net.ipv4.ip_forward = 1`**:
    - This setting allows the system to **forward** IPv4 network traffic between different network interfaces (like from the container network to the external network or other containers).
    - It's necessary for Kubernetes because it enables pods to communicate with each other, and also allows traffic to go outside the node (e.g., to the internet).
    - Kubernetes often needs to route network traffic between pods on different nodes (servers) in a cluster. By enabling IP forwarding, you're allowing the system to route traffic between pods that may be on different machines. Without this, pods won't be able to communicate with each other if they are on separate nodes.

Why we need to apply these settings:

Kubernetes often uses **network bridges** to connect pods (small isolated units in Kubernetes that run applications) together. When pods send traffic to each other, it goes through these bridges. For **network security and policies** (like preventing pods from talking to each other), Kubernetes relies on **iptables rules** to inspect and control that traffic.

If this setting is not enabled, iptables rules would **not apply** to the traffic flowing between pods, and Kubernetes wouldn't be able to enforce network policies.

---

## **3. Install CRI-O Runtime**

### Add CRI-O Repository and Install

```bash
sudo apt-get update -y
sudo apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates

curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

```

**Why:** Installs CRI-O, a lightweight container runtime used by Kubernetes.

`sudo apt-get install -y software-properties-common gpg curl apt-transport-https ca-certificates`:

### 1. **`software-properties-common`**

- **What it does**: This package provides a set of tools for managing software repositories in Ubuntu-based systems.
- **Why we need it**: It includes the `add-apt-repository` command, which is used to add Personal Package Archives (PPAs) or other third-party repositories to your system. It's also required for managing custom repositories in some scenarios, such as installing Kubernetes or Docker.
- **Use case**: When you want to add external repositories (like Docker, Kubernetes, etc.) to your system, this package provides the functionality to do so.

### 2. **`gpg`**

- **What it does**: This is the **GNU Privacy Guard (GPG)** tool, used for encrypting and signing data, including package files.
- **Why we need it**: When installing packages from third-party repositories (such as Docker or Kubernetes), it's important to verify that the packages are authentic and haven't been tampered with. GPG keys help verify the integrity of packages by signing them, and this package enables that verification.
- **Use case**: Installing packages securely from official or third-party repositories requires GPG to ensure the authenticity of the software.

### 3. **`curl`**

- **What it does**: `curl` is a command-line tool used for transferring data with URLs. It supports protocols like HTTP, HTTPS, FTP, etc.
- **Why we need it**: `curl` is used in many scripts and commands to download files from the internet, especially when setting up repositories or downloading installation scripts.
- **Use case**: You’ll use `curl` to download scripts, files, or even GPG keys needed to configure repositories (for example, downloading the Docker installation script or Kubernetes setup).

### 4. **`apt-transport-https`**

- **What it does**: This package allows the **APT** package manager to communicate with repositories over HTTPS instead of HTTP.
- **Why we need it**: Some repositories, especially third-party ones (like Docker, Kubernetes, etc.), require HTTPS for secure communication. Installing this package ensures that APT can securely communicate with those repositories and download packages.
- **Use case**: When setting up Kubernetes or Docker repositories, they often require HTTPS to ensure the integrity and security of the packages being downloaded.

### 5. **`ca-certificates`**

- **What it does**: This package contains a collection of **Certificate Authority (CA) certificates**, which are used to verify the authenticity of SSL/TLS connections.
- **Why we need it**: When using `apt` to fetch packages over HTTPS, the system needs to verify that the server you're downloading from is trusted. The `ca-certificates` package ensures that the system can validate SSL/TLS certificates and establish secure connections.
- **Use case**: You’ll need this package to ensure that `curl` and APT can securely fetch files over HTTPS, especially from trusted repositories.

To check if they are installed or not: `dpkg -l | grep software-properties-common`

**`curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key`**

- **`curl`**: A command-line tool used to transfer data from or to a server. In this case, it’s fetching data from a URL.
- **`fsSL`**: These are options used with `curl`:
    - **`f`**: Fail silently on server errors (like 404 or 500), preventing `curl` from printing errors to the console.
    - **`s`**: Silent mode, which prevents `curl` from displaying progress information.
    - **`S`**: Show an error message if it fails.
    - **`L`**: Follow redirects if the URL is redirected to another location (for example, HTTP -> HTTPS).
- **`https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key`**: This is the URL from which you're fetching the GPG key. The key is used to verify the authenticity of the CRI-O package repository.
    - CRI-O is an **Open Container Initiative (OCI) compatible container runtime** used in Kubernetes clusters to run containers.
    - The URL provides a GPG **public key** (a cryptographic key) for the Kubernetes package repository that hosts CRI-O packages.
- **`|` (Pipe Operator)**
    - The pipe operator (`|`) takes the output of the `curl` command (the GPG key) and passes it as input to the next command (`gpg`).
- **`gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg`**
    - **`gpg`**: This is the GNU Privacy Guard (GPG) tool. It is used for encrypting, signing, and verifying data.
    - **`-dearmor`**: This option tells `gpg` to convert the input key from ASCII-armored format (a textual representation of the key) into a binary format that APT can use for verifying the authenticity of packages.
    - **`o /etc/apt/keyrings/cri-o-apt-keyring.gpg`**: This option specifies the output file where the converted (binary) GPG key will be stored. In this case, it's saving it to the directory `/etc/apt/keyrings/`, which is a standard location for storing trusted GPG keys in modern Debian-based systems.
    
    When you add a third-party repository to your system (like the Kubernetes or CRI-O repository), you want to ensure that the packages you're downloading and installing are from a trusted source and haven't been tampered with.
    
    [GPG Key Concept](https://www.notion.so/GPG-Key-Concept-16a5182282d580f48eedc482bb2e51bb?pvs=21)
    

### Command Breakdown

```bash
bash
Copy code
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list
```

1. **`echo "..."`**:
    - `echo` is a command that prints text to the standard output (usually the terminal).
    - Inside the `echo` command, you are writing the repository URL and instructions for APT to use the public key for verifying packages from this repository.
2. **`deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] ...`**:
    - `deb` tells APT that this is a **Debian-based repository**, meaning it's a repository that contains `.deb` packages (which is the format used by Debian, Ubuntu, and similar Linux distributions).
    - `[signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg]`: This part specifies the **location of the GPG key** that should be used to verify the authenticity of the packages from this repository. The key is located at `/etc/apt/keyrings/cri-o-apt-keyring.gpg`.
        - APT will use this key to check if the packages in this repository are **signed and verified**.
        - This is a security measure to ensure that the software you're downloading is authentic and hasn't been tampered with.
3. **`https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/`**:
    - This is the actual URL of the repository that you are adding to your system.
    - The repository contains packages for **CRI-O**, a container runtime for Kubernetes.
    - The path `addons:/cri-o:/prerelease:/main/deb/` indicates that it's specifically for CRI-O (the Kubernetes container runtime) and this is the **prerelease** version, which is often used for testing or early access to features before the stable release.
4. **`/`**:
    - The `/` at the end is a standard part of the repository URL in the Debian-based system. It’s the root of the repository where all the packages are stored.
5. **`| tee /etc/apt/sources.list.d/cri-o.list`**:
    - The pipe (`|`) takes the output of the `echo` command (which is the repository configuration) and sends it to the `tee` command.
    - `tee` is used to both **display** the output on the screen and **write** it to a file.
    - In this case, it writes the repository information into the file `/etc/apt/sources.list.d/cri-o.list`.
        - This file is where APT looks to find additional package repositories. By adding this file, you are telling your system to use this **CRI-O repository** for package installation.

### What Does This Command Do?

This command adds the **CRI-O repository** to your system's list of repositories, so you can install and update the CRI-O package directly from the Kubernetes package source.

Here’s what happens in the sequence:

1. The command adds the **CRI-O repository** URL to a new file in `/etc/apt/sources.list.d/` (specifically `/etc/apt/sources.list.d/cri-o.list`).
2. The `signed-by` option specifies the GPG key (`/etc/apt/keyrings/cri-o-apt-keyring.gpg`) that should be used to verify packages from this repository.
3. This means that when you install or update CRI-O from this repository using `apt`, it will only allow packages that have been **signed** with the private key that corresponds to the public key you’ve imported.

### Why Is This Necessary?

- **Security**: By specifying the `signed-by` option and providing the path to the GPG key, you're telling APT to only trust packages from this repository if they are signed with the matching GPG key. This helps protect your system from **malicious packages** that could be added to the repository by an attacker.
- **Convenience**: This configuration allows you to easily install and update CRI-O (or any other software from that repository) using standard `apt` commands, just like you would with any other software from the default Ubuntu or Debian repositories.

### After Running This Command

Once you run this command, you need to **update the APT package list** to fetch the information from the newly added repository. You can do this by running:

```bash

sudo apt update
```

This command will make sure that your system is aware of the new packages available in the **CRI-O repository**.

Now you can install CRI-O or any other package provided by this repository using:

```bash

sudo apt install cri-o
```

This will install the CRI-O container runtime on your system.

### Start and Enable CRI-O

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

```

**Why:** Ensures CRI-O starts automatically after reboot.

**`systemctl enable crio --now`** enables the CRI-O container runtime to start automatically at boot and also starts it immediately.

---

## **4. Install Kubernetes Tools (Kubeadm, Kubelet, Kubectl)**

### Add Kubernetes Repository

```bash
KUBERNETES_VERSION=1.30

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" |
    sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y

```

**Why:** Adds the official Kubernetes repository for installing `kubeadm`, `kubelet`, and `kubectl`.

### 1. Set Kubernetes Version

```bash
KUBERNETES_VERSION=1.30
```

- **Explanation**: This sets an environment variable `KUBERNETES_VERSION` to `1.30`, which represents the Kubernetes version you want to install or use. The version specified here will be dynamically inserted into the next commands.

### 2. Create Keyrings Directory

```bash
sudo mkdir -p /etc/apt/keyrings
```

- **Explanation**: This creates the directory `/etc/apt/keyrings` if it doesn't already exist. This directory is used to store the GPG keyring that will be used to verify the integrity of the packages when you install Kubernetes components.
- The `p` flag ensures that no error occurs if the directory already exists.

### 3. Download Kubernetes GPG Key and Save It

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key |
    sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- **Explanation**:
    - `curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key`: This command uses `curl` to download the GPG public key for the Kubernetes repository. This key is used to verify the authenticity and integrity of the packages when you install them.
        - The URL contains `v$KUBERNETES_VERSION`, so it will download the key for the version specified earlier (in this case, `v1.30`).
        - The `fsSL` flags:
            - `f`: Fail silently on server errors (no output).
            - `s`: Silent mode (no progress bar).
            - `S`: Show error messages if there is an issue.
            - `L`: Follow redirects if the URL redirects to another address.
    - `sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`:
        - This converts the downloaded GPG key from ASCII-armored format (a text format) into a binary format (`-dearmor`) and saves it as `/etc/apt/keyrings/kubernetes-apt-keyring.gpg`.
        - The GPG key is used by `apt` to verify the packages from the Kubernetes repository, ensuring that you are installing genuine, unmodified software.

### 4. Add Kubernetes Repository to `apt` Sources

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /" |
    sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- **Explanation**:
    - `echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/ /"`: This command constructs a line of text to specify the Kubernetes APT repository URL. The `$KUBERNETES_VERSION` variable will be replaced with `1.30`, so the final URL will be `https://pkgs.k8s.io/core:/stable:/v1.30/deb/`.
        - `deb`: This indicates that the repository contains Debian packages.
        - `[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg]`: This tells `apt` to use the Kubernetes GPG key (`/etc/apt/keyrings/kubernetes-apt-keyring.gpg`) to verify the authenticity of the packages from this repository.
    - `sudo tee /etc/apt/sources.list.d/kubernetes.list`: This writes the repository configuration into the file `/etc/apt/sources.list.d/kubernetes.list`. This is the APT configuration file that tells `apt` where to find Kubernetes packages.
        - `tee` is used to redirect the output of the `echo` command to a file with `sudo` permissions.

### 5. Update APT Package List

```bash
sudo apt-get update -y
```

- **Explanation**:
    - This command updates the local package index from the repositories configured in your system (including the newly added Kubernetes repository). It fetches information about the available packages from all the repositories listed in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/*`, ensuring that your system knows about the latest available packages and updates.
    - The `y` flag automatically confirms the update, meaning it won't ask you for confirmation before downloading the package lists.

### Install Tools

```bash
sudo apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1 kubeadm=1.30.0-1.1
```

**Why:** Installs the Kubernetes command-line tools necessary for cluster management.

---

## **5. Configure Kubelet**

### Set Node IP Address

```bash
sudo apt-get install -y jq
local_ip="$(ip --json addr show ens4 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```

```yaml
local_ip="172.24.131.61"
echo "KUBELET_EXTRA_ARGS=--node-ip=$local_ip" | sudo tee /etc/default/kubelet > /dev/null

```

1. Install jq

```bash
sudo apt-get install -y jq
```

- **Explanation**: This command installs the `jq` utility on your system, which is a powerful tool for working with JSON data from the command line.
    - `jq` allows you to parse, filter, and manipulate JSON data in various ways. In your case, it's used to extract specific information (the local IP address) from the output of the `ip` command.
    - The `y` flag automatically confirms the installation without asking for confirmation.

### 2. Extract the Local IP Address

```bash
local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
```

- **Explanation**: This command extracts the local IP address of the system using two tools: `ip` and `jq`.
    - `ip --json addr show eth0`: This command shows the network interface (`eth0` in this case) information in JSON format.
        - `-json`: Outputs the `ip` command results in JSON format, which is easier to parse programmatically.
    - `jq -r '.[0].addr_info[] | select(.family == "inet") | .local'`: This part uses `jq` to process the JSON data produced by the `ip` command:
        - `.[0].addr_info[]`: Refers to the first (and usually only) entry in the list of address info for the `eth0` interface. This is an array of addresses, and we're iterating over each one.
        - `select(.family == "inet")`: Filters the entries to include only those with the IP family `inet`, which corresponds to IPv4 addresses (not IPv6).
        - `.local`: Extracts the local IP address from the selected entry.
    - The result of this operation (the local IP address) is saved into the `local_ip` variable.

### 3. Configure Kubelet to Use the Local IP Address

```bash
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
```

- **Explanation**: This command writes a configuration setting for the Kubernetes `kubelet` into the `/etc/default/kubelet` file.
    - `cat > /etc/default/kubelet`: This redirects the following content into the `/etc/default/kubelet` file, effectively setting environment variables and configuration options for the `kubelet` service.
    - `KUBELET_EXTRA_ARGS=--node-ip=$local_ip`: This line sets the `KUBELET_EXTRA_ARGS` environment variable to include the `-node-ip` flag, which specifies the IP address that the `kubelet` will use to identify itself in the cluster.
        - `$local_ip` will be replaced with the local IP address extracted from the previous command, ensuring that the `kubelet` uses the correct IP address for the node.

### Why is this Important?

### **Kubelet and Node IP Address**

- In Kubernetes, the `kubelet` is the primary agent that runs on each node in the cluster. It ensures that containers are running in the desired state.
- The `-node-ip` flag is used to explicitly specify the IP address that the `kubelet` will bind to. If your node has multiple network interfaces or IP addresses (e.g., internal and external IPs), it's crucial to specify the correct one to avoid issues with node communication or networking.
    - By setting the `node-ip` to the local IP address (`$local_ip`), you're ensuring that the `kubelet` binds to the correct network interface for internal communication within the Kubernetes cluster.

### **Automatic Detection of IP Address**

- The command `local_ip="$(ip --json addr show eth0 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"` automates the process of detecting the local IP address of your machine.
    - This is particularly useful in cloud environments or machines with multiple network interfaces where the IP address can change or vary. By automating this, you don't have to manually configure the IP address each time.

### Summary of What Happens:

- **Step 1**: `jq` is installed to process JSON data.
- **Step 2**: The local IP address of the machine is automatically extracted using the `ip` command and `jq`.
- **Step 3**: The `kubelet` configuration is updated to ensure the `kubelet` uses the correct local IP address by modifying the `/etc/default/kubelet` file.

After running these commands, the `kubelet` will start with the `--node-ip` flag set to the local IP address of your system, helping it register and communicate with the Kubernetes control plane.

Why this method to extract the ip?

[Network Interfaces](https://www.notion.so/Network-Interfaces-16a5182282d5808ebcd5cddcab328ea3?pvs=21)

[ip using jq](https://www.notion.so/ip-using-jq-16a5182282d58071b7bee73501a677bd?pvs=21)

---

## **6. Initialize the Kubernetes Cluster (Master Node)**

### Initialize Kubernetes

```bash
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"

sudo kubeadm init --control-plane-endpoint=$IPADDR \
    --apiserver-cert-extra-sans=$IPADDR \
    --pod-network-cidr=$POD_CIDR --node-name $NODENAME \
    --ignore-preflight-errors Swap
```

On Private Ip this works for me:
```
`IPADDR=$(hostname -I | awk '{print $1}')
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"`

`sudo kubeadm init --control-plane-endpoint=$IPADDR \
--apiserver-cert-extra-sans=$IPADDR \
--pod-network-cidr=$POD_CIDR --node-name $NODENAME \
--ignore-preflight-errors Swap`
```

### Full Command Breakdown:

### 1. **`IPADDR=$(curl ifconfig.me && echo "")`**

- **Purpose**: This command fetches the **public IP address** of your machine and stores it in the `IPADDR` variable.
- **How it works**:
    - `curl ifconfig.me`: This uses `curl` to make a request to the `ifconfig.me` service, which returns the public IP address of the machine.
    - `&& echo ""`: This ensures that after the IP is fetched, an empty string is printed, which ensures the correct result of the `curl` command is captured.
- **Why it's needed**: In a multi-node Kubernetes setup, this IP address will be used as the **control-plane endpoint** for the Kubernetes master node, allowing other nodes to communicate with the API server.

### 2. **`NODENAME=$(hostname -s)`**

- **Purpose**: This command fetches the short name of the current machine (hostname) and assigns it to the variable `NODENAME`.
- **How it works**:
    - `hostname -s`: This returns the **short hostname** of the machine (e.g., `k8s-master` for a master node or `worker-node-1` for a worker node).
- **Why it's needed**: The `node-name` is passed to `kubeadm` during the cluster initialization process. This helps Kubernetes identify the node's name in the cluster.

### 3. **`POD_CIDR="192.168.0.0/16"`**

- **Purpose**: This defines the CIDR block (IP range) for the pods in the Kubernetes cluster.
- **How it works**:
    - `POD_CIDR="192.168.0.0/16"`: This assigns a **subnet** to the pods in the cluster. A `/16` CIDR means that there are 65,536 possible IP addresses for pods.
- **Why it's needed**: The `-pod-network-cidr` option is used to specify the IP range that will be allocated to pods in the cluster. This allows Kubernetes to allocate unique IP addresses to each pod.
    - **Note**: The value you choose for the pod CIDR depends on the networking solution you use (e.g., Calico, Flannel). Different solutions may have different CIDR block requirements.

### 4. **`sudo kubeadm init --control-plane-endpoint=$IPADDR --apiserver-cert-extra-sans=$IPADDR --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap`**

This is the main command that initializes the **Kubernetes master node** and sets up the control plane. Here's what each option does:

- **`sudo kubeadm init`**:
    - This runs the `kubeadm init` command with elevated privileges (`sudo`), which is necessary for Kubernetes cluster setup.
    - It initializes the Kubernetes control plane (master node) on the machine.
- **`-control-plane-endpoint=$IPADDR`**:
    - This specifies the **control-plane endpoint** (master node’s IP address) for the cluster. The value `$IPADDR` is the **public IP** of your machine, which is dynamically fetched in the earlier step.
    - Other nodes in the cluster (workers) will use this IP to communicate with the master node's API server.
- **`-apiserver-cert-extra-sans=$IPADDR`**:
    - This option is used to configure additional **Subject Alternative Names (SANs)** for the API server's TLS certificate. SANs are used in TLS certificates to allow multiple domain names or IP addresses to be associated with the same certificate.
    - Here, it adds `$IPADDR` (the public IP address of the master node) as a valid SAN for the Kubernetes API server certificate. This ensures that when other nodes or clients try to reach the Kubernetes API server, they won't get certificate errors related to the IP address.
- **`-pod-network-cidr=$POD_CIDR`**:
    - This sets the **IP range for pods** in the Kubernetes cluster. The specified `POD_CIDR` (`192.168.0.0/16`) means Kubernetes will allocate IPs to pods in this range.
    - This is important because the network plugin (e.g., Flannel, Calico) will need to know which IP range to allocate to pods.
- **`-node-name $NODENAME`**:
    - This specifies the **node name** for the Kubernetes node being initialized. The value `$NODENAME` is derived from the hostname of the machine using `hostname -s`.
    - The node name is used by Kubernetes to identify the node within the cluster.
- **`-ignore-preflight-errors Swap`**:
    - This option tells `kubeadm` to ignore the **preflight error** related to **swap space**.
    - By default, Kubernetes requires **swap** to be disabled on all nodes for performance and stability reasons. If swap is enabled, you would typically get a preflight error during `kubeadm init`.
    - `-ignore-preflight-errors Swap` disables this check, allowing you to proceed even if swap is enabled on the system.
    - **Important**: It's recommended to disable swap before setting up Kubernetes, but this flag allows you to bypass that step for situations where you may not want to disable swap.

### What This Command Does:

- **Initializes the Kubernetes master node**: The `kubeadm init` command sets up the control plane for the Kubernetes cluster on the node. It configures the API server, controller manager, scheduler, and other components necessary to run the cluster.
- **Control-plane endpoint**: The public IP address of the node is set as the control-plane endpoint, allowing other nodes to communicate with the API server.
- **Pod network**: The `-pod-network-cidr` option defines the range of IPs that can be used for pods. The actual network plugin (like Calico, Flannel, or Weave) will use this range.
- **Node name**: The node's short hostname is set as the name of the node in the cluster.
- **Swap handling**: The command will ignore swap-related errors, even if swap is not disabled.

### After Running `kubeadm init`:

- Once the master node is initialized, you will get a command for joining worker nodes to the cluster (a `kubeadm join` command).
- You'll also need to install a **pod network plugin** (e.g., Calico, Flannel) to enable pod-to-pod communication.
    - Example command: `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

### Configure kubectl for User Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 1. **`mkdir -p $HOME/.kube`**

- **Purpose**: Creates the `.kube` directory in your home directory if it doesn't already exist.
- **Explanation**:
    - `$HOME`: This environment variable points to the current user's home directory (e.g., `/home/username` or `/Users/username`).
    - `.kube`: This is the directory where Kubernetes configuration files are typically stored. It will hold the `config` file that `kubectl` uses to connect to the Kubernetes cluster.
    - `p`: This option ensures that if the directory path does not exist (in this case, the `.kube` directory), it will be created without errors. If the directory already exists, no error will occur.

**Effect**: This command ensures the `.kube` directory exists, allowing subsequent commands to store the Kubernetes config file there.

### 2. **`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`**

- **Purpose**: Copies the Kubernetes admin configuration file (`admin.conf`) to your user's `.kube` directory, so that `kubectl` can use it.
- **Explanation**:
    - `/etc/kubernetes/admin.conf`: This is the default location of the **Kubernetes admin configuration file**. It is created when you run `kubeadm init` and contains the necessary information for connecting to the Kubernetes cluster (such as the API server URL, credentials, etc.).
    - `$HOME/.kube/config`: This is where Kubernetes stores the configuration file used by `kubectl` to interact with the cluster.
    - `i`: The `i` flag stands for "interactive." It ensures that `cp` asks for confirmation if the file already exists, preventing overwriting without warning.

**Effect**: This command copies the admin configuration file, allowing `kubectl` to use it for accessing the Kubernetes API server.

### 3. **`sudo chown $(id -u):$(id -g) $HOME/.kube/config`**

- **Purpose**: Changes the ownership of the copied Kubernetes config file so that your user can access and modify it.
- **Explanation**:
    - `sudo`: This is required because the original file (`admin.conf`) is owned by the `root` user, and you need elevated privileges to change the file's ownership.
    - `chown`: This command is used to change the ownership of a file or directory.
    - `$(id -u)`: This command returns your **user ID** (UID). It ensures that the ownership is set to the current user.
    - `$(id -g)`: This command returns your **group ID** (GID). It ensures that the ownership is set to the current user's group.
    - `$HOME/.kube/config`: This is the file whose ownership is being changed.

**Effect**: This command changes the ownership of the Kubernetes configuration file so that your user (instead of `root`) has read and write access to it. This is necessary because without this step, only `root` would have access to the file, and you wouldn't be able to run `kubectl` without using `sudo`.

### **Why are these steps necessary?**

When you initialize a Kubernetes master node with `kubeadm init`, the configuration file (`admin.conf`) is created under `/etc/kubernetes/admin.conf`. However, this file is owned by the `root` user, and only `root` can read it by default. Since you want to interact with the Kubernetes cluster using `kubectl` (which you run as your user), you need to copy the configuration file to your home directory and ensure that your user has ownership of the file.

Once you've completed these steps, you'll be able to run `kubectl` as your regular user to manage the Kubernetes cluster.

### **What happens after these steps?**

1. **Configuration File**: The file at `$HOME/.kube/config` will now contain the Kubernetes configuration required to access the cluster. This is where `kubectl` looks for the necessary credentials and cluster information.
2. **User Access**: After changing the file ownership, you can use `kubectl` without `sudo` to interact with your Kubernetes cluster.
3. **Running kubectl**: Now you can start running `kubectl` commands like:
    
    ```bash
    kubectl get nodes
    kubectl get pods --all-namespaces
    ```
    

### **Summary**:

- **Step 1** (`mkdir`): Creates the `.kube` directory where Kubernetes config files will be stored.
- **Step 2** (`cp`): Copies the Kubernetes admin configuration file (`admin.conf`) to your user's `.kube` directory.
- **Step 3** (`chown`): Changes the ownership of the Kubernetes config file to your user, allowing you to use `kubectl` without requiring `sudo`.

These steps enable you to use `kubectl` to manage your Kubernetes cluster from your local machine.

---

## **7. Join Worker Nodes**

### Generate Join Command (Master Node)

```bash
kubeadm token create --print-join-command

```

**Why:** Provides the command for worker nodes to join the cluster.

### Run Join Command on Worker Nodes

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash <HASH>

```

### **Command Breakdown**

- **`kubeadm`**: This is the command-line tool for managing Kubernetes cluster lifecycle tasks such as initialization, joining nodes, and upgrading clusters.
- **`token create`**: This subcommand creates a **token** used by other nodes (typically worker nodes) to join the cluster securely.
- **`-print-join-command`**: This flag tells `kubeadm` to not just create the token but also print the **`kubeadm join`** command that you need to run on the worker nodes to join the cluster.

### **What Happens When You Run This Command?**

1. **Creates a Token**: A temporary **token** is generated. This token is used by worker nodes to authenticate and securely join the cluster.
2. **Prints the Join Command**: The command prints out the exact `kubeadm join` command that you need to run on each worker node. This join command includes:
    - The **control-plane endpoint** (API server's IP or hostname).
    - The **token** that worker nodes will use to authenticate with the Kubernetes master node.
    - A **hash** of the API server’s certificate for security.

Token is valid for 24hrs.

---

## **8. Configure Networking**

### Install Network Plugin

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

```

You're absolutely right to notice that you’ve already enabled **`br_netfilter`** and **`overlay`** modules on your system, but these configurations are not the same as installing and configuring a **network plugin** (CNI plugin) for Kubernetes. Let's clarify what each of these modules does and why you still need to install a **network plugin** despite having configured these system modules.

### 1. **br_netfilter Module**:

The **`br_netfilter`** module is a kernel module that enables the **bridge network filtering** features for Linux bridges. This is particularly important when you are using Kubernetes in a setup where containers/pods communicate via Linux bridges (i.e., in a setup where networking uses **virtual Ethernet interfaces** and **bridged networks**).

When you enable the **`br_netfilter`** module, you’re allowing Kubernetes (and Docker) to have proper control over packet filtering for **bridge interfaces**. It ensures that iptables (or similar tools) can properly control the traffic flowing between containers and between containers and the outside world.

However, enabling **`br_netfilter`** alone does **not** provide actual networking between pods. It only ensures that the system allows the traffic to be managed via iptables and other filtering mechanisms.

### What does **`br_netfilter`** do?

- **Packet Filtering for Bridges**: It allows the Linux kernel to filter packets that are being bridged. Kubernetes uses this feature to manage pod communication.
- **iptables Support for Networking**: It ensures that Kubernetes can properly set up iptables rules for pod-to-pod traffic and pod-to-node traffic, especially when the networking involves virtual network bridges.

But even though **`br_netfilter`** ensures packet filtering is enabled, **it still doesn’t implement the actual pod network** that allows Kubernetes to manage and route traffic between pods across nodes. That’s where the **CNI plugin** comes in.

### 2. **overlay Module**:

The **`overlay`** module is a network driver used for **overlay networking** in containerized environments. When you use an overlay network, each container or pod gets its own unique IP address, and the communication between these containers (even across nodes) is encapsulated within a virtual network.

In Kubernetes, the **`overlay`** module allows the creation of virtual networks that can span across multiple nodes. This is a necessary piece for enabling **multi-host networking**, where pods on different machines can communicate as though they are on the same local network.

- **What does the overlay module do?**
    - It enables the use of **overlay networks**, where network traffic between pods on different nodes is encapsulated in virtual tunnels (e.g., using VXLAN or GRE tunneling).
    - Without this module, Kubernetes wouldn’t be able to handle communication between pods that span multiple physical machines in the cluster.

But like **`br_netfilter`**, the **`overlay`** module still doesn’t solve the higher-level **pod networking logic** required by Kubernetes. It only sets up the **underlying transport layer** for network traffic between pods across nodes. You still need a **CNI plugin** to manage the actual **IP address assignment**, **routing**, and **network policies**.

### 3. **What’s Missing Without a CNI Plugin?**

Even though you’ve enabled **`br_netfilter`** and **`overlay`**, Kubernetes still needs a **CNI plugin** to implement the complete network stack that Kubernetes requires for pods and services. Here’s why:

### 1. **Pod IP Assignment and Routing**:

- A CNI plugin manages the assignment of IP addresses to pods. For example, in a multi-node cluster, a pod running on **Node A** should have a unique IP address that can be routed to a pod running on **Node B**.
- CNI plugins like **Calico**, **Flannel**, or **Weave** provide an **IP address management system (IPAM)** that ensures each pod gets a unique IP, and that traffic can be correctly routed between pods on different nodes.

### 2. **Network Communication**:

- The CNI plugin ensures that there is network connectivity between all pods, whether they are on the same node or different nodes. It handles the routing of traffic between the different network namespaces (which are created for pods).
- The **overlay module** provides the capability to tunnel traffic between nodes, but the CNI plugin manages the full network stack, making sure that the traffic reaches the correct pod on the correct node.

### 3. **Network Policies**:

- Many CNI plugins, like **Calico**, provide **network policies** that allow you to control which pods can communicate with each other. Without a CNI plugin, Kubernetes doesn’t have a way to enforce these policies.
- For example, with Calico, you can define a policy like: "Only pods in the `frontend` namespace can communicate with pods in the `backend` namespace", and Calico enforces this by creating iptables rules or using eBPF for faster filtering.

### 4. **DNS for Services**:

- Kubernetes needs to provide a DNS service to resolve the names of services and pods. A CNI plugin ensures the network is correctly configured so that the Kubernetes DNS system can resolve names like `myservice.my-namespace.svc.cluster.local` to the correct pod IPs.
- Without the proper networking setup provided by a CNI plugin, DNS resolution across pods and services may not work.

### 4. **Conclusion: Why the Network Plugin is Still Required**

In summary, enabling **`br_netfilter`** and **`overlay`** provides the kernel-level infrastructure that Kubernetes can use to manage low-level network operations like filtering traffic and tunneling packets between nodes. However, these modules don’t **implement Kubernetes-specific networking**, such as **IP address management**, **pod-to-pod communication across nodes**, **network policies**, and **service discovery**.

The **CNI plugin** fills this gap and provides the actual functionality that Kubernetes needs to run its networking. Without a CNI plugin:

- Pods won’t get IP addresses that can be routed to other pods.
- Pods won’t be able to communicate across nodes.
- Kubernetes won’t be able to enforce **network policies**.
- DNS resolution for services will likely fail.
- The cluster won’t function correctly.

Therefore, while enabling **`br_netfilter`** and **`overlay`** is important, installing a CNI plugin is absolutely essential for Kubernetes networking to work properly.

---

## **9. Setup Metrics Server**

### Deploy Metrics Server

```bash
kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml

```

**Why:** Enables resource metrics for nodes and pods.

---

## **10. Verification**

### Check Node Status

```bash
kubectl get nodes

```

**Why:** Verifies that all nodes are correctly joined and ready.

### Check Cluster Information

```bash
kubectl cluster-info

```

**Why:** Confirms the cluster is operational.

### View Metrics

```bash
kubectl top nodes

```

**Why:** Displays resource usage metrics.

---

Following these steps ensures a robust Kubernetes cluster setup on Ubuntu, with essential tools and configurations explained for clarity.