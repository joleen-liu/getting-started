# Managed AWS cluster

Create a Zetaris compute cluster in your AWS account, sign in to its Gateway and run a query. This guide follows the AWS setup, connection and teardown workflow demonstrated in the recordings on 13 October 2026.

For Docker Compose on your own computer, see [Install locally](updated_zetaris_installation_guide.md). For Docker Compose on a VM you manage, see [Self-managed EC2 and Docker Compose](self-managed-ec2-installation-guide.md).

## Before you start

- Sign in to Zetaris Cloud Gateway and the AWS account that will own the resources.
- Use an AWS identity with permission to create and delete the supplied CloudFormation stack and its resources. Review the IAM permissions requested by the stack with your AWS administrator if needed.
- Review the AWS cost notice under **Compute > Credentials**. AWS bills these resources directly to your AWS account, separately from your Zetaris tier. The estimate depends on its stated region, date and assumptions.

AWS account setup creates networking, a storage bucket and access roles. Billable resources can incur charges from setup completion until you delete them, even when no compute cluster is running. Suspending a cluster does not remove the account setup or retained storage.

## 1. Add and verify your AWS account

1. In Zetaris Cloud Gateway, open **Compute > Credentials**. The page is titled **AWS Accounts**.
2. Click **Add AWS Account**, select **Quickstart**, enter a friendly name for the account and click **Continue**.
3. Choose your **AWS region** from the supported options. Review the displayed AWS cost information, then click **Launch Stack**.
4. The AWS Console opens in a new tab on **Quick create stack**, with the template and parameters filled in. Check that you are signed in to the intended AWS account and that the region and stack details match the setup.
5. Review the IAM capabilities. Tick the acknowledgement that CloudFormation may create IAM resources with customised names, then click **Create stack**.
6. Follow the stack's progress in CloudFormation. Wait for the stack to reach `CREATE_COMPLETE`.
7. Return to the Zetaris setup tab, continue with **I've already launched the stack**, and complete **Verify AWS Account** using **Verify**.
8. Confirm that the account appears under **Compute > Credentials > Active** with status **Verified**. It is now available for cluster creation.

Keep the stack name and region with your session notes. You will need the corresponding stack during teardown. If creation or verification fails, inspect the CloudFormation events and the Zetaris verification message before continuing.

## 2. Create your compute cluster

1. Open **Compute > Clusters** and click **Create Cluster**.
2. In **AWS Account**, select the verified Quickstart account you just added.
3. Enter a **Cluster Name**. Keep this name for the deletion confirmation later.
4. Choose the **Spark cluster size**. The demonstrated session used **Mini**, a trial configuration with two nodes and four vCPUs in total. Use the current form's options and limits for your account. Node type and node count are fixed for Mini; other sizes expose their own sizing controls.
5. Review **Networking**. For a Quickstart account, the form reads the VPC and subnets from the CloudFormation stack outputs, so you do not need to select them manually.
6. Review the configuration and cost information shown for your account and selected size, then click **Create Cluster**.
7. Open the cluster's **Lifecycle** tab and wait for all seven provisioning stages to succeed. Continue when the cluster is **Active**.

Provisioning takes time. Keep the Lifecycle page open to follow progress. If a stage fails, use its displayed error and the cluster's events to resolve the failure before connecting.

## 3. Connect to the Gateway and run a query

1. Open **Compute > Clusters**, select your cluster and open **Overview**.
2. Under **Access**, open the **Sign-in URL**. This opens the Gateway for this cluster.
3. Under **Organisations** on the cluster Overview, find your organisation's administrator email and password. Use those credentials to sign in to the cluster Gateway. Keep them private and out of source files, prompts and recordings you share publicly.
4. In the Gateway, open **SQL Editor** and run:

```sql
SELECT 1
```

5. Confirm that the result contains `1`. This was demonstrated successfully in the recording and confirms that you can sign in and execute SQL on the cluster.

The **Access** section also supplies the cluster hostname and **JDBC endpoint**. If you connect a client or coding assistant, copy the endpoint from your own cluster and follow [Client setup](../connections/client-setup.md). Use the cluster's organisation credentials rather than the credentials from a local Docker installation.

Check the cluster's **Schedule** if you need it to remain available during your session. A scheduled suspension or the **Suspend** action pauses compute availability; it does not complete AWS cleanup.

## 4. Delete the compute cluster

1. In Zetaris Cloud Gateway, open **Compute > Clusters** and select your cluster.
2. Click **Delete**. In **Delete compute?**, read the warning and type the exact cluster name shown in the dialog, then click **Delete** to confirm.
3. Follow **Lifecycle > Termination Steps** while the status is **Terminating**. The displayed stages are **Draining nodes**, **Deleting cluster** and **Releasing resources**.
4. Wait for termination to finish. Return to **Compute > Clusters** and confirm that this cluster is no longer in the active list.

Cluster deletion terminates its EKS cluster and associated IAM roles. The confirmation dialog explicitly says that EFS and S3 are preserved. Keep any data you need before proceeding with the AWS account cleanup below.

## 5. Delete the CloudFormation stack and mark the credential deleted

Do this after the compute cluster has finished terminating. If other active clusters use the same AWS credential configuration, finish their teardown before removing that shared setup.

1. Open **Compute > Credentials** and find the AWS account used by the cluster. Click its delete icon.
2. The confirmation dialog displays the **Stack name** and **Region**. Click **Open in AWS CloudFormation** to open that exact stack in a new tab.
3. In AWS, check the account, region and stack name against the Zetaris dialog. Click **Delete stack**.
4. Read the AWS deletion warning, enter the stack name if prompted and confirm **Delete stack**. AWS deletes stack resources according to their deletion policies.
5. Wait for the stack status to reach `DELETE_COMPLETE`. If it reaches `DELETE_FAILED`, inspect the stack's **Events** and resolve the reported resource issue before continuing.
6. Return to the Zetaris credential deletion dialog and click **Mark Deleted**.
7. Confirm that the credential no longer appears under **Active**. Use the **Deleted** tab to review deleted configurations.

Zetaris cannot delete this CloudFormation stack for you. You must use your own AWS identity. **Mark Deleted** records the credential's deletion in Zetaris; it does not delete AWS resources.

## 6. Check retained resources and charges

Review the AWS account for the EFS file system, S3 data and any other retained resources from this setup. Stack deletion follows each resource's deletion policy, so do not assume that deleting the stack removed retained data.

Keep the data you need and remove billable resources you no longer need. AWS charges continue for any billable resources left in the account. Completion means the cluster has terminated, the stack has reached `DELETE_COMPLETE`, the credential is marked deleted, and you have checked the retained resources.

The managed AWS cost information is separate from the dated EC2 estimate in the [self-managed VM guide](self-managed-ec2-installation-guide.md). Use the current AWS account notice and your AWS billing information for this workflow.
