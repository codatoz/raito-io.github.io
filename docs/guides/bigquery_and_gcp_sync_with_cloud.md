---
title: Google Cloud BigQuery - First sync with Raito Cloud
nav_order: 20
parent: Guides
permalink: /docs/guide/bigquery
---

# Synchronizing Google Bigquery for the first time

In this guide, we'll walk you through an example of how to connect Raito Cloud to your BigQuery data warehouse through the Raito CLI. We'll 
- make sure that Raito CLI is installed and available
- log into Raito Cloud and create a data source
- create a service account and assign the correct IAM Roles
- configure Raito CLI to connect to Raito Cloud and synchronize with the previously-created data source
- run a first sync
  
For this guide, you will need access to Raito Cloud and you also need access to GCP and optionally GSuite
If you don't have access to Raito Cloud yet, [request a trial](https://www.raito.io/trial){:target="_blank"}. 
If you don't have access to GCP, you can [request a 30-day free trial](https://cloud.google.com/docs/get-started){:target="_blank"}. 


## Raito CLI installation

To install the Raito CLI, simply run the following command in a terminal window:
```bash
$> brew install raito-io/tap/cli
```

Check that everything is correctly installed by running
```bash
$> raito --version
```

If you want more information about the installation process, or you need to troubleshoot an issue, you can find [more information here](/docs/cli/installation). 

## Create a data source in Raito Cloud

Now that the CLI is working, sign in to your Raito Cloud instance. 

In the left navigation pane, go to `Data Sources` > `All data sources`. You should see a button on the top-right named `Add data source`. This will guide you through a short wizard to create a new data source. The main things that you will need to configure are: 

* `Data source type`. Select Google Cloud Platform.
* `Data source name`. Give your data source a good descriptive name, separating it from other data sources. For this example, we'll choose 'Google Cloud Test'.
* `Data source description`. Accompany your data source with a meaningful description.
* `Connection method`. Select whether you want to use the Raito hosted cloud version of the CLI or one managed by yourself, which is recommended. In this example we indeed select 'CLI'

Now that we have our GCP Data Source set up, repeat the same step to create a data source of type BigQuery. You will notice that this time, the wizard has an additional step `Select a Google Cloud Platform data source` where you will have to select the data source created in the previous step. This will ensure proper sharing of identities across your various BigQuery projects as well as other GCP service data sources. Note that this selection is optional.

## Create a GCP service account 

If you do not have a service account created already, you can go ahead and create one either in the Google Cloud console or through the command line in on of your GCP projects.

```sh
gcloud iam service-accounts create raito-cli --display-name="Service account for raito-cli"
```

Then obtain a key file for that service account

```sh
gcloud iam service-accounts keys create ${JSON_KEY_PATH} --iam-account=raito-cli@${PROJECT_ID}.iam.gserviceaccount.com
```

Now we need to assign this service account the correct roles so it can run the Raito BigQuery and GCP connectors. For the GCP connector, we create a custom role with the necessary permissions (or you can choose to use an existing role with these permissions):

Add the following to raito-role.yaml

```yaml
title: "RaitoGcpRole"
description: "Role for the Raito GCP connector"
stage: GA
includedPermissions:
- resourcemanager.organizations.get
- resourcemanager.organizations.getIamPolicy
- resourcemanager.organizations.setIamPolicy
- resourcemanager.folders.get
- resourcemanager.folders.list
- resourcemanager.folders.getIamPolicy
- resourcemanager.folders.setIamPolicy
- resourcemanager.projects.get
- resourcemanager.projects.list
- resourcemanager.projects.getIamPolicy
- resourcemanager.projects.setIamPolicy
- iam.serviceAccounts.list
```

Now create the role using this definition using your GCP organization ID, and assign it to the service account

```sh
gcloud iam roles create RaitoGcpRole --organization=${ORGANIZATION_ID} --file=${YAML_FILE_PATH}

gcloud organizations add-iam-policy-binding ${ORGANIZATION_ID} \
    --member=serviceAccount:raito-cli@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=organizations/${ORGANIZATION_ID}/roles/RaitoGcpRole
```

In the GCP project containing the BigQuery assets to be imported into Raito cloud, you will also need to grant the ServiceAccount BigQuery permissions.
To do so we create an additional custom role.

Add the following to raito-bigquery-role.yaml

```yaml
title: "RaitoBigQueryRole"
description: "Role for the Raito BigQuery connector"
stage: GA
includedPermissions:
- bigquery.datasets.createTagBinding
- bigquery.datasets.deleteTagBinding
- bigquery.datasets.get
- bigquery.datasets.getIamPolicy
- bigquery.datasets.listEffectiveTags
- bigquery.datasets.listSharedDatasetUsage
- bigquery.datasets.listTagBindings
- bigquery.datasets.setIamPolicy
- bigquery.datasets.updateTag
- bigquery.jobs.create
- bigquery.jobs.get
- bigquery.jobs.list
- bigquery.jobs.listAll
- bigquery.jobs.listExecutionMetadata
- bigquery.jobs.update
- bigquery.routines.get
- bigquery.routines.list
- bigquery.rowAccessPolicies.create
- bigquery.rowAccessPolicies.delete
- bigquery.rowAccessPolicies.getIamPolicy
- bigquery.rowAccessPolicies.list
- bigquery.rowAccessPolicies.setIamPolicy
- bigquery.rowAccessPolicies.update
- bigquery.tables.createSnapshot
- bigquery.tables.createTagBinding
- bigquery.tables.deleteTagBinding
- bigquery.tables.get
- bigquery.tables.getData
- bigquery.tables.getIamPolicy
- bigquery.tables.list
- bigquery.tables.listEffectiveTags
- bigquery.tables.listTagBindings
- bigquery.tables.setCategory
- bigquery.tables.setColumnDataPolicy
- bigquery.tables.setIamPolicy
- bigquery.tables.updateTag
- resourcemanager.projects.get
- resourcemanager.projects.list
```

Now create the role using this definition using your GCP organization ID, and assign it to the service account

```sh
gcloud iam roles create RaitoBigQueryRole --organization=${ORGANIZATION_ID} --file=${YAML_FILE_PATH_BIGQUERY}

gcloud organizations add-iam-policy-binding ${ORGANIZATION_ID} \
    --member=serviceAccount:raito-cli@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=organizations/${ORGANIZATION_ID}/roles/RaitoBigQueryRole
```

## Set up domain-wide delegation in GSuite

Domain-wide delegation is required for the service account created in the previous step to access the GSuite directory API on behalf of one of your administrators. This is needed to read users, groups and group memberships and import them into Raito Cloud. To do this, 
* Visit https://admin.google.com/
* Go to Security >> Access and data control >> API controls 
* Under Domain wide delegation go to `Manage domain wide delegation`
* Click on add new
* Fill in the client ID of the service account you previously created (typically a number with 21 digits)
* As oAuth scopes, provide `https://www.googleapis.com/auth/admin.directory.group.readonly,https://www.googleapis.com/auth/admin.directory.group.member.readonly,https://www.googleapis.com/auth/admin.directory.user.readonly`
* Click `authorize` and verify if your new entry is present in the API clients table
* Finally go to `Account >> Account Settings` and take note of the Customer ID

## Raito CLI Configuration

To configure your Raito CLI to synchronize your GCP organization and BigQuery assets, start by creating a file with the name `raito.yml` and edit it to look like this:

{% raw %}
```yaml
api-user: "{{RAITO_USER}}"
api-secret: "{{RAITO_API_KEY}}"
domain: "{{DOMAIN}}"

targets:
  - name: gcp
    connector-name: raito-io/cli-plugin-gcp/gcp
    connector-version: latest
    data-source-id: "<data-source-id>"
    identity-store-id: "<identity-store-id>"
    
    gcp-serviceaccount-json-location: <gcp-service-account-json>
    gcp-organization-id: <organization-id>

    gsuite-identity-store-sync: true
    gsuite-customer-id: <gsuite-customer-id>
    gsuite-impersonate-subject: <admin@gsuite.domain>

  - name: bigquery1
    connector-name: raito-io/cli-plugin-gcp/bigquery
    connector-version: latest
    data-source-id: "<data-source-id>"
    identity-store-id: "<identity-store-id>"
    
    gcp-serviceaccount-json-location: <gcp-service-account-json>
    gcp-project-id: <project-id>    
```
{% endraw %}

It contains
- a section to configure the connection to Raito Cloud: `api-user`, `api-secret`, and `domain`. `domain` is the part of the URL from your Raito Cloud instance (e.g. https://`domain`.raito.cloud). `api-user` and `api-secret` are the login credentials for your Raito Cloud instance.
- `targets` has one target defined for the GCP connector and one for BigQuery. You can copy paste this section from the snippet that is shown on the page of the respective newly created data source in Raito cloud. The first part defines the target, connector and corresponding object ID's in Raito Cloud (i.e. `data-source-id` and `identity-store-id`). The second part is the configuration specific to the connectors.

Feel free to customize this configuration further. Find more information in the sections about [general configuration](/docs/cli/configuration#command-specific-parameters),  [GCP-specific configuration](/docs/cli/connectors/googlecloud/platform#gcp-specific-cli-parameters) and [BigQuery-specific configuration](/docs/cli/connectors/googlecloud/bigquery#bigquery-specific-cli-parameters). 
Remember that you can use double curly brackets to reference environment variables, like we did for the `api-user` field and others in the example.

## Raito run

Now that our data source is set up and we have our Raito CLI configuration file, we can run the Raito CLI with:

```bash
$> raito run
```

This will download all data objects, users, access controls (nameless bindings) and data usage information from BigQuery and GCP and upload it to Raito Cloud. It will also get the access controls created in Raito Cloud and push them as nameless bindings to BigQuery and GCP, but since you've started out with an empty instance, this is not relevant at this point. 

See [here](/docs/cli/intro) for more information about what happens exactly. 

## Check results in Raito Cloud

When the `raito run` command finished successfully, go back to Raito Cloud. 

On the dashboard you will now see some initial insights that we extract from the data that was synchronized. If you go to `Data Sources` and visit the data sources that you have created before, you should be able to see when the last sync was done in the `General information` section. When you scroll down, you can also navigate through the data objects in your BigQuery warehouse.

When you go to `Identities` in the navigation bar, you can see all the users imported from GSuite. Under `Access Controls`, under grants, you have an overview of all the IAM Role grants both on GCP organization level as well as your BigQuery tables and datasets. If you click on one, you get a detailed view of who belongs to that access control, and what they have access to with which permissions. 

Now that you have synchronized your GCP organization and the first BigQuery project, you can repeat the steps to connect all your other BigQuery projects to the same GCP data source by creating new BigQuery data sources in Raito and configuring them using the same steps as before.

{% include slack.html %}
