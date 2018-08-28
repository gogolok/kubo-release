## Deploying CFCR with a Cloud Provider

Kubernetes exposes the concept of a [Cloud Provider](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/)
which interfaces with the IAAS to provision TCP Load Balancers, Nodes, Networking Routes, and Persistent Volumes.

CFCR can configure Kubernetes with a Cloud Provider through the following methods.
In each example it is assumed that you already have access to a BOSH Director.

### GCP

1. Create a service account and IAM profiles for your master and worker nodes.
   [This Terraform script](https://github.com/cloudfoundry/bosh-bootloader/blob/master/plan-patches/cfcr-gcp/terraform/cfcr_iam_override.tf)
   can be used as a reference guide.

   **Note: The service accounts should be used per CFCR deployment (NOT per director)**

2. Save the service account email addresses into a vars file that will be used to create the cloud-config

```
$ export deployment_name="your deployment name"

$ cat ${deployment_name}-cc-vars.yml

   cfcr_master_service_account_address: <master-service-account-email>
   cfcr_worker_service_account_address: <worker-service-account-email>
   deployment_name: <deployment-name>
```

3. Add a cloud config for the deployment with BOSH [generic configs](https://bosh.io/docs/configs/)

```
$ export KD="path to kubo-deployment repo"

$ bosh update-config --name ${deployment_name} \
   ${KD}/manifests/cloud-config/iaas/gcp/use-vm-extensions.yml \
   --type cloud \
   --vars-file ${deployment_name}-cc-vars.yml
```

4. Deploy CFCR

```
$ bosh deploy -d ${deployment_name} \
   ${KD}/manifests/cfcr.yml \
   -o ${KD}/manifests/ops-files/iaas/gcp/cloud-provider.yml \
   -o ${KD}/manifests/ops-files/use-vm-extensions.yml \
   -o ${KD}/manifests/ops-files/rename.yml \
   -v deployment_name=${deployment_name}
```

5. To test that the cloud provider has been configured correctly, create a simple nginx deployment with an external load balancer

```
$ kubectl apply -f https://github.com/cloudfoundry-incubator/kubo-ci/raw/master/specs/nginx-lb.yml

# wait for the nginx service to get an external-ip
$ kubectl get services

$ export external_ip=$(kubectl get service/nginx -o jsonpath={.status.loadBalancer.ingress[0].ip})
$ curl http://${external_ip}:80
```

### AWS
1. Create IAM profiles for your master and worker nodes.
   [This Terraform script](https://github.com/cloudfoundry/bosh-bootloader/blob/master/plan-patches/cfcr-aws/terraform/cfcr_iam_override.tf)
   can be used as a reference guide.

   **Note: The profiles should be used per CFCR deployment (NOT per director)**

2. Save the profile into a vars file that will be used to create the cloud-config

```
$ export deployment_name="your deployment name"

$ cat ${deployment_name}-cc-vars.yml

   master_iam_instance_profile: <master-iam-profile-name>
   worker_iam_instance_profile: <worker-iam-profile-name>
   cfcr_master_target_pool: <list-of-elbs-for-master>
   deployment_name: <deployment-name>
```

3. Add a cloud config for the deployment with BOSH [generic configs](https://bosh.io/docs/configs/)

```
$ export KD="path to kubo-deployment repo"

$ bosh update-config --name ${deployment_name} \
   ${KD}/manifests/cloud-config/iaas/aws/use-vm-extensions.yml \
   --type cloud \
   --vars-file ${deployment_name}-cc-vars.yml
```

4. Deploy CFCR

```
$ bosh deploy -d ${deployment_name} \
   ${KD}/manifests/cfcr.yml \
   -o ${KD}/manifests/ops-files/iaas/aws/cloud-provider.yml \
   -o ${KD}/manifests/ops-files/use-vm-extensions.yml \
   -o ${KD}/manifests/ops-files/rename.yml \
   -v deployment_name=${deployment_name}
```

5. To test that the cloud provider has been configured correctly, create a simple nginx deployment with an external load balancer

```
$ kubectl apply -f https://github.com/cloudfoundry-incubator/kubo-ci/raw/master/specs/nginx-lb.yml

# wait for the nginx service to get an external-ip
$ kubectl get services

$ export external_ip=$(kubectl get service/nginx -o jsonpath={.status.loadBalancer.ingress[0].hostname})
$ curl http://${external_ip}:80
```