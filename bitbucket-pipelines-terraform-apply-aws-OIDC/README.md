# Running Terraform Apply on Bitbucket Pipelines with OIDC

## 1. Create IAM Role
- Name: `bitbucket-oidc-role`
- Trust Relationship:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<aws-account-number>:oidc-provider/api.bitbucket.org/2.0/workspaces/<bitbucket-workspace-name>/pipelines-config/identity/oidc"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "api.bitbucket.org/2.0/workspaces/<bitbucket-workspace-name>/pipelines-config/identity/oidc:aud": "ari:cloud:bitbucket::workspace/<bitbucket-workspace-id>"
                }
            }
        }
    ]
}
```
then add your IAM permissions to this role.
## 2. BitBucket Pipeline
- File: `bitbucket-pipelines.yml`
- Content:
```yaml
pipelines:
  branches:
    main:
    - step:
        name: "Terraform plan"
        image: docker.io/hashicorp/terraform:1.7
        artifacts:
        - .terraform/**
        - .terraform.lock.hcl
        - tf-plan.output
        oidc: true
        script:
        - apk update
        # install aws cli v2 and JSON parser
        - apk add 'aws-cli>2' jq
        - aws --version

        # initial AWS credentials
        # https://aws.amazon.com/blogs/apn/using-bitbucket-pipelines-and-openid-connect-to-deploy-to-amazon-s3/
        - |-
          aws sts assume-role-with-web-identity \
            --role-arn arn:aws:iam::<aws-account-number>:role/bitbucket-oidc-role \
            --role-session-name sfas-terraform-plan-session  \
            --web-identity-token "$BITBUCKET_STEP_OIDC_TOKEN" \
            --duration-seconds 900 > credentials.json
        - cat credentials.json
        - export AWS_ACCESS_KEY_ID=$(cat credentials.json | jq -r '.Credentials.AccessKeyId')
        - export AWS_SECRET_ACCESS_KEY=$(cat credentials.json | jq -r '.Credentials.SecretAccessKey')
        - export AWS_SESSION_TOKEN=$(cat credentials.json | jq -r '.Credentials.SessionToken')
        - export AWS_REGION=ap-southeast-2
        - aws sts get-caller-identity | cat

        # Terraform plan
        - terraform -version
        - terraform init
        - terraform plan -out ./tf-plan.output
        - ls -lha 
    - step:
        trigger: manual
        name: "Terraform apply"
        image: docker.io/hashicorp/terraform:1.7
        oidc: true
        script:
        - apk update
        # install aws cli v2 and JSON parser
        - apk add 'aws-cli>2' jq
        - aws --version

        # initial AWS credentials
        # https://aws.amazon.com/blogs/apn/using-bitbucket-pipelines-and-openid-connect-to-deploy-to-amazon-s3/
        - |-
          aws sts assume-role-with-web-identity \
            --role-arn arn:aws:iam::<aws-account-number>:role/bitbucket-oidc-role \
            --role-session-name sfas-terraform-apply-session  \
            --web-identity-token "$BITBUCKET_STEP_OIDC_TOKEN" \
            --duration-seconds 3600 > credentials.json
        - cat credentials.json
        - export AWS_ACCESS_KEY_ID=$(cat credentials.json | jq -r '.Credentials.AccessKeyId')
        - export AWS_SECRET_ACCESS_KEY=$(cat credentials.json | jq -r '.Credentials.SecretAccessKey')
        - export AWS_SESSION_TOKEN=$(cat credentials.json | jq -r '.Credentials.SessionToken')
        - export AWS_REGION=ap-southeast-2
        - aws sts get-caller-identity | cat

        # Terraform apply
        - terraform -version
        - ls -lha
        - terraform apply -auto-approve ./tf-plan.output
```
