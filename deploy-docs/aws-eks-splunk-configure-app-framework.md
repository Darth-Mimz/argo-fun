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

4. Link the ServiceAccount to the splunk-operator pod

On line 49992 of the splunk-operator-namespace.yaml CR.. add the name of the newly created service account 

```
      serviceAccountName: splunk-operator-controller-manager, s3-splunk-appframework-reader-sa
```

