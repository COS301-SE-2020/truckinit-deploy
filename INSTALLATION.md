# Technical Installation Manual

## System Overview
The Truckin-IT system is composed of two primary systems, the API and the mobile apps. The API is intended to be hosted on a Kubernetes cluster managing Docker containers, with the API components built using the Spring framework (running Kotlin). The mobile apps all operate the same way: The apps are built using Dart and Flutter which handles its own compilation and packaging for deployment to an app store or repository.

## Cluster Deployment

### Overview
The main purpose of this section of the installation manual is to provide detailed instructions on how to setup a Kubernetes
  cluster which can then be used to host the **Truckin-IT** backend system.

> **Note**: The steps found in this section make a number of assumptions 
  (such as using a debian based OS for the cluster nodes) for the sake of simplicity. 
  This by no means implies that the system cannot be deployed in a different manner.

### Cluster Setup

#### Kubeadm
> **Note**: All commands should be run on the control-plane/master node unless otherwise stipulated

[Kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
  is used to create the Kubernetes (k8s) cluster which runs all the microservices used in
  the system.

1. Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

2. Enable bridged network traffic
```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

3. Install Docker
```bash
sudo apt install docker.io
sudo systemctl enable --now docker
```

4. Install required Kubeadm packages
```bash
sudo apt install -y apt-transport-https curl gnupg2
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

5. Restart Kubelet
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

6. Create cluster
```bash
kubeadm init
```

7. Copy kube config
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

8. Install a Pod Network
```bash
# Required for communication between pods.
# Cluster DNS (CoreDNS) will not startup before this network is installed
# Using Calico (recommended by Kubeadm)

kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

# Verify if installation is succesful by checking if CoreDNS has started and is running
kubectl get pods --all-namespaces
```

9. Enable scheduling on control-plane node (optional if deploying a multi-node cluster)
```bash
# Used to allow pods to be scheduled to current master/control-plane node
kubectl taint nodes --all node-role.kubernetes.io/master-
```

10. Allow communication in firewall for required ports
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

11. Add nodes to cluster
```bash
# Follow step 1, 2, 3, 4, 10 (on new node)

# Get join command (on control-plane node)
kubeadm token create --print-join-command

# Join node to cluster using command above (on new node)
```

12. Install Nginx-ingress controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/baremetal/deploy.yaml
```

### Truckin-IT System Deployment

1. Clone the deployment git repository
```bash
https://github.com/COS301-SE-2020/truckinit-deploy.git

cd truckinit-deploy
```

1. Create Secrets Configurations
```bash
cd secrets

# Option 1: Setting up new secrets
cp .secrets-example.yml ./secrets.yml

# Option 2: Copying secrets
cp /tmp/secrets.yml ./
 
cd ..
```

2. Deploy Secrets
```bash
kubectl apply -f secrets/secrets.yml
```

3. Create persistent volumes
```bash
kubectl apply -f volumes/persistent-volumes.yml
```

4. Create cluster issuer
```bash
kubectl apply -f cert-manager/cluster-issuer.yml
```

5. Create service account *(optional, used for GitLab integration)*
```bash
kubectl apply -f service-accounts/gitlab-admin-service-account.yml
```

6. Create persistent volumes
```bash
kubectl apply -f volumes/postgres-persistent-volume-prod.yml
```

7. Create deployments, services, and persistent volume claims
```bash
kubectl apply -f deployments/
```

## API Server deployment

### Overview
Once a cluster has been prepared, an image of the API server application can be deployed to it to begin serving content. A docker container registry (such as DockerHub or Gitlab container registry) is used to host the API image which is run on the Kubernetes cluster. When changes are made to the API and a new image is pushed it can then be deployed (updated) on the cluster.

## API Development Environment

The main purpose of this section of the installation manual is to provide detailed instructions on how to setup the backend API for development.

A docker image can be built for the backend API (which we make use of to deploy new versions of the API on the Kubernetes cluster).

> **Note**: The steps found in this section make a number of assumptions 
  (such as using a debian ) for the sake of simplicity. 
  This by no means implies that the system cannot be deployed in a different manner.

### Prerequisites

This project requires two dependencies, an authentication server as well as an SQL database. We have opted to use Keycloak
and PostgreSQL, but in theory it should be very simple to replace these with your own preferences.

1. Install Java [JDK](https://www.oracle.com/java/technologies/javase-downloads.html) version 1.8

2. Install [Gradle](https://gradle.org/install/) version 6.6

3. Install [Docker](https://docs.docker.com/get-docker/) version 18.10

4. Install [Docker Compose](https://docs.docker.com/compose/install/) version 1.26.2

3. Clone the git repository
```bash
https://github.com/COS301-SE-2020/truckinit-api.git

cd truckinit-api
```

### Getting Started

The following ports should be vacant:

* `8080`: for Tapi (Truckin-IT API)
* `5533`: for the PostgreSQL instance
* `5555`: for the Keycloak server

To create the development environment, run the following commands in the root directory:

1. `make dev-up`
2. `./gradlew bootRun`

`make dev-up` will create the PosgreSQL and Keycloak instances, whereas `./gradlew bootRun` will run Tapi.

#### Keycloak

In order to use the Keycloak Admin Console, navigate to `localhost:5555` and then click on "Administration Console". 
You will be prompted to enter a username and password:

* Username: `admin`
* Password: `admin`

Here you can configure Keycloak further. By default, we have created a realm named Truckinit with a four demo users.

#### PostgreSQL

The PostgreSQL instance has two users and databases namely, `tapi` and `keycloak`. Both databases share the password
`password`.

#### Further Configuration

All development configuration files are located in the `env/dev/` directory. Here you can edit the `docker-compose.yml`
file, database initialization scripts in `db_scripts/` as well as the Keycloak realm settings for the Truckinit realm in
the `truckinit-realm-export.json`.

#### Cleanup

In order to stop the PostgreSQL and Keycloak instances, run `make dev-down`. Furthermore, `make dev-clean` will remove
the docker volumes used by PostgreSQL.

## Mobile Development Environment

### Setup
_NB: A Google Developer account is required for map functionality (using their location API)._

#### Setting up Flutter

> Note: If you are not already familiar with Flutter basics, please take a look at their [starter guide](https://flutter.dev/docs/get-started/codelab)

1. Install [Flutter 1.17.3](https://flutter.dev/docs/development/tools/sdk/releases) for build and run ([guide](https://flutter.dev/docs/get-started/install))

2. If you want to run unit tests or isolate controller classes, install the [Dart SDK](https://dart.dev/get-dart)

3. 
    * Android: [Enable USB debugging](https://www.embarcadero.com/starthere/xe5/mobdevsetup/android/en/enabling_usb_debugging_on_an_android_device.html) on your device or [install an emulator](https://developers.arcgis.com/android/10-2/sample-code/emulator/)
    * iOS: You will need XCode and an Apple developer license to run [on a device](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/RunningonaDevice.html#//apple_ref/doc/uid/TP40010215-CH55-SW1) or [in the simulator](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/RunningintheSimulator.html#//apple_ref/doc/uid/TP40010215-CH54-SW1)

4. You can perform Flutter development with most popular IDEs. We like using [VSCode](https://code.visualstudio.com/) ([Flutter guide](https://flutter.dev/docs/get-started/test-drive?tab=vscode)), which is nothing like [Visual Studio IDE](https://devrant.com/rants/2066988/after-wasting-most-of-the-afternoon-trying-to-fix-visual-studio-i-can-honestly-s)

#### Setting up the Repository

1. Clone this repository (see Clone button above)
2. Download the Google Services configuration files for Android and iOS as needed\*
3. Copy the Android file to `android/app/`, and the iOS file to `ios/Runner`.

\* From the [Firebase Console](https://console.firebase.google.com/u/0/project/_/settings/general), select the target project and download `google_services.json` (Android) and `GoogleService-Info.plist` (iOS).

#### Application Package Deployment
_Note: These apps are currently only intended for use on an Android device. iOS deployment would follow similar steps but will not be explicitly detailed here_

1. Ensure all configurations have been fulfilled in the `common/ui/lib/secrets.dart` and `android/app/google_services.json` files.
2. Build all generated classes using `common/core/tools/build.bat` or `.../build.sh` based on your current operating system.
1. Produce the multi-target Android app installer bundle* using:
```
flutter build appbundle
```

\* To produce a .apk file for targeted installation to a specific device, use:
```
flutter build apk
```
That .apk can then be used in the below steps for installation to Android via a development computer.

## Mobile Application

### Installation-Package Export
It is assumed that the mobile apps have been compiled using [the above configuration](#mobile-development-environment) and either uploaded to a platform-specific "app store" (e.g. Google Play Store) or prepared on a computer with debug-bridging capabilities that can manually deploy to a device.

### Package Installation
Note: Currently the iOS operating system does not allow manual app deployment to a device outside of live testing conditions with a connected computer.
- iOS:
  -  Should the (Customer, Manager or Trucker) app be available on the iOS App Store, it can be searched for by name.
  -  Upon finding the correct app, click the "install" button to install onto the current device.
-  Android (via app store):
   -  Search for the (Customer, Manager or Trucker) app on the Google Play store, or use one of the following links:
      - [Customer App](https://play.google.com/store/apps/details?id=com.truckin-it.fleet-manager) | [Manager App](https://play.google.com/store/apps/details?id=com.truckin-it.customer) | [Trucker App](https://play.google.com/store/apps/details?id=com.truckin-it.trucker)
   -  Upon finding the correct app, click the "install" button to install onto the current device.
-  Android (via development computer):
   -  Open the target app's source directory in a terminal.
   -  To install directly to the mobile device:
      -  Connect the intended device to the computer via USB
      -  Ensure USB-debugging is enabled in the device's settings
      -  In a terminal, run `flutter install` (which will automatically target your device)
   -  To install via an exported app package:
      -  Export the desired app using `flutter build apk`, producing `customer.apk`, `manager.apk`, or `trucker.apk` (`customer.apk` will be used for this example)
      -  Locate the exported package in ``
      -  Copy the file to your device by your preferred means
      -  Open a file explorer on the device and find the `customer.apk` file
      -  Launch the file to install the app
         -  If the device does not allow the installation, you will have to enable app installation from non-secure sources in the device settings.
