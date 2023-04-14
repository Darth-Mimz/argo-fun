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
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetMultiRegionAccessPointPolicyStatus",
                "s3:GetMultiRegionAccessPointRoutes",
                "s3:GetMultiRegionAccessPointPolicy",
                "s3:GetMultiRegionAccessPoint"
            ],
            "Resource": "arn:aws:s3::<aws-account-id>:accesspoint/*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:GetObjectVersionTagging",
                "s3:GetStorageLensConfigurationTagging",
                "s3:GetObjectAcl",
                "s3:GetBucketObjectLockConfiguration",
                "s3:GetIntelligentTieringConfiguration",
                "s3:GetObjectVersionAcl",
                "s3:GetBucketPolicyStatus",
                "s3:GetObjectRetention",
                "s3:GetBucketWebsite",
                "s3:GetJobTagging",
                "s3:GetObjectAttributes",
                "s3:GetObjectLegalHold",
                "s3:GetBucketNotification",
                "s3:DescribeMultiRegionAccessPointOperation",
                "s3:GetReplicationConfiguration",
                "s3:ListMultipartUploadParts",
                "s3:GetObject",
                "s3:DescribeJob",
                "s3:GetAnalyticsConfiguration",
                "s3:GetObjectVersionForReplication",
                "s3:GetAccessPointForObjectLambda",
                "s3:GetStorageLensDashboard",
                "s3:GetLifecycleConfiguration",
                "s3:GetInventoryConfiguration",
                "s3:GetBucketTagging",
                "s3:GetAccessPointPolicyForObjectLambda",
                "s3:GetBucketLogging",
                "s3:ListBucketVersions",
                "s3:ListBucket",
                "s3:GetAccelerateConfiguration",
                "s3:GetObjectVersionAttributes",
                "s3:GetBucketPolicy",
                "s3:GetEncryptionConfiguration",
                "s3:GetObjectVersionTorrent",
                "s3:GetBucketRequestPayment",
                "s3:GetAccessPointPolicyStatus",
                "s3:GetObjectTagging",
                "s3:GetMetricsConfiguration",
                "s3:GetBucketOwnershipControls",
                "s3:GetBucketPublicAccessBlock",
                "s3:ListBucketMultipartUploads",
                "s3:GetAccessPointPolicyStatusForObjectLambda",
                "s3:GetBucketVersioning",
                "s3:GetBucketAcl",
                "s3:GetAccessPointConfigurationForObjectLambda",
                "s3:GetObjectTorrent",
                "s3:GetStorageLensConfiguration",
                "s3:GetBucketCORS",
                "s3:GetBucketLocation",
                "s3:GetAccessPointPolicy",
                "s3:GetObjectVersion"
            ],
            "Resource": [
                "arn:aws:s3:us-west-2:<aws-account-id>:async-request/mrap/*/*",
                "arn:aws:s3:*:<aws-account-id>:accesspoint/*",
                "arn:aws:s3:::purplesec-splunk-apps",
                "arn:aws:s3:::*/*",
                "arn:aws:s3:*:<aws-account-id>:job/*",
                "arn:aws:s3-object-lambda:*:<aws-account-id>:accesspoint/*",
                "arn:aws:s3:*:<aws-account-id>:storage-lens/*"
            ]
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": [
                "s3:ListStorageLensConfigurations",
                "s3:ListAccessPointsForObjectLambda",
                "s3:GetAccessPoint",
                "s3:GetAccountPublicAccessBlock",
                "s3:ListAllMyBuckets",
                "s3:ListAccessPoints",
                "s3:ListJobs",
                "s3:ListMultiRegionAccessPoints"
            ],
            "Resource": "*"
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