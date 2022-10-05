# Phase 1 - Set up the Mirror Registry

### Prerequisites 

- Access to Bastion ("Staging" or "Utility" VM) 
- RHEL 8 Minimal installation, registered and up-to-date (yum update) 
- sudo should be configured  

If not already, register the host:

```
subscription-manager register --username=my@email.com --password=xxyyyzzz
```

### DNS

If there is no DNS, set up DNS on this server and create an A record for the
congtainer mirror registry, e.g. mirror.poc.local, pointing to this host's IP address.  See "Example setting up DNS zone" in the Appendix. 
Or use localhost for now. 


### Proxy

If the system is behind an HTTP proxy set up the proxy configuration: 

```
export NO_PROXY=bastion.lan,api.ocp.lan,apps.ocp.lan
export HTTP_PROXY=<proxy server IP>:8080
export HTTPS_PROXY=<proxy server IP>:8080
# Add these to ~/bashrc 
```

Note: If you use the above proxy settings and you see "Internal Server" errors, be sure to append the endpoint hostnames to the `NO_PROXY` env var. 

Note: add the details in `/etc/rhsm/rhsm.conf` as follows (see: https://access.redhat.com/solutions/65300):

```
# an http proxy server to use (enter server FQDN)
proxy_hostname = myproxy.example.com 

# port for http proxy server
proxy_port = 8080

# user name for authenticating to an http proxy, if needed
proxy_user = proxy_username

# password for basic http proxy auth, if needed
proxy_password = proxy_password
```

### Yum install

```
dnf repolist
sudo yum install podman httpd-tools openssl
```

Install tmux and other useful commands 

```
sudo yum install tmux net-tools 
```


### Environment variables

Set some env. variables 

```
DNS_SERVER=10.0.1.8           # Set to IP of DNS server
REGISTRY_SERVER=bastion.lan   # Set to hostname of the mirror server 
```

Add the above to `~/.bashrc` 


## Install tools

### Install oc CLI

```
# All commands are run as user (non-root) unless otherwise stated 
curl -s https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz -o - | sudo tar xzvf - -C /usr/local/bin oc

oc version
Client Version: 4.11.0
Kustomize Version: v4.5.4
```

### Configure pull secrets 

Fetch your pull secret and store in a `pull-secret.txt` file.  If possible, ask the customer to do this with their own Red Hat account. 

https://console.redhat.com/openshift/install/pull-secret 


## Mirror Registry 

### Install mirror script

Ensure enough storage is mounted, e.g. to /mnt, to store the images. 
Depending on what you require to install, 30-50 GB is needed for the basic image content or 500GB+ for all image content. 

Download the mirror install script:

```
curl -sL https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz -o - | tar xzvf -
```

### Install Mirror Registry 



Install the mirror registry 

```
#### mkdir $HOME/quay-root        # or store the registry data in a location with enough space
sudo ./mirror-registry install  --quayHostname $REGISTRY_SERVER  --quayRoot $HOME/quay-root
```

Note the username “init” and the auto generated password.

```	
REGISTRY_PW='xxxxxxxyyyyyyyyyyzzzzzzzzzzz'
# Add this to ~/.bashrc 
echo REGISTRY_PW="'xxxxxxxyyyyyyyyyyzzzzzzzzzzz'" >> ~/.bashrc  
```

Note, how to remove the registry again, if needed

```
sudo ./mirror-registry uninstall --quayRoot $HOME/quay-root --autoApprove
```

Note: If you see a "_segmentation violation_" error, be sure to add the folowing to /etc/hosts:

```
127.0.0.1   <output of hostname command>
```
See [this page on the issue](https://github.com/quay/mirror-registry/issues/60)

### Registry root certs

Set up the certs in the bastion so that `oc mirror` can connect to all required registries. 

```
sudo cp $HOME/quay-root/quay-rootCA/rootCA* /etc/pki/ca-trust/source/anchors/ -v
sudo update-ca-trust extract
```

Use the username and password generated during installation to log into the mirror registry by running the following command:

```
podman login --authfile pull-secret.txt \
  -u init \
  -p $REGISTRY_PW \
  $REGISTRY_SERVER:8443 \
  --tls-verify=false
```
This will create the file pull-secret.txt with the needed pull secret. 

Generate the base64-encoded username and password.  Ensure it's in the pull-secret file. 

```
echo -n init:$REGISTRY_PW | base64 -w0
REGISTRY_AUTH=`echo -n init:$REGISTRY_PW | base64 -w0` 
```

Generate yaml file (optional) 

```
sudo yum install jq -y 
cat ./pull-secret.txt | jq .  > pull-secret.json 
```

Save the pull-seceret files 

```
mkdir ~/.config/containers 
cp pull-secret.json  ~/.config/containers/auth.json.  

mkdir ~/.docker   
cp pull-secret.json  ~/.docker/config.json.      # Needed by "oc mirror" 
```

Note: Both of these files are needed. 

### Test registry is up

```
curl -k https://$REGISTRY_SERVER:8443/health/instance
```

### Log into the Quay UI with the created password (see above) 

```
echo https://$REGISTRY_SERVER:8443/ 
```

## Populate the registry 

### Install tools

#### Install oc mirror plugin 

See: https://github.com/openshift/oc-mirror 

```
curl -L -s -o - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/oc-mirror.tar.gz | \
sudo tar x -C /usr/local/bin -vzf - oc-mirror && \
sudo chmod +x /usr/local/bin/oc-mirror

oc mirror help
```
See [this blog](https://cloud.redhat.com/blog/mirroring-openshift-registries-the-easy-way) for more on the mirror plugin. 


### Generate the image set config file

Note: be sure to have the correct access to both/all registries: username/password/pull secrets/certs.  Note: but in oc-mirror ... still requires file: `~/.docker/config.json` 

```
oc mirror init --registry $REGISTRY_SERVER:8443/ocp4/openshift4/mirror/oc-mirror-metadata > imageset-config-init.yaml  
```

Edit the config file generated (`imageset-config-init.yaml `) or use this example:

```
cat > imageset-config.yaml <<END
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  registry:
    imageURL: $REGISTRY_SERVER:8443/ocp4/openshift4/mirror/oc-mirror-metadata
    #skipTLS: false
mirror:
  platform:
    channels:
    - name: candidate-4.11
      minVersion: 4.11.0
#      maxVersion: 4.11.2
      shortestPath: true 
      type: ocp
#    graph: true
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.11
    packages:
    - name: cincinnati-operator
    - name: local-storage-operator
      channels:
      - name: stable
    - name: ocs-operator
    - name: kubevirt-hyperconverged
      channels:
      - name: stable
  additionalImages:
  - name: registry.redhat.io/ubi8/ubi:latest
  helm: {}
END
```

See the reference list of operator names in the Appendix 

See [this tool](https://access.redhat.com/labs/ocpupgradegraph/update_path?channel=stable-4.10&arch=x86_64&is_show_hot_fix=true&current_ocp_version=4.9.12&target_ocp_version=4.10.22) for all versions and upgrade paths.
 

### Mirror the content

```
oc mirror --config=./imageset-config.yaml docker://$REGISTRY_SERVER:8443 --dry-run 
# This will take some time!
# Note that this command requires ~/docker/config.json   See above. 
```

### Optional: How to extract information from the mirror registry (examples) 

```
oc mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.11

OR

oc mirror list operators --catalog bastion.lan:8443/steve/redhat/redhat-operator-index@sha256:f07deab29ed8694391f999b9da409d58fae02b66029306e2101fe868944de648

oc-mirror list operators --catalogs --version=4.11
registry.redhat.io/redhat/redhat-operator-index:v4.11
registry.redhat.io/redhat/certified-operator-index:v4.11
registry.redhat.io/redhat/community-operator-index:v4.11
registry.redhat.io/redhat/redhat-marketplace-index:v4.11

oc-mirror list releases --version=4.11
oc mirror list releases --channel fast-4.11
oc-mirror list releases --channels --version=4.11
# See: https://github.com/openshift/oc-mirror#releases 

# Extract binary from release (example) 
oc adm release extract --registry-config "pull-secret.json" --command=openshift-baremetal-install --to "/usr/local/bin/" $VERSION

```
Fetch release versions and channels: 

oc mirror list operators --package advanced-cluster-management --catalog registry.redhat.io/redhat/redhat-operator-index:v4.11             
NAME                         DISPLAY NAME                                DEFAULT CHANNEL                                                                                  
advanced-cluster-management  Advanced Cluster Management for Kubernetes  release-2.5
PACKAGE                      CHANNEL      HEAD
advanced-cluster-management  release-2.5  advanced-cluster-management.v2.5.1
```

# Extract OCP version from the Release Image 

oc adm release info -o template --template '{{.metadata.version}}' --insecure=true $REGISTRY_SERVER:8443/ocp4/openshift4/openshift/release-images@sha256:97410a5db655a9d3017b735c2c0747c849d09ff551765e49d5272b80c024a844; echo

List Operators that are available to this cluster: 

```
oc get packagemanifests -n openshift-marketplace
```

# Find the image sha by logging into the registry with your browser, as above, after populating the registry and going to /ocp4/openshift4/release-images -> Tags. 
```


Find the generated catalogSource & imageContentSourcePolicy in the "results-*" dir 

```
ls -ltr oc-mirror-workspace/ | tail -1    # Find the newest dir
```
Later, the yaml config in this dir will be applied to the OCP cluster. 


Verify that the Operators are available in the OCP console (how to do this on the CLI?)

If installing ACM and you need MCH, annotate MCH as there's a doc bug. 

```
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  annotations:
    installer.open-cluster-management.io/mce-subscription-spec: '{"source": "redhat-operator-index"}'
```


# Phase 2 - Install OpenShift 

### Install OpenShift using agent-based installer binary 

This is based on the blog: https://github.com/schmaustech/agent-installer-blog/blob/main/README.md 

The blog describes how to fetch (or build) the binary. 


### Get the openshift-install binary that is `agant-based` capable 

Recommend to build the binary in advance.  Follow the blog (note that 6GB of RAM is needed to build the binary!)  

Check which OCP release & OCP version image the openshift-install binary is configured for. 

Example:

```
./openshift-installer version
built from commit ddf29b4b5bdb297648ce1b71b916b8cc1212f19a
release image bastion.lan:8443/ocp4/openshift4/openshift/release-images@sha256:300bce8246cf880e792e106607925de0a404484637627edf5f517375517d54a4
release architecture amd64
```
- Note that the (dev preview) binary may be primed to install from 'openshift.org' (the dev registry).  If this is the case this needs to be changed to point to your mirrored registry using the "billi-release.sh" script in the Appendix. 


#### Add the extra values in the install-config.yaml

```
The pull secret for the Internet registry (see: bastion.lan:8443 below, under pullSecret) 
The certificate (quay-rootCA/rootCA.pem)
```



### Verify the release image in the registry 

Fetch the release image SHA ID from the registry UI (or the `oc adm` command below) and set the env. variable.
It can be found in the UI e.g. under `/ocp4/openshift4/openshift/release-images:v4.11` 

```
RELEASE_IMAGE=sha256:97410a5db655a9d3017b735c2c0747c849d09ff551765e49d5272b80c024a844
```

```
oc adm release info -o template --template '{{.metadata.version}}' --insecure=true \
$REGISTRY_SERVER:8443/ocp4/openshift4/openshift/release-images@$RELEASE_IMAGE; echo
```
- Fetch the sha value from the Registry UI. The Release Image is usually in the "release-images' repo. 


## OpenShift installation

### Generate the ISO boot image

First, boot each type of node in your env. to determine:
- Primary NIC name, e.g. ens192
- Primary NIC mac address 

Use the network interface name and its mac address to configure the install-config.yaml file

Set the following fields: 

- sshKey 
- platform.baremetal.hosts
- additionalTrustBundle
- imageContentSources 

Get the mirror registry certificate from `quay-rootCA/rootCA.pem` in Quay's data dir. 

ROOT_CA=`~/quay-root/quay-rootCA/rootCA.pem`
SSH_PUB_KEY=`tail -1 ~/.ssh/authorized_keys`     # needed below 
SSH_KEY-`cat ~/.ssh/id_rsa`


Optional: Copy the ssh private key to the bastion 
 
```
scp ~/.ssh/id_rsa  user@bastion:.ssh/
```

Set up ssh config

```
cat > ~/.ssh/config <<END
StrictHostKeyChecking no
UserKnownHostsFile=/dev/null
ConnectTimeout=15
ServerAliveInterval=120
END

chmod 600 ~/.ssh/config
```

Create a directory to hold the configuration files (install-config.yaml and agent-config.yaml)

```
mkdir cluster-manifests.src
```

Example:

```
cat > cluster-manifests.src/install-config.yaml <<END
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: poc
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.1.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    hosts:
      - name: master1
        role: master
        bootMACAddress: 00:50:56:00:00:01
      - name: master2
        role: master
        bootMACAddress: 00:50:56:00:00:02
      - name: master3
        role: master
        bootMACAddress: 00:50:56:00:00:03
      - name: worker1
        role: worker
        bootMACAddress: 00:50:56:00:00:11
      - name: worker2
        role: worker
        bootMACAddress: 00:50:56:00:00:12
    apiVIP: "10.0.1.216"
    ingressVIP: "10.0.1.226"
pullSecret: |
  {
    "auths": {
      "$REGISTRY_SERVER:8443": {
        "auth": "$REGISTRY_AUTH"
      }
    }
  }
additionalTrustBundle: |
`echo "$ROOT_CA" | sed -e 's/^/  /'`
sshKey: |
  $SSH_PUB_KEY
imageContentSources:
- mirrors:
  - $REGISTRY_SERVER:8443/ocp4/openshift4/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - $REGISTRY_SERVER:8444/ocp4/openshift4/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
END
```
  
To create agent-config.yaml, change the following fields:

- rendezvousIP 
- hosts[].interfaces.macAddress
- hosts[].interfaces.mac-address
- hosts[].networkConfig.interfaces.name
- hosts[].networkConfig.interfaces.ipv4.address[0].prefix-length 
- hosts[].networkConfig.interfaces.dns-resolver.config.server[]
  
The `rendezvousIP` determines where the Assisted Installer Service will run. 
  
Example agent config file:

```
cat > cluster-manifests.src/agent-config.yaml <<END
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: poc
rendezvousIP: 10.0.1.51
hosts:
  - hostname: master1
    interfaces:
     - name: ens192
       macAddress: 00:50:56:00:00:01
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          mac-address: 00:50:56:00:00:01
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.51
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - $DNS_SERVER
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: ens192
  - hostname: master2
    interfaces:
     - name: ens192
       macAddress: 00:50:56:00:00:02
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          mac-address: 00:50:56:00:00:02
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.52
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - $DNS_SERVER
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: ens192
  - hostname: master3
    interfaces:
     - name: ens192
       macAddress: 00:50:56:00:00:03
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          mac-address: 00:50:56:00:00:03
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.53
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - $DNS_SERVER
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: ens192
  - hostname: worker1
    interfaces:
     - name: ens192
       macAddress: 00:50:56:00:00:11
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          mac-address: 00:50:56:00:00:11
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.61
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - $DNS_SERVER
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: ens192
  - hostname: worker2
    interfaces:
     - name: ens192
       macAddress: 00:50:56:00:00:12
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          mac-address: 00:50:56:00:00:12
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.62
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - $DNS_SERVER 
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: ens192
END
```

## Installation execution

```
sudo dnf -y install nmstate    # Needed by "openshift-install agent" 
```

Generate the boot image iso. 
Note: We sure to unset the proxy vars, otherwise data may be fecthed from the internet and cause problems. 

```
unset no_proxy
unset http_proxy
unset https_proxy
```

```
rm -rf cluster-manifests && cp -rp cluster-manifests.src cluster-manifests && \
bin/openshift-install agent create image --log-level debug --dir cluster-manifests
```

Note: The nodes (VMs/hosts) that are being installed need to have the same date/time set.  If NTP is not configured (e.g. in vSphere), then set the times manually on all of them. If the nodes have times which are out of sync, then the installation will not start or will fail. 

Now, boot all the nodes using the created agent.iso image found in cluster-manifests/.


## Disabling the default OperatorHub sources

After installation, in a restricted network environment, you must disable the default catalogs as a cluster administrator. 

```
oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```


### Apply the catalogSource & imageContentSourcePolicy yaml

The generated yaml files are found under the `oc-mirror-workspace` dir which was generated by the "oc mirror" command above in phase 1. 

Example: 
```
oc apply -f oc-mirror-workspace/results-1661347016 
```

After applying, the OperatorHub in the OCP Console should show some available Operators which are served from the internal mirror server. 


## Appendix 

### Reference list of Operator names from a 4.11 Release 

Add these operator names to the 'imageset-config.yaml', created above.

```
NAME                                          DISPLAY NAME                                           DEFAULT CHANNEL
3scale-operator                               Red Hat Integration - 3scale                           threescale-2.12
advanced-cluster-management                   Advanced Cluster Management for Kubernetes             release-2.5
amq-online                                    Red Hat Integration - AMQ Online                       stable
amq-streams                                   Red Hat Integration - AMQ Streams                      stable
amq7-interconnect-operator                    Red Hat Integration - AMQ Interconnect                 1.10.x
ansible-automation-platform-operator          Ansible Automation Platform                            stable-2.2-cluster-scoped
ansible-cloud-addons-operator                 Ansible Cloud Addons                                   stable-2.2-cluster-scoped
apicast-operator                              Red Hat Integration - 3scale APIcast gateway           threescale-2.12
aws-efs-csi-driver-operator                   AWS EFS CSI Driver Operator                            stable
aws-load-balancer-operator                    AWS Load Balancer Operator                             stable-v0.1
bamoe-businessautomation-operator             IBM Business Automation                                8.x-stable
bamoe-kogito-operator                         IBM BAMOE Kogito Operator                              8.x
bare-metal-event-relay                        Bare Metal Event Relay                                 stable
businessautomation-operator                   Business Automation                                    stable
cincinnati-operator                           OpenShift Update Service                               v1
cluster-kube-descheduler-operator             Kube Descheduler Operator                              stable
cluster-logging                               Red Hat OpenShift Logging                              stable
clusterresourceoverride                       ClusterResourceOverride Operator                       stable
compliance-operator                           Compliance Operator                                    release-0.1
container-security-operator                   Red Hat Quay Container Security Operator               stable-3.7
costmanagement-metrics-operator               Cost Management Metrics Operator                       stable
cryostat-operator                             Cryostat Operator                                      stable
datagrid                                      Data Grid                                              8.3.x
devspaces                                     Red Hat OpenShift Dev Spaces                           stable
devworkspace-operator                         DevWorkspace Operator                                  fast
dpu-network-operator                          DPU Network Operator                                   stable
eap                                           JBoss EAP                                              stable
elasticsearch-operator                        OpenShift Elasticsearch Operator                       stable
external-dns-operator                         ExternalDNS Operator                                   stable-v1.0
file-integrity-operator                       File Integrity Operator                                release-0.1
fuse-apicurito                                Red Hat Integration - API Designer                     fuse-apicurito-7.11.x
fuse-console                                  Red Hat Integration - Fuse Console                     7.11.x
fuse-online                                   Red Hat Integration - Fuse Online                      latest
gatekeeper-operator-product                   Gatekeeper Operator                                    stable
integration-operator                          Red Hat Integration                                    1.x
jaeger-product                                Red Hat OpenShift distributed tracing platform         stable
jws-operator                                  JWS Operator                                           alpha
kiali-ossm                                    Kiali Operator                                         stable
klusterlet-product                            Klusterlet                                             backplane-2.0
kubernetes-nmstate-operator                   Kubernetes NMState Operator                            stable
kubevirt-hyperconverged                       OpenShift Virtualization                               stable
local-storage-operator                        Local Storage                                          stable
loki-operator                                 Loki Operator                                          stable
mcg-operator                                  NooBaa Operator                                        stable-4.10
metallb-operator                              MetalLB Operator                                       stable
mtc-operator                                  Migration Toolkit for Containers Operator              release-v1.7
mtv-operator                                  Migration Toolkit for Virtualization Operator          release-v2.3
multicluster-engine                           multicluster engine for Kubernetes                     stable-2.0
nfd                                           Node Feature Discovery Operator                        stable
node-healthcheck-operator                     Node Health Check Operator                             stable
node-maintenance-operator                     Node Maintenance Operator                              stable
node-observability-operator                   Node Observability Operator                            alpha
numaresources-operator                        numaresources-operator                                 4.11
ocs-operator                                  OpenShift Container Storage                            stable-4.10
odf-csi-addons-operator                       CSI Addons                                             stable-4.10
odf-lvm-operator                              ODF LVM Operator                                       stable-4.10
odf-multicluster-orchestrator                 ODF Multicluster Orchestrator                          stable-4.10
odf-operator                                  OpenShift Data Foundation                              stable-4.10
odr-cluster-operator                          Openshift DR Cluster Operator                          stable-4.10
odr-hub-operator                              Openshift DR Hub Operator                              stable-4.10
openshift-cert-manager-operator               cert-manager Operator for Red Hat OpenShift            tech-preview
openshift-custom-metrics-autoscaler-operator  Custom Metrics Autoscaler                              stable
openshift-gitops-operator                     Red Hat OpenShift GitOps                               latest
openshift-pipelines-operator-rh               Red Hat OpenShift Pipelines                            pipelines-1.8
openshift-secondary-scheduler-operator        Secondary Scheduler Operator for Red Hat OpenShift     stable
openshift-special-resource-operator           Special Resource Operator                              stable
opentelemetry-product                         Red Hat OpenShift distributed tracing data collection  stable
ptp-operator                                  PTP Operator                                           stable
quay-bridge-operator                          Red Hat Quay Bridge Operator                           stable-3.7
quay-operator                                 Red Hat Quay                                           stable-3.7
red-hat-camel-k                               Red Hat Integration - Camel K                          1.6.x
redhat-oadp-operator                          OADP Operator                                          stable-1.0
rh-service-binding-operator                   Service Binding Operator                               stable
rhacs-operator                                Advanced Cluster Security for Kubernetes               latest
rhpam-kogito-operator                         RHPAM Kogito Operator                                  7.x
rhsso-operator                                Red Hat Single Sign-On Operator                        stable
sandboxed-containers-operator                 OpenShift sandboxed containers Operator                stable-1.3
self-node-remediation                         Self Node Remediation Operator                         stable
serverless-operator                           Red Hat OpenShift Serverless                           stable
service-registry-operator                     Red Hat Integration - Service Registry Operator        2.0.x
servicemeshoperator                           Red Hat OpenShift Service Mesh                         stable
skupper-operator                              Skupper                                                alpha
sriov-network-operator                        SR-IOV Network Operator                                stable
submariner                                    Submariner                                             stable-0.12
tang-operator                                 Tang                                                   0.0.24
topology-aware-lifecycle-manager              Topology Aware Lifecycle Manager                       stable
vertical-pod-autoscaler                       VerticalPodAutoscaler                                  stable
volsync-product                               VolSync                                                stable
web-terminal                                  Web Terminal                                           fast
windows-machine-config-operator               Windows Machine Config Operator                        stable
```

### Installing virtctl

View the ConsoleCLIDownload object by running the following command

```
$ oc get ConsoleCLIDownload virtctl-clidownloads-kubevirt-hyperconverged -o yaml

tar -xvf <virtctl-version-distribution.arch>.tar.gz

chmod +x <virtctl-file-name>
```

Download the virtctl client by using the link listed for your distribution.



### Billi-release.sh script 

This script can build the installer from source and change the embedded release image URL and OCP version in the openshift-install binary. 

For this script to work, install the following. 

```
sudo yum install golang git make zip`
```


```
#!/usr/bin/env bash

function verify_if_release_image_exists() {
    local release_image=$1
    if [ ! "$(podman manifest inspect ${release_image})" ]; then
        echo "Cannot download image ${release_image}"
    fi
}

function clone_agent_repo() {
    
    local commit=$1

    if [[ ! -d "installer" ]]; then 
        git clone https://github.com/openshift/installer.git
    fi
    
    if [[ ! -z "${commit}" ]]; then
        echo "Building from commit $commit"
    fi

    pushd installer
    git checkout $commit
    popd
}

function build_agent_installer() {
    pushd installer
    # Steve
    export GOCACHE=/tmp/.cache/go-build
    hack/build.sh
    popd
}

function patch_openshift_install_release_version() {
    local version=$1

    local res=$(grep -oba ._RELEASE_VERSION_LOCATION_.XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX openshift-install)
    local location=${res%%:*}

    # If the release marker was found then it means that the version is missing
    if [[ ! -z ${location} ]]; then
        echo "Patching openshift-install with version ${version}"
        printf "${version}\0" | dd of=openshift-install bs=1 seek=${location} conv=notrunc &> /dev/null 
    else
        echo "Version already patched"
    fi
}

function patch_openshift_install_release_image() {
    local image=$1

    local res=$(grep -oba ._RELEASE_IMAGE_LOCATION_.XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX openshift-install)
    local location=${res%%:*}

    # If the release marker was found then it means that the image is missing
    if [[ ! -z ${location} ]]; then
        echo "Patching openshift-install with image ${image}"
        printf "${image}\0" | dd of=openshift-install bs=1 seek=${location} conv=notrunc &> /dev/null 
    else
    	local res=$(grep -oba bastion.lan:8443/steve openshift-install)
    	local location=${res%%:*}
    	if [[ ! -z ${location} ]]; then
        	echo "Patching openshift-install with image ${image}"
        	printf "${image}\0" | dd of=openshift-install bs=1 seek=${location} conv=notrunc &> /dev/null 
    	else
       		echo "Image already patched"
	fi
    fi
}

function complete_release() {
    echo 
    echo "BILLI installer release completed"
    echo 

    ./openshift-install version
}

if [ "$#" -ne 3 ]; then
    echo "Usage: billi-release2.sh <release image>"
    echo "[Arguments]"
    echo "      <commit>            Commit, e.g. 'agent-installer'"
    echo "      <release image>     The desired OpenShift release image location/url"
    echo "      <version>           The desired OpenShift version, e.g. 4.11.5"
    echo
    echo "example: ./billi-release.sh quay.io/openshift-release-dev/ocp-release@sha256:300bce8246cf880e792e106607925de0a404484637627edf5f517375517d54a4"

    exit 1
fi

commit=$1
release_image=$1
release_version=$3

verify_if_release_image_exists $release_image
clone_agent_repo $commit
build_agent_installer
patch_openshift_install_release_version $release_version
patch_openshift_install_release_image $release_image
complete_release
```

Example: how to run this script: 

```
./billi-release.sh agent-installer registry.example.com:8443/ocp4/openshift4/openshift/release-image@sha256:xxxyyzzz 4.11.5 
# Fetch the SHAR ID of the release image you want to use from the registry. 
```

### Example ImageContentSourcePolicy that works

```
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  labels:
    operators.openshift.org/catalog: "true"
  name: operator-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - bastion.lan:8443/steve/rhceph
    source: registry.redhat.io/rhceph
  - mirrors:
    - bastion.lan:8443/steve/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - bastion.lan:8443/steve/rhel8
    source: registry.redhat.io/rhel8
  - mirrors:
    - bastion.lan:8443/steve/redhat
    source: registry.redhat.io/redhat
  - mirrors:
    - bastion.lan:8443/steve/rhacm2-tech-preview
    source: registry.redhat.io/rhacm2-tech-preview
  - mirrors:
    - bastion.lan:8443/steve/multicluster-engine
    source: registry.redhat.io/multicluster-engine
  - mirrors:
    - bastion.lan:8443/steve/odf4
    source: registry.redhat.io/odf4
  - mirrors:
    - bastion.lan:8443/steve/rhacm2
    source: registry.redhat.io/rhacm2
```

### Example setting up DNS zone

Steps from https://www.linuxtechi.com/setup-bind-server-centos-8-rhel-8/ 

```
LOCAL_DOMAIN=poc.local
LOCAL_CIDR=192.168.0.0/24
LOCAL_REVERSE=0.168.192       # if needed 
NTP_IP=192.168.0.10 
DNS_IP=192.168.0.20
MIRROR_IP=192.68.0.30
```

Install Bind 

```
sudo dnf install bind bind-utils -y 
sudo systemctl start named
sudo systemctl enable named
sudo systemctl status named
```

Set up the `allow` confg 

```
cp /etc/named.conf  /etc/named.bak

# In the conf file, comment out
sed -i -e "s#listen-on port 53.*#// \1#" /etc/named.conf
sed -i -e "s#listen-on-v6 port 53.*#// \1#" /etc/named.conf

// listen-on port 53 { 127.0.0.1; }; 
// listen-on-v6 port 53 { ::1; };
# and

allow-query { localhost; $LOCAL_CIDR; };   # <= your subnet 
```

Define the forward zone 

```
cat >> /etc/named.conf <<END

//forward zone
zone "$LOCAL_DOMAIN" IN {
     type master;
     file "$LOCAL_DOMAIN.db";
     allow-update { none; };
     allow-query { any; };
};

//backward zone
zone "$LOCAL_REVERSE.in-addr.arpa" IN {
     type master;
     file "$LOCAL_DOMAIN.rev";
     allow-update { none; };
     allow-query { any; };
};
END
```

Set up the zone config 

```

$TTL 86400
@ IN SOA dns-primary.$LOCAL_DOMAIN. admin.$LOCAL_DOMAIN. (
                                                2020011800 ;Serial
                                                3600 ;Refresh
                                                1800 ;Retry
                                                604800 ;Expire
                                                86400 ;Minimum TTL
)

;Name Server Information
@ IN NS dns-primary.$LOCAL_DOMAIN.

;IP Address for Name Server
dns-primary IN A $192.168.0.20

;A Record for the following Host name
dns   IN   A  $DNS_IP
ntp   IN   A  $NTP_IP
mirror IN  A  $MIRROR_IP 

END
```

Finally

```
sudo chown named:named /var/named/linuxtechi.local.db
sudo chown named:named /var/named/linuxtechi.local.rev

# Verify 
sudo named-checkconf
named-checkzone $LOCAL_DOMAIN /var/named/$LOCAL_DOMAIN.db
# named-checkzone 192.168.43.35 /var/named/linuxtechi.local.rev

systemctl restart named

sudo firewall-cmd  --add-service=dns --zone=public  --permanent
sudo firewall-cmd --reload
```

### Configure XRDP for Remote Desktop access 

See: https://linuxconfig.org/how-to-work-with-dnf-package-groups 

```
sudo -i 
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

dnf groupinstall "Server with GUI" -y 

systemctl start xrdp
systemctl status xrdp
systemctl enable xrdp

firewall-cmd --permanent --add-port=3389/tcp
firewall-cmd --reload
exit 
```

Connect in the usual way to the server.  Log in with a user / password. 


### Configure tmux

```
cat > ~/.tmux.conf <<END
# remap prefix from 'C-b' to 'C-a'
unbind C-b
set-option -g prefix C-a
bind-key C-a send-prefix

bind j resize-pane -D 5
bind k resize-pane -U 5
bind h resize-pane -L 10
bind l resize-pane -R 10

#set -g mode-mouse on 
#set -g mouse-resize-pane on

bind-key R source-file ~/.tmux.conf \; display-message "source-file done"
END 
```

