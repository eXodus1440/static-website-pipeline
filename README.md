# static-website-pipeline
Static website pipeline based on Jekyll



First you need to deploy the lambda function which will handle simple URLs for the CloudFront Distribution
```
aws cloudformation deploy \
  --template-file jekyll-lambda-edge.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --stack-name cloudfront-simple-urls \
  --region us-east-1
```

Grab the lambda ARN with
```
aws cloudformation describe-stacks --stack-name cloudfront-simple-urls --region us-east-1 --output text --query 'Stacks[*].Outputs[*].[ExportName,OutputValue]'
```

Deploy the Jekyll Pipeline, replacing the parameter overrides values with your own
Update <LambdaEdgeFunctionARN> with details from the above command
```
aws cloudformation deploy \
  --template-file jekyll-build-pipeline.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --stack-name my-jekyll-website \
  --parameter-overrides \
    LambdaEdgeStackARN=<LambdaEdgeFunctionARN> \
    DomainName=jekyll-demo.com \
    WebsiteURL=www.jekyll-demo.com \
  --region eu-west-1
```

The new pipeline details can be grabbed with
```
aws cloudformation describe-stacks --stack-name my-jekyll-website --region eu-west-1 --output text --query 'Stacks[*].Outputs[*].[OutputKey,OutputValue]'
```

Create a user, add to the repo-rw group, and generate credentials to access the CodeCommit repo via HTTPS
Update <CodeCommitGroup> with details from the above command
```
aws iam create-user --user-name Steve
aws iam add-user-to-group --group-name <CodeCommitGroup> --user-name Steve
aws iam create-service-specific-credential --user-name Steve --service-name codecommit.amazonaws.com
aws iam create-access-key --user-name Steve --output text --query 'AccessKey.[UserName,AccessKeyId,SecretAccessKey]'
```

Add the newly created access-key and secret-key to your awscli credentials file as a new profile
```
[default]
aws_access_key_id = <your default aws_access_key_id>
aws_secret_access_key = <your default aws_secret_access_key>

[codecommit]
aws_access_key_id = <newly created aws_access_key_id>
aws_secret_access_key = <newly created aws_secret_access_key>
```

Set up the credential helper
[https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html]
[https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-windows.html]
```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git config --global credential.helper "!aws --profile codecommit codecommit credential-helper $@"
git config --global credential.UseHttpPath true
```
Note: you can configure profiles per repository instead of globally by using ```--local``` instead of ```--global```

Connect to the CodeCommit repository and create a local clone
```
git clone https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/my-jekyll-website-repo my-jekyll-website
```

# Create new Jekyll site
docker run -v .:/srv/jekyll jekyll/jekyll new #ToDo test/verify command from machine with docker

To kick off the build pipeline, we'll need to commit the CodeBuild buildspec and Jekyll website into the repo
```
git add .
git commit -m "initial commit"
git push
```

# Testing the new Jekyll Website
Browse to the Staging S3 bucket URL
Browse to the CloudFront URL
Browse to your website URL/domain apex

# Removing the Website and Pipeline

Start with removing the user account
[https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_manage.html#id_users_deleting_cli]
[https://vrnchndk.in/2017/01/17/delete-iam-user-completely-using-aws-cli/]
```
aws iam list-access-keys --user-name Steve --output text --query 'AccessKeyMetadata[*].AccessKeyId'
aws iam delete-access-key --user-name Steve --access-key-id <AccessKeyId>
aws iam list-service-specific-credentials --user-name Steve --output text --query 'ServiceSpecificCredentials[*].ServiceSpecificCredentialId'
aws iam delete-service-specific-credential --user-name Steve --service-specific-credential-id <ServiceSpecificCredentialId>
aws iam list-groups-for-user --user-name Steve --output text --query 'Groups[*].GroupName'
aws iam remove-user-from-group --group-name my-jekyll-website-codecommit-repo-rw --user-name Steve
aws iam delete-user --user-name Steve
```

Next, empty the S3 buckets
```
aws s3 rm s3://my-jekyll-website-website-staging --recursive
aws s3 rm s3://my-jekyll-website-website-prod --recursive
aws s3 rm s3://my-jekyll-website-codepipeline-artifacts --recursive
```
Note: This will remove all website and CodePipeline build assets so make sure you've created a backup if you need to retain those items.

Then, remove the CloudFormation Stack
```
aws cloudformation delete-stack \
  --stack-name my-jekyll-website
  --region eu-west-1
```
Note: This will remove all infrastructure including the CodeCommit repo, so make sure you have a backup if you need to retain that code.

Verify that no other CloudFront Distros are using the Lambda@Edge function
```
aws cloudfront list-distributions --query 'DistributionList.Items[*].[Id,DefaultCacheBehavior.LambdaFunctionAssociations.Items[*].LambdaFunctionARN]'
```

Finally, remove the Simplify-URL Lambda function stack
```
aws cloudformation delete-stack \
  --stack-name cloudfront-simple-urls \
  --region us-east-1
```

aws cloudformation delete-stack --stack-name my-jekyll-website
aws cloudformation delete-stack --stack-name cloudfront-simple-urls2 --region us-east-1

#ToDo - add approval trigger from staging to production
#ToDo - add budget and budget alerting
#ToDo - add logging and lifecycle to keep storage down
