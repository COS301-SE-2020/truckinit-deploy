## Team Enigma
This project is a product by team Enigma for the "COS301 - Software Engineering" (University of Pretoria) 2020 Capstone Project Assignment.
- For more information about the team, [check out our website](https://enigmasoftware.co.za/)!
- For our other related project modules, see the following:
  - Mobile Apps ([gitlab](https://gitlab.com/team-enigma/mobile/), [github](https://github.com/COS301-SE-2020/truckinit-mobile))
  - Cluster Deployment ([gitlab](https://gitlab.com/team-enigma/deploy/), [github](https://github.com/COS301-SE-2020/truckinit-deploy))


# Truckin-IT Deployment
> **Note**: This repository serves to hold configuration files only and as such does not contain all of the information 
  regarding our project and its progress, to view more about our project please navigate to either our 
  [API](https://gitlab.com/team-enigma/api)  or [Mobile](https://gitlab.com/team-enigma/mobile) repository.

*A centralized repository for the Truckin-IT [Docker](https://www.docker.com/) images and
  [Kubernetes](https://kubernetes.io/) cluster configuration files*

**Please be aware that this repository is being mirrored from [GitLab](https://gitlab.com/team-enigma/deploy) → 
  [GitHub](https://github.com/COS301-SE-2020/truckinit-deploy). As a result, if you are viewing this repository on 
  GitHub, we suggest that you rather navigate to the GitLab repository.**


## Our Team
For more about us, [check out our website!](https://enigmasoftware.co.za/)

<table>
<tr>
  <td>
  <img alt="Malick Burger" src="https://enigmasoftware.co.za/img/malick/malick.png" width="150pt">
  </td>
  <td>
    <h3>Malick Burger</h3>
    <p><em>Systems Administration and Deployment</em></p>
    <p><b>Active on this repository</b></p>
    <p>(<a href="https://enigmasoftware.co.za/profiles/malick">see profile</a>)</p>
  </td>
</tr>

<tr>
  <td>
  <img alt="Daneel Nortier" src="https://enigmasoftware.co.za/img/daneel/daneel.jpeg" width="150pt">
  </td>
  <td>
    <h3>Daneel Nortier</h3>
    <p><em>Machine Learning and Optimization</em></p>
    <p><b>Active on this repository</b></p>
    <p>(<a href="https://enigmasoftware.co.za/profiles/daneel">see profile</a>)</p>
  </td>
</tr>

<tr>
  <td>
  <img alt="Francois Snyman" src="https://enigmasoftware.co.za/img/francois/francois.jpg" width="150pt">
  </td>
  <td>
    <h3>Francois Snyman</h3>
    <p><em>Mobile & Web Development</em></p>
    <p>(<a href="https://enigmasoftware.co.za/profiles/francois">see profile</a>)</p>
  </td>
</tr>

<tr>
  <td>
  <img alt="Xander Terblanche" src="https://enigmasoftware.co.za/img/xander/xander.jpg" width="150pt">
  </td>
  <td>
    <h3>Xander Terblanche</h3>
    <p><em>UI/UX and Mobile Development</em></p>
    <p>(<a href="https://enigmasoftware.co.za/profiles/xander">see profile</a>)</p>
  </td>
</tr>

<tr>
  <td>
  <img alt="Preston van Tonder" src="https://enigmasoftware.co.za/img/preston/profile.png" width="150pt">
  </td>
  <td>
    <h3>Preston van Tonder</h3>
    <p><em>API Design and DevOps</em></p>
    <p><b>Active on this repository</b></p>
    <p>(<a href="https://enigmasoftware.co.za/profiles/preston">see profile</a>)</p>
  </td>
</tr>
</table>

# Setup

## Overview
This repository contains the yaml configuration files and documentation required to 
  deploy and maintain the **Truckin-IT** backend system on a Kubernetes (k8s) cluster.

The main purpose of this document is to provide detailed instructions on how to setup a k8s 
  cluster which can then be used to host the **Truckin-IT** backend system.

> **Note**: The steps found in this documentation make a number of assumptions 
  (such as using a debian based image for the cluster nodes) for the sake of simplicity. 
  This by no means implies that the system cannot be deployed in a different manner.

## Cluster Setup

### Kubeadm
> **Note**: All commands should be run on the control-plane/master node unless otherwise stipulated

[Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
  is used to create the Kubernetes (k8s) cluster which runs all the microservices used in
  the system.

1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

2. Install initial apt packages
```bash
sudo apt install -y git nmap zsh apt-transport-https curl gnupg2 docker.io vim
```

3. Install oh-my-zsh
```bash
sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
chsh -s /bin/zsh
```

4. Enable bridged network traffic
```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

5. Enable Docker
```bash
sudo systemctl enable --now docker
```

6. Install Kubeadm
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

7. Restart Kubelet
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

8. Create cluster
```bash
kubeadm init
```

9. Copy kube config
```bash
# Don't just copy if your user already has a .kube/config setup

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# The details of this .kube/config need to be copied to all machines that will be used to
# develop and deploy on

```

* Example Kube Config with multiple clusters (kubectl use-context <context> to change)
```yaml
apiVersion: v1
clusters:
  - name: cluster-truckinit
    cluster:
      certificate-authority-data: LS0<REDACTED>
      server: https://<REDACTED>

  - name: cluster-local
    cluster:
      certificate-authority-data: LS0<REDACTED>
      server: https://127.0.0.1:16443

contexts:
  - name: context-truckinit
    context:
      cluster: cluster-truckinit
      user: user-truckinit

  - name: context-local
    context:
      cluster: cluster-local
      user: user-local

users:
  - name: user-truckinit
    user:
      auth-provider:
        config:
          access-token: ya29<REDACTED>
          cmd-args: config config-helper --format=json
          cmd-path: /usr/lib/google-cloud-sdk/bin/gcloud
          expiry: "2020-06-16T13:52:11Z"
          expiry-key: '{.credential.token_expiry}'
          token-key: '{.credential.access_token}'
        name: gcp

  - name: user-local
    user:
      username: admin
      password: N1R<REDACTED>

current-context: context-truckinit
kind: Config
preferences: {}
```

10. Install a Pod Network
```bash
# Required for communication between pods.
# Cluster DNS (CoreDNS) will not startup before this network is installed
# Using Calico (recommended by Kubeadm)

kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

# Verify if installation is succesful by checking if CoreDNS has started and is running
kubectl get pods --all-namespaces
```

11. Enable scheduling on control-plane node (optional if deploying a multi-node cluster)
```bash
# Used to allow pods to be scheduled to current master/control-plane node
kubectl taint nodes --all node-role.kubernetes.io/master-
```

12. Install Helm package manager
```bash
wget https://get.helm.sh/helm-canary-linux-amd64.tar.gz
mkdir tmp-helm
mv helm-*.tar.gz tmp-helm/
cd tmp-helm
tar xvf helm-*.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/
cd ../
rm -rf tmp-helm
```

13. Allow communication in firewall for required ports
```bash
# Control-Plane Node(s)
# Port Range      Purpose                       Used By
# 6443            Kubernetes API Server         All
# 2379-2380       Etcd Server Client API        kube-apiserver, etcd
# 10250           Kubelet API                   Self, Control Plane
# 10251           Kube-Scheduler                Self (no need to expose)
# 10252           Kube-Controller-Manager       Self (no need to expose)

# Worker Node(s) (including control-plane node if untainted)
# Port Range      Purpose                       Used By
# 10250           Kubelet API                   Self, Control Plane
# 30000-32767     NodePort Services             All
```

14. Add nodes to cluster
```bash
# Follow step 1, 2, 3, 4, 10 (on new node)

# Get join command (on control-plane node)
kubeadm token create --print-join-command

# Join node to cluster using command above (on new node)
```

15. Install MetalLB
```bash
# MetalLB is used to create bare metal load-balancers

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

16. Install Nginx-ingress controller
```bash
kubectl create namespace ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx/
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx
```

17. Install Cert-Manager
```bash
# Used to generate letsencrypt certificates in the ingress

kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.1 \
  --set installCRDs=true
```

18. Install Loki
```bash
helm repo add loki https://grafana.github.io/loki/charts
helm repo update
helm upgrade --install loki loki/loki-stack
```

19. Install Litmus (optional, used for chaos testing)
```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v1.7.0.yaml

# Verify that the chaos operator is running
kubectl get pods -n litmus
```

20. Install Generic Chaos Experiment (optional, used for chaos testing)
```bash
kubectl apply -f https://hub.litmuschaos.io/api/chaos/1.7.0?file=charts/generic/experiments.yaml
```

## Truckin-IT System Deployment

1. Create Secrets Configurations
```bash
cd secrets

# Option 1: Setting up new secrets
cp .secrets-example.yml ./secrets.yml

# Option 2: Copying secrets
cp /tmp/secrets.yml ./
```

2. Deploy Secrets
```bash
kubectl apply -f secrets/secrets.yml
```
3. Deploy ConfigMaps
```bash
kubectl apply -f configmaps/
```

4. Create persistent volumes
```bash
kubectl apply -f volumes/postgres-persistent-volume-prod.yml
```

5. Create deployments, services, and persistent volume claims
```bash
kubectl apply -f deployments/
```

6. Deploy MetalLB Configmap
```bash
# Set node(s) IP addresses in configmap first

kubectl apply -f metallb/configmap.yml
```

7. Create cluster issuer
```bash
kubectl apply -f cert-manager/cluster-issuer.yml
```

8. Deploy Ingress
```bash
kubectl apply -f ingress/ingress.yml
```

9. Create Litmus Service Accounts (optional, used for chaos testing)
```bash
kubectl apply -f litmus/chaos-engine.yml
```

10. Create Litmus Chaos Engine (optional, used for chaos testing)
```bash
kubectl apply -f litmus/rbac.yml
```
