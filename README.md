# ML Platform MPI on EKS PoC

## ML on EKS Architecture

[TODO]

## Prepare the EKS Cluster

### Download command line tools

1. AWS CLI: https://aws.amazon.com/cli/

Configure the AWS CLI to connect to cn-northwest-1 AWS region, set the *default* profile and default region.

1. kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/
2. eksctl: https://eksctl.io/

Please use the newest version (above 0.15.0-rc.0) of the eksctl tools downloaded from https://github.com/weaveworks/eksctl/releases/tag/0.15.0-rc.0

1. heml: https://docs.aws.amazon.com/eks/latest/userguide/helm.html

## Create EKS Cluster

1. Check the eksctl version must be above 0.15.0-rc.0:

```
eks version
```


2. Run eksctl to create a EKS cluster:

```
eksctl create cluster --name ml-cluster-1 --region cn-northwest-1 --managed --asg-access
```

This command will create a EKS cluster with one managed node group, please wait about 30m for the CloudFormation deployment completed.

3. Run kubectl to test the kubeconfig has been update by eksctl:

```
kubectl get node
```

The eksctl will automatically update the kubeconfig file on your computer, and you can use the kubectl now

4. Create a Node Group using g4dn.xlarge instance type

```
eksctl create ng --cluster ml-cluster-1 --name ng-g4dn-gpu \
     --node-type g4dn.xlarge --nodes-min 1 --nodes-max 4 --managed --asg-access
```

This will create a Managed NodeGroup with g4dn.xlarge instance type

5. Apply Nvidia Plugin(Please use kubectl command under the workshop root dir):

```
kubectl apply -f nvidia-plugin/nvidia-device-plugin.yml
```

6. Apply Kubernetes Metrics Server YAML to EKS cluster (Please use kubectl command under the workshop root dir):

```
kubectl apply -f metrics-server/
```

Make sure the deployment has been successed:

```
kubectl get deployment metrics-server -n kube-system
```


7. Deploy Prometheus and Grafana:

Deploy Prometheus

```
kubectl create namespace prometheus
helm install prometheus stable/prometheus \
 --namespace prometheus \
 --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
kubectl get pods -n prometheus
```

Deploy Grafana:

```
kubectl create namespace grafana
helm install grafana stable/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set adminPassword='EKS!sAWSome' \
    --set datasources."datasources\.yaml".apiVersion=1 \
    --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
    --set datasources."datasources\.yaml".datasources[0].type=prometheus \
    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources."datasources\.yaml".datasources[0].access=proxy \
    --set datasources."datasources\.yaml".datasources[0].isDefault=true \
    --set service.type=LoadBalancer
```

Get the Grafana dashboard link:

```
kubectl get all -n grafana
```

Login the Grafana(user/password admin/EKS!sAWSome) and install the dashboards 3131 3146 11840:

8. Deploy S3 Goofys FUSE CSI Driver:
    1. Get an AWS AK/SK for S3 access which have the privilege to create S3 bucket, list/put/get S3 objects;
    2. create a ***secret.yaml*** from ***secret.yaml.template*** and filling the ***accessKeyId*** and ***secrectAccessKey***

    3. Deploy the Goofys FUSE CSI driver and Kubernetes PVC:
```
kubectl apply s3-csi-driver/
kubectl apply goofys-pvc
kubectl get pvc
```

9. Deploy EKS Cluster Autoscaler:

```
kubectl apply -f cluster-autoscaler/
kubectl get deployment -n kube-system
```


10. Deploy Kubernetes MPI Operator:
```
kubectl apply -f mpi-operator/
    kubectl get all -n mpi-operator
```


## Run Tensorflow Distribution Training Job

1. Copy the ImageNet Tensorflow format data to S3 bucket which Kubernetes PVC has been created such as pvc-xxxxx-xxxx-xxxxx-xxxx/csi-fs/

2. Create a output Bucket for saving the Tensorflow Checkpoint and model files.
3. Modify the mpi-job/goofys-csi-ml-dist.yaml file S3 bucket name to your own created, and rename the file mpi-job/secret.yaml.tempalte file to mpi-job/secret.yaml mofiy the AWS AK/SK.

4. Deploy the MPI Job:
```
kubectl apply mpi-job/
kubectl get mpijob
kubectl get pod
```
you will find there will only have one pod runing becuase the lack of GPUs

5. Find the Pending Pod why it is in Pending status:
```
kubectl describe ml-goofys-dist-worker-1
```

6. You also can check the AWS console for the Autoscaling Group to find History that it has started more two g4dn.xlarge instance to EKS cluster:

7. Wait the node to be readied in Kubernetes Cluster:
```
kubectl get node
```
8. And you can monitor the Cluster’s GPUs status on GPU monitoring Dashboard:

9. Find your MPI operator launcher POD to check the MPI logs:
```
kubectl get pod
```
10. Check the Launcher POD logs, you can find current training job Sample Throughput about 650 Sample/Second for 3 T4 GPUs and 220 Samples/s per GPU:

```
kubectl logs -f ml-goofys-dist-launcher-xxxx
```

11. This training job will complete in half hours when it loop for1000 batches, another 15 mins the Autoscalor will adjust the EKS Nodegroup to 1 g4dn.xlarge instance.
```
eksctl get ng --cluster ml-cluster-1
```



