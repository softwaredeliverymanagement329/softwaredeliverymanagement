# Changes made on FHT AWS account

1. Created a bucket for static client hosting. test.cs240.click
1. Created a web certificate for cs240.click for cloudfront to enable HTTPS
1. Added Route 53 DNS records to authorize the web certificate generation (\_c54655b34f31243aa575eb5008eb16ca.cs240.click)
1. Created a cloudfront distribution for the bucket
1. Created an IAM identity provider for GitHub so I can push to the bucket (token.actions.githubusercontent.com/devops329)
1. Added a Route 53 record for cloudfront (test.cs240.click)
1. Created role for CD connection to github actions (cs329-githubaction-cd). I used the Roles create role wizard to generate a Web Identity trusted entity type.
1. Created a ECR registry and uploaded an image

# Changes made on my account

## AWS Organizations

- Created an organization on my account. I don't this this did much. Maybe it enabled my ability to use IAM Identity Center
- Opened IAM Identity Center in the AWS console.
- Created a user lee-cs to see if I could use organizations to get temporary credentials. Way to complicated and didn't seem to work.
- Tried Cloudshell, but I really need to run it from my dev environment.
- It seems like getting an access token is the only way. bummer.
- Probably the best way to do

## Certificate Manager

I didn't have to do anything here since I already had one.

## s3

1. Created a bucket for static client hosting. test.leesjensen.com. Nothing special. Used all the defaults. Kept private.

## CloudFront

1. Created couldfront distribution for the bucket.
1. Add the CNAME test.leesjensen.com
1. Origin domain: test.leesjensen.com.s3.us-east-2.amazonaws.com
1. Changed name to test.leesjensen.com
1. Set default root object to index.html
1. Public access
1. Redirect HTTP to HTTPS
1. Do not enable WAF
1. Use only North America and Europe
1. Set origin access to Legacy and used the leesjensen.com existing Origin access identity. Told it to update the bucket policy. This caused the following update to the bucket policy.
   ```json
   {
     "Sid": "1",
     "Effect": "Allow",
     "Principal": {
       "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E2RZXH5AWP6HVG"
     },
     "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::test.leesjensen.com/*"
   }
   ```
1. Support HTTP/3
1. Use a custom web certificate that I already had for \*.leesjensen.com

## Route 53

1. Added a Route 53 record for cloudfront (test.leesjensen.com)

## IAM

1. Create a role for the GitHub Action.
   1. Choose Web Identity
   1. Create a new Identity Provider - OpenID Connect
   1. You register the GitHub IP with AWS IAM and specify the AWS rights you are granting for an authenticated user.
      1. provider URL: https://token.actions.githubusercontent.com. Press Get Thumbprint. I don't believe this is used anymore, but it makes me press it.
      1. Audience: sts.amazonaws.com
      1. Configure the IAM role
      1. Permissions policy S3
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Principal": {
              "Federated": "arn:aws:iam::464152414144:oidc-provider/token.actions.githubusercontent.com"
            },
            "Condition": {
              "StringEquals": {
                "token.actions.githubusercontent.com:aud": ["sts.amazonaws.com"]
              },
              "StringLike": {
                "token.actions.githubusercontent.com:sub": ["repo:devops329/*"]
              }
            }
          }
        ]
      }
      ```

## GitHub Action

With this in place I can now enhance the GitHub Action Workflow.

First you need to add the permission to use OIDC tokens in the script.

```yaml
permissions:
  id-token: write
  contents: read
```

Next add AWS CLI command with the acquired OIDC token

```yaml
- name: Create OIDC token to AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    audience: sts.amazonaws.com
    aws-region: us-east-1
    role-to-assume: arn:aws:iam::464152414144:role/GitHubAction-CD
- name: Push to AWS S3
  run: |
    ls -la
    aws s3 ls s3://test.leesjensen.com
    aws s3 cp dist s3://test.leesjensen.com --recursive
```

## ECR

### Create repository

- Navigate to ECR. Expand the menu
- Select public registry
- Select `create repository`
- Made it public
- Gave it the name `webserver`

### Create an image

- Create a simple react app
  ```sh
  mkdir webserver && cd webserver
  npm init -y
  npm install express
  ```
- Write the express code

  ```js
  const express = require('express');

  const app = express();

  app.get('*', (req, res) => {
    console.log(req.path);
    res.send('Hello, World!');
  });

  app.listen(3000, () => {
    console.log('running on 3000');
  });
  ```

- Create docker file

  ```sh
  docker init

  # Follow all the prompts
  ? What application platform does your project use? Node
  ? What version of Node do you want to use? 20.11.1
  ? Which package manager do you want to use? npm
  ? What command do you want to use to start the app? node index.js
  ? What port does your server listen on? 3000

  CREATED: .dockerignore
  CREATED: Dockerfile
  CREATED: compose.yaml
  CREATED: README.Docker.md
  ```

  This created some really useful stuff such as `.dockerignore`, `Dockerfile`, `compose.yaml`, and even a `README.docker.md` that describes how to build and deploy. This might be overkill, but it does expose all new things that Docker provides.

  The docker file does several important steps:

  - Used a base docker image that has the correct node version
  - Specifies thw working dir of the app
  - `npm ci` to install all the dependencies
  - Copies the files into the container
  - Exposes port 3000 from the container
  - Starts up the app. In this case I am using Vite to host the server

- Build the image. I can either use docker compose or docker build. Compose will build and start up the application and allows for a `service` to be defined that launches multiple containers. Build just creates the image.

  ```sh
  docker build -t webserver .

  docker image ls --all
  REPOSITORY                        TAG       IMAGE ID       CREATED         SIZE
  webserver                         latest    0cd8f858dfe7   9 seconds ago   176MB
  ```

  You can also specify the target platform

  ```sh
  docker build --platform=linux/amd64 -t webserver:linux-latest .
  ```

- Test that it works. This creates the container from the image, names it `webserver`, runs it in the background, and opens up port 3000.

  ```sh
  docker run -d --name webserver -p 3000:3000 webserver
  ```

- To remove a running container

  ```sh
  docker kill webserver
  docker container rm webserver

  # or simply

  docker container rm -f webserver
  ```

### Deploy image to ECR

If you open your repository in ECR there is an option to `View push commands`. This gives you everything you need.

First create a temporary login with ECR so that docker can push.

```sh
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 464152414144.dkr.ecr.us-east-2.amazonaws.com

Login Succeeded
```

Tag my image in the local registry

```sh
docker tag webserver:latest 464152414144.dkr.ecr.us-east-2.amazonaws.com/webserver:latest
```

push the image to ECR

```sh
docker push 464152414144.dkr.ecr.us-east-2.amazonaws.com/webserver:latest
```

## ECS/Fargate

- Created a task definition that references my image using fargate.
- Create a cluster named webserver and specified fargate.
- Create a service in the cluster
  - Chose fargate
  - Chose webserver task definition
- Pressed deploy create service
