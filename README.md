### Prerequisites

- [Kubectl](https://kubernetes.io/docs/tasks/tools/#install-kubectl)
- [Aws CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)


### Provisioning a Kubernetes Cluster

There are many tools for doing this, we will be using [eksctl](https://github.com/weaveworks/eksctl).

#### Dowload eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/0.44.0/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

#### Download the AWS IAM authenticator

We will use this to ensure that we have access to the necessary AWS resources.

```
curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/linux/amd64/aws-iam-authenticator
chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin
```

#### Create a Custom Managed Key (CMK)

The CMK is use by EKS to encrypt our kubernetes secrets.

```
aws kms create-alias --alias-name alias/kubeflow-demo --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
export MASTER_ARN=$(aws kms describe-key --key-id alias/demo --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```

#### Define the cluster config file

We make a .yaml file to define the cluster configuration. Note that we used 8 nodes of t3.small instances since kubeflow requires quite a lot of compute. Using fewer nodes you might not be able to finish the demo.

```
cat << EOF > kubeflow-demo.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${AWS_CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.17"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: kubeflow-mng
  desiredCapacity: 8
  instanceType: t3.small

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

#### Provision Cluster with Eksctl

```
eksctl create cluster -f kubeflow-demo.yaml
```

### Installing Kubeflow on the Cluster

#### Download Kfctl

```
curl --silent --location "https://github.com/kubeflow/kfctl/releases/download/v1.0.1/kfctl_v1.0.1-0-gf3edb9b_linux.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/kfctl /usr/local/bin
```

#### Set Some Environment Variables

We set our file structure and the location of the kfctl config.

```
cat << EoF > kf-install.sh
export KF_NAME=\${AWS_CLUSTER_NAME}

export BASE_DIR=${HOME}/environment
export KF_DIR=\${BASE_DIR}/\${KF_NAME}

# export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_aws_cognito.v1.0.1.yaml"
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_aws.v1.0.1.yaml"

export CONFIG_FILE=\${KF_DIR}/kfctl_aws.yaml
EoF

source kf-install.sh
```

#### Get the kfctl Config

```
mkdir -p ${KF_DIR}
cd ${KF_DIR}
wget -O kfctl_aws.yaml $CONFIG_URI
```

#### Finding the Role Name of our Nodegroup

Here we are using IAM roles for instance groups, which means that we need to give our nodes permission to certain resoures. For this, we need the role name of the nodegroup. If you want to use IAM roles for service accounts that is also possible.

```
export NODE_INSTANCE_ROLENAME=$(aws iam list-roles \
    | jq -r ".Roles[] \
    | select(.RoleName \
    | startswith(\"eksctl-$AWS_CLUSTER_NAME\") and contains(\"NodeInstanceRole\")) \
    .RoleName")
```

#### Modyfiying the Kfctl Config

We set the cluster name, cluster region, and our nodegroup role name in the config file.

```
sed -i -e 's/kubeflow-aws/'"$AWS_CLUSTER_NAME"'/' ${CONFIG_FILE}
sed -i "s@us-west-2@$AWS_REGION@" ${CONFIG_FILE}

sed -i "s@- eksctl-kubeflow-demo-nodegroup-ng-a2-NodeInstanceRole-xxxxxxx@- ${NODE_INSTANCE_ROLENAME}@" ${CONFIG_FILE}
```

#### Launching Kubeflow

```
cd ${KF_DIR}
kfctl apply -V -f ${CONFIG_FILE}
```

#### Assign Permissions
```
aws iam attach-role-policy --role-name $NODE_INSTANCE_ROLENAME --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess

aws iam attach-role-policy --role-name $NODE_INSTANCE_ROLENAME --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```


#### Accessing the Dashboard

In order to access the dashboard we port-forward to the cluster. We will have our dashboard at localhost:8080.

```
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```

#### Creating Notebook

Once you access the dashboard, specify the namespace name and select notebook servers. Create a new notebook server in your namespace. You can specify a docker image with useful packages already installed. We use the following image: ```527798164940.dkr.ecr.us-west-2.amazonaws.com/tensorflow-1.15.2-notebook-cpu:1.0.0```. We also set the total amount of CPU reserved by the server to 1. The server will take few moments to initialize.

#### Cloning Repo

Once the notebook server is running press connect and then "Python 3" under "Notebooks". This will open up a jupyter notebook. Simply enter ```!git clone https://github.com/cpiehl1/kf-demo.git``` into the first cell and run it. This should clone our repo into the current directory. Go into the "kf-demo" folder, open the file "nb.ipynb" and follow the instructions inside.

### Useful Resources

- [EKS workshop](https://www.eksworkshop.com/)
- [Kubeflow documentaion](https://www.kubeflow.org/)


