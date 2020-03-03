# Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

## Technologies:
**Docker**: application containerization
<br>**Kubernetes(AWS EKS)**: for deployment, scaling, and management of containerized applications.
<br>**CloudFormation**: Automation of creation infrastructure on AWS
<br>**AWS CodePipeline**: for automatic integration and deployment (CI/CD).
<br>**Python**: Example web application with JWT auth.

## Example web application

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'.
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token.

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).

## Improve this project steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

_____
# AWS EKS Walkthrough

## Create a Kubernetes (EKS) Cluster
Create an EKS cluster named `simple-jwt-api`.

##Set Up an IAM Role for the Cluster
The next steps are provided to quickly set up an IAM role for your cluster.

1. Create an IAM role that CodeBuild can use to interact with EKS:

* Set an environment variable `ACCOUNT_ID` to the value of your AWS account id. You can do this with awscli:
```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```
* Create a role policy document that allows the actions "eks:Describe*" and "ssm:GetParameters". You can do this by setting an environment variable with the role policy:
```
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"
```

* Create a role named 'FlaskDeployCBKubectlRole' using the role policy document:
```
aws iam create-role --role-name FlaskDeployCBKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'
```
* Create a role policy document that also allows the actions "eks:Describe*" and "ssm:GetParameters". You can create the document in your tmp directory:
```
 echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "eks:Describe*", "ssm:GetParameters" ], "Resource": "*" } ] }' > /tmp/iam-role-policy
```
you can file this file in the root or this repository.

* Attach the policy to the 'UdacityFlaskDeployCBKubectlRole'. You can do this using awscli:
```
aws iam put-role-policy --role-name FlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
```

2. Grant the role access to the cluster. The 'aws-auth ConfigMap' is used to grant role based access control to your cluster.

* Get the current configmap and save it to a file:
```
 kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
```

* In the data/mapRoles section of this document add, replacing <ACCOUNT_ID> with your account id:
```
- rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
    username: build
    groups:
      - system:masters
```

* Now update your cluster's configmap:
```
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

3. Generate a GitHub access token. A Github acces token will allow CodePipeline to monitor when a repo is changed.

4. The file buildspec.yml instructs CodeBuild. We need a way to pass your jwt secret to the app in kubernetes securly. You will be using AWS Parameter Store to do this. First add the following to your buildspec.yml file:

```
env:
  parameter-store:         
    JWT_SECRET: JWT_SECRET
```

This lets CodeBuild know to set an evironment variable based on a value in the parameter-store.

5. Put secret into AWS Parameter Store
```
aws ssm put-parameter --name JWT_SECRET --value "YourJWTSecret" --type SecureString
```

6. Modify CloudFormation template.

There is file named ci-cd-codepipeline.cfn.yml, this the the template file you will use to create your CodePipeline pipeline. Open this file and go to the 'Parameters' section. These are parameters that will accept values when you create a stack. Fill in the 'Default' value for the following:

* EksClusterName : use the name of the EKS cluster you created above
* GitSourceRepo : use the name of your project's github repo.
* GitHubUser : use your github user name
* KubectlRoleName : use the name of the role you created for kubectl above

7. Create a stack for CodePipeline by applying CloudFormation template

8. Check the pipeline works. Once the stack is successfully created, commit a change to the master branch of your github repo. Then, in the aws console go to the CodePipeline UI. You should see that the build is running.

9. Test your endpoints.

To test your api endpoints, get the external ip for your service:
```
kubectl get services simple-jwt-api -o wide
```
Now use the external ip url to test the app:
```
export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST <EXTERNAL-IP URL>/auth  | jq -r '.token'`
curl --request GET '<EXTERNAL-IP URL>/contents' -H "Authorization: Bearer ${TOKEN}" | jq
```
