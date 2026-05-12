---
layout: docu
title: Quack on WebAssembly
---

## CloudFormation Stack

We provide an example template for initializing a CloudFormation stack.
Please [inspect the content of the file](https://test-wasm-carlo.s3.us-east-1.amazonaws.com/duckdb-quack-ec2-template.yaml) before using it.

### Deploy

To deploy a Quack demo with the name `my-quack-demo` in the `us-east-2` region, run:

```bash
aws cloudformation create-stack \
  --stack-name my-quack-demo \
  --template-url https://test-wasm-carlo.s3.us-east-1.amazonaws.com/duckdb-quack-ec2-template.yaml \
  --region us-east-2
```

Wait approx. 2 minutes for the deployment to complete.

### Destroy

```bash
aws cloudformation delete-stack \
  --stack-name my-quack-demo \
  --region us-east-2
```

### Web UI

To see the status of your CloudFormation stack on the AWS web user inferface, visit the [CloudFormation console](https://console.aws.amazon.com/cloudformation/home).
