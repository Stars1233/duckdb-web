---
layout: docu
title: Quack on WebAssembly
---

## CloudFormation Stack

We provide an example template for initializing a CloudFormation stack, based on pre-baked public AMI, based at the moment on Ubuntu that will:
* via duckdb, install and load quack, create a randomized token, and do `quack_serve` on the 0.0.0.0:1294 port.
* setup via nginx a proxy between 0.0.0.0:1294 and the incoming :443 port
* obtain a valid TLS certificate via Let's Encrypt

All together, this allows to expose the local `0.0.0.0:1294` port to the public EC2 instance IP via HTTPS.

The template is reachable at <https://duckdb-quack-ami.s3.us-east-1.amazonaws.com/quack.yaml>, and is backed by images for (at the moment) 3 regions:

* `us-east-1`
* `us-east-2`
* `eu-central-1`

Only inputs are the type of the instance (default t3.micro) and the name of the stack (needs to be unique on the same org/region).

### Deploy via Web UI

1. Visit <https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https://duckdb-quack-ami.s3.us-east-1.amazonaws.com/quack.yaml&stackName=my-duckdb-quack-demo>
2. Log-in, select a name, possibly an instance class, and click **Launch Stack**.

> This will incur billing. Make sure you check the stack page afterward, and delete the non-relevant anymore stacks.

### Deploy via `aws` CLI

To deploy a Quack demo with the name `my-quack-demo` in the `us-east-2` region, run:

```bash
aws cloudformation create-stack \
  --stack-name my-quack-demo \
  --template-url https://duckdb-quack-ami.s3.us-east-1.amazonaws.com/quack.yaml \
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

To see the status of your CloudFormation stack on the AWS web user interface, visit the [CloudFormation console](https://console.aws.amazon.com/cloudformation/home).
