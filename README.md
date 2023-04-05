# Bootstrap ArgoCD via Terraform Cloud Notifications

A [Harness CI](https://www.harness.io/products/continuous-integration) pipeline that allows bootstrapping [Argo CD](https://argo-cd.readthedocs.io/en/stable/) once terraform completes the provisioning, in this demo we will provision a GKE cluster using terraform.

## Prerequisites

- [Terraform Cloud Account](https://app.terraform.io/public/signup/account)
- [Google Cloud Account](https://cloud.google.com)
- [Harness Free Tier](https://app.harness.io/auth/#/signup/?module=ci&utm_source=internal&utm_medium=social&utm_campaign=community&utm_content=kamesh-ci-demos&utm_term=tutorial)

## Git Repositories

The demo uses the following git repositories a sources,

- [vanilla-gke](https://github.com/harness-apps/vanilla-gke) - the terraform source repository that will be used with terraform cloud to provision the GKE.
- [bootstrap-argocd](https://github.com/harness-apps/bootstrap-gke) - the repository that holds kubernetes manifests to bootstrap argo CD on to the GKE cluster
  
Please make sure you fork those repositories to your Git accounts. For reference sake we will refer the repositories as `$TFC_GKE_REPO` and `$ARGOCD_BOOTSTRAP_REPO`.

### Fork and Clone the Repositories

To make fork and clone easier we will use [gh CLI](https://cli.github.com/)

Navigate to the directory where you want to clone the sources typically it will be `$HOME/git`.

Clone and fork `vanilla-gke` repo,

```shell
gh repo clone harness-apps/vanilla-gke
cd vanilla-gke
gh repo fork
export TFC_GKE_REPO="$PWD"
```

Clone and fork `bootstrap-argocd` repo,

```shell
cd ..
gh repo clone harness-apps/bootstrap-argocd
cd bootstrap-argocd
gh repo fork
export ARGOCD_BOOTSTRAP_REPO="$PWD"
```

## Harness CI

If you don't have Harness Free Tier account, please do sign up for one via [Harness Free Tier Sign Up](https://app.harness.io/auth/#/signup/?module=ci&utm_source=internal&utm_medium=social&utm_campaign=community&utm_content=kamesh-ci-demos&utm_term=tutorial).

### Create Harness Project

We will be using a project named `terraform_integration_demos`, let us create the project using Harness Web Console,

![New Harness Project](./docs/images/new_harness_project.png)

Update the details as shown,

![New Harness Project Details](./docs/images/new_harness_project_details.png)

Follow the wizard leaving rest to defaults and on the last screen choose **Continuous Integration**,

![Use CI](docs/images/use_ci_module.png)

Click **Go to Module** to go to project home page.

### Define New Pipeline

Click **Pipelines** to define a new pipeline,

![Get Started with CI](docs/images/get_started_with_ci.png)

For this demo will be doing manual clone, hence disable the clone,

![Disable](docs/images/disable_clone.png)

Click on **Pipelines** and delete the default **Build pipeline**,

![Delete Pipeline](docs/images/delete_pipeline.png)

### Add `harnessImage` Docker Registry Connector

As part of pipelines we will be pulling image from DockerHub. Let us configure an `harnesImage` connector as described in <https://developer.harness.io/docs/platform/connectors/connect-to-harness-container-image-registry-using-docker-connector/>. The pipelines we create as part of the later section will use this connector.

### Configure GitHub

#### GitHub Credentials

Create a [GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) for the account where you have have forked the repositories `$TFC_GKE_REPO` and `$ARGOCD_BOOTSTRAP_REPO`. We will refer to the token as `$GITHUB_PAT`.

From the **Project Setup** click **Secrets**,

![New Text Secret](docs/images/new_text_secret.png)

Update the encrypted text secret details as shown,

![GitHub PAT Secret](docs/images/github_pat_secret_details.png)

Click **Save** to save the secret,

![Project Secrets](docs/images/project_secrets_list_1.png)

#### Connector

As we need to clone the sources from GitHub, we need to define a **GitHub Connector**,  from the **Project Setup** click **Connectors**,

![New Connector](docs/images/new_connector.png)

From connector list select **GitHub**,

![New GitHub Connector](docs/images/new_github_connector.png)

Enter the name as **GitHub**,

![GitHub Connector Overview](docs/images/github_connector_overview.png)

Click **Continue** to enter the connector details,

![GitHub Connector Details](docs/images/github_connector_details.png)

Click **Continue** and update the GitHub Connector credentials,

![GitHub Connector Credentials](docs/images/gh_connector_credentials.png)

When selecting the **Personal Access Token** make sure you select the `GitHub PAT` that we defined in previous section,

![GitHub PAT Secret](docs/images/gh_project_secret.png)

Click **Continue** and use select **Connect through Harness Platform**,

![Connect through Harness Platform](docs/images/use_harness_platform.png)

Click **Save and Continue** to run the connection test, if all went well the connection should successful,

![GH Connection Success](docs/images/gh_connector_success.png)

## Google Cloud Service Account Secret

We need Google Service Account(GSA) credentials(JSON Key) to query the GKE cluster details and create resources on it.

### Set environment

```shell
export GCP_PROJECT="the Google Cloud Project where Kubernetes Cluster is created"
export GSA_KEY_FILE="path where to store the key file"
```

### Create SA

```shell
gcloud iam service-accounts create gke-user \
  --description "GKE User" \
  --display-name "gke-user"
```

### IAM Binding

Add permissions to the user to be able to provision kubernetes resources,

```shell
gcloud projects add-iam-policy-binding $GCP_PROJECT \
  --member="serviceAccount:$GSA_NAME@$GCP_PROJECT.iam.gserviceaccount.com" \
  --role="roles/container.admin"
```

### Download And Save GSA Key

> IMPORTANT: Only security admins can create the JSON keys. Ensure the Google Cloud user you are using has **Security Admin** role.

```shell
gcloud iam service-accounts keys create "${GSA_KEY_FILE}" \
    --iam-account="gke-user@${GCP_PROJECT}.iam.gserviceaccount.com"
```

### GSA Secret

Get back to the **Project Setup** click **Secrets**,

![New File Secret](docs/images/new_file_secret.png)

Add the GSA secret details as shown,

![GSA Secret Details](docs/images/gsa_secret_details.png)

>**IMPORTANT**: When you browse and select make sure you select the `$GSA_KEY_FILE` as the file for the secret.

Click **Save** to save the secret,

![Project Secrets](docs/images/project_secrets_list_2.png)

## Terraform Workspace

On your terraform cloud account create a new workspace called **vanilla-gke**. Update the workspace settings to use Version Control and make it point to `$TFC_GKE_REPO`.

![TFC Workspace VCS](docs/images/tfc_workspace_vcs.png)

Configure the workspace with following variables,

![TFC Workspace Variables](docs/images/tfc_workspace_variables.png)

For more details on available variables, check [Terraform Inputs](https://github.com/harness-apps/vanilla-gke#inputs).

>**IMPORTANT**: The `GOOGLE_CREDENTIALS` is Google Service Account JSON Key with permissions to create GKE cluster. Please check the <https://github.com/harness-apps/vanilla-gke#pre-requisites> for the required roles and permissions. This key will be used by Terraform to create the GKE cluster. When you add the key to terraform variables, you need to make it as base64 encoded e.g. `cat YOUR_GOOGLE_CREDENTIALS_KEY_FILE | tr -d \\n`

Going forward we will refer to the Terraform Workspace as `$TF_WORKSPACE`

Your terraform organization as `$TF_CLOUD_ORGANIZATION`,

![TFC Cloud Organization](docs/images/tfc_cloud_org.png)

Since we need to pull the outputs of terraform run in our CI pipeline we need Terraform API Token.

From your terraform user settings **Create an API token**,

![Terraform API Token](docs/images/tfc_api_token.png)

Make sure to copy the value and save it, we will refer to this token value as `$TF_TOKEN_app_terraform_io` as part of the Harness CI pipeline.

## Harness CI Pipeline

Getting back to Harness web console, navigate to your project **terraform_integration_demos**, click **Pipelines** and **Create a Pipeline** --> **Import From Git**,

![New CI Pipeline Import](docs/images/new_pipeline_import.png)

Update the pipeline details as shown,

![Pipeline Details](docs/images/import_pipeline_details.png)

>**IMPORTANT**: Make sure the **Name** of the pipeline is `bootstrap argocd pipeline` to make the import succeed with defaults.

![Pipeline Import Successful](docs/images/import_pipeline_successful.png)

Click the `bootstrap argocd pipeline` from the list to open the **Pipeline Studio** and click on the stage **Bootstrap Argo CD** to bring up the pipeline steps,

![Pipeline Steps](docs/images/pipeline_stages_and_steps.png)

You can click on each step to see the details.

The Pipeline uses the following secrets,

- `google_application_credentials` - the GSA credentials to manipulate GKE
- `terraform_cloud_api_token` - the value of `$TF_TOKEN_app_terraform_io`
- `terraform_workspace` - the value `$TF_WORKSPACE`
- `terraform_cloud_organization` - the value `$TF_CLOUD_ORGANIZATION`

We already added `google_application_credentials` secret as part of the [earlier](#gsa-secret) section. Following the similar pattern let us add the `terraform_cloud_api_token`, `terraform_workspace` and `terraform_cloud_organization` as text secrets.

>**HINT**:
> From the **Project Setup** click **Secrets**,
>
>![New Text Secret](docs/images/new_text_secret.png)
>

![all terraform secrets](docs/images/tfc_secrets.png)

## Notification Trigger

For the Harness CI pipelines to listen to Terraform Cloud Events we need to define a **Trigger**, navigate back to pipelines and select the **bootstrap argocd pipeline**  --> **Triggers**,

![Pipeline Triggers](docs/images/pipeline_triggers.png)

Click **Add New Trigger** to add a new webhook trigger(Type: `Custom`),

![Custom Webhook Trigger](docs/images/custom_webhook_trigger.png)

On the **Configuration** page enter the name of the trigger to be `tfc notification`,

![TFC Notification Config](docs/images/tfc_notification_config.png)

Leave rest of the fields to defaults and click **Continue**, leave the **Conditions** to defaults and click **Continue**.

On the **Pipeline Input** update the **Pipeline Reference Branch** to be set to **main**

![Pipeline Input](docs/images/trigger_pipeline_input.png)

>**NOTE**: The **Pipeline Reference Branch** does not have any implication with this demo as we do manual clone of resources.

Click **Create Trigger** to create and save the trigger.

![Trigger List](docs/images/trigger_list.png)

### Copy Webhook URL

Copy the WebHook url,

![Webhook URL](docs/images/copy_webhook_url.png)

Let us refer to this value as `$TRIGGER_WEBHOOK_URL`.

## Terraform Notification

On your terraform cloud console navigate to the workspace **Settings** --> **Notifications**,

![TFC Notifications](docs/images/tfc_notification.png)

Click **Create Notification** and select **Webhook** as the **Destination**,

![Webhook](docs/images/tfc_webhook_dest.png)

Update the notification details as shown,

![TFC Webhook Details](docs/images/tfc_webhook_notification_details.png)

Since we need to bootstrap argo CD only on create events we set the triggers to happen only on **Completed**,

![Trigger Events](docs/images/tfc_webhook_triggers.png)

Click **Create Notification** to finish the creation of notification.

![TFC Webhook Creation Success](docs/images/tfc_webhook_notification_success.png)

>**NOTE**: The creation would have fired a notification, if the cluster is not ready yet the pipeline would have failed.

**Congratulations!!**. With this setup any new or updates thats done to the `$TFC_GKE_REPO` will trigger a plan and apply on Terraform Cloud. A **Completed** plan will trigger the `bootstrap argocd pipline` to run and apply the manifests from `$BOOTSTRAP_ARGOCD_REPO` on the GKE cluster.

An example of successful pipeline run

![Pipeline Success](docs/images/successful_pipeline.png)

## References

- [Terraform Cloud Variables](https://developer.hashicorp.com/terraform/cloud-docs/api-docs/variables)
- [Terraform Cloud Variable Set](https://developer.hashicorp.com/terraform/cloud-docs/api-docs/variable-sets)
- [Terraform Notifications](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/notifications)

## License

[Apache 2.0](./LICENSE)
