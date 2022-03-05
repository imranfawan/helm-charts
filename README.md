
# Kubernetes Secert Manager Operator

## About this Repository

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

Note that the <IAM_ROLE_ARN> must have the correct policy attached for it to read the desired AWS secrets.

