# Configure App Framework 

Deploy apps to a search head cluster via the AppFramework feature of the splunk-operator. 

https://splunk.github.io/splunk-operator/AppFramework.html

In the example, the Tenable app/add-on will be installed. 
Tenable Add-On for Splunk: https://splunkbase.splunk.com/app/4060
Tenable App for Splunk: https://splunkbase.splunk.com/app/4061

NOTE: The default `splunk-operator` "operator" already has a persistent volume/persistent volume claim configured. 

# Deploy 

1. Create S3 bucket/folders 

Login to SplunkBase and download the apps above. Create an S3 bucket/folder structure for the apps and upload them.

```
aws s3api create-bucket --bucket splunk-apps --region <REGION>
aws s3api put-object --bucket splunk-apps --key vuln-mgmt/
aws s3 cp /local/path/to/splunk-app.tgz s3://splunk-apps/vuln-mgmt/your-file
```

2. Create access policy 

Note: This policy needs to be reviewed before any production use

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::<my-bucket>",
                "arn:aws:s3:::<my-bucket>/*"
            ]
        }
    ]
}
```

Create the policy

```
aws iam create-policy \
   --policy-name YOUR-IAM-POLICY \
   --policy-document file://iam-policy.json
```

3. Create an IAM role and associate it with a Kubernetes service account

A `--role-name` flag can be added to specify the name of the custom role that is created, otherwise ekstcl will set this. 

```
eksctl create iamserviceaccount \
  --name my-service-account \
  --namespace <my-namespace> \
  --cluster <my-cluster> 
  --role-name "my-role" \
  --attach-policy-arn arn:aws:iam::111122223333:policy/my-policy --approve
```


4. ..? 

Tried associating the role ARN with the splunk-operator pod by adding it in the annotation .. but could not seem to get this working properly. I was able to get this working by creating an IAM user, attaching the same policy, creating access keys, and manually configuring a secret with them via kubectl. Once I did this, I noticed some errors in the splunk-operator pod about a password failing when it tried to validate if apps were installed or not; this is likely because I modified the kubernetes-generated secret to something easier to type via the Splunk GUI. I tried modifying the secret via `kubectl edit secret..` to what I had changed it to, but the secret stored on the standalone SH in `/mnt/spunk-secrets/password` wasn't getting updated to match. After I changed the secret back to the original value, the AppFramework properly deployed packages. 

To Do: 
- Figure out how IAM roles are applied to pods within the EKS cluster
- Auth to s3 bucket via IAM role vs. hardcoded creds. 