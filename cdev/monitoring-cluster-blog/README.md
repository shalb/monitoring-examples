* [Goal](#goal)
* [Requirements](#requirements)
   * [OS](#os)
   * [Docker](#docker)
   * [AWS account](#aws-account)
* [Create and deploy our project](#create-and-deploy-our-project)
   * [Get example code](#get-example-code)
   * [Create S3 bucket to store our project's state](#create-s3-bucket-to-store-our-projects-state)
   * [Customize project's settings](#customize-projects-settings)
      * [Select AWS Region](#select-aws-region)
      * [Set unique cluster name](#set-unique-cluster-name)
      * [Set our SSH key](#set-our-ssh-key)
      * [Set ArgoCD password](#set-argocd-password)
      * [Set Grafana password](#set-grafana-password)
   * [Run bash in Clusterdev container](#run-bash-in-clusterdev-container)
   * [Deploy our project](#deploy-our-project)
* [Destroy our project](#destroy-our-project)
* [Conclusion](#conclusion)

# Goal
In this article we will learn how to deploy project which contains test monitoring environment.  
This project will be deployed by [Clusterdev](https://cluster.dev/) to [AWS](https://aws.amazon.com/),  
managed by [Kubernetes](https://kubernetes.io/) cluster([k3s](https://rancher.com/docs/k3s/latest/en/)) and monitored by [Community monitoring stack](https://github.com/prometheus-community/helm-charts/tree/kube-prometheus-stack-35.0.3/charts/kube-prometheus-stack).

# Requirements
## OS
We should have some client host with [Ubuntu 20.04](https://releases.ubuntu.com/20.04/) to use this manual without any customization.  

## Docker
We should install [Docker](https://docs.docker.com/engine/install/ubuntu/) to client host.

## AWS account
Login or [register](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/) new AWS account.  
We should [select](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/select-region.html) AWS [region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-available-regions) to deploy our cluster in that region.  
Add [programmatic access key](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys) for [new](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) or existing user.  
Open `bash` terminal on client host.  
Get example environment file `env` to set our AWS credentials:
```bash
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/env > env
```
Add programmatic access key to environment file `env`:
```bash
editor env
```

# Create and deploy our project
## Get example code
```bash
mkdir -p cdev && mv env cdev/ && cd cdev && chmod 777 ./
alias cdev='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env clusterdev/cluster.dev:v0.6.3'
cdev project create https://github.com/shalb/cdev-aws-k3s-test
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/stack.yaml > stack.yaml
curl https://raw.githubusercontent.com/shalb/monitoring-examples/main/cdev/monitoring-cluster-blog/project.yaml > project.yaml
```

## Create S3 bucket to store our project's state
Go to [S3](https://s3.console.aws.amazon.com/s3/buckets) and [create](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html) new bucket.  
Replace value of `state_bucket_name` key in config file `project.yaml` by state bucket's name:  
```bash
editor project.yaml
```

## Customize project's settings
We will set all needed settings of our project in config file `project.yaml`  
We should customize all variables, which have `# example` comment in the end of line.

### Select AWS Region
We should replace value of [region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html#concepts-available-regions) key in config file `project.yaml` by our region.  

### Set unique cluster name
By default we will use `cluster.dev` domain as root domain for cluster [ingresses](https://kubernetes.github.io/ingress-nginx/).  
We should replace value of `cluster_name` key by unique string in config file `project.yaml`, because default ingress will use it in resulting DNS name.  
This command may help us to generate random name and check is it in use:  
```bash
CLUSTER_NAME=$(echo "$(tr -dc a-z0-9 </dev/urandom | head -c 5)") 
dig argocd.${CLUSTER_NAME}.cluster.dev | grep -q "^${CLUSTER_NAME}" || echo "OK to use cluster_name: ${CLUSTER_NAME}"
```
We should see message: `OK to use cluster_name: ...`

### Set our SSH key
We should have access to cluster nodes via [SSH](https://en.wikipedia.org/wiki/Secure_Shell).  
To add existing SSH key we should replace value of `public_key` key in config file `project.yaml`.  
If we have no SSH key, then we should [create it](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html).  

### Set ArgoCD password
In our project we have [ArgoCD](https://argo-cd.readthedocs.io/). It will help us to deploy our applications.  
To secure ArgoCD we should replace value of `argocd_server_admin_password` key  
by unique password in config file `project.yaml`. Default value is bcrypted `password` string.  
To encrypt our custom password we may use [online tool](https://www.browserling.com/tools/bcrypt) or encrypt it by command:
```bash
docker run -it --entrypoint="" clusterdev/cluster.dev:v0.6.3 apt install -y apache2-utils && htpasswd -bnBC 10 "" myPassword | tr -d ':\n' ; echo ''
```

### Set Grafana password
We should add custom password for [Grafana](https://grafana.com/docs/grafana/latest/).  
To secure Grafana we should replace value of `grafana_password` key  
by unique password in config file `project.yaml`.  
This command may help us to generate random password:  
```bash
echo "$(tr -dc a-zA-Z0-9,._! </dev/urandom | head -c 20)"
```

## Run bash in Clusterdev container
To avoid installation of all needed tools directly to client host - we will run all commands inside Clusterdev container.  
We should run next commands to execute `bash` inside cdev container and proceed to deploy:  
```bash
alias cdev_bash='docker run -it -v $(pwd):/workspace/cluster-dev --env-file=env --network=host --entrypoint="" clusterdev/cluster.dev:v0.6.3 bash'
cdev_bash
```

## Deploy our project
Now we should deploy our project to AWS via `cdev` command:
```bash
cdev apply -l debug | tee apply.log
```
Successful deploy should provide further instructions how to access Kubernetes and URLs of ArgoCD, Grafana web UIs.  
In some cases we should wait some time to access those web UIs.  
DNS update delays may be source of problem.  
In such case we able to forward all needed services via `kubectl` to client host:  
```bash
kubectl port-forward svc/argocd-server -n argocd 18080:443  > /dev/null 2>&1 &
kubectl port-forward svc/monitoring-grafana -n monitoring 28080:80  > /dev/null 2>&1 &
```
We may test our forwards via `curl`:  
```bash
curl 127.0.0.1:18080
curl 127.0.0.1:28080
```
If we see no errors from `curl`, than client host should access those same endpoints via any browser.

# Destroy our project
If we want to destroy our cluster - we should run command:
```bash
cdev apply -l debug
cdev destroy -l debug | tee destroy.log
```

# Conclusion
Now we able to deploy and destroy basic project with monitoring stack by simple commands to save our time.  
This project allows us to use current project as test environment for monitoring related articles  
and test many useful monitoring cases before applying it to production environments.
