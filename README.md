# tvcio

[tvc.io](https://tvc.io) hugo site, hosted on S3.
Will hopefully be a blog if I get round to posting anything.


### Cloudformation Deployment

Run the following locally

```
aws cloudformation update-stack \
  --stack-name <<insert name>>  \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=AcmCertificateArn,UsePreviousValue=true \
  --template-body "$(cat cf.yml)"
```