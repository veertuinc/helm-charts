# Anka Build Cloud Helm Chart for AWS

## Before You Begin

- Our recommended (and default) approach is to use the [`aws-load-balancer-controller`](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/) to create a NodePort and then ALB in AWS with a specific hostname. Pods must also have IPs from the VPC subnets. We configure [`amazon-vpc-cni-k8s`](https://github.com/aws/amazon-vpc-cni-k8s) plugin for this purpose in our kops deployment.
- By default local-storage is used for the Registry and ETCD. If the service pods are placed on a different kubernetes node, data will be orphaned on the previous and Anka VM Templates, Instances, etc will seem missing and be orphaned. This is not a good idea unless you only have a single node kubernetes cluster. For the registry: EFS is available as an alternative (see the comments below in the yaml and section "Using EFS") which can be cross-az. For ETCD, since it's sensitive to disk speed, you can set up a cluster with [Bitnami's ETCD Helm Chart](https://bitnami.com/stack/etcd/helm) which spans the entire cluster and prevents this.

## Usage

[Helm](https://helm.sh) must be installed to use the charts. Please refer to Helm's [documentation](https://helm.sh/docs) to get started.

1. Create namespace:

    ```bash
    cat << BLOCK > namespace.yaml
    ---
    apiVersion: v1
    kind: Namespace
    metadata:
      name: anka-build-cloud
    BLOCK
    kubectl apply -f namespace.yaml
    ```

1. Create `anka-build-cloud.yaml`

    ```yaml
      # Note: You cannot leave a section empty; Comment out "controller:" or "registry:" if everything under it is also commented out.

      controller:
        version: '1.30.0'
        replicaCount: 3
        #
        #= Automatically create an AWS ALB requires Kubernetes cluster with AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/
        #= Comment out ingressALBHostname if you don't wish to set up the AWS ALB (you will need to deploy your own services)
        ingressALBHostname: controller.k8s.myDomain.com
        #
        #= SG (ID or Name) must be inside of the VPC that's being used by the cluster
        #= Comment out if you want to automatically create an SG for this ALB
        ingressALBSecurityGroup: default
        #
        #= Change this to a URL your nodes can access
        ANKA_ANKA_REGISTRY: http://registry.k8s.myDomain.com:8089
        # If using Bitnami helm chart: ANKA_ETCD_ENDPOINTS: 'etcd-0.etcd-headless.anka-build-cloud.svc.cluster.local:2379,etcd-1.etcd-headless.anka-build-cloud.svc.cluster.local:2379,etcd-2.etcd-headless.anka-build-cloud.svc.cluster.local:2379'
        # ANKA_LOCAL_ANKA_REGISTRY: 'http://registry.anka-build-cloud.svc.cluster.local:8089'
        #
        # ANKA_ENABLE_CENTRAL_LOGGING: true

      registry:
        version: '1.30.0'
        replicaCount: 1 #= don't use more than 1 unless you have some sort of network storage that the entire cluster can access, no matter where the registry pods are.
        #
        #= Set volumeClaimUseLocalStorage to false (or comment out) if you already have a volume available (you'll need your own pvc for it too)
        volumeClaimUseLocalStorage: true
        #
        #= Use EFS to handle multi-node clusters and HA for Registry (requires that https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html is setup)
        # volumeClaimUseEFS: true
        # volumeClaimFileSystemId: fs-XXXXXXXXXXXXX
        #
        #= Change the volumeClaimName if you have your own PV/PVC set up for the registry with a different name (defaults to registry-data)
        volumeClaimName: 'registry-data'
        volumeClaimCapacityStorageSize: 200Gi
        #
        #= Automatically create an AWS ALB requires Kubernetes cluster with AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/
        #= Comment out ingressALBHostname if you don't wish to set up the AWS ALB (you will need to deploy your own services)
        ingressALBHostname: registry.k8s.myDomain.com
        #
        #= SG (ID or Name) must be inside of the VPC that's being used by the cluster
        #= Comment out if you want to automatically create an SG for this ALB
        ingressALBSecurityGroup: default

      etcd:
        version: '1.30.0'
        #= Whether or not to run a single pod with etcd in it. Disable this if you are running an etcd cluster already.
        enabled: true
        volumeClaimUseLocalStorage: true
        volumeClaimName: 'etcd-data'
        volumeClaimCapacityStorageSize: 10Gi
        ETCD_QUOTA_BACKEND_BYTES: '6294967296' # must be a string
    ```

2. Once Helm has been set up correctly, add the repo as follows:

        helm repo add anka-build-cloud https://veertuinc.github.com/anka-build-cloud-helm-chart

3. If you had already added this repo earlier, run `helm repo update` to retrieve the latest versions of the packages. You can then run `helm search repo anka-build-cloud` to see the charts.

    To install the anka-build-cloud chart:

        helm upgrade -i --namespace anka-build-cloud -f anka-build-cloud.yaml anka-build-cloud

    To uninstall the chart:

        helm delete anka-build-cloud

4. Change your `controller.k8s.myDomain.com` and `controller.k8s.myDomain.com` to point to the ALB that was set up for each (AWS > EC2 > Target Groups).

### Using EFS

To use EFS, you need to go through instructions in https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html. Then, once set up in kubernetes, create the EFS, mount targets for the various AZs, and finally your storageClass for it. Here is an example of how to script this (changing `CLUSTER_NAME` and anything else):

```bash
CLUSTER_NAME="k8s.myDomain.com"
DEFAULT_K8S_SG_NAME="default"
VPC_ID="$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}" | grep VpcId | grep -Eo '[^:]*$' | sed 's/"//g' | sed 's/,//g' | xargs)"
DEFAULT_SG_ID="$(aws ec2 describe-security-groups --filter "Name=vpc-id,Values=${VPC_ID}" "Name=group-name,Values=${DEFAULT_K8S_SG_NAME}" | grep GroupId | tail -1 | grep -Eo '[^:]*$' | sed 's/"//g' | sed 's/,//g' | xargs)"
# install aws-fs-csi-driver
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.3"
# add node SG to default cluster SG
NODES_SG_ID="$(aws ec2 describe-security-groups --filter "Name=group-name,Values=nodes.${CLUSTER_NAME}" --query "SecurityGroups[0].GroupId" --output text)"
aws ec2 authorize-security-group-ingress --group-id "${DEFAULT_SG_ID}" --protocol tcp --port 2049 --source-group "${NODES_SG_ID}"
# create or get existing EFS ID
if ! aws efs describe-file-systems | grep anka-build-cloud >/dev/null; then
  FS_ID=$(aws efs create-file-system --performance-mode generalPurpose --throughput-mode bursting \
    --tags Key=Name,Value=anka-build-cloud --query "FileSystemId" --output text)
else
  FS_ID="$(aws efs describe-file-systems --query "FileSystems[?Name=='anka-build-cloud'].FileSystemId" --output text)"
  for MT_ID in $(aws efs describe-mount-targets --file-system-id "${FS_ID}" --query "MountTargets[].MountTargetId" --output text); do
    aws efs delete-mount-target --mount-target-id "${MT_ID}"
  done
fi
sleep 20
# create mount targets for each AZ/subnet
for SN_ID in $(aws ec2 describe-subnets \
  --filters "Name=tag:KubernetesCluster,Values=${CLUSTER_NAME}" \
  --query "Subnets[].SubnetId" --output text); do 
  echo "SUBNET ID: ${SN_ID}"
  aws efs create-mount-target --file-system-id "${FS_ID}" --subnet-id "${SN_ID}" --security-groups "${DEFAULT_SG_ID}"
done
while [[ "$(aws efs describe-mount-targets --file-system-id ${FS_ID} --query "MountTargets[0].LifeCycleState" --output text)" == 'creating' ]]; do
  echo "mount target for efs still creating..."
  sleep 10
done
# create EFS StorageClass
cat << EOF > efs-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
EOF
kubectl apply -f ./efs-storageclass.yaml
```
