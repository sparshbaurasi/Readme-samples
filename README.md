# Aws OIDC configuration

OpenID Connect (OIDC) allows your GitHub Actions workflows to access resources in Amazon Web Services (AWS), without needing to store the AWS credentials as long-lived GitHub secrets.This guide explains how to configure AWS to trust GitHub's OIDC as a federated identity, and includes a workflow example for the `aws-actions/configure-aws-credentials` that uses tokens to authenticate to AWS and access resources.

There are mainly four steps performed in order to configure OIDC from github to aws which are as follows:

## Adding the identity provider to AWS

   _There are multiple methods for adding the identity provider to AWS fue of which are given below_

  1. By AWS console:
    
  1. Open the IAM console at https://console.aws.amazon.com/iam/
  2. In the navigation pane, choose Identity providers, and then choose Add provider.
  3. For Configure provider, choose OpenID Connect.
  4. For Provider URL, type the URL of the IdP give `https://token.actions.githubusercontent.com` as input.
  5. Choose Get thumbprint to verify the server certificate of your IdP.
  6. For the "Audience": Use `sts.amazonaws.com` if you are using the official action.

  2. Using terraform:

  Run the following terraform script for creating the identity provider:
      

      resource "aws_iam_openid_connect_provider" "main" {
       url             = "https://token.actions.githubusercontent.com"
       client_id_list  = ["sts.amazonaws.com"]
       thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
      }


## Configuring the role and trust policy

Create an IAM role with the following trust policy

```json
 {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": {aws_iam_openid_connect_provider.arn}
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:userName/repoName:ref:refs/heads/branchName"
                }
            }
        }
    ]
}
```

## Updating your GitHub Actions workflow
  
  * Add permissions settings for the token like following:

   For workflow:
    
    
    permissions:
        id-token: write # This is required for requesting the JWT
        contents: read  # This is required for actions/checkout
    
    
   For actions:


    permissions:
        id-token: write # This is required for requesting the JWT
    



  * Use the `aws-actions/configure-aws-credentials` action to exchange the OIDC token (JWT) for a cloud access token.
    

      name: AWS example workflow
      on:
        push 
      permissions:
            id-token: write   # This is required for requesting the JWT
            contents: read    # This is required for actions/checkout
      jobs:
        build:
          runs-on: ubuntu-latest
          steps:
            - name: Git clone the repository
              uses: actions/checkout@v3
            - name: configure aws credentials
              uses: aws-actions/configure-aws-credentials@v1
              with:
                role-to-assume: Role arn that we created in 2nd step
                role-session-name: samplerolesession
                aws-region:  AWS_REGION
    