# Windows Setup Guide

## Essential Software

### Basic Applications

- Notepad++ v8.8.5 (no root cert)
- Chrome
- Bitwarden extension

### Development Tools

- WSL ([Microsoft Installation Guide](https://learn.microsoft.com/en-us/windows/wsl/install))
  > Note: Manual enabling of WSL supporting features required despite documentation
- wsl.conf (note hostname. this should be all lowercase. microk8s warns about hostnames containing uppercase letters and the kube-ovn addon fails to install with a non-standard hostname)
- ```
  [boot]
  systemd=true
  
  [user]
  default={lowercase}
  
  [network]
  generateHosts = false
  hostname={follow the DNS label standard as defined in RFC 1123 - lowercase, only "-" seperator, must start with an alphabetic character, must end with an alphanumeric character}
  ```

### Package Managers

#### Chocolatey

Install with PowerShell:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

#### .NET Installation

Modified from chocolatey install and PowerShell script ([source](https://dotnet.microsoft.com/en-us/download/dotnet/scripts)):

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://builds.dotnet.microsoft.com/dotnet/scripts/v1/dotnet-install.ps1'))
```

### Development Applications

- DBeaver (via chocolatey)
- VS Code (via chocolatey)
  - Extensions:
    - WSL
    - C#
    - C# DevKit (requires .NET)
    - GitLens
    - Vim
    - Containers
    - Kubernetes

## Network Configuration

### Microsoft Loopback Adapter Setup

> For microk8s configuration (30.30.30.3)

Setup Resources:

- [Windows 11 Setup Guide](https://www.linkedin.com/pulse/how-create-microsoft-loopback-adapter-windows-11-buddhika-wijesooriya-3mgle)
- [DNS Registration Info](https://learn.microsoft.com/en-us/answers/questions/344170/loopback-adapters-keep-getting-registered-in-dns-a)
- [Windows Server Configuration](https://docs.progress.com/bundle/loadmaster-technical-note-configuring-dsr-ltsf/page/Add-a-loopback-interface-on-Windows-Server-2012-2016-and-2019.html)

Configuration Steps:

1. Optionally rename adapter to "Loopback" in Network Connections
2. Add additional IP addresses:

```cmd
netsh int ip add address Loopback address=30.30.30.10 mask=255.255.255.0
```

## WSL Configuration

### MicroK8s Setup

Installation and basic setup:

```bash
sudo snap install microk8s --classic --channel=latest/stable
microk8s config view > ~/.kube/config
microk8s enable hostpath-storage
mount --make-rshared / # This is to prevent the error message "path "/var/run/netns" is mounted on "/" but it is not a shared or slave mount" when enabling kube-ovn addon"
microk8s enable kube-ovn --force # This addon is fully supported by metallb. The default calico networking pluggin is "mostly" supported https://metallb.io/installation/network-addons/. It does not seem to allow access to k8s services from the host which I've been unable to debug and may be related to one of the known issues
microk8s enable ingress
microk8s enable metallb 30.30.30.1-30.30.30.10 # Ip Range of the MS Test Loopback adapter added above


```

Testing and Features:

- Network testing: `kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot`
- Enable ingress: `PUBLISH_STATUS_ADDRESS=$CURRENT_IP microk8s enable ingress`
- Enable metallb: `microk8s enable metallb 30.30.30.3-30.30.30.10`

> Note: As of 2025.09.09, access to mikrok8s services via metallb in WSL from host is not working

#### Network Change Procedure

When IP address changes:

```bash
# Refresh root CA and certificates
sudo microk8s refresh-certs --cert ca.crt
# Update container-registry.local in /etc/hosts
```

#### Registry Configuration

- Add insecure-registries to `/etc/docker/daemon.json`
- Check hosts.toml file for container-registry.local:32000 after restart

### Additional Software Installation

#### Browser and Development Tools

- [Firefox](https://support.mozilla.org/en-US/kb/install-firefox-linux#w_install-firefox-deb-package-for-debian-based-distributions)
- [Docker CE](https://docs.docker.com/engine/install/ubuntu/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

#### Fonts

- [Linux Font Management Guide](https://linuxconfig.org/how-to-install-and-manage-fonts-on-linux)
- [FiraCode Installation](https://github.com/tonsky/FiraCode/wiki/Linux-instructions#installing-with-a-package-manager)
- [Nerd Fonts](https://www.nerdfonts.com/)
  - [FiraCode Nerd Font Download](https://github.com/ryanoasis/nerd-fonts/releases/download/v3.4.0/FiraCode.zip)
  - [Linux Font Installation Guide](https://dev.to/pulkitsingh/install-nerd-fonts-or-any-fonts-easily-in-linux-2e3l)

#### Development Tools

- [Starship](https://starship.rs/) (config at `~/.config/starship.toml`)
- [NVM](https://github.com/nvm-sh/nvm?tab=readme-ov-file#install--update-script)
- [Helm](https://helm.sh/docs/intro/install/)

### Kubernetes Applications

#### Gitea

Enable storage and install:
```bash
microk8s enable hostpath-storage
helm install gitea gitea-charts/gitea --values
```

- [Default password information](https://gitea.com/gitea/helm-gitea#gitea)

#### ArgoCD

Installation and setup:

```bash
# Get initial password
argocd admin initial-password -n argocd
# Configure LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer", "loadBalancerIP": "30.30.30.2"}}'
```

#### SSL Configuration

Install SSL certificates:

```bash
sudo apt-get install ssl-cert
```

### Post-install MetalLB Configuration

Configure load balancing for Gitea services:

```bash
kubectl patch svc gitea-http -n gitea -p '{ "metadata": { "annotations": { "metallb.io/allow-shared-ip": "gitea-30.30.30.3"}}, "spec": {"loadBalancerIP": "30.30.30.3"}}'
kubectl patch svc gitea-ssh -n gitea -p '{ "metadata": { "annotations": { "metallb.io/allow-shared-ip": "gitea-30.30.30.3"}}, "spec": {"loadBalancerIP": "30.30.30.3"}}'
```



