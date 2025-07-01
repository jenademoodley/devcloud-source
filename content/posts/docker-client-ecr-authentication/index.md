+++
date = 2022-02-10
draft = false
title = 'How does the Docker client authenticate to ECR?'
url = 'how-does-the-docker-client-authenticate-to-ecr'
weight = 10
showAuthor = false
authors = ["jenademoodley"]
showHero = true
heroStyle = 'background'
categories = ['AWS']
tags = ['Docker', 'ECR', 'EKS', 'ECS']
+++

If you have been using ECR, or plan to use ECR for your private images, you should be aware that ECR is closely integrated with IAM. You need an IAM user/role with appropriate permissions to access an ECR repository, and you can pull, push, and authenticate to the ECR repository using AWS API calls. A basic IAM policy with a list of actions needed to interact with a policy is detailed as below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetRepositoryPolicy",
                "ecr:DescribeRepositories",
                "ecr:ListImages",
                "ecr:DescribeImages",
                "ecr:BatchGetImage",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:ListTagsForResource",
                "ecr:DescribeImageScanFindings",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "*"
        }
    ]
}
```

The [AWS ECR documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/security-iam-awsmanpol.html) provides several example policies which you can attach to your IAM user/role which will be accessing ECR.

You can also place a policy on the repository itself, which is used to limit users and accounts which can access the repository (this is somewhat similar to the effect of an S3 bucket policy). IAM is an excellent source of Authentication (confirms the identity of the user/role making the call) and Authorization (is this identity allowed to make this API call), which are both necessary when securing your image. However, this indicates that an IAM user/role is needed in order to run these API calls. Additionally, the average user would generally interact with these images using a docker client, not using the APIs directly. This begs the question, how exactly does the docker client access an ECR repository is IAM credentials are needed? To investigate this behaviour, lets first analyse how an API call actually works, and how the Docker client this functionality.

## Signature Version 4

Signature Version 4 (or SigV4) is the process by which authentication information is added to an API request. We will not go into the process in exact detail, but the general overview is that the HTTP API request includes the API call you would like to make, as well as the AWS Access Key and a signature which is calculated using a hashing algorithm, current date and time, and the Secret Key. The request also needs to include headers for the current date and time (`X-Amz-Date`), the hashing algorithm (`AWS4-HMAC-SHA256`), and the session token if you are using an IAM role. The [AWS documentation for SigV4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html) explains this process in more detail.

Coming back to Docker using IAM authentication, there are two issues which we can identify:

1. Docker would need to be able to translate the Docker push and pull commands into the appropriate ECR API calls. This means Docker needs to be able to obtain the AWS credentials and needs to be built with the AWS eco system in mind.

2. Docker has support for a bearer authentication token (explained below) which can contain an `Authorization` header, but would not be able to contain the `X-Amz-Date` and `AWS4-HMAC-SHA256` headers.

In order to address these issues, we leverage the use of the bearer token, and allow the ECR endpoint to handle some of the work.


## The AWS GetLoginPassword API and Docker Authorization Token

In order to log into your ECR endpoint, the ECR console provides the below command which you need to run:

```bash
aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
```

As seen in the command, the AWS CLI is used to obtain the login password used to login to the repository. This allows the AWS CLI to handle obtaining the credentials, and the login password provided would include these credentials. The docker login command would then use this password and return an authorization token which includes the credentials, as well as the required headers which are needed for the API calls. The token is then saved to the `~/.docker/config.json` file. The contents of this file after logging in would be similar to below:

```json
{
    "auths": {
        "<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com": {
            "auth": "xxxxxx...."
        }
    }
}
```

The Docker client would then use this authorization token whenever it interacts with the ECR repository. The authorization token would contain the credentials of the IAM user/role which was used to login to the repository.  When the docker client invokes any action against the repository with this token, it is able to determine access rights of the IAM user/role. In this way, Docker is able to interact with the repository with the security advantages of IAM.

The authentication token is also encrypted. It is returned encrypted from ECR when running the docker login command, hence you would not be able to unencrypt this token. However, if you are able to obtain this token, you can run docker commands using that token. This is why the legacy `get-login` command is no longer recommended. The [AWS CLI documentation for `get-login`](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login.html) mentions the following:

> This command displays docker login commands to stdout with authentication credentials. Your credentials could be visible by other users on your system in a process list display or a command history. If you are not on a secure system, you should consider this risk and login interactively. For more information, see get-authorization-token.

However, as the token is based on your IAM user/role credentials, if you change the IAM user/role used by your machine to access ECR, you need to login to the docker registry again so that a new token can be created using the updated IAM credentials. Additionally, the token is temporary, and is only valid for 12 hours. Once this time period has elapsed, the token would have expired, and you need to login again.

## Why use this method?

The reason behind this approach is that ECR is intended to be easily integrated into your environment, and you can use your existing Docker setup to manage. In this way, you do not need to modify your Docker client in order to take advantage of ECR. The only addition that you would need is the AWS CLI, however you can instead opt to use the [ECR credential helper](https://github.com/awslabs/amazon-ecr-credential-helper) instead, which can manage your credentials for you. You should look at utilizing a credential helper in your Docker environment to improve security because, by default, the token is stored in plain text in the `~/.docker/config.json` file. It also ensures that credentials are periodically refreshed so that the token is always valid. This is useful in setups where you have automated the running of your Docker containers, for example a Jenkins server.

## Conclusion

The main take away from this blog post is to highlight how ECR and IAM work with Docker authorization tokens to allow you to securely access your images by leveraging IAM. Understanding the process helps us to take precautions such as ensuring get-login-password is used instead of get-login, or a credential helper is used to safe guard the token. It can also help troubleshoot any possible issues which may arise, for example, failed image pulls when the IAM role of an instance changed. This can go a long way in furthering your knowledge of setting up a Docker/ECR environment in the AWS cloud, and should make your journey a lot easier.