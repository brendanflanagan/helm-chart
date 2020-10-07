# IllumiDesk Helm Chart

:warning: Draft Status :warning:

# Overview

Use this [helm chart](https://helm.sh/docs/topics/charts/) to install IllumiDesk into your Cluster. This chart depends on the [jupyterhub](https://zero-to-jupyterhub.readthedocs.io/en/latest/).

This setup pulls images defined in the `illumidesk/values.yaml` file from `DockerHub`. To push new versions of these images or to change the image's tag(s) (useful for testing), then follow the instructions in the [build images section](#build-images).  

## TL;DR
     $ helm repo add illumidesk https://illumidesk.github.io/helm-chart/
     $ helm repo update
     $ helm upgrade --install $RELEASE illumidesk --namespace $NAMESPACE --values example-config/values.yaml

![Load Balancer Example](https://illumidesk-storage.s3-us-west-2.amazonaws.com/TLDR.gif)

## Prerequsites

- [helm >= v3](https://github.com/kubernetes/helm)
- [Kubectl >= 1.17](https://kubernetes.io/docs/tasks/tools/install-kubectl/)




## Installing the chart

Create a copy of _**example-config/values.yaml.example**_ file and update it with your setup. 

* NOTE: to get a token use  ``` openssl rand -hex 32``` 
    * Here is an example of a basic load balancer setup setup
        ```bash 
                jupyterhub:
                    proxy:
                        secretToken: your_token
                        service:
                        type: LoadBalancer
                alb_ingress_controller:
                enabled: false
                allow_external_dns: 
                enabled: false
                allow_efs: 
                enabled: false
        ```
    * Here is another example of a basic setup using nodeport
        ```bash
            jupyterhub:
            proxy:
                secretToken: your_token
                service:
                type: NodePort
                nodePorts:
                    http: 30791
                    https: 30792
            alb_ingress_controller:
            enabled: false
            allow_external_dns:
            enabled: false
            allow_efs:
            enabled: false
        ```

Create a namespace for your chart to install on:
  * kubernetes namespace defaults to the ```default ``` namespace

        $ kubectl create namespace $NAMESPACE

* (Optional) For alb ingress controller the following must be created
  *  Create **IAM policy** using aws cli for alb ingress policy
        ```Bash    
            `aws iam create-policy \
            --policy-name ALBIngressControllerIAMPolicy \
            --policy-document file://IAM/alb-policy.json` 
        ```
  * Associate the cluster with an oidc-provider to create IAM entities to describe an external identity provider service if you have not done so
        
        eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster={YOUR_CLUSTER}  --approve
  * create an IAM service account and attach it to your cluster
        
        eksctl create iamserviceaccount --cluster={YOUR_CLUSTER} --name=alb-ingress-controller --namespace={YOUR_NAMESPACE} --attach-policy-arn=arn:aws:iam::XXXXXXXXXXXX:policy/ALBIngressControllerIAMPolicy
* (Optional) For the external dns the following must be created
  *  Create **IAM policy** using aws cli for alb ingress policy
        ```Bash    
            `aws iam create-policy \
            --policy-name AllowExternalDNSUpdates \
            --policy-document file://IAM/dns-policy.json` 
        ```
  * create an IAM service account and attach it to your cluster
        
        eksctl create iamserviceaccount --cluster={YOUR_CLUSTER} --name=external-dns --namespace={YOUR_NAMESPACE} --attach-policy-arn=arn:aws:iam::XXXXXXXXXXXX:policy/AllowExternalDNSUpdates
  
Add Illumidesk repository to HELM:

    $ helm repo add illumidesk https://illumidesk.github.io/helm-chart/
    $ helm repo update

Install a release of the illumidesk helm chart
   
    $ RELEASE=jhub
    $ NAMESPACE=jhub 
    $ helm upgrade --install $RELEASE illumidesk --namespace $NAMESPACE --values example-config/values.yaml

## Uninstall the Chart
    $ helm uninstall $RELEASE -n $NAMESPACE

## Configuration 
The following tables lists the configurable parameters of the chart and their default values.

| Parameter                                                                            | Description                                                                                                                              | Default                                                                                                                                          |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| rbac.enabled                                                                         | gives applications only as much access they need to the kubernetes API                                                                   | TRUE                                                                                                                                             |
| jupyterhub.proxy.secretToken                                                         | 32-byte cryptographically secure randomly generated string used to secure communications between the hub and the configurable-http-proxy | value from running openssl rand -hex 32                                                                                                          |
| jupyterhub.proxy.service.type                                                        | Kubernetes service to use to access jupyterhub                                                                                           | LoadBalancer                                                                                                                                     |
| alb_ingress_controller.enabled                                                       | allows creation of aws application load balancer                                                                                         | FALSE                                                                                                                                            |
| alb_ingress_controller.enable_irsa                                                   | allow passing of IAM role arn if you are not using eksctl                                                                                | FALSE                                                                                                                                            |
| alb_ingress_controller.aws_access_key                                                | AWS Access Key used to authenticate with aws API                                                                                         | value from aws configure get aws_access_key_id                                                                                                   |
| alb_ingress_controller.aws_secret_token                                              | AWS Secret Token used to authenticate with aws API                                                                                       | value from aws configure aws_secret_access_key                                                                                                   |
| alb_ingress_controller.cluster_name                                                  | EKS Cluster where aws resources should be created                                                                                        | value from eksctl get cluster                                                                                                                    |
| alb_ingress_controller.cluster_vpc                                                   | Cluster VPC ID alb ingress controller uses to create aws resources                                                                       | value from aws eks describe-cluster --name cluster_name --query "cluster.resourcesVpcConfig.vpcId" --output text                                 |
| alb_ingress_controller.aws_region                                                    | aws region where your cluster is located                                                                                                 | value from aws configure region                                                                                                                  |
| alb_ingress_controller.host                                                          | Host name configured by ingress resource that uses alb                                                                                   |                                                                                                                                                  |
| alb_ingress_controller.service_accounts.annotations.eks.amazonaws.com/role-arn       | assuming 'enable_irsa:true' pass the role arn for the alb ingress controller                                                             |                                                                                                                                                  |
| alb_ingress_controller.ingress.annotations.kubernetes.io/ingress.class               | determines which controller the ingress manifest uses                                                                                    | ALB                                                                                                                                              |
| alb_ingress_controller.ingress.annotations.alb.ingress.kubernetes.io/target-type     | determines how to create target groups                                                                                                   | IP                                                                                                                                               |
| alb_ingress_controller.ingress.annotations.alb.ingress.kubernetes.io/scheme          | determines whether the load balancer is internal or internet facing                                                                      | internet-facing                                                                                                                                  |
| alb_ingress_controller.ingress.annotations.alb.ingress.kubernetes.io/subnets         | subnets that are part of your cluster vpc. At least 2 required                                                                           | value from aws ec2 describe-subnets --filter "Name=vpc-id,Values=vpc-xxxxxxxxxxxx" "Name=tag:Name,Values=*Public*" --query "Subnets[*].SubnetId" |
| alb_ingress_controller.ingress.annotations.alb.ingress.kubernetes.io/tags            | ingress tags to tag ALB/Target Groups/Security group                                                                                     |                                                                                                                                                  |
| alb_ingress_controller.ingress.annotations.alb.ingress.kubernetes.io/certificate-arn | certificate managaged by aws                                                                                                             | certifcate managed by aws                                                                                                                        |
| allow_external_dns.enabled                                                           | makes Kubernetes resources discoverable via public DNS Servers                                                                           | FALSE                                                                                                                                            |
| allow_external_dns.enable_irsa                                                       | allow passing of IAM role arn if you are not using eksctl                                                                                |                                                                                                                                                  |
| allow_external_dns.domainFilter                                                      | aws route 53 hosted zonezone                                                                                                             | value from aws route53 list-hosted-zones --query "HostedZones[*].Name"                                                                           |
| allow_external_dns.txtOwnerID                                                        | identifies externalDNS instance                                                                                                          |                                                                                                                                                  |
| allow_external_dns.service_account.annotations.eks.amazonaws.com/role-arn            | assuming 'enable_irsa:true' pass the role arn for the external dns                                                                       |                                                                                                                                                  |
| allow_efs.enabled                                                                    | Enables creation of EFS manifests                                                                                                        | FALSE                                                                                                                                            |
| allow_efs.efs                                                                        | efs file system url                                                                                                                      | FALSE                                                                                                                                            |
| allow_efs.orgName                                                                    | organization name to configure efs path                                                                                                  | /                                                                                                                                                |
| nginx_ingress_controller.enabled                                                     | allows creation of nginx ingress controller                                                                                              | FALSE                                                                                                                                            |
| nginx_ingress_controller.host                                                        | Host name configured by ingress resource that uses nginx                                                                                 |                                                                                                                                                  |
| nginx_ingress_controller.certificate_arn                                             | certificate managaged by aws                                                                                                             | certifcate managed by aws                                                                                                                        |
| nginx_ingress_controller.vpc_cidr                                                    | CIDR of your cluster vpc                                                                                                                 | XXX.XXX.XXX/XX                                                                                                                                   |


## Validate the Helm Chart

* For nodeport you will need to use your one of your node ips and also the port you defined in your values file. 
  * Open up your browser and use the **NODE_IP:NODE_PORT**
  * Use the following command to list out your nodes:
        
        $ kubectl get nodes -o wide 
          

* For load balancer you will need to get the external IP for proxy-public 
  * Use this command to view your services and then paste the loadbalancer dns that is is the external ip of proxy-public

            $ kubectl get svc -n $NAMESPACE

* For Application Load Balancer, you must have specified the host in your values file
    *  Verify the dns has propgates your domain

            $ dig $HOST 

    * Open up your browser and paste the value for your host


 
## Cleanup

```bash
    helm delete <release name> --purge
```

## Images

### Singleuser Images

By default this chart sets the `singleuser` image to [illumidesk/base-notebook](https://hub.docker.com/r/illumidesk/base-notebook). However, any image maintained in the [illumidesk/docker-stacks](https://github.com/illumidesk/docker-stacks) repo is compatible with this chart.

The `illumidesk/docker-stacks` images are based off of the `jupyterh/docker-stacks` conventions. You can therefore use any of the images in the `jupyterh/docker-stacks` repo which are [also available in dockerhub](https://hub.docker.com/u/jupyter).

To set an alternate image for end-users, update the `singleuser.image` key in the `illumidesk/values.yaml` file.

### JupyterHub Images

There are two Dockerfiles to create two version of the JupyterHub image (`illumidesk/jupyterhub`):

- `illumidesk/jupyterhub`: standard JupyterHub image that uses Python 3.8 and installs the illumidesk package. The illumidesk package contains customized authenticators and spawners.
- `illumidesk/k8s-hub`: inherits from the above image and defines the `NB_USER`, `NB_UID`, and `NB_GID` to run the container.

#### Quick Build/Push

    make build-push-jhubs

This command creates requirements.txt with `pip-compile`, builds docker images, and pushes them to the DockerHub registry.

Enter `make help` for additional options.

### The Hard Way

1. Setup virtualenv:

```bash
    virtualenv -p python3 venv
    source venv/bin/activate
    python3 -m pip install dev-requirements.txt
```

1. Build requirements.txt:

```bash
    pip-compile images/jupyterhub/requirements.in
```

> **Note**: The above command will overwrite the existing requirements.txt file.

2. Build the base JupyterHub image (illumidesk/jupyterhub:py3.8):

```bash
    docker build -t illumidesk/jupyterhub:py3.8 \
      images/jupyterhub/.
```

3. Build the JupyterHub Kubernetes image (illumidesk/k8s-hub:py3.8):

```bash
    docker build -t illumidesk/k8s-hub:py3.8 -f \
      images/jupyterhub/Dockerfile.k8s \
      images/jupyterhub/.
```

4. Push images to registry (DockerHub by default):

```bash
    docker push illumidesk/jupyterhub:py3.8
    docker push illumidesk/k8s-hub:py3.8
```

5. Update `jupyterhub.image.name` with image name. The image name should include the full image namespace and tag.

6. Install IllumiDesk with ```helm``` as inatructed in the first section.





