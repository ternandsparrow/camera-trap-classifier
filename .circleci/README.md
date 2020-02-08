You need the following env vars defined for CircleCI:

  1. `PROD_S3_BUCKET` as the AWS S3 bucket that we should deploy prod code (`master` branch) to
  1. `DEV_S3_BUCKET` as the AWS S3 bucket that we should deploy dev code (`develop` branch) to
  1. `AWS_ACCESS_KEY_ID` for a IAM user to can access the S3 bucket
  1. `AWS_SECRET_ACCESS_KEY` for a IAM user to can access the S3 bucket
  1. `AWS_DEFAULT_REGION` probably as `ap-southeast-2`

The AWS IAM policy for the user should look something like this, assuming the target bucket is `example.bucket.com`
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObjectAcl",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::example.bucket.com/*",
                "arn:aws:s3:::example.bucket.com"
            ]
        }
    ]
}
```
