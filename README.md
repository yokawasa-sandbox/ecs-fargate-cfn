# ecs-fargate-cfn
test project for ecs fargate. The code in this repository is largely based on [awstut-fa](https://github.com/awstut-an-r/awstut-fa/tree/main/018) (relevant [article](https://awstut.com/en/2022/01/25/introduction-to-fargate-with-cloudformation/)).

## Architecture

![fa-018-diagram](https://user-images.githubusercontent.com/84276199/190931404-d2cacdf3-98c6-4e7d-887b-91ede36de44e.png)

## Requirements

* AWS CLI
* S3 Bucket(Here, the bucket name is *my-bucket* and region is *ap-northeast-1*)

## Usage

### Tempalte File Modification

Modify the following locations in fa-018.yaml.

```yaml
Parameters:
  TemplateBucketName:
    Type: String
    Default: my-ecs-fargate-cfn-template
```

### Upload  Template Files to S3 Bucket

```bash
export BUCKET_NAME=my-ecs-fargate-cfn-template

# Create s3 Bucket
aws s3api create-bucket --bucket ${BUCKET_NAME} \
    --region ap-northeast-1 --create-bucket-configuration LocationConstraint=ap-northeast-1

# Upload Cfn template files to s3 bucket
cd cloudformation
aws s3 cp . s3://${BUCKET_NAME}/ecs-fargate-cfn/ --recursive
```

### CloudFormation Stack Creation

#### Create ECR stack

```bash
aws cloudformation create-stack \
--stack-name ecs-fargate-cfn-ecr \
--template-url https://${BUCKET_NAME}.s3.ap-northeast-1.amazonaws.com/ecs-fargate-cfn/ecr.yaml
```

#### Push Image to ECR Repository

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com

cd task

docker build -t ecs-fargate-cfn .

docker tag ecs-fargate-cfn:latest ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/ecs-fargate-cfn:latest
```

Check if the container image that you've built can work as expected

```
docker run -d -p 8888:80 ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/ecs-fargate-cfn   
curl localhost:8888
```

Here is an expected output:

```
<html>
  <head>
  </head>
  <body>
    <h1>ecs-fargate-cfn index.html</h1>
  </body>
</html>
```

Finally push the image to the ECR

```bash
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/ecs-fargate-cfn:latest
```

#### Create other stacks to run ECS fargate tasks

```bash
aws cloudformation create-stack \
--stack-name ecs-fargate-cfn \
--template-url https://${BUCKET_NAME}.s3.ap-northeast-1.amazonaws.com/ecs-fargate-cfn/main.yaml \
--capabilities CAPABILITY_IAM
```

After checking the resources for each stack, the information for the main resources created this time is as follows

- ECS cluster: `ecs-fargate-cfn-cluster`
- ECS service 1 name: `ecs-fargate-cfn-service1`
- ECS service 2 name: `ecs-fargate-cfn-service2`

You can see that there are two services running in the cluster.

