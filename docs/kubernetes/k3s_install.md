---
title: "k3s"
date: 2021-07-23T00:00:00+00:00
draft: true
---

# Préambule

Pour pouvoir créer la solution, un certain nombre d'outils sera nécessaire.
Le but de ce projet est d'installer la stack par le réseau.

# Création de l'environement de boot PXE

Installation des packages nécessaire au déploiment par le réseau.

Il est nécessaire de récupérer les images suivantes :

- [ISO MicroOS](https://download.opensuse.org/ports/aarch64/tumbleweed/iso/openSUSE-MicroOS-DVD-aarch64-Current.iso)
- [RAW MicroOS Raspberry Pi](https://download.opensuse.org/ports/aarch64/tumbleweed/appliances/openSUSE-MicroOS.aarch64-ContainerHost-RaspberryPi.raw.xz)

## serveur DNSMASQ

Le serveur DNSMASQ embarque un serveur DNS ainsi qu'un server TFTP

Installation du package

```
# zypper install dnsmasq
```

Configurer DNSMASQ avec les fichiers suivants :

- [/etc/dnsmasq.conf](k3s/dnsmasq/dnsmasq.conf)
- [/etc/dnsmasq-dns.conf](k3s/dnsmasq/dnsmasq-dns.conf)
- [/etc/dnsmasq-hosts.conf](k3s/dnsmasq/dnsmasq-hosts.conf)

Afin d'initialiser l'environnment de boot par le réseau, il faut préparer les sources dont nous allons avoir besoin :

```
# mkdir -p /m/{iso,raw_rootfs,raw_bootfs}
# losetup -f openSUSE-MicroOS.aarch64-16.0.0-ContainerHost-RaspberryPi-Snapshot20210710.raw
# kpartx -a -v -f /dev/loop0
# mount openSUSE-MicroOS-DVD-aarch64-Current.iso /m/iso
# mount /dev/mapper/loop0p1 /m/raw_bootfs
# mount /dev/mapper/loop0p2 /m/raw_rootfs
```

Pour la configuration et le deploiment du server TFTP, il est nécessaire de copier les fichiers suivants au niveau du serveur :

```
cp -r /m/raw_rootfs/boot/vc/*         /srv/tftpboot
cp -r /m/raw_rootfs/boot              /srv/tftpboot
cp -r /m/iso/boot/aarch64             /srv/tftpboot/boot
cp -r /m/iso/EFI                      /srv/tftpboot
mkdir -p /srv/tftpboot/dtb/broadcom/
cp -r /srv/tftpboot/bcm*.dtb /srv/tftpboot/dtb/broadcom
find /srv/tftpboot -type d -exec chmod 755 {} +
find /srv/tftpboot -type f -exec chmod 644 {} +
```

Il faut ajouter le fichier [grub.cfg](k3s/grub.cfg) dans /srv/tftpboot/EFI/BOOT/grub.cfg

Une fois fait, démarrer le service

```
# systemctl enable --now dnsmasq
```

## serveur APACHE

Installation du serveur web apache2

```
# zypper install apache2
```

```
mkdir -p /srv/www/htdocs/{MicroOs,profile}
```

Copier le contenu de l'[ISO](https://download.opensuse.org/ports/aarch64/tumbleweed/iso/openSUSE-MicroOS-DVD-aarch64-Current.iso) dans /srv/www/htdocs/MicroOs

Et mettre le fichier [autoinst.xml](k3s/profile/autoinst.xml) dans /srv/www/htdocs/profile.

Puis démarrer le service

```
# systemctl enable --now apache2
```

La préparation du système d'installation par le réseau est opérationnel.
/!\ bien mettre les @MAC des rpi4 dans la conf DNSMASQ

check autoinst.xml configuration

```
# jing /usr/share/YaST2/schema/autoyast/rng/profile.rng /srv/www/htdocs/profile/autoinst.xml
```

# Deploiement manuel


sur kmaster01

```
INSTALL_K3S_EXEC="-v=0 \
                  --node-taint CriticalAddonsOnly=true:NoExecute \
                  --no-deploy local-storage \
                  --cluster-domain=mon-lab.org \
                  --tls-san=192.168.4.160 \
                  --cluster-init" \
/usr/bin/k3s-install
```

sur kmaster02

```
INSTALL_K3S_EXEC="-v=0 \
                  --node-taint CriticalAddonsOnly=true:NoExecute \
                  --no-deploy local-storage \
                  --bind-address=192.168.4.162 \
                  --advertise-address=192.168.4.162 \
                  --node-ip=192.168.4.162 \
                  --node-external-ip=192.168.4.159 \
                  --cluster-domain=k3s.mon-lab.org \
                  --tls-san=192.168.4.159 \
                  --tls-san=192.168.4.160 \
                  --server=https://192.168.4.161:6443 \
                  --token=MON_TOKEN_A_PENSER_A_METTRE_A_JOUR" \
/usr/bin/k3s-install
```

sur kmaster03

```
INSTALL_K3S_EXEC="-v=0 \
                  --node-taint CriticalAddonsOnly=true:NoExecute \
                  --no-deploy local-storage \
                  --bind-address=192.168.4.163 \
                  --advertise-address=192.168.4.163 \
                  --node-ip=192.168.4.163 \
                  --node-external-ip=192.168.4.159 \
                  --cluster-domain=k3s.mon-lab.org \
                  --tls-san=192.168.4.159 \
                  --tls-san=192.168.4.160 \
                  --server=https://192.168.4.161:6443 \
                  --token=MON_TOKEN_A_PENSER_A_METTRE_A_JOUR" \
/usr/bin/k3s-install
```

sur kworker01

```
K3S_URL="https://192.168.4.160:6443" \
K3S_TOKEN="MON_TOKEN_A_PENSER_A_METTRE_A_JOUR" \
INSTALL_K3S_EXEC="-v=0 --node-ip=192.168.4.164" \
/usr/bin/k3s-install
```

sur kworker02

```
K3S_URL="https://192.168.4.160:6443" \
K3S_TOKEN="MON_TOKEN_A_PENSER_A_METTRE_A_JOUR" \
INSTALL_K3S_EXEC="-v=0 --node-ip=192.168.4.165" \
/usr/bin/k3s-install
```





# TODO...

Juste pour avoir une idée des commandes k3s-install :

```
# Usage:
#   curl ... | ENV_VAR=... sh -
#       or
#   ENV_VAR=... ./install.sh
#
# Example:
#   Installing a server without traefik:
#     curl ... | INSTALL_K3S_EXEC="--disable=traefik" sh -
#   Installing an agent to point at a server:
#     curl ... | K3S_TOKEN=xxx K3S_URL=https://server-url:6443 sh -
#
# Environment variables:
#   - K3S_*
#     Environment variables which begin with K3S_ will be preserved for the
#     systemd service to use. Setting K3S_URL without explicitly setting
#     a systemd exec command will default the command to "agent", and we
#     enforce that K3S_TOKEN or K3S_CLUSTER_SECRET is also set.
#
#   - INSTALL_K3S_SKIP_DOWNLOAD
#     If set to true will not download k3s hash or binary.
#
#   - INSTALL_K3S_FORCE_RESTART
#     If set to true will always restart the K3s service
#
#   - INSTALL_K3S_SYMLINK
#     If set to 'skip' will not create symlinks, 'force' will overwrite,
#     default will symlink if command does not exist in path.
#
#   - INSTALL_K3S_SKIP_ENABLE
#     If set to true will not enable or start k3s service.
#
#   - INSTALL_K3S_SKIP_START
#     If set to true will not start k3s service.
#
#   - INSTALL_K3S_VERSION
#     Version of k3s to download from github. Will attempt to download from the
#     stable channel if not specified.
#
#   - INSTALL_K3S_COMMIT
#     Commit of k3s to download from temporary cloud storage.
#     * (for developer & QA use)
#
#   - INSTALL_K3S_BIN_DIR
#     Directory to install k3s binary, links, and uninstall script to, or use
#     /usr/local/bin as the default
#
#   - INSTALL_K3S_BIN_DIR_READ_ONLY
#     If set to true will not write files to INSTALL_K3S_BIN_DIR, forces
#     setting INSTALL_K3S_SKIP_DOWNLOAD=true
#
#   - INSTALL_K3S_SYSTEMD_DIR
#     Directory to install systemd service and environment files to, or use
#     /etc/systemd/system as the default
#
#   - INSTALL_K3S_EXEC or script arguments
#     Command with flags to use for launching k3s in the systemd service, if
#     the command is not specified will default to "agent" if K3S_URL is set
#     or "server" if not. The final systemd command resolves to a combination
#     of EXEC and script args ($@).
#
#     The following commands result in the same behavior:
#       curl ... | INSTALL_K3S_EXEC="--disable=traefik" sh -s -
#       curl ... | INSTALL_K3S_EXEC="server --disable=traefik" sh -s -
#       curl ... | INSTALL_K3S_EXEC="server" sh -s - --disable=traefik
#       curl ... | sh -s - server --disable=traefik
#       curl ... | sh -s - --disable=traefik
#
#   - INSTALL_K3S_NAME
#     Name of systemd service to create, will default from the k3s exec command
#     if not specified. If specified the name will be prefixed with 'k3s-'.
#
#   - INSTALL_K3S_TYPE
#     Type of systemd service to create, will default from the k3s exec command
#     if not specified.
#
#   - INSTALL_K3S_SELINUX_WARN
#     If set to true will continue if k3s-selinux policy is not found.
#
#   - INSTALL_K3S_SKIP_SELINUX_RPM
#     If set to true will skip automatic installation of the k3s RPM.
#
#   - INSTALL_K3S_CHANNEL_URL
#     Channel URL for fetching k3s download URL.
#     Defaults to 'https://update.k3s.io/v1-release/channels'.
#
#   - INSTALL_K3S_CHANNEL
#     Channel to use for fetching k3s download URL.
#     Defaults to 'stable'.
```

Juste pour avoir une idée des commandes k3s server :

```
k3s server --help
NAME:
   k3s server - Run management server

USAGE:
   k3s server [OPTIONS]

OPTIONS:
   --config FILE, -c FILE                     (config) Load configuration from FILE (default: "/etc/rancher/k3s/config.yaml") [$K3S_CONFIG_FILE]
   --debug                                    (logging) Turn on debug logs [$K3S_DEBUG]
   -v value                                   (logging) Number for the log level verbosity (default: 0)
   --vmodule value                            (logging) Comma-separated list of pattern=N settings for file-filtered logging
   --log value, -l value                      (logging) Log to file
   --alsologtostderr                          (logging) Log to standard error as well as file (if set)
   --bind-address value                       (listener) k3s bind address (default: 0.0.0.0)
   --https-listen-port value                  (listener) HTTPS listen port (default: 6443)
   --advertise-address value                  (listener) IPv4 address that apiserver uses to advertise to members of the cluster (default: node-external-ip/node-ip)
   --advertise-port value                     (listener) Port that apiserver uses to advertise to members of the cluster (default: listen-port) (default: 0)
   --tls-san value                            (listener) Add additional hostnames or IPv4/IPv6 addresses as Subject Alternative Names on the server TLS cert
   --data-dir value, -d value                 (data) Folder to hold state default /var/lib/rancher/k3s or ${HOME}/.rancher/k3s if not root
   --cluster-cidr value                       (networking) IPv4/IPv6 network CIDRs to use for pod IPs (default: 10.42.0.0/16)
   --service-cidr value                       (networking) IPv4/IPv6 network CIDRs to use for service IPs (default: 10.43.0.0/16)
   --service-node-port-range value            (networking) Port range to reserve for services with NodePort visibility (default: "30000-32767")
   --cluster-dns value                        (networking) IPv4 Cluster IP for coredns service. Should be in your service-cidr range (default: 10.43.0.10)
   --cluster-domain value                     (networking) Cluster Domain (default: "cluster.local")
   --flannel-backend value                    (networking) One of 'none', 'vxlan', 'ipsec', 'host-gw', or 'wireguard' (default: "vxlan")
   --token value, -t value                    (cluster) Shared secret used to join a server or agent to a cluster [$K3S_TOKEN]
   --token-file value                         (cluster) File containing the cluster-secret/token [$K3S_TOKEN_FILE]
   --write-kubeconfig value, -o value         (client) Write kubeconfig for admin client to this file [$K3S_KUBECONFIG_OUTPUT]
   --write-kubeconfig-mode value              (client) Write kubeconfig with this mode [$K3S_KUBECONFIG_MODE]
   --kube-apiserver-arg value                 (flags) Customized flag for kube-apiserver process
   --kube-scheduler-arg value                 (flags) Customized flag for kube-scheduler process
   --kube-controller-manager-arg value        (flags) Customized flag for kube-controller-manager process
   --kube-cloud-controller-manager-arg value  (flags) Customized flag for kube-cloud-controller-manager process
   --datastore-endpoint value                 (db) Specify etcd, Mysql, Postgres, or Sqlite (default) data source name [$K3S_DATASTORE_ENDPOINT]
   --datastore-cafile value                   (db) TLS Certificate Authority file used to secure datastore backend communication [$K3S_DATASTORE_CAFILE]
   --datastore-certfile value                 (db) TLS certification file used to secure datastore backend communication [$K3S_DATASTORE_CERTFILE]
   --datastore-keyfile value                  (db) TLS key file used to secure datastore backend communication [$K3S_DATASTORE_KEYFILE]
   --etcd-expose-metrics                      (db) Expose etcd metrics to client interface. (Default false)
   --etcd-disable-snapshots                   (db) Disable automatic etcd snapshots
   --etcd-snapshot-name value                 (db) Set the base name of etcd snapshots. Default: etcd-snapshot-<unix-timestamp> (default: "etcd-snapshot")
   --etcd-snapshot-schedule-cron value        (db) Snapshot interval time in cron spec. eg. every 5 hours '* */5 * * *' (default: "0 */12 * * *")
   --etcd-snapshot-retention value            (db) Number of snapshots to retain (default: 5)
   --etcd-snapshot-dir value                  (db) Directory to save db snapshots. (Default location: ${data-dir}/db/snapshots)
   --etcd-s3                                  (db) Enable backup to S3
   --etcd-s3-endpoint value                   (db) S3 endpoint url (default: "s3.amazonaws.com")
   --etcd-s3-endpoint-ca value                (db) S3 custom CA cert to connect to S3 endpoint
   --etcd-s3-skip-ssl-verify                  (db) Disables S3 SSL certificate validation
   --etcd-s3-access-key value                 (db) S3 access key [$AWS_ACCESS_KEY_ID]
   --etcd-s3-secret-key value                 (db) S3 secret key [$AWS_SECRET_ACCESS_KEY]
   --etcd-s3-bucket value                     (db) S3 bucket name
   --etcd-s3-region value                     (db) S3 region / bucket location (optional) (default: "us-east-1")
   --etcd-s3-folder value                     (db) S3 folder
   --default-local-storage-path value         (storage) Default local storage path for local provisioner storage class
   --disable value                            (components) Do not deploy packaged components and delete any deployed components (valid items: coredns, servicelb, traefik, local-storage, metrics-server)
   --disable-scheduler                        (components) Disable Kubernetes default scheduler
   --disable-cloud-controller                 (components) Disable k3s default cloud controller manager
   --disable-kube-proxy                       (components) Disable running kube-proxy
   --disable-network-policy                   (components) Disable k3s default network policy controller
   --node-name value                          (agent/node) Node name [$K3S_NODE_NAME]
   --with-node-id                             (agent/node) Append id to node name
   --node-label value                         (agent/node) Registering and starting kubelet with set of labels
   --node-taint value                         (agent/node) Registering kubelet with set of taints
   --image-credential-provider-bin-dir value  (agent/node) The path to the directory where credential provider plugin binaries are located (default: "/var/lib/rancher/credentialprovider/bin")
   --image-credential-provider-config value   (agent/node) The path to the credential provider plugin config file (default: "/var/lib/rancher/credentialprovider/config.yaml")
   --docker                                   (agent/runtime) Use docker instead of containerd
   --container-runtime-endpoint value         (agent/runtime) Disable embedded containerd and use alternative CRI implementation
   --pause-image value                        (agent/runtime) Customized pause image for containerd or docker sandbox (default: "rancher/pause:3.1")
   --snapshotter value                        (agent/runtime) Override default containerd snapshotter (default: "overlayfs")
   --private-registry value                   (agent/runtime) Private registry configuration file (default: "/etc/rancher/k3s/registries.yaml")
   --node-ip value, -i value                  (agent/networking) IPv4/IPv6 addresses to advertise for node
   --node-external-ip value                   (agent/networking) IPv4/IPv6 external IP addresses to advertise for node
   --resolv-conf value                        (agent/networking) Kubelet resolv.conf file [$K3S_RESOLV_CONF]
   --flannel-iface value                      (agent/networking) Override default flannel interface
   --flannel-conf value                       (agent/networking) Override default flannel config file
   --kubelet-arg value                        (agent/flags) Customized flag for kubelet process
   --kube-proxy-arg value                     (agent/flags) Customized flag for kube-proxy process
   --protect-kernel-defaults                  (agent/node) Kernel tuning behavior. If set, error if kernel tunables are different than kubelet defaults.
   --rootless                                 (experimental) Run rootless
   --agent-token value                        (experimental/cluster) Shared secret used to join agents to the cluster, but not servers [$K3S_AGENT_TOKEN]
   --agent-token-file value                   (experimental/cluster) File containing the agent secret [$K3S_AGENT_TOKEN_FILE]
   --server value, -s value                   (experimental/cluster) Server to connect to, used to join a cluster [$K3S_URL]
   --cluster-init                             (experimental/cluster) Initialize a new cluster using embedded Etcd [$K3S_CLUSTER_INIT]
   --cluster-reset                            (experimental/cluster) Forget all peers and become sole member of a new cluster [$K3S_CLUSTER_RESET]
   --cluster-reset-restore-path value         (db) Path to snapshot file to be restored
   --secrets-encryption                       (experimental) Enable Secret encryption at rest
   --system-default-registry value            (image) Private registry to be used for all system images [$K3S_SYSTEM_DEFAULT_REGISTRY]
   --selinux                                  (agent/node) Enable SELinux in containerd [$K3S_SELINUX]
   --lb-server-port value                     (agent/node) Local port for supervisor client load-balancer. If the supervisor and apiserver are not colocated an additional port 1 less than this port will also be used for the apiserver client load-balancer. (default: 6444) [$K3S_LB_SERVER_PORT]
   --no-flannel                               (deprecated) use --flannel-backend=none
   --no-deploy value                          (deprecated) Do not deploy packaged components (valid items: coredns, servicelb, traefik, local-storage, metrics-server)
   --cluster-secret value                     (deprecated) use --token [$K3S_CLUSTER_SECRET]
```


```
   k3s agent - Run node agent
NAME:
USAGE:
   k3s agent [OPTIONS]
OPTIONS:
   --config FILE, -c FILE                     (config) Load configuration from FILE (default: "/etc/rancher/k3s/config.yaml") [$K3S_CONFIG_FILE]
   --debug                                    (logging) Turn on debug logs [$K3S_DEBUG]
   -v value                                   (logging) Number for the log level verbosity (default: 0)
   --vmodule value                            (logging) Comma-separated list of pattern=N settings for file-filtered logging
   --log value, -l value                      (logging) Log to file
   --alsologtostderr                          (logging) Log to standard error as well as file (if set)
   --token value, -t value                    (cluster) Token to use for authentication [$K3S_TOKEN]
   --token-file value                         (cluster) Token file to use for authentication [$K3S_TOKEN_FILE]
   --server value, -s value                   (cluster) Server to connect to [$K3S_URL]
   --data-dir value, -d value                 (agent/data) Folder to hold state (default: "/var/lib/rancher/k3s")
   --node-name value                          (agent/node) Node name [$K3S_NODE_NAME]
   --with-node-id                             (agent/node) Append id to node name
   --node-label value                         (agent/node) Registering and starting kubelet with set of labels
   --node-taint value                         (agent/node) Registering kubelet with set of taints
   --image-credential-provider-bin-dir value  (agent/node) The path to the directory where credential provider plugin binaries are located (default: "/var/lib/rancher/credentialprovider/bin")
   --image-credential-provider-config value   (agent/node) The path to the credential provider plugin config file (default: "/var/lib/rancher/credentialprovider/config.yaml")
   --docker                                   (agent/runtime) Use docker instead of containerd
   --container-runtime-endpoint value         (agent/runtime) Disable embedded containerd and use alternative CRI implementation
   --pause-image value                        (agent/runtime) Customized pause image for containerd or docker sandbox (default: "rancher/pause:3.1")
   --snapshotter value                        (agent/runtime) Override default containerd snapshotter (default: "overlayfs")
   --private-registry value                   (agent/runtime) Private registry configuration file (default: "/etc/rancher/k3s/registries.yaml")
   --node-ip value, -i value                  (agent/networking) IPv4/IPv6 addresses to advertise for node
   --node-external-ip value                   (agent/networking) IPv4/IPv6 external IP addresses to advertise for node
   --resolv-conf value                        (agent/networking) Kubelet resolv.conf file [$K3S_RESOLV_CONF]
   --flannel-iface value                      (agent/networking) Override default flannel interface
   --flannel-conf value                       (agent/networking) Override default flannel config file
   --kubelet-arg value                        (agent/flags) Customized flag for kubelet process
   --kube-proxy-arg value                     (agent/flags) Customized flag for kube-proxy process
   --protect-kernel-defaults                  (agent/node) Kernel tuning behavior. If set, error if kernel tunables are different than kubelet defaults.
   --rootless                                 (experimental) Run rootless
   --selinux                                  (agent/node) Enable SELinux in containerd [$K3S_SELINUX]
   --lb-server-port value                     (agent/node) Local port for supervisor client load-balancer. If the supervisor and apiserver are not colocated an additional port 1 less than this port will also be used for the apiserver client load-balancer. (default: 6444) [$K3S_LB_SERVER_PORT]
   --no-flannel                               (deprecated) use --flannel-backend=none
   --cluster-secret value                     (deprecated) use --token [$K3S_CLUSTER_SECRET]
```


On first master :

```
k3s server --cluster-init --bind-address 192.168.4.161:6443 --cluster-domain mon-lab.org --selinux
```

On others masters :

```
k3s server --token K10dfdb6d2fa3f0cf8cd0632423c593f8694cca7e12a6c0a43a9ed3d626dceabc3a::server:546f7db44b56afc65c8f305f87f64618 --server https://192.168.4.160:6443
```

k3s server --server https://192.168.4.160:6443 --token K103f037da7534c8cb75ac5e8281cc39f179340cc2ea3c1e52ddc9aa0b12b2ec8fe::server:6592241765825cbd5b3860c4a79060c3


k3s server --cluster-init --bind-address 192.168.4.160 --cluster-domain mon-lab.org --flannel-backend=none



systemctl set-property user.slice CPUAccounting=yes
systemctl set-property user.slice MemoryAccounting=yes





k3s server --token K10aebf895f0ecf67e93d40e9a3ca6667f8456458122ab648f42a5f1e620ad3e134::server:1633538e00c4e6439ac9be3abfa5639c --bind-address 192.168.4.162 --server https://192.168.4.161:6443

# Install Dashboard

```
# helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
```

# Install Calico

kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

kubectl apply -f https://raw.githubusercontent.com/hashicorp/learn-terraform-provision-eks-cluster/master/kubernetes-dashboard-admin.rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/chopperchan/k8s-dashboard/master/kubernetes-dashboard-admin.rbac.yaml



les variables d'env /etc/systemd/system/k3s.service.env

IP local





Sur kmaster01 en init

INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--cluster-init --bind-address=192.168.4.161 --cluster-domain=mon-lab.org --tls-san=k3sapi.mon-lab.org" k3s-install

systemctl stop k3s

enlever "--cluster-init " du fichier /etc/systemd/system/k3s.service

systemctl daemon-reload

systemctl start k3s

cat /var/lib/rancher/k3s/server/token


join des masters

INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--bind-address=192.168.4.162 --server https://192.168.4.161:6443 --token K1028ec6cbdf5ea54f9442572f84d0c91811e13399e22449d5bf6236eb91c259fae::server:b500fdcd9a985f1b53567fc62530a039" k3s-install

INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--bind-address=192.168.4.163 --server https://192.168.4.161:6443 --token K1028ec6cbdf5ea54f9442572f84d0c91811e13399e22449d5bf6236eb91c259fae::server:b500fdcd9a985f1b53567fc62530a039" k3s-install

join des workers

INSTALL_K3S_SELINUX_WARN=true K3S_URL=https://k3sapi.mon-lab.org:6443 K3S_TOKEN="K1028ec6cbdf5ea54f9442572f84d0c91811e13399e22449d5bf6236eb91c259fae::server:b500fdcd9a985f1b53567fc62530a039" k3s-install



--token K1028ec6cbdf5ea54f9442572f84d0c91811e13399e22449d5bf6236eb91c259fae::server:b500fdcd9a985f1b53567fc62530a039 --bind-address 192.168.4.162 --server https://192.168.4.161:6443


debug install

debut install nouvelle configuration
------------------------------------

1:03

fin install nouvelle configuration
----------------------------------

1:32


29 minutes



debug install

debut install nouvelle configuration
------------------------------------

11:23

fin install nouvelle configuration
----------------------------------

11:55




kmaster01:~ # INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--bind-address=192.168.4.161 --advertise-address=192.168.4.160 --write-kubeconfig-mode=0644 --cluster-domain=mon-lab.org --node-external-ip=192.168.4.160  --tls-san=192.168.4.160 --cluster-init" k3s-install
[INFO]  Finding release for channel stable
[INFO]  Using v1.21.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s


kmaster02:~ # INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--bind-address=192.168.4.162 --advertise-address=192.168.4.160  --node-external-ip=192.168.4.160 --server=https://k3sapi.mon-lab.org:6443 --token=${TOKEN}" k3s-install
[INFO]  Finding release for channel stable
[INFO]  Using v1.21.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

kmaster03:~ # INSTALL_K3S_SELINUX_WARN=true INSTALL_K3S_EXEC="--bind-address=192.168.4.163 --advertise-address=192.168.4.160  --node-external-ip=192.168.4.160 --server=https://k3sapi.mon-lab.org:6443 --token=${TOKEN}" k3s-install
[INFO]  Finding release for channel stable
[INFO]  Using v1.21.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

kworker01:~ # INSTALL_K3S_SELINUX_WARN=true K3S_URL="https://k3sapi.mon-lab.org:6443" K3S_TOKEN="${TOKEN}" k3s-install
[INFO]  Finding release for channel stable
[INFO]  Using v1.21.3+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.21.3+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
































TOKEN="K10882efbda353594001d9f2a6d44e1ba6fc9ac29413c21cbe349f677480c673b79::server:9f6a08003a67d6564f1de1a7aff7867d"

INSTALL_K3S_SELINUX_WARN=true \
INSTALL_K3S_EXEC="--bind-address=192.168.4.161 \
                  --write-kubeconfig-mode=0644 \
                  --advertise-address=192.168.4.160 \
                  --cluster-domain=mon-lab.org \
                  --node-external-ip=192.168.4.160 \
                  --tls-san=192.168.4.160 \
                  --cluster-init" k3s-install

INSTALL_K3S_SELINUX_WARN=true \
INSTALL_K3S_EXEC="--bind-address=192.168.4.162 \
                  --write-kubeconfig-mode=0644 \
                  --advertise-address=192.168.4.160  \
                  --node-external-ip=192.168.4.160 \
                  --server=https://k3sapi.mon-lab.org:6443 \
                  --token=${TOKEN}" k3s-install

INSTALL_K3S_SELINUX_WARN=true \
INSTALL_K3S_EXEC="--bind-address=192.168.4.163 \
                  --write-kubeconfig-mode=0644 \
                  --advertise-address=192.168.4.160 \
                  --node-external-ip=192.168.4.160 \
                  --server=https://k3sapi.mon-lab.org:6443 \
                  --token=${TOKEN}" k3s-install

INSTALL_K3S_SELINUX_WARN=true \
K3S_URL="https://k3sapi.mon-lab.org:6443" \
K3S_TOKEN="${TOKEN}" k3s-install



# génération SSL version 1

## Generate the Certificate Files

```
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=[master-ip-address]" -days [number] -out ca.crt
openssl genrsa -out server.key 2048
```

## Create the Certificate Configuration File

```
create csr.conf
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = [country]
ST = [state]
L = [city]
O = [company]
OU = [organization-unit]
CN = [common-name]

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = [MASTER_IP]
IP.2 = [MASTER_CLUSTER_IP]

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

```
openssl req -new -key server.key -out server.csr -config csr.conf
```

## Generate the Certificate

```
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000 -extensions v3_ext -extfile csr.conf
```

```
openssl x509 -noout -text -in ./server.crt
```

https://blog.wescale.fr/2018/12/13/tutoriel-ajouter-un-user-dans-kubernetes-avec-cfssl/

# génération SSL version 2

## Installation binaire

Installer cfssl (version 1.6.0 à date)

```
https://github.com/cloudflare/cfssl/releases/tag/v1.6.0
cfssl
cfssljson
```

## Create a Certificate Authority

ca.json

```
{
  "CN": "Example Company Root CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
  {
    "C": "US",
    "L": "New York",
    "ST": "New York",
    "O": "Example Company",
    "OU": "Example Company Root CA"
  }
 ]
}
```

Générer le CA avec la commande suivante :

```
cfssl gencert -initca ca.json | cfssljson -bare ca
```

## Create the Configuration File

cfssl.json

```
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "intermediate_ca": {
        "usages": [
            "signing",
            "digital signature",
            "key encipherment",
            "cert sign",
            "crl sign",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h",
        "ca_constraint": {
            "is_ca": true,
            "max_path_len": 0,
            "max_path_len_zero": true
        }
      },
      "peer": {
        "usages": [
            "signing",
            "digital signature",
            "key encipherment",
            "client auth",
            "server auth"
        ],
        "expiry": "8760h"
      },
      "server": {
        "usages": [
          "signing",
          "digital signing",
          "key encipherment",
          "server auth"
        ],
        "expiry": "8760h"
      },
      "client": {
        "usages": [
          "signing",
          "digital signature",
          "key encipherment",
          "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
```

## Create an Intermediate Certificate Authority

intermediate-ca.json

```
{
  "CN": "K3S mon-lab.org Intermediate CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "FR",
      "L": "La Madeleine",
      "ST": "La Madeleine",
      "O": "Mon-Lab Company",
      "OU": "Mon-Lab Company Intermediate CA"
    }
  ],
  "ca": {
    "expiry": "42720h"
  }
}
```

## Sign the Certificate

```
cfssl gencert -initca intermediate-ca.json | cfssljson -bare intermediate_ca
cfssl sign -ca ca.pem -ca-key ca-key.pem -config cfssl.json -profile intermediate_ca intermediate_ca.csr | cfssljson -bare intermediate_ca
```

## Generate Host Certificates

k3sapi.json

```
{
  "CN": "k3sapi.mon-lab.org",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "FR",
      "L": "La Madeleine",
      "O": "Mon-lab",
      "OU": "Mon-lab Company Intermediate CA",
      "ST": "La Madeleine"
    }

  ],
  "hosts": [
    "k3sapi.mon-lab.org",
    "kmaster01.mon-lab.org",
    "kmaster02.mon-lab.org",
    "kmaster03.mon-lab.org",
    "localhost"
  ]
}
```

```
cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=peer   k3sapi.json | cfssljson -bare k3sapi-peer
cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=server k3sapi.json | cfssljson -bare k3sapi-server
cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=client k3sapi.json | cfssljson -bare k3sapi-client
```


```
mfatrez@Mac-mini-de-Matthieu certs % cfssl gencert -initca ca.json | cfssljson -bare ca
2021/07/27 09:49:29 [INFO] generating a new CA key and certificate from CSR
2021/07/27 09:49:29 [INFO] generate received request
2021/07/27 09:49:29 [INFO] received CSR
2021/07/27 09:49:29 [INFO] generating key: rsa-2048
2021/07/27 09:49:29 [INFO] encoded CSR
2021/07/27 09:49:29 [INFO] signed certificate with serial number 679261756739298370276388692010628555600110216694
mfatrez@Mac-mini-de-Matthieu certs % cfssl gencert -initca intermediate-ca.json | cfssljson -bare intermediate_ca
2021/07/27 09:49:35 [INFO] generating a new CA key and certificate from CSR
2021/07/27 09:49:35 [INFO] generate received request
2021/07/27 09:49:35 [INFO] received CSR
2021/07/27 09:49:35 [INFO] generating key: rsa-2048
2021/07/27 09:49:35 [INFO] encoded CSR
2021/07/27 09:49:35 [INFO] signed certificate with serial number 31944400668280568493406719187369439594056418004
mfatrez@Mac-mini-de-Matthieu certs % cfssl sign -ca ca.pem -ca-key ca-key.pem -config cfssl.json -profile intermediate_ca intermediate_ca.csr | cfssljson -bare intermediate_ca
2021/07/27 09:49:39 [INFO] signed certificate with serial number 31501736480666624564184193192952772846945421046
mfatrez@Mac-mini-de-Matthieu certs % cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=peer   k3sapi.json | cfssljson -bare k3sapi-peer
2021/07/27 09:49:45 [INFO] generate received request
2021/07/27 09:49:45 [INFO] received CSR
2021/07/27 09:49:45 [INFO] generating key: rsa-2048
2021/07/27 09:49:45 [INFO] encoded CSR
2021/07/27 09:49:45 [INFO] signed certificate with serial number 554178241005183240979256329865754249256241679080
mfatrez@Mac-mini-de-Matthieu certs % cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=server k3sapi.json | cfssljson -bare k3sapi-server
2021/07/27 09:49:49 [INFO] generate received request
2021/07/27 09:49:49 [INFO] received CSR
2021/07/27 09:49:49 [INFO] generating key: rsa-2048
2021/07/27 09:49:50 [INFO] encoded CSR
2021/07/27 09:49:50 [INFO] signed certificate with serial number 396997979496631030568187188758113509430781327471
mfatrez@Mac-mini-de-Matthieu certs % cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config cfssl.json -profile=client k3sapi.json | cfssljson -bare k3sapi-client
2021/07/27 09:49:53 [INFO] generate received request
2021/07/27 09:49:53 [INFO] received CSR
2021/07/27 09:49:53 [INFO] generating key: rsa-2048
2021/07/27 09:49:54 [INFO] encoded CSR
2021/07/27 09:49:54 [INFO] signed certificate with serial number 491534613748577578920868660222737348726831371141
```







```
mfatrez@Mac-mini-de-Matthieu templates % kubectl config set clusters.cluster.server https://192.168.4.160:6443
Property "clusters.cluster.server" set.
mfatrez@Mac-mini-de-Matthieu templates % kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.4.161:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```


```
mfatrez@Mac-mini-de-Matthieu templates % kubectl config set clusters.default.server https://192.168.4.160:6443
Property "clusters.default.server" set.
mfatrez@Mac-mini-de-Matthieu templates % kubectl config view
apiVersion: v1
clusters:
- cluster:
    server: https://192.168.4.160:6443
  name: cluster
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.4.160:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```







```

mfatrez@Mac-mini-de-Matthieu SynoKube % helm repo add minio https://operator.min.io/
"minio" has been added to your repositories
```

```
mfatrez@Mac-mini-de-Matthieu SynoKube % helm install --namespace minio-operator --create-namespace --generate-name minio/minio-operator
NAME: minio-operator-1627427413
LAST DEPLOYED: Wed Jul 28 01:10:18 2021
NAMESPACE: minio-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the JWT for logging in to the console:
  kubectl get secret $(kubectl get serviceaccount console-sa --namespace minio-operator -o jsonpath="{.secrets[0].name}") --namespace minio-operator -o jsonpath="{.data.token}" | base64 --decode
2. Get the Operator Console URL by running these commands:
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  echo "Visit the Operator Console at http://127.0.0.1:9090"
```


```
kubectl apply -f https://github.com/minio/operator/blob/master/examples/tenant.yaml
```


Install 5 nodes

start 01h15
End   01h42 27 minutes

# helm repo ls

```
NAME                	URL
elastic             	https://helm.elastic.co
kubernetes-dashboard	https://kubernetes.github.io/dashboard/
projectcalico       	https://docs.projectcalico.org/charts
minio               	https://operator.min.io/
bitnami             	https://charts.bitnami.com/bitnami
k8s-at-home         	https://k8s-at-home.com/charts/
ingress-nginx        https://kubernetes.github.io/ingress-nginx
```

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add gitea-charts https://dl.gitea.io/charts/

helm install gitea gitea-charts/gitea





helm repo add ActiveGlobalhelm3-library https://git.rancher.io/helm3-charts
helm repo add ActiveGlobalk8s-at-home https://k8s-at-home.com/charts
helm repo add ActiveGloballibrary https://git.rancher.io/charts
helm repo add ActiveGlobalsystem-library https://git.rancher.io/system-charts




mfatrez@Mac-mini-de-Matthieu etcd-download-test % kubectl taint node kmaster01 k3s-controlplane=true:NoSchedule
node/kmaster01 tainted
mfatrez@Mac-mini-de-Matthieu etcd-download-test % kubectl taint node kmaster02 k3s-controlplane=true:NoSchedule
node/kmaster02 tainted
mfatrez@Mac-mini-de-Matthieu etcd-download-test % kubectl taint node kmaster03 k3s-controlplane=true:NoSchedule
node/kmaster03 tainted


```
for i in 02 03
do
  kubectl taint node kmaster${node_id} CriticalAddonsOnly=true:NoExecute
  kubectl label node/kmaster${node_id} node.longhorn.io/create-default-disk='config'
done
```





```
mfatrez@Mac-mini-de-Matthieu etcd-download-test % kubectl create ns kubernetes-dashboard
mfatrez@Mac-mini-de-Matthieu etcd-download-test % helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -n kubernetes-dashboard
NAME: kubernetes-dashboard
LAST DEPLOYED: Wed Jul 28 22:58:21 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*********************************************************************************
*** PLEASE BE PATIENT: kubernetes-dashboard may take a few minutes to install ***
*********************************************************************************

Get the Kubernetes Dashboard URL by running:
  export POD_NAME=$(kubectl get pods -n default -l "app.kubernetes.io/name=kubernetes-dashboard,app.kubernetes.io/instance=kubernetes-dashboard" -o jsonpath="{.items[0].metadata.name}")
  echo https://127.0.0.1:8443/
  kubectl -n default port-forward $POD_NAME 8443:8443

mfatrez@Mac-mini-de-Matthieu etcd-download-test % kubectl apply -f /Users/mfatrez/Documents/Perso/SynoKube/ansible/roles/k3s-cluster/files/dashboard.token4admin.yaml
```






```
mfatrez@Mac-mini-de-Matthieu etcd-download-test % helm install home-assistant k8s-at-home/home-assistant
NAME: home-assistant
LAST DEPLOYED: Wed Jul 28 22:58:37 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=home-assistant,app.kubernetes.io/instance=home-assistant" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:8123
```













```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.1.2/deploy/prerequisite/longhorn-nfs-installation.yaml
```




```
kubectl create deploy nginx --image nginx
kubectl expose deploy nginx --port 80
```

```
cat <<EOF> nginx_ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    ingress.kubernetes.io/ssl-redirect: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
EOF
```

```
kubectl apply -f nginx_ingress.yaml
```



```
cat <<EOF> deploy1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-un
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-un
  template:
    metadata:
      labels:
        app: hello-un
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.7
        ports:
        - containerPort: 8080
        env:
        - name: MESSAGE
          value: Je Suis UN
EOF
```

```
cat <<EOF> deploy2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deux
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-deux
  template:
    metadata:
      labels:
        app: hello-deux
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.7
        ports:
        - containerPort: 8080
        env:
        - name: MESSAGE
          value: Je Suis DEUX
EOF
```

```
cat <<EOF> svc.yaml
apiVersion: apps/v1
kind: Service
metadata:
  name: hello-un
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-un
---
apiVersion: apps/v1
kind: Service
metadata:
  name: hello-deux
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-deux
EOF
```

```
cat <<EOF> ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - http:
      paths:
      - path: /hw1
        pathType: Prefix
        backend:
          service:
            name: hello-un
            port:
              number: 80
  - host: hw2.kub
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-deux
            port:
              number: 80
EOF
```




calicoctl

```
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.20.0/calicoctl-linux-arm64"
```



Installation 1 master + 4 workers

INSTALL_K3S_EXEC="-v=0 --node-taint CriticalAddonsOnly=true:NoExecute --no-deploy local-storage" k3s-install


K3S_URL="https://192.168.4.160:6443" \
K3S_TOKEN="K10160f758e6633622c52aa0f364eca5e410af958fcf27d2dc2ab12f86085330ac1::server:8862989726e241df3c932b71376f34a7" \
INSTALL_K3S_EXEC="-v=0" \
k3s-install


```
# kubectl create deploy nginx --image containous/whoami
deployment.apps/nginx created
```

```
# kubectl expose deploy nginx --port 80 --type=LoadBalancer
service/nginx exposed
```

file: ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

```
# kubectl apply -f ingress.yaml
ingress.networking.k8s.io/nginx-ingress created
```









```
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
```

Premier resultat, rien à voir avec les suivants...

```
# ---
# apiVersion: v1
# kind: Service
# metadata:
#   name: nginx-service
#   labels:
#     run: nginx
# spec:
#   ports:
#     - port: 80
#       protocol: TCP
#   selector:
#     app:  nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: test-html
```









INSTALLATION METALLB

% helm install metallb metallb/metallb -f ../helm/metallb.values
W0803 10:07:02.073570   13978 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0803 10:07:02.079204   13978 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0803 10:07:02.217999   13978 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0803 10:07:02.218887   13978 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
NAME: metallb
LAST DEPLOYED: Tue Aug  3 10:07:01 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
MetalLB is now running in the cluster.
LoadBalancer Services in your cluster are now available on the IPs you
defined in MetalLB's configuration:

config:
  address-pools:
  - addresses:
    - 192.168.4.192/27
    name: default
    protocol: layer2

To see IP assignments, try `kubectl get services`.





mfatrez@Mac-mini-de-Matthieu test3 % kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system

namespace/longhorn-system created
W0803 10:58:59.520586   17429 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0803 10:58:59.677478   17429 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
NAME: longhorn
LAST DEPLOYED: Tue Aug  3 10:58:59 2021
NAMESPACE: longhorn-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Longhorn is now installed on the cluster!

Please wait a few minutes for other Longhorn components such as CSI deployments, Engine Images, and Instance Managers to be initialized.

Visit our documentation at https://longhorn.io/docs/









###############################################################################################################################################################################################################
PORTWORX mais faut des disques complets ... trop luxe pour moi.

kubectl create deploy whoami --image containous/whoami --replicas 3
kubectl label nodes kworker01 kworker02 kmaster03 px/metadata-node=true


###############################################################################################################################################################################################################


Enable query logs in coreDNS

```
kubectl get configmap -n kube-system coredns -o json |  kubectl get configmap -n kube-system coredns -o json | sed -e 's_loadbalance_log\\n    loadbalance_g' | kubectl apply -f -
```

Voir les logs CoreDNS

```
kubectl -n kube-system get svc -l k8s-app=kube-dns
```

Check DNS on

```
kubectl -n kube-system get pods -l k8s-app=kube-dns --no-headers -o custom-columns=NAME:.metadata.name,HOSTIP:.status.hostIP | while read pod host
do
  echo "Pod ${pod} on host ${host}"
  kubectl -n kube-system exec $pod -c kubedns -- cat /etc/resolv.conf
done
```





###############################################################################################################################################################################################################
# NETWORK PROBLEM
###############################################################################################################################################################################################################

mfatrez@Mac-mini-de-Matthieu files % vi ds-dnstest.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnstest
spec:
  selector:
      matchLabels:
        name: dnstest
  template:
    metadata:
      labels:
        name: dnstest
    spec:
      tolerations:
      - operator: Exists
      containers:
      - image: busybox:1.28
        imagePullPolicy: Always
        name: alpine
        command: ["sh", "-c", "tail -f /dev/null"]
        terminationMessagePath: /dev/termination-log

mfatrez@Mac-mini-de-Matthieu files % kubectl create -f ds-dnstest.yml
daemonset.apps/dnstest created

mfatrez@Mac-mini-de-Matthieu files % kubectl rollout status ds/dnstest -w
daemon set "dnstest" successfully rolled out

mfatrez@Mac-mini-de-Matthieu files % export DOMAIN=www.google.com; echo "=> Start DNS resolve test"; kubectl get pods -l name=dnstest --no-headers -o custom-columns=NAME:.metadata.name,HOSTIP:.status.hostIP | while read pod host; do kubectl exec $pod -- /bin/sh -c "nslookup $DOMAIN > /dev/null 2>&1"; RC=$?; if [ $RC -ne 0 ]; then echo $host cannot resolve $DOMAIN; fi; done; echo "=> End DNS resolve test"

=> Start DNS resolve test
command terminated with exit code 1
192.168.4.165 cannot resolve www.google.com
command terminated with exit code 1
192.168.4.164 cannot resolve www.google.com
=> End DNS resolve test

mfatrez@Mac-mini-de-Matthieu ~ % kubectl delete ds/dnstest
daemonset.apps "dnstest" deleted









###############################################################################################################################################################################################################
# TEST AVEC DATASTORE EXTERNE
###############################################################################################################################################################################################################

# INSTALLATION PACKAGES

```
infranet:~ # zypper in postgresql10-server
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 9 NEW packages are going to be installed:
  libicu60_2 libicu60_2-ledata libpq5 postgresql postgresql-server postgresql10 postgresql10-server postgresql13 postgresql13-server

The following 2 recommended packages were automatically selected:
  postgresql13 postgresql13-server

9 new packages to install.
Overall download size: 19.3 MiB. Already cached: 0 B. After the operation, additional 87.8 MiB will be used.
Continue? [y/n/v/...? shows all options] (y): y
Retrieving package libicu60_2-ledata-60.2-3.9.1.noarch                                                                       (1/9),   6.2 MiB ( 25.7 MiB unpacked)
Retrieving: libicu60_2-ledata-60.2-3.9.1.noarch.rpm ............................................................................................[done (4.9 MiB/s)]
Retrieving package libicu60_2-60.2-3.9.1.aarch64                                                                             (2/9),   1.5 MiB (  5.1 MiB unpacked)
Retrieving: libicu60_2-60.2-3.9.1.aarch64.rpm ..................................................................................................[done (5.3 MiB/s)]
Retrieving package libpq5-13.3-5.13.1.aarch64                                                                                (3/9), 172.8 KiB (713.6 KiB unpacked)
Retrieving: libpq5-13.3-5.13.1.aarch64.rpm .................................................................................................................[done]
Retrieving package postgresql-13-10.3.3.noarch                                                                               (4/9),  14.4 KiB (  713   B unpacked)
Retrieving: postgresql-13-10.3.3.noarch.rpm ................................................................................................................[done]
Retrieving package postgresql10-10.17-8.35.1.aarch64                                                                         (5/9),   1.4 MiB (  6.6 MiB unpacked)
Retrieving: postgresql10-10.17-8.35.1.aarch64.rpm ..........................................................................................................[done]
Retrieving package postgresql13-13.3-5.13.1.aarch64                                                                          (6/9),   1.5 MiB (  7.1 MiB unpacked)
Retrieving: postgresql13-13.3-5.13.1.aarch64.rpm ...............................................................................................[done (6.3 MiB/s)]
Retrieving package postgresql-server-13-10.3.3.noarch                                                                        (7/9),  21.6 KiB (  4.6 KiB unpacked)
Retrieving: postgresql-server-13-10.3.3.noarch.rpm .........................................................................................................[done]
Retrieving package postgresql10-server-10.17-8.35.1.aarch64                                                                  (8/9),   4.0 MiB ( 20.5 MiB unpacked)
Retrieving: postgresql10-server-10.17-8.35.1.aarch64.rpm .......................................................................................[done (6.2 MiB/s)]
Retrieving package postgresql13-server-13.3-5.13.1.aarch64                                                                   (9/9),   4.4 MiB ( 22.1 MiB unpacked)
Retrieving: postgresql13-server-13.3-5.13.1.aarch64.rpm ....................................................................................................[done]

Checking for file conflicts: ...............................................................................................................................[done]
(1/9) Installing: libicu60_2-ledata-60.2-3.9.1.noarch ......................................................................................................[done]
(2/9) Installing: libicu60_2-60.2-3.9.1.aarch64 ............................................................................................................[done]
(3/9) Installing: libpq5-13.3-5.13.1.aarch64 ...............................................................................................................[done]
(4/9) Installing: postgresql-13-10.3.3.noarch ..............................................................................................................[done]
(5/9) Installing: postgresql10-10.17-8.35.1.aarch64 ........................................................................................................[done]
(6/9) Installing: postgresql13-13.3-5.13.1.aarch64 .........................................................................................................[done]
(7/9) Installing: postgresql-server-13-10.3.3.noarch .......................................................................................................[done]
Additional rpm output:
Updating /etc/sysconfig/postgresql ...


(8/9) Installing: postgresql10-server-10.17-8.35.1.aarch64 .................................................................................................[done]
(9/9) Installing: postgresql13-server-13.3-5.13.1.aarch64 ..................................................................................................[done]
Executing %posttrans scripts ...............................................................................................................................[done]
```

# START AND ENABLE SERVICE

Mofifier le fichier /etc/sysconfig/postgresql

```
POSTGRES_OPTIONS="-h 192.168.4.251 -i"
```

Modifier le fichier /var/lib/pgsql/data/pg_hba.conf
```
Remplacer :
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident

Par :
# IPv4 local connections:
host    all             all             0.0.0.0/0            md5
```

```
infranet:~ # systemctl enable --now postgresql
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql.service → /usr/lib/systemd/system/postgresql.service.
```

```
infranet:~ # ss -nlaput | grep LISTEN
tcp    LISTEN  0        32               0.0.0.0:53              0.0.0.0:*       users:(("dnsmasq",pid=26094,fd=7))
tcp    LISTEN  0        128              0.0.0.0:22              0.0.0.0:*       users:(("sshd",pid=1391,fd=3))
tcp    LISTEN  0        128              0.0.0.0:5432            0.0.0.0:*       users:(("postgres",pid=17478,fd=6))
tcp    LISTEN  0        128                    *:80                    *:*       users:(("httpd-prefork",pid=27801,fd=4),("httpd-prefork",pid=27002,fd=4),("httpd-prefork",pid=27001,fd=4),("httpd-prefork",pid=17001,fd=4),("httpd-prefork",pid=15903,fd=4),("httpd-prefork",pid=10631,fd=4),("httpd-prefork",pid=10630,fd=4),("httpd-prefork",pid=10629,fd=4),("httpd-prefork",pid=3862,fd=4),("httpd-prefork",pid=3861,fd=4),("httpd-prefork",pid=1420,fd=4))
tcp    LISTEN  0        32                  [::]:53                 [::]:*       users:(("dnsmasq",pid=26094,fd=10))
tcp    LISTEN  0        128                 [::]:22                 [::]:*       users:(("sshd",pid=1391,fd=4))
tcp    LISTEN  0        128                 [::]:5432               [::]:*       users:(("postgres",pid=17478,fd=7))
```


# CREATE USER AND SCHEMA

Quick and Dirty



```
infranet:~ # sudo -u postgres psql
psql (13.3)
Type "help" for help.

postgres=# create database k3s;
CREATE DATABASE
postgres=# create user thermometre with encrypted password 'Tr0u-Du_cUL';
CREATE ROLE
postgres=# grant all privileges on database k3s to thermometre;
GRANT
```

Utile:

select pg_terminate_backend(pg_stat_activity.pid) from pg_stat_activity where pg_stat_activity.datname = 'k3s';
drop database k3s;
create database k3s;
grant all privileges on database k3s to thermometre;






POUR LE PROBLEME DE DNS : WORKAROUND

REMOVE ALL IPTABLES ON ALL NODES AND REBOOT ALL

```
iptables -F
iptables -X
iptables -F -t nat
iptables -X -t nat
```
