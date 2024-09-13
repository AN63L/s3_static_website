# Guide to setting up an S3 static website with Gitlab

Instructions to build a globally accessible S3 static website.

## Required resources

- Gitlab Repo
- Protected branches
- buildspec.yml file in the source repo
- CodeCommit Mirror Repo (global)
- S3 Website Bucket (same region as build bucket)
- S3 Build Bucket (same region as s3 website bucket)
- ACM Certificate 
- Cloudfront Distribution (global)
- Route 53 to point to CloudFront distribution (global)
- Codebuild project
- CodePipeline

You may want to create a specific bucket for logging

## Pipeline structure

The code of the <secure branch> branch is mirrored inside CodeCommit in a repository 

A new push to the <secure branch> branch will trigger a push to the CodeCommit repo and subsequently the CodePipeline 

During the build phase, the new build code (dist folder) will be uploaded to the S3 bucket and the cloudfront distribution will be invalidated to force a renewal of the cached data.

## Repositories

The main repository for your code should be stored in Gitlab. You will create a mirror for this repository in CodeCommit. View the lambda setup instructions if you have trouble with this. Don’t forget to protect your main branches !

## Source files & buildspec.yml

The source code must contain a file called buildspec.yml. This buidlspec file will build the source code, upload it to s3 and invalidate the cloudfront distribution. Here is a sample: 

```
version: 0.2

phases:
  install:
    commands:
      - npm i npm@latest -g
      - pip install --upgrade pip
      - pip install --upgrade awscli
  pre_build:
    commands:
      - echo Pre_build Phase
      - npm install
  build:
    commands:
      - echo Build Phase
      - npm run generate
  post_build:
    commands:
      - echo PostBuild Phase
      - aws s3 sync ./dist s3://bucket.com
      - aws cloudfront create-invalidation --distribution-id "DISTRIBUTION_ID" --paths "/*"
```

## S3 Buckets

### Build bucket

Like the lambda pipelines, you need to create a specific s3 bucket for Codepipeline to put the artifacts in. 

### Static Website Bucket

1. Create bucket: 
- With YOUR DOMAIN NAME (such as www.domain.com)
  - Do not enable bucket versioning
  - Have default encryption enabled
  - (optional) enable server access logging 
  - No transfer acceleration
  - No requester pays

1. Enable static website hosting:
   - from properties tab
   - Index and Error document must be the same. By default it is index.html but it might depend (with nuxt, a build results in a 200.html file for example)

2. Enable correct permissions
- Enable public access from permissions tab
- Add the following bucket policy (DO NOT FORGET TO UPDATE THE BUCKET URI):
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::bucket.com/*"
        }
    ]
}
```

- Leave ACL as default
- Leave CORS as default

### CDN - Cloudfront & Route53

Route53 handles the DNS configuration and Cloudfront the CDN. Certificates are managed via ACM.

1. Create a cloudfront distribution:
- NEVER SELECT A PROVIDED INPUT FOR THE ORIGIN !!!! It must follow the format:  http://domain.com.s3-website.eu-west-1.amazonaws.com
- No origin shield
- Use all edge locations
- Use default HTTP versions
- Add recommended TLS security policy
- Redirect HTTP to HTTPS
- Compress objects automatically
- Allow all HTTP Methods, don’t cache option
- Use caching optimized cache policy
- (optional) Do not add response headers policy => this will be handled by an edge lambda (create a lambda for that if you haven't already)
- (optional) Add to a cloud distribution ACL if you have one
- (optional) In functions association, add in origin response the edge lambda + check the latest version

2. Create your certificate in us-east-1 to ensure that Cloudfront can use it. 

3. You will need to create a DNS entry pointing to your cloudfront distribution in route 53 (use the helper if needed)

4. Test you configuration via these tools: 
- [securityheaders.com](https://securityheaders.com/)
- [Mozilla observatory](https://observatory.mozilla.org/analyze/avumi.com)

Adding the security headers using a default header response from Cloudfront will result in partial headers being provided. For example, CSP and Permissions-Policy headers are not included by cloudfront and cannot be added as custom headers. The only way to avoid this is to use lambdas to add the relevant security headers.

This will work but is incomplete: https://miketabor.com/how-to-add-security-headers-to-an-aws-s3-static-website/