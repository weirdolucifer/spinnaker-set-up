# Spinnaker Set up

## Install halyard on Ubuntu

Get the latest version of Halyard:

```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
```

Install it:

```
sudo bash InstallHalyard.sh
```

Check whether Halyard was installed properly:

```
hal -v
```

Run `. ~/.bashrc` to enable command completion.

Update Halyard on Ubuntu

```
sudo update-halyard
```

## Install and configure AWS EKS

### Download and install kubectl

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

Verify the installation of kubectl

```
kubectl help
```

### Download and install aws-iam-authenticator

```
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator

chmod +x ./aws-iam-authenticator

mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

aws-iam-authenticator help
```

### Install awscli

```
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo apt install python3-pip awscli
pip3 install --upgrade awscli
aws --version
aws configure
```

### Install eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl help
```

### Verify Halyard

Spinnaker 1.19.x requires Halyard 1.32.0 or latest

```
hal -v
```

### Add and configure Kubernetes accounts

```
hal config provider kubernetes enable
```

Create a cluster named as eks-spinnaker

```
eksctl create cluster --name <CLUSTER_NAME> --version 1.14 --region us-east-2 --nodegroup-name <NODEGROUP-NAME> --node-type t3.large --nodes 2 --nodes-min 2 --nodes-max 2 --ssh-access --ssh-public-key <SSH_KEY_NAME> --write-kubeconfig=false
```

If cluster is already created:

```
aws eks --region us-east-2  update-kubeconfig --name <ACCOUNT_NAME>
```

Assign the Kubernetes context to CONTEXT

```
CONTEXT=$(kubectl config current-context)
```

Create a service account for the Amazon EKS cluster

```
kubectl apply --context $CONTEXT -f https://www.spinnaker.io/downloads/kubernetes/service-account.yml
```

Extract the secret token of the spinnaker-service-account

```
TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)
```

Set the user entry in kubeconfig:

```
kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
```

Add <ACCOUNT_NAME> as a Kubernetes provider:

```
hal config provider kubernetes account add <ACCOUNT_NAME> --context $CONTEXT
```

### Enable artifact support

```
hal config features edit --artifacts true
```

### Configure Spinnaker to install in Kubernetes

```
hal config deploy edit --type distributed --account-name <ACCOUNT_NAME>
```

### Configure Spinnaker to use AWS S3

Set up/ Check AWS credential

```
vi ~/.aws/credentials
```

```
export YOUR_ACCESS_KEY_ID=<YOUR_KEY>

hal config storage s3 edit --access-key-id $YOUR_ACCESS_KEY_ID --secret-access-key --region us-east-2

hal config storage edit --type s3
```

### Choose the Spinnaker version

```
hal version list
export VERSION=<LATEST_STABLE_VERSION>
hal config version edit --version $VERSION
```

### Install Spinnaker on the eks-spinnaker Amazon EKS cluster:

```
hal deploy apply
```

Verify Spinnaker installation

```
kubectl -n spinnaker get svc
```

### Expose Spinnaker using Elastic Load Balancer:

```
export NAMESPACE=spinnaker

# Expose Gate and Deck
kubectl -n ${NAMESPACE} expose service spin-gate --type LoadBalancer \
  --port 80 --target-port 8084 --name spin-gate-public

kubectl -n ${NAMESPACE} expose service spin-deck --type LoadBalancer \
  --port 80 --target-port 9000 --name spin-deck-public

export API_URL=$(kubectl -n $NAMESPACE get svc spin-gate-public \
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

export UI_URL=$(kubectl -n $NAMESPACE get svc spin-deck-public \
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Configure the URL for Gate
hal config security api edit --override-base-url http://${API_URL}

# Configure the URL for Deck
hal config security ui edit --override-base-url http://${UI_URL}

# Apply your changes to Spinnaker
hal deploy apply
```

It can take several moments for Spinnaker to restart.

You can verify that the Spinnaker Pods have restarted and check their status:

```
kubectl -n spinnaker get pods
kubectl -n spinnaker get svc
```

### Log in to Spinnaker console

```
kubectl -n $NAMESPACE get svc spin-deck-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Enable docker-registry and it to your kubernetes account:

```
hal config provider docker-registry enable

ADDRESS=index.docker.io
REPOSITORIES=library/nginx
USERNAME=<USERNAME>
PASSWORD=<PASSWORD>

hal config provider docker-registry account add <DOCKER_REGISTRY_NAME> --address $ADDRESS --repositories $REPOSITORIES --username $USERNAME --password $PASSWORD

hal config provider kubernetes account edit <ACCOUNT_NAME> --add-docker-registry <DOCKER_REGISTRY_NAME>

hal deploy apply
```

### Adding more Kubernetes accounts:

##### Follow these steps:

* Add and configure Kubernetes accounts [here](#add-and-configure-kubernetes-accounts)
* Configure Spinnaker to install in Kubernetes
* If you wish to add docker-registry to this account follow the step - [Enable docker-registry and it to your kubernetes account]
* Run `hal deploy apply`. This command takes some time to reflect the newly created account on Deck. Alternatively, you can try `hal deploy apply --service-names clouddriver` and `hal deploy apply --service-names gate` to boost this process to some extent.


