# Configuring Workload identity federation between GCP and Azure AKS

Workload identity federation is the process of impersonating an identity in one cloud provider from the other without long lived keys. In this configuration-azure-assume-gcp we will walk through how to setup federation between a workload (provider-gcp-compute) running on Azure AKS to GCP.

Here are the steps we need to complete on the GCP side as a prerequisite. We'll also walk you through the customizations needed for provider-gcp-compute so it can use the impersonated service account on the GCP side.

# GCP
## Enable Additional GCP Services
The Security Token Service API and the IAM Service Account Credentials API need to be enabled in our GCP project.

```bash
gcloud services enable sts.googleapis.com
gcloud services enable iamcredentials.googleapis.com
```

## Create GCP Service Account

Now we need to create the GCP Service Account which provider-gcp-compute will authenticate with in order to Create,Update,Delete GCP resources.

```bash
gcloud iam service-accounts create configuration-azure-assume-gcp \
    --description="configuration-azure-assume-gcp for providers which runs on Azure." \
    --display-name="configuration-azure-assume-gcp"
```

We need to ensure that the new Service Account has the appropriate perssion to generate access token in GCP.

```bash
gcloud projects add-iam-policy-binding ${GCP_PROJECT_NAME} \
--member="serviceAccount:configuration-azure-assume-gcp@${GCP_PROJECT_NAME}.iam.gserviceaccount.com" \
--role="roles/iam.serviceAccountTokenCreator"
```

We need to ensure that the new Service Account has the appropriate permission to Create,Update,Delete resources in the desired project in GCP.

```bash
gcloud projects add-iam-policy-binding ${GCP_PROJECT_NAME} \
--member="serviceAccount:configuration-azure-assume-gcp@${GCP_PROJECT_NAME}.iam.gserviceaccount.com" \
--role="roles/compute.admin"
```

## Create GCP Workload Identity Pool

Next, we need to create the GCP Workload Identity Pool.

```bash
gcloud iam workload-identity-pools create configuration-azure-assume-gcp \
    --description="Workload Identity Pool for Providers on Azure." \
    --display-name="Providers Azure Pool" \
    --location="global"
```

## Create Workload Identity Provider

After creating the GCP Workload Identity Pool we need to create the Workload Identity Provider.
This example does not cover how to harden the Identity Provider, which can be achieved by passing the `--attribute-condition` flag.

THe OIDC-Usser-URL can be found in the Azure AKS Cluster `status.atProvider.oidcIssuerUrl`

```bash
gcloud iam workload-identity-pools  \
    providers create-oidc configuration-azure-assume-gcp \
    --location="global"  \
    --workload-identity-pool="configuration-azure-assume-gcp"  \
    --issuer-uri="${GCP_OIDC_ISSUER_URL}"  \
    --allowed-audiences="api://AzureADTokenExchange"  \
    --attribute-mapping="google.subject=assertion.sub"
```

## Download Client library

The Client library config needs to be downloaded, this can be achieved by navigating to the Workload Identity Pool and selecting the CONNECTED SERVICE ACCOUNTS Tab on the right side. The Client library config can be obtained from there. The file does NOT contain sensitive information.

## Bind GCP Service Account Impersonation.

Now, we will create an IAM policy binding.
The below command allows all identites from our pool to impersonate the GCP Service Account we created earlier by binding the IAM role `roles/iam.workloadIdentityUser` to the GCP service account.

This example does not cover how to harden with granular scoping for service account impersonation.

```bash
gcloud iam service-accounts add-iam-policy-binding \
    configuration-azure-assume-gcp@${GCP_PROJECT_NAME}.iam.gserviceaccount.com \
    --member "principalSet://iam.googleapis.com/projects/${GCP_PROJECT_NUMBER}/locations/global/workloadIdentityPools/configuration-azure-assume-gcp/*" \
    --role "roles/iam.workloadIdentityUser"
```

# Azure

On the Azure side, you'll need an AKS cluster with an OIDC provider enabled to use Workload Identity.
This allows you to assign an IAM role to a Kubernetes service account for role assumption.
We won't go into detailed setup instructions here, as there are various configurations available for implementing this.

this part is implemented in this configuration

## Create configmap with GCP Client library


Earlier, we downloaded the GCP Client library for the Workload Pool.
Now, we need to create a ConfigMap using its content.
Just to note, this file does NOT contain any sensitive information.

```bash
kubectl -n upbound-system create configmap gcp-default-credentials --from-file=gcp_default_credentials.json
```

## Create DeploymentRuntimeConfig

We need to create a DeploymentRuntimeConfig to customize the name of the service account and add the required labels for workload identity.
Additionally, we need to configure provider-gcp-compute to locate the client library JSON file, which will be used later for impersonating a GCP service account.

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: upbound-provider-gcp-compute
spec:
  serviceAccountTemplate:
    metadata:
      name: upbound-provider-gcp-compute
      annotations:
        azure.workload.identity/client-id: "upbound-provider-gcp-compute"
  deploymentTemplate:
    spec:
      selector: {}
      replicas: 1
      template:
        metadata:
          labels:
            azure.workload.identity/use: "true"
        spec:
          containers:
            - name: package-runtime
              env:
                - name: GOOGLE_APPLICATION_CREDENTIALS
                  value: /tmp/gcp_default_credentials.json
              volumeMounts:
                - mountPath: /tmp/
                  name: gcp
          volumes:
            - name: gcp
              configMap:
                name: gcp-default-credentials
                items:
                  - key: gcp_default_credentials.json
                    path: gcp_default_credentials.json
```

## Create ProviderConfig for provider-gcp

We need to create a ProviderConfig to allow impersonation of the service account in GCP.

```yaml
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    impersonateServiceAccount:
      name: configuration-azure-assume-gcp@${GCP_PROJECT_NAME}.iam.gserviceaccount.com
    source: ImpersonateServiceAccount
  projectID: ${GCP_PROJECT_NAME}
```
