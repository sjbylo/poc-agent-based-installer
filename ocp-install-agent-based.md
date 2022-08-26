# POC Phase 1

### Prerequisites 

- Access to Bastion (Staging Utility VM) 
- RHEL Base installation, registered and up-to-date (yum update -y) 

## Install tools

### oc

```
curl -s https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz -o - | tar xzvf - -C /usr/local/bin oc
```

### Required tools

```
sudo yum -y install podman httpd-tools openssl
```

### Configure pull secrets 

Fetch your pull-secret and store in a pull-secret.text file.  If needed, ask the customer to do this with their Red Hat account. 

https://console.redhat.com/openshift/install/pull-secret 



## Install Mirror Registry 

## Install mirror script

Ensure enough storage is mounted (500GB+), e.g. to /mnt, to store the images 

```
curl -sL https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/mirror-registry/latest/mirror-registry.tar.gz -o - | tar xzvf -
```


## Install Mirror Registry 

Set some env. variables 

```
BASTION_HOST=bastion.lan  # Set to hostname of the mirror server 
DNS_SERVER=10.0.1.8      # Set to IP of DNS server
```

```
sudo ./mirror-registry install   --quayHostname $BASTION_HOST   --quayRoot /mnt/quay-root 
```

Note down username “init” and password.

```	
REG_PW=xxxxxxxyyyyyyyyyyzzzzzzzzzzz
```

Note, how to remove it again, if needed

```
sudo ./mirror-registry uninstall --quayRoot /mnt/quay-root --autoApprove
```

### Set up registry root certs

```
sudo cp /mnt/quay-root/quay-rootCA/rootCA* /etc/pki/ca-trust/source/anchors/ -v
sudo update-ca-trust extract
```

Use the username and password generated during installation to log into the mirror registry by running the following command:

```
podman login --authfile pull-secret.txt \
  -u init \
  -p <password> \
  $BASTION_HOST:8443> \
  --tls-verify=false
```

Generate the base64-encoded username and password.  Ensure it's in the pull-secret

```
echo -n '<user_name>:<password>' | base64 -w0
```

Generate yaml file

```
cat ./pull-secret.text | jq .  > <path>/<pull_secret_file_in_json>
```

Save the file as:
```
~/containers/auth.json
```


### Test registry is up

```
curl -k https://$BASTION_HOST:8443/health/instance
```

### Log into Quay UI with

```
https://$BASTION_HOST:8443/ 
```

### Set up environment variables

TBD 

## Populate the registry 

### Install tools


### Install oc plugin 

```
curl -s -o - https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/oc-mirror.tar.gz | \
sudo tar x -C /usr/local/bin -vzf - oc-mirror && \
sudo chmod +x /usr/local/bin/oc-mirror

oc mirror help
```

### How to extract information from the mirror registry (examples) 

```
oc mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.11

oc mirror list releases --channel fast-4.11

# Extract OCP version from the Release Image 

oc adm release info -o template --template '{{.metadata.version}}' --insecure=true $BASTION_HOST:8443/poc-mirror/openshift/release-images@sha256:97410a5db655a9d3017b735c2c0747c849d09ff551765e49d5272b80c024a844; echo

# Find the image URL by logging into the registry with your browser, as above, after populating the registry.  
```

### Generate the image set config file

```
oc mirror init --registry $BASTION_HOST:8443/poc-mirror/mirror/oc-mirror-metadata > imageset-config-init.yaml
```

Edit the config file.  Here is an example:

```
cat > imageset-config.yaml <<END
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  registry:
    imageURL: $BASTION_HOST:8443/poc-mirror/mirror/oc-mirror-metadata
    skipTLS: false
mirror:
  platform:
    channels:
    - name: candidate-4.11
      minVersion: 4.11.0
      maxVersion: 4.11.2
      type: ocp
    graph: true
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

### Mirror the content

```
oc mirror --config=./imageset-config.yaml docker://$BASTION_HOST:8443/poc-mirror --dry-run 
# This will take some time!
```

### Find the generated catalogSource & imageContentSourcePolicy in the "results-*" dir 

```
ls -ltr oc-mirror-workspace/ | tail -1    # Find the newest dir
```
Later, the yaml config in this dir will be applied to the OCP cluster. 


Verify that the Operators are available in the OCP console (how to do this on the CLI?)


# POC Phase 2


### Install OpenShift using agent-based installer binary 

This is based on the blog: https://github.com/schmaustech/agent-installer-blog/blob/main/README.md 

The blog describes how to build the binary. 


### Get the openshift-install binary that is agant-based capable 

Recommend to build the binary in advance.  Follow the blog (note that 6GB of RAM is needed to build the binary!).  

Check which OCP release & OCP version image the openshift-install binary is configured for. 

Example:

```
./openshift-installer version
built from commit ddf29b4b5bdb297648ce1b71b916b8cc1212f19a
release image $BASTION_HOST:8443/poc-mirror/openshift/release-images@sha256:300bce8246cf880e792e106607925de0a404484637627edf5f517375517d54a4
release architecture amd64
```
- Note that the (dev preview) binary may be primed to install from 'openshift.org' (the dev registry).  If this is the case this needs to be changed to point to your mirrored registry using the "billi-release.sh" script in the Appendix. 



#### Add the extra values in the install-config.yaml

```
The pull secret for the Internet registry (see: $BASTION_HOST:8443 below, under pullSecret) 
The certificate (quay-rootCA/rootCA.pem)
```
FIXME: Is this still needed? 

### Verify the release image in the registry 

```
oc adm release info -o template --template '{{.metadata.version}}' --insecure=true \
$BASTION_HOST:8443/poc-mirror/openshift/release-images@sha256:97410a5db655a9d3017b735c2c0747c849d09ff551765e49d5272b80c024a844; echo
```
- Fetch the sha value from the Registry UI. The Release Image is usually in the "release-images' repo. 


## OpenShift installation

### Generate the ISO boot image

Boot each type of node to determine:
- Primary interface name, e.g. ens192
- Primary mac address 

Use the network interface name and its mac address to configure the install-config.yaml file

Set the following fields: 

- sshKey 
- platform.baremetal.hosts
- additionalTrustBundle
- imageContentSources 

Get the mirror registry certificate from `quay-rootCA/rootCA.pem`

Example:

```
cat > install-config.yaml <<END
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
      "$BASTION_HOST:8443": {
        "auth": "aW5pdDo0MDNXXXXXXFmWTYYYYYYUXY4V0ZkQk5FazlEeUlubA=="
      }
    }
  }
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIDxzCCAq+gAwIBAgIUD6h49srl+AVZKBuOkFcJGSq5jxEwDQYJKoZIhvcNAQEL
  BQAwZTELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9y
  azENMAsGA1UECgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xFDASBgNVBAMMC2Jh
  c3Rpb24ubGFuYYYYYYYYYYYYYYYYMjUyMFoXDTI1MDYxMjEyMjUyMFowZTELMAkG
  A1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9yazENMAsGA1UE
  CgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xFDASBgNVBAMMC2Jhc3Rpb24ubGFu
  MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAoN4BgJRbOOYICY/1R6EB
  QVcGcNKdZAIrE96gHH0zRvI3viFKPtpC37zjgVIYZyJFNAAsqlWVqHnhnRunYc3j
  iNSiAHIqnOs8ynfZM1xaCEGIuS8GH49k2x+A9YSB0N0e7pppGrNZ8773cKdHtuHB
  QPt4ZLO1L9KeTQbSZWkacItBZB8zH66DXeLrl/EYnS38/YCbXYuNq1PEaj8fj59U
  hsfK3mNzfj8XOIgLEZRIsXXXXXXXXXXXXZH0KrRMYzVn44R7OFmty/2k3Kf3scmn
  KhDqtDeEBfjQXeUn0rlo3544Om4x44yBSc/+8euO1lrhpOz/9epdSBzASt2RsI72
  xQIDAQABo28wbTALBgNVHQ8EBAMCAuQwEwYDVR0lBAwwCgYIKwYBBQUHAwEwFgYD
  VR0RBA8wDYILYmFzdGlvbi5sYW4wEgYDVR0TAQH/BAgwBgEB/wIBATAdBgNVHQ4E
  FgQUf3cDW/Bas95abk9FVk1hTRFQ6V8wDQYJKoZIhvcNAQELBQADggEBAIB+APnD
  6rSHg5Rr6A5L+H047Q2128P3w7qscGAiZTJG7Mq4OzJc5Ww7gtJ7Ox5Pk/n/qnWS
  FnKrpMRqyGICy7OJQ4FBHmY+OYCiv2D4UdbSD6Zk07xTClZheZhPkmofNTzLnxjf
  6rZloYjpBbTFT4OtKTqqL+RNcGC0lhS9Uz3fdSVWUbrPBlKO82t9+ursRQl1jR4F
  ntqUEJnGfkpADTzU49lXMrXqBrDKvOs4Z6ZUd4Z1g0rjqmjVDj4QMsocx0PaCjCP
  0wpVBgDIWSLdZ4W2O3e3KXvDtWfVL0qX/cutG/uwdg543Dm9nRjCG2TXhHBpM/iZ
  wlpL0ZqWXmp6w38=
  -----END CERTIFICATE-----
sshKey: |
  ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABXXXXXXXXXXXXXXXXXXXXXIFebRYnjXdsP3uex+Tti9g3WE+SriQhh9FcpkdZQkLV1nF0VaOKjVEF3lif7DeJfOlMwO0x8wLSqFUv94JccnlG8+Nwab+UJ3yinQXC3r1dJ330uoT44Qc0dHn8fiJm0jCDozcvVV9dPUOkONcGkBMWmZgxbjeBW1JgtM6t1NTB1Zu7yVpG1P+Ot4jlBxREqzGx/O3UGk97CJMncaT7wfgODovp0yo86lzc1UshChXYv6JeO360rmvILsmnOdZlzSVYiq+czSWztMDQMfT9fOCp8SZH1M2/puhRf+w7vcyAAgzF30BHYgU9CyIE/tkZ sbylo@ovpn-117-52.sin2.redhat.com
imageContentSources:
- mirrors:
  - $BASTION_HOST:8443/poc-mirror/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - $BASTION_HOST:8444/poc-mirror/openshift/release-images
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
  
Example agent config file:

```
cat > agent-config.yaml <<END
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
END
```


### Apply the catalogSource & imageContentSourcePolicy yaml

The generated yaml files are found under the `oc-mirror-workspace` dir which was generated by the "oc mirror" command above. 

Example: 
```
oc apply -f oc-mirror-workspace/results-1661347016 
```

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



