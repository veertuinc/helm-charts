# Anka Build Cloud Helm Chart for AWS

## Synopsis

- By default local-storage is used for the Registry. If the Registry pod is placed on a different node, data will be orphaned and not reachable. This is not a good idea unless you only have a single node kubernetes cluster.
- 

## Usage

[Helm](https://helm.sh) must be installed to use the charts. Please refer to Helm's [documentation](https://helm.sh/docs) to get started.

1. Create `anka-build-cloud.yaml`

    ```yaml
      # Note: You cannot leave a section empty; Comment out "controller:" or "registry:" if everything under it is also commented out.

      controller:
        # version: '1.30.0'
        replicaCount: 2

        #= Automatically create an AWS ALB requires Kubernetes cluster with AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/
        #== Comment out ingressALBHostname if you don't wish to set up the AWS ALB (you will need to deploy your own services)
        ingressALBHostname: controller.k8s.myDomain.com
        #= SG (ID or Name) must be inside of the VPC that's being used by the cluster
        #== Comment out if you want to automatically create an SG for this ALB
        ingressALBSecurityGroup: default

        #= Change this to a URL your nodes can access
        ANKA_ANKA_REGISTRY: http://registry.k8s.myDomain.com:8089
        # ANKA_ETCD_ENDPOINTS: 'etcd-0.etcd-headless.anka-build-cloud.svc.cluster.local:2379,etcd-1.etcd-headless.anka-build-cloud.svc.cluster.local:2379,etcd-2.etcd-headless.anka-build-cloud.svc.cluster.local:2379'
        # ANKA_LOCAL_ANKA_REGISTRY: 'http://registry.anka-build-cloud.svc.cluster.local:8089'

      registry:
        # version: '1.30.0'
        # replicaCount: 1 #= don't use more than 1 unless you have some sort of network storage that the entire cluster can access, no matter where the registry pods are.
        #= Set volumeClaimUseLocalStorage to false (or comment out) if you already have a volume available (you'll need your own pvc for it too)
        volumeClaimUseLocalStorage: true
        #= Use EFS to handle multi-node clusters and HA for Registry (requires that https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html is setup)
        # volumeClaimUseEFS: true
        # volumeClaimFileSystemId: fs-XXXXXXXXXXXXX
        #= Change the volumeClaimName if you have your own PV/PVC set up for the registry with a different name (defaults to registry-data)
        # volumeClaimName: 'registry-data'
        # volumeClaimCapacityStorageSize: 200Gi

        #= Automatically create an AWS ALB requires Kubernetes cluster with AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/
        #== Comment out ingressALBHostname if you don't wish to set up the AWS ALB (you will need to deploy your own services)
        ingressALBHostname: registry.k8s.myDomain.com
        #= SG (ID or Name) must be inside of the VPC that's being used by the cluster
        #== Comment out if you want to automatically create an SG for this ALB
        ingressALBSecurityGroup: default
    ```

2. Once Helm has been set up correctly, add the repo as follows:

        helm repo add anka-build-cloud https://veertuinc.github.com/anka-build-cloud-helm-chart

3. If you had already added this repo earlier, run `helm repo update` to retrieve the latest versions of the packages. You can then run `helm search repo anka-build-cloud` to see the charts.

    To install the anka-build-cloud chart:

        helm upgrade -i -f anka-build-cloud.yaml anka-build-cloud anka-build-cloud

    To uninstall the chart:

        helm delete anka-build-cloud

4. Change your `controller.k8s.myDomain.com` and `controller.k8s.myDomain.com` to point to the ALB that was set up for each (AWS > EC2 > Target Groups).

### EFS

To use EFS, you need to go through instructions in https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html. Then, once 