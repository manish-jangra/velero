Velero is an open-source tool largely used for backing up applications running in Kubernetes clusters to object storage like AWS S3 and restoring them on the same cluster and on a different cluster.

## Prerequisite:
1. AWS CLI
2. Configure AWS CLI with **ACCESS_KEY** and **SECRET_ACCESS_KEY**
3. Velero CLI

## AWS CLI Installation
>**Reference** https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

## AWS CLI Configuration
Run the following command to configure AWS CLI and follow the prompts.
`aws configure`

## Velero Installation
>**Reference** https://velero.io/docs/v1.8/basic-install/

## Configure AWS Resources
1. Create Storage Bucket for storing backups
`export BUCKET=BUCKET_NAME`
`export REGION=AWS_REGION`
`aws s3api create-bucket --bucket $BUCKET --region $REGION --create-bucket-configuration LocationConstraint=$REGION`

2. Create IAM User
`aws iam create-user --user-name velero`
3. Create policy.json file for the IAM User
```json
cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF
```
4. Attach the IAM policy to the user
```
aws iam put-user-policy \
  --user-name velero \
  --policy-name velero \
  --policy-document file://velero-policy.json
```
5. Create Access Key for the IAM user
```
aws iam create-access-key --user-name velero > /tmp/key.json

AWS_ACCESS_ID=`cat /tmp/key.json | jq .AccessKey.AccessKeyId | sed s/\"//g`
AWS_ACCESS_KEY=`cat /tmp/key.json | jq .AccessKey.SecretAccessKey | sed s/\"//g`
```
6. Export Variables
```
export AWS_ACCESS_ID=$AWS_ACCESS_ID
export AWS_ACCESS_KEY=$AWS_ACCESS_KEY
```
7. Create Velero user credentials
```
cat > /tmp/credentials-velero <<EOF
[default]
aws_access_key_id=$AWS_ACCESS_ID
aws_secret_access_key=$AWS_ACCESS_KEY
EOF
```
8. Log in to Kubernetes Cluster and Install Velero
```
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.5.2 \
    --bucket $BUCKET \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --secret-file /tmp/credentials-velero
```
9. Check the Velero pod and its logs
```
kubectl -n velero get pods
kubectl logs deployment/velero -n velero
```
10. Create a backup of a namespace using velero including persistent volumes and persistent volume claims
```
velero backup create default-namespace-backup --include-namespaces default --snapshot-volumes
```
11. Verify the backup for any possible errors 
```
# describe
velero backup describe default-namespace-backup

# logs
velero backup logs default-namespace-backup
```
12. Restore backup in Kubernetes Cluster
```
velero restore create --from-backup default-namespace-backup
```