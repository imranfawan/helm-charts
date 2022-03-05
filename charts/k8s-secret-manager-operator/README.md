
# Kubernetes Secert Manager Operator

## About this Chart

This repoistory contains the helm chart templates that deploys a kubernetes 'Operator' for the purpose of dynamically creating a kubernetes secret bases on a given AWS secret.

The operator dynamically scans a newly created custom CRD resource (secretmanager) that defines
which AWS secret to fetch. The operator will then automatically create a new kubernetes secret based on the AWS secret that was fetched.

## Pre-requisites

The operator leverages the capability of mapping kubernetes service accounts to an IAM role, this is achieved through creating an [OIDC IAM Provider (OpenID Connect)](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html). 

Any pod that uses such a service account will inherit the IAM role credentials that the service account is mapped. As far as this operator is concerned then it needs to reference a service account that is authenticated to have permissions to read secrets from AWS Secret Manager. 

Below are the steps to create an IAM role service account using [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

Run the following to create the 'iamserviceaccount'

```bash
eksctl create iamserviceaccount \
              --name <SERVICE_ACCOUNT> \
              --namespace <NAMESPACE> \
              --cluster <CLUSTER_> \
              --attach-policy-arn <IAM_ROLE_ARN> --approve 
```

Note that the IAM policy belonging to the <IAM_ROLE_ARN> must already exist in AWS and it must have permissions to read the desired secrets. This is required so that any account created with this IAM service account is able to read the desired AWS secrets.

## Deploying the Helm Chart

The raw helm templates are in the repository, however the simplest way to deploy the chart is to pull it from github and execute against your customised 'values.yaml' file. Below are the steps: 

Ensure the abpve pre-requisites is done first, which is to ensure you have created a iam service account.

1) Customise the values.yaml as appropriate.

```bash
replicaCount: 1

image:
  registry: imranawan/k8s-secret-manager-operator
  tag: 1.0.0
  pullPolicy: Always

aws:
  accountId: "<account_id>"

serviceAccountName: read-aws-secrets

rbac:
 clusterRoleName: read-aws-secrets

```

Other than the AWS account ID, you can leave the values as they are. However ensure that the IAM service account you created in the pre-requisites matches serviceAcountName in the values.yaml file. In this case it is set as 'read-aws-secrets'.

Once the values.yaml file is amended accordingly you can deploy the chart as follows:


```bash
helm repo add imranfawan https://imranfawan.github.io/helm-charts/
helm repo update
helm upgrade --install secret-manager imranfawan/k8s-secret-manager-operator --namespace secret-manager -f values.yaml
```

You can deploy to your desired namespace. In the above example the chart is deployed to a namespace called 'secret-manager'.

Several kubernetes resources are now deployed as follows after installing the chart: 

* Custom Resource Definition (CRD)
* Deployment (This deployment scans any resources that are of type secretmanager as per the CRD. Any new resources of this type it will create a k8s secret)
* ClusterRole - To give permissions to the controller to read / write secrets
* ClusterROleBinding - The IAM service account is binded to the cluster role.

Once deployed you should see a Deployment created called 'secret-manager'. This is the deployment that belongs to the controller


## Deploying the SecretManager Resource

We can now create a manifestation of the CRD that is of kind: SecretManager as follows:

```bash
---
apiVersion: kubernetes-client.io/v1
kind: SecretManager
metadata:
  name: <SECRET_NAME>           # Name of the SecretManager resource. Note, that the k8s secret created will also take this name.
spec:
  awsSecret: <AWS_SECRET_NAME>  # Name of the secret in AWS Secret Manager
  region: <AWS_REGION>          # Region of where the AWS secret is located e.g eu-west-2
  namespace: <NAMESPACE>        # Namespace of where the k8s secret will be created
  secretType: <SECRET_TYPE>     # The type of secret to be created e.g opaque or docker
```

Replace the placholders as appropriate and execute in desired namespace as follows: 

```bash
kubectl create -f <name_of_secret_manager_manifest>.yaml -n <namspace>
```

Once a resource of type SecretManager is created as above, the controller will pick this up and create a corresponding kubernetes secret as per the name of the SecretManager resource.

Finally verify the kubernetes secret is created along with its correct content as below:

```bash
kubectl get secret <secret_name> -o yaml -n <namspace>
```
