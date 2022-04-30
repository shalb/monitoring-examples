# Goal
In this article we will learn how to run [Kubernetes](https://kubernetes.io/) cluster in [AWS](https://aws.amazon.com/) by [Clusterdev](https://cluster.dev/).  
We will deploy monitoring code to learn how to monitor Kubernetes clusters by [Prometheus](https://prometheus.io/).


# Requirements
## OS
We should have [Ubuntu 20.04](https://releases.ubuntu.com/20.04/) host to use this manual without any customization.

## Docker
Install [Docker](https://docs.docker.com/engine/install/ubuntu/).

## AWS account
Login or register new [AWS account](https://aws.amazon.com/).  
We should pick region to deploy. All things should be deployed to this region.  
Add [programmatic access key](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keysv) for [new](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) or existing user.  
Get example environment file:
```bash
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/env > env
```
Add programmatic access key to environment file:
```bash
editor env
```

# Create basic project
## Get example code
```bash
mkdir -p cdev && mv env cdev/ && cd cdev && chmod 777 ./
alias cdev='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env clusterdev/cluster.dev:v0.6.3'
alias aws='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env --entrypoint=aws clusterdev/cluster.dev:v0.6.3'
alias kubectl='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env --network=host --entrypoint=kubectl clusterdev/cluster.dev:v0.6.3'
alias cdev_bash='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env --entrypoint="" clusterdev/cluster.dev:v0.6.3 bash'
cdev project create https://github.com/shalb/cdev-aws-k3s-test
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/stack.yaml > stack.yaml
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/project.yaml > project.yaml
```

## Create S3 bucket to store our project's state
Go to [S3](https://s3.console.aws.amazon.com/s3/buckets) and [create](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html) new bucket.  
Replace value of `state_bucket_name` key in config file `project.yaml` by our bucket name:
```bash
editor project.yaml
```

## Customize mandatory settings
We will set all needed settings in config file `project.yaml`  

We should set region via `region` key in config file `project.yaml`  

By default we will use `cluster.dev` domain as root domain for cluster [ingresses](https://kubernetes.github.io/ingress-nginx/).  
We should set unique value for `cluster_name` in `project.yaml`, because default ingress will use it in resulting DNS name.  
This command may help us to generate random name: `echo "$(tr -dc a-z0-9 </dev/urandom | head -c 5)"`  

We should have access to cluster nodes via [SSH](https://en.wikipedia.org/wiki/Secure_Shell).  
To add existing SSH key we should add it to `public_key` key in `project.yaml`.  
If we have no SSH key, then we should [create it](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html).  

In our default cluster we will have [ArgoCD](https://argo-cd.readthedocs.io/). It will help us to deploy our applications.  
To secure ArgoCD we should set unique password as value of `argocd_server_admin_password` key in `project.yaml`.  
Default value is bcrypted `password` string. We should change it to our custom password.  
To encrypt our custom password we may use [online tool](https://www.browserling.com/tools/bcrypt) or run command:
```bash
docker run -it --entrypoint="" clusterdev/cluster.dev:v0.6.3 apt install -y apache2-utils && htpasswd -bnBC 10 "" myPassword | tr -d ':\n' ; echo ''
```

## Deploy our cluster
```bash
cdev apply -l debug | tee apply.log
```

## Get kubeconfig
```bash
aws s3 cp s3://gelo-cdev-state/test/kubeconfig ./kubeconfig
```

# Deploy basic monitoring stack
```bash
curl https://raw.githubusercontent.com/shalb/monitoring-examples/cdev/cdev/monitoring-cluster-blog/kube-prometheus-stack/kube-prometheus-stack-crds.yaml > kube-prometheus-stack-crds.yaml
curl https://raw.githubusercontent.com/shalb/monitoring-examples/cdev/cdev/monitoring-cluster-blog/kube-prometheus-stack/kube-prometheus-stack.yaml > kube-prometheus-stack.yaml
kubectl -f kube-prometheus-stack-crds.yaml apply
kubectl -f kube-prometheus-stack.yaml apply
```

# Destroy cluster
```bash
cdev destroy -l debug | tee destroy.log
```

# Usefull commands
```bash
kubectl port-forward svc/argocd-server -n argocd 28080:443
kubectl port-forward svc/monitoring-grafana -n monitoring 28080:80
```
