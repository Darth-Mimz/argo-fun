# argo-fun
Repo to host necessary code for learning how to use argocd

the "deploy-docs" folder contains documentation for manually deploying resources and should be executed in the following order: 

1. [Deploying the Splunk Operator on Amazon EKS](deploy-docs/aws-eks-splunk-operator-install.md)
2. [Exposing your AWS EKS Splunk Deployment](deploy-docs/aws-eks-splunk-exposing-services.md)
3. [Configuring auto-cert issuance with LetsEncrypt](deploy-docs/tls-configuration-letsencrypt.md)
4. [Configuring AppFramework on a Standalone Search Head](deploy-docs/aws-esk-splunk-configure-app-framework.md)

### Helpful Docs 
1. Understanding persistent volume claims - https://aws.amazon.com/blogs/storage/persistent-storage-for-kubernetes/
2. 

### To Do
1. Add docs for deploying argocd
2. Add docs for integrating repo w/ argocd
3. Update initial deployment steps to include necessary IAM privs for S3 buckets (AppFramework, SmartStore)

