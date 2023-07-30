# AWS & Bitbucket Pipelines OICD Integration
We may want to use AWS Access Key to access resources in AWS, but we may also want to enforce Least Privilege principle to avoid accidental access. With OICD integration, we make sure that only Bitbucket can assume the role we are assigning to a particular identity. Unlike Access Key where it can be stolen by someone and use it to access AWS resources.

## Create Identidy Provider
1. In Bitbucket, create a new repository, go to `Repository settings` -> `Pipelines` -> `OpenID Connect`, then take note of the values.
1. In AWS, go to `IAM` -> `Identity providers` -> `Add provider`, then use the following values:
   - Provider type: `OpenID Connect`
   - Identity provider URL: <paste the value from step 1>
   - Audience: <paste the value from step 1>
   then click `Get thumbprint`, then save.

## Create IAM Role
Go to AWS IAM Role and create a new one, use the following values (adjust as necessary):
- Trust relationship:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "Federated": "arn:aws:iam::<your AWS account>:oidc-provider/api.bitbucket.org/2.0/workspaces/<your Bitbucket workspace>/pipelines-config/identity/oidc"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                  "StringEquals": {
                      "api.bitbucket.org/2.0/workspaces/<your Bitbucket workspace>/pipelines-config/identity/oidc:aud": "ari:cloud:bitbucket::workspace/<your Bitbucket workspace id (UUID)>"
                  },
                  "StringLike": {
                      "api.bitbucket.org/2.0/workspaces/<your Bitbucket workspace>/pipelines-config/identity/oidc:sub": "<your Bitbucket repository id including brackets (UUID)>:*"
                  }
              }
          }
      ]
  }
  ```
- Add IAM Permissions you want for this Role

**Note:** for the Bitbucket workspace and repository IDs, you will find it from the same page in `Bitbucket repository` -> `Repository settings` -> `Pipelines` -> `OpenID Connect`.

## Links
- https://support.atlassian.com/bitbucket-cloud/docs/deploy-on-aws-using-bitbucket-pipelines-openid-connect/

