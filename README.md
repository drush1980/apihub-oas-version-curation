# Apigee API hub: API Version Curation using OAS

This repository provides an example of a custom curation process for [Apigee API hub](https://cloud.google.com/apigee/docs/apihub/what-is-api-hub), built using Google [Application Integration](https://cloud.google.com/application-integration/docs/overview). 

## Background

API hub's [Curation](https://cloud.google.com/apigee/docs/apihub/curations) feature allows users to transform and enrich API metadata that is ingested from Apigee projects or [hub plugins](https://cloud.google.com/apigee/docs/apihub/plugins).

A typical use case for curation involves retrieving additional metadata from an ancillary system while importing APIs from the primary source. For example, [this tutorial](https://cloud.google.com/apigee/docs/apihub/tutorials/enrich-api-data) shows how to retrieve API specification files stored in Cloud Storage buckets.

Another purpose of curation is to use specific attributes from the source metadata to derive the unique fingerprint of an API, to prevent duplication of entries in the hub's catalog. For example, if a customer creates two Apigee proxies for the same API named `proxyName-v1` and `proxyName-v2`, we typically don't want this to result in two separate [API](https://cloud.google.com/apigee/docs/apihub/apis-intro) entries in the hub. Instead, we only want a single API (i.e. `proxyName`) containing two [*versions*](https://cloud.google.com/apigee/docs/apihub/versions-intro) (`v1` and `v2`). A custom curation can be used to achieve this.

## Setting API version using a custom curation

API hub allows users to [automatically import existing Apigee API proxies](https://cloud.google.com/apigee/docs/apihub/auto-register-apigee-proxies) into the catalog. This sample curation shows how to use the version number from an [OpenAPI](https://swagger.io/resources/open-api/) spec attached to the proxy as the version number in API hub. The curation logic implemented here performs the following steps for each ingested API proxy:

* Looks for an OpenAPI specification contained in the API metadata, which should be present if the proxy bundle in Apigee contains a spec resource
    * If both YAML and JSON formatted OAS specs are present, the curation will prefer the YAML file
* Parses and extracts the version number from the spec's [Info object](https://spec.openapis.org/oas/v3.2.0.html#info-object)
* Sets the version number for the hub entry to that value

For example, if the deployed API proxy bundle contains an OAS spec with the following:
```
title: Example Pet Store App
summary: A pet store manager.
description: This is an example server for a pet store.
termsOfService: https://example.com/terms/
contact:
  name: API Support
  url: https://www.example.com/support
  email: support@example.com
license:
  name: Apache 2.0
  url: https://www.apache.org/licenses/LICENSE-2.0.html
version: 1.0.1
...
```
the resulting entry in the hub will contain a version named `1.0.1`.

## Application Integration Overview

The Application Integration flow that implements this curation contains 3 components:
* An [API trigger](https://cloud.google.com/application-integration/docs/configure-api-trigger) called by the API hub, which receives the API metadata
* A [Data Transformer](https://cloud.google.com/application-integration/docs/configure-data-transformer-script-task) script task which modifies the version info using the value extracted from the spec.  The mapping is performed using a [Jsonnet](https://jsonnet.org/) template.
* A [Data Mapping](https://cloud.google.com/application-integration/docs/configure-data-mapping-task) task which replaces the metadata in the output with the modified content

## Prerequisites

To implement this sample, you'll need a GCP project with Apigee, API hub, and Application Integration activated. 
For more info, refer to the following documentation:
* [Provision Apigee](https://cloud.google.com/apigee/docs/api-platform/get-started/provisioning-intro)
* [Provision API hub](https://cloud.google.com/apigee/docs/apihub/provision)
* [Set up Application Integration](https://cloud.google.com/application-integration/docs/setup-application-integration)

You will also need to associate your Apigee Organization with the hub to automatically import API proxies. If you provisioned Apigee and API hub in a new project, this association is configured automatically. However, if you want to associate an existing Apigee Org from a different project, this can be done manually. For more info, see [Auto-register Apigee proxies](https://cloud.google.com/apigee/docs/apihub/auto-register-apigee-proxies).

## Application Integration Setup

To create the curation flow, perform the following steps:

1. Clone this repo 
2. In the Google Cloud Console, go to the [Application Integration](https://console.cloud.google.com/integrations) page
3. Create a new integration by clicking the `Create Integration` button
    - Enter the name `apihub-oas-version-curation` on the `Create Integration` screen
    - Select a region for the integration from the list of supported regions
    - Click `Create`
    
    This opens the integration in the integration designer
4. In the integration designer, click the three dots in the upper right corner and then select `Upload Integration`
5. In the file browser dialog, select the `apihub-oas-version-curation.json` file from this repo, and then click Open. A new version of the integration is created using the uploaded file
6. In the integration designer, click the `Publish` button to deploy the integration

## Apigee API hub Setup

Next you'll register a new custom curation within Apigee API hub and associate it with the Apigee plugin.

First, let's register the custom curation:

1. In the Google Cloud Console, navigate to the [Apigee API hub page](https://console.cloud.google.com/apigee/api-hub/get-started)
2. In the left navigation menu, under `API hub`, click `Settings`
3. Select the `Curations` tab, and then click `Create Curation`, and then `Start`
4. Enter a `Display name` and `Description`, and then from the dropdown list, select the integration you created previously (`apihub-oas-version-curation`) and the trigger (`api_trigger/apihub-oas-version-curation_API_1`)
5. Click `Create curation`

Next, you need to associate this new curation with the ingestion plugin for your Apigee Org. To associate your new curation:

1. Switch to the `Plugins` tab
2. If your Apigee Org is correctly associated with the hub, you should see a plugin entry named `Apigee X and Hybrid` with the corresponding project id for your Org
3. Click the three dots next to the plugin, and select `See details`.  The `Plugin instance details` dialog should appear.
4. Under `Curation`, select the pencil icon. Choose the new curation from the drop down, and select `Save`

## Testing the Curation Flow

To test the curation flow, either [create a new API proxy from an OpenAPI specification](https://cloud.google.com/apigee/docs/api-platform/tutorials/create-api-proxy-openapi-spec), or [add an OAS resource to an existing proxy](https://cloud.google.com/apigee/docs/api-platform/develop/resource-files#create-ui). Then simply [deploy](https://cloud.google.com/apigee/docs/api-platform/deploy/ui-deploy-new) the latest revision of the proxy.

To verify if the curation worked, go to the [APIs page](https://console.cloud.google.com/apigee/api-hub/apis) in API hub and look for an entry with the same name as the proxy.  Select the API and scroll down to the `Versions` section.  You should see a version that matches what is declared in the OAS file.

## Viewing the Integration log

You can view logs for each execution of the curation using the Logs viewer inside Application Integration.

In the Google Cloud console, go to the [Application Integration page](https://console.cloud.google.com/integrations), and in the navigation menu click `Logs`.  You can expand each entry that corresponds to `apihub-oas-version-curation` to view the steps, and compare the API metadata before and after curation.

For more information on how to view Application Integration Logs, see [here](https://cloud.google.com/application-integration/docs/integration-execution-logs).

## Disclaimer

This implementation is not an official Google product, nor is it part of an official Google product. The code samples in this repository are for demonstrative purposes only.