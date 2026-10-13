# Managed AWS cluster

Use this guide to create a managed Zetaris compute cluster in your AWS account from Zetaris Cloud. The setup uses an AWS CloudFormation Quickstart handoff to configure access and provision AWS resources. This path does not install the Docker Compose bundle or ask you to manage an EC2 virtual machine.

For Docker Compose on your own computer, see [Install locally](updated_zetaris_installation_guide.md). To install the bundle on an EC2 virtual machine that you manage, see [Self-managed EC2 and Docker Compose](self-managed-ec2-installation-guide.md).

## Before you start

- Sign in to Zetaris Cloud and to the AWS account that should own the resources.
- Make sure your AWS account can launch the supplied CloudFormation stack and acknowledge the IAM permissions it requests. Ask your AWS administrator if you cannot review or launch the stack.
- Review the current cluster cost estimate before provisioning. The estimate depends on the selected region and configuration; it is not a quote.

## 1. Configure AWS access

1. In Zetaris Cloud, start the AWS credential setup from the managed cluster flow and continue to its AWS Quickstart CloudFormation handoff.
2. In the AWS account that should own the cluster resources, review the account, region, stack details and requested IAM permissions. Acknowledge the IAM resource creation and submit the stack.
3. Wait for the CloudFormation operation to finish successfully. Return to Zetaris Cloud and finish registering the AWS credential configuration. Proceed to cluster creation only after that configuration is available to select.

If the stack does not finish successfully, resolve that setup before requesting a cluster. Do not treat the submitted stack request as a ready credential.

## 2. Configure and create the cluster

1. In Zetaris Cloud, start compute cluster creation using the AWS credential configuration you just set up.
2. Choose a region, cluster size and name from the options currently shown. The available choices can change; do not use the EC2 instance sizes in the separate self-managed VM guide as managed-cluster sizing.
3. Review the current estimate shown for the selected configuration. Check its region, date and included assumptions. AWS resource charges are billed to your AWS account. The estimate does not cover any separate Zetaris licence or platform charges; check your Zetaris account or agreement for those.
4. Submit the request and wait until all seven displayed provisioning stages report success and the cluster is ready. A submitted request or an in-progress stage does not mean the cluster is ready to use.

If the cluster screen offers a schedule or suspend/resume controls, use them for temporary compute availability. Suspension is not teardown: the CloudFormation stack, storage and other AWS resources may remain and may continue to incur charges.

## 3. Open the Gateway and verify access

1. Open the cluster's Gateway from its details in Zetaris Cloud and sign in using the account flow shown there.
2. In the Gateway SQL Editor, run this read-only query:

```sql
SELECT 1
```

A successful result confirms that you can sign in to the Gateway and run a query. The query creates no tables or other objects.

## 4. Tear down the cluster and AWS setup

Complete these steps in order:

1. In Zetaris Cloud, delete the compute cluster and wait for its deletion to finish.
2. In the AWS account used for setup, find the corresponding Quickstart CloudFormation stack and delete it. Wait for the stack deletion to finish.
3. After the stack deletion completes, return to Zetaris Cloud and delete the AWS credential configuration. Zetaris Cloud cannot delete a credential configuration while an active cluster still uses it; if deletion is blocked, check for clusters that still use that configuration.

Deleting the compute cluster can leave EFS file systems, S3 buckets or data, and other retained AWS resources. After the stack operation, review the AWS account for retained resources and remove only those you no longer need. Check data and backup retention before deleting EFS or S3 contents. AWS charges can continue for billable resources until they are removed. Cluster deletion alone does not confirm full AWS cleanup.

The managed-cluster estimate is separate from the dated EC2 and Docker Compose estimate in the [self-managed VM guide](self-managed-ec2-installation-guide.md). Check the current estimate and your AWS account for current resource costs.
