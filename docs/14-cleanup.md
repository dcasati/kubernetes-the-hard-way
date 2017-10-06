# Cleaning Up

In this labs you will delete the compute resources created during this tutorial.

## Resource Group

You can delete a Resource Group and all of the resources under it will also be removed. 

> A word of caution here. This command will delete _everything_ we've created so. If you do not wish to do that please skip this step and follow the next sections. 

```
az group delete --no-wait --name kubernetes-the-hard-way
```

> output

```
az group delete --no-wait -y --name kubernetes-the-hard-way
Are you sure you want to perform this operation? (y/n): y
```
> If you do not want to be prompted to confirm the operation you can append the `-y` flag to the previous command.

## Compute Instances

Delete the controller and worker compute instances:

```
gcloud -q compute instances delete \
  controller-0 controller-1 controller-2 \
  worker-0 worker-1 worker-2
```

## Networking

Delete the external load balancer network resources:

```
gcloud -q compute forwarding-rules delete kubernetes-forwarding-rule \
  --region $(gcloud config get-value compute/region)
```

```
gcloud -q compute target-pools delete kubernetes-target-pool
```

Delete the `kubernetes-the-hard-way` static IP address:

```
gcloud -q compute addresses delete kubernetes-the-hard-way
```

Delete the `kubernetes-the-hard-way` firewall rules:

```
gcloud -q compute firewall-rules delete \
  kubernetes-the-hard-way-allow-nginx-service \
  kubernetes-the-hard-way-allow-internal \
  kubernetes-the-hard-way-allow-external
```

Delete the Pod network routes:

```
gcloud -q compute routes delete \
  kubernetes-route-10-200-0-0-24 \
  kubernetes-route-10-200-1-0-24 \
  kubernetes-route-10-200-2-0-24
```

Delete the `kubernetes` subnet:

```
gcloud -q compute networks subnets delete kubernetes
```

Delete the `kubernetes-the-hard-way` network VPC:

```
gcloud -q compute networks delete kubernetes-the-hard-way
```
