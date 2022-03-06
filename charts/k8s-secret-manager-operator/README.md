
# Kubernetes Secert Manager Operator

## About this Chart

This repoistory contains the helm chart templates that deploys a kubernetes 'Operator' for the purpose of dynamically creating a kubernetes secret bases on a given AWS secret.

The operator dynamically scans a newly created custom CRD resource (secretmanager) that defines
which AWS secret to fetch. The operator will then automatically create a new kubernetes secret based on the AWS secret that was fetched.

## Pre-requisites

The operator leverages the capability of mapping kubernetes service accounts to an IAM role, this is achieved through creating an [OIDC IAM Provider (OpenID Connect)](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html). 

Any pod that uses such a service account will inherit the IAM role credentials that the service account is mapped. As far as this operator is concerned then it needs to reference a service account that is authenticated to have permissions to read secrets from AWS Secret Manager. 

Below are the steps to create an IAM role service account using [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

Ensure you have installed the eksctl cli. 

First associate the oidc provider to your cluster:

```bash
eksctl utils associate-iam-oidc-provider \
                --name <CLUSTER_NAME> \
                --approve \
```

Run the following to create the 'iamserviceaccount'

```bash
eksctl create iamserviceaccount \
              --name <IAM_SERVICE_ACCOUNT> \
              --namespace <NAMESPACE> \
              --cluster <CLUSTER_> \
              --attach-policy-arn <IAM_ROLE_ARN> --approve 
```

Note that the IAM policy belonging to the <IAM_ROLE_ARN> must already exist in AWS and it must have permissions to read the desired secrets. This is required so that any account created with this IAM service account is able to read the desired AWS secrets. Also ensure the <NAMESPACE> is the same namespace where the operator helm chart will be deployed to.


## Deploying the Helm Chart

The raw helm templates are in the repository, however the simplest way to deploy the chart is to pull it from github and execute against your customised 'values.yaml' file. Below are the steps: 

Ensure the above pre-requisites is done first, which is to ensure you have created a iam service account.

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

When running the Helm chart you will need to override the AWS Account ID and the name of the IAM service account in the values.yaml file. Remeber to replace <AWS_ACCOUNT_ID> and
<IAM_SERVICE_ACCOUNT> with your AWS Account ID and IAM Service Account created above. Below are the commands to run the chart.

```bash
helm repo add imranfawan https://imranfawan.github.io/helm-charts/
helm repo update
helm upgrade  --install secret-manager \
              --set aws.accountId=<AWS_ACCOUNT_ID> \
              --set serviceAccountName=<IAM_SERVICE_ACCOUNT> \
              imranfawan/k8s-secret-manager-operator --namespace <NAMESPACE>
```

Other than the AWS account ID, you can leave the values as they are. However ensure that the IAM service account you created in the pre-requisites matches serviceAcountName in the values.yaml file. In this case it is set as 'read-aws-secrets'.

Once the values.yaml file is amended accordingly you can deploy the chart as follows:

```bash
helm upgrade --install secret-manager imranfawan/k8s-secret-manager-operator --namespace <NAMESPACE> -f </path_to_your_values.yaml>
```


Several kubernetes resources are now deployed as follows after installing the chart: 

* Custom Resource Definition (CRD)
* Deployment (This deployment scans any resources that are of type secretmanager as per the CRD. Any new resources of this type it will create a k8s secret)
* ClusterRole - To give permissions to the operator to read / write secrets
* ClusterRoleBinding - The IAM service account is binded to the cluster role.

Once deployed you should see a Deployment created called 'secret-manager'. This is the deployment that belongs to the operator

```bash
NAME          	NAMESPACE     	REVISION	UPDATED                                	STATUS  	CHART                            	APP VERSION
secret-manager	secret-manager	1       	2022-03-05 17:31:33.120666889 +0000 UTC	deployed	k8s-secret-manager-operator-1.0.0	1.0.0     
```

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

You should now see a new resource of type 'SecretManager' as below:

```bash
kubectl get secretmanager -n <namspace>
```

## Verify the K8s Secret is created.
Once a resource of type SecretManager is created as above, the controller will pick this up and create a corresponding kubernetes secret as per the name of the SecretManager resource.

Finally verify the kubernetes secret is created along with its correct content as below:

```bash
kubectl get secret <secret_name> -o yaml -n <namspace>
```

## Useful Tasks

TO delete the IAM service account, execute the following: 

```bash
eksctl delete iamserviceaccount --cluster <CLUSTER_NAME> <IAM_SERVICE_ACCOUNT_NAME>
```

You can minismise the polling the operator does to the K8s API server by de-scaling its Deployment to 0, and scale back to 1 whenever you need to create a K8s secret.

To scale to 0:

```bash
kubectl scale --replicas=0 deployment/<deployment_name> -n <namespace>
```

To scale to 1:

```bash
kubectl scale --replicas=1 deployment/<deployment_name> -n <namespace>
```


## Useful Links

It's possible specify the IAM accounts to be created in the EKS Cluster configuration. You can find more on [IAM Roles for Service Accounts](https://eksctl.io/usage/iamserviceaccounts/)
