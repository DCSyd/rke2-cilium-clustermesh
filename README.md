# rke2-cilium-clustermesh
Create RKE2 custom cluster from Rancher with --> additional cluster config. CNI, hubble, clustermesh all will be up. 
  1. CA cert and key same for all clusters
  2. Connectivity between clusters (here I am using node ports for connecting clusters)
  3. Pod cidr and service cidr unique for each cluster defined in RKE2 config
  4. Matching pod cidr to be included in cilium operator as well (clusterPoolIPv4PodCIDR)

#Install cilium cli installed along with kubeconfig for all clusters

      CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
      CLI_ARCH=amd64
      if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
      curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
      sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
      sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
      rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
    
https://docs.cilium.io/en/v1.13/gettingstarted/k8s-install-default/#install-the-cilium-cli


#Check clustermesh status, needs kubeconfigs for each cluster and connectivity
    
    cilium clustermesh status --context cl-1
    cilium clustermesh status --context cl-2

Then connect mesh from one side (access to both kubeconfig/api is must)

    cilium clustermesh connect --context cl-1 --source-endpoint 10.128.0.53:32379 --destination-endpoint 10.128.0.54:32379 --destination-context cl-2

    cilium clustermesh status --context cl-1

    ⚠️  Service type NodePort detected! Service may fail when nodes are removed from the cluster!
    ✅ Cluster access information is available:
      - 10.128.0.53:32379
    ✅ Service "clustermesh-apiserver" of type "NodePort" found
    ✅ All 1 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
    🔌 Cluster Connections:
    - cl-2: 1/1 configured, 1/1 connected
    🔀 Global services: [ min:13 / avg:13.0 / max:13 ]


Create deployment and service in both clusters with same name and in same namespace name

For nginx pod to return POD name

Use init container busybox with emptydir to  /usr/share/nginx/html/ to enter pod name
      Command --
      sh -c 'echo "my name is $HOSTNAME" > /usr/share/nginx/html/index.html'

      volumeMounts:
       - mountPath: /usr/share/nginx/html
          name: html
          
    volumes:
      - emptyDir: {}
        name: html

Then create service with below annotation on both clusters
    service.cilium.io/global: "true"

Now test by curl to service and it will be load balanced between clusters
    curl test.test.svc
   

In this setup no encryption used.


