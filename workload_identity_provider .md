# Workload Identity Federation (WIF)
Workload Identity Federation (WIF) is a **keyless authentication mechanism** that allows applications running outside Google Cloud (such as on-premises, other clouds, or CI/CD platforms like GitHub Actions) to securely access Google Cloud resources **without using long-lived service account JSON keys**.

## What is Workload Identity Federation?

- Instead of using a static service account key file, your external workload authenticates with an **external identity provider (IdP)** (e.g., GitHub, AWS, Azure AD).
- The workload receives a token (usually an OIDC token) from the IdP.
- This token is exchanged with Google Cloud’s **Security Token Service (STS)** for a **short-lived Google Cloud access token**.
- The access token impersonates a Google Cloud **service account** with assigned permissions.
- This approach eliminates the need to manage, store, or rotate long-lived JSON keys, improving security and reducing operational overhead.


## How Workload Identity Federation Works (High-Level Flow)

1. **Setup:**
    - Create a **Workload Identity Pool** in your Google Cloud project to represent external identities.
    - Create an **Identity Provider** within the pool that trusts tokens from your external IdP.
    - Create a Google Cloud **service account** with the required permissions.
    - Grant the identity pool permission to **impersonate** this service account.
2. **Authentication:**
    - Your workload authenticates with the external IdP and obtains an OIDC token.
    - The workload exchanges this token with Google Cloud STS.
    - STS validates the token against the configured identity pool and returns a short-lived Google Cloud access token.
3. **Access:**
    - The workload uses this access token to call Google Cloud APIs, acting as the impersonated service account.

## Why Replace Service Account JSON Keys with Workload Identity Federation?

| Aspect | Service Account JSON Keys | Workload Identity Federation |
| :-- | :-- | :-- |
| Credential Type | Long-lived JSON key files | Short-lived tokens issued via external IdP |
| Security Risk | Risk of key leakage and misuse | Reduced risk; no static keys to manage |
| Credential Rotation | Manual rotation required | Automatic token expiration and renewal |
| Management Overhead | High (secure storage, rotation, auditing) | Minimal (configure once, no key management) |
| Granularity | Broad access per service account | Fine-grained access based on external identity attributes |
| Use Cases | Legacy workflows, local development | Modern CI/CD, multi-cloud, external workloads |

## How to Replace Service Account JSON Authentication with Workload Identity Federation

### Step 1: Create a Workload Identity Pool

```bash
gcloud iam workload-identity-pools create "my-pool" \
  --project=PROJECT_ID \
  --location="global" \
  --display-name="My workload identity pool"
```


### Step 2: Create an Identity Provider in the Pool

Configure it to trust your external IdP (e.g., GitHub, AWS, Azure AD):

```bash
gcloud iam workload-identity-pools providers create-oidc "my-provider" \
  --project=PROJECT_ID \
  --location="global" \
  --workload-identity-pool="my-pool" \
  --display-name="My OIDC provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```


### Step 3: Create a Google Cloud Service Account

```bash
gcloud iam service-accounts create my-service-account \
  --project=PROJECT_ID \
  --display-name="My Service Account"
```


### Step 4: Grant the Workload Identity Pool Permission to Impersonate the Service Account

```bash
gcloud iam service-accounts add-iam-policy-binding my-service-account@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/my-pool/*"
```


### Step 5: Configure Your Workload or CI/CD Pipeline to Use WIF

- In GitHub Actions, use the [`google-github-actions/auth`](https://github.com/google-github-actions/auth) action with `workload_identity_provider` and `service_account` inputs.
- Your workload will request tokens from the external IdP, then exchange them for Google Cloud credentials transparently.


## Summary

| Step | Description |
| :-- | :-- |
| Create Workload Identity Pool | Organize external identities in Google Cloud |
| Create Identity Provider | Trust tokens from external IdP (e.g., GitHub) |
| Create Service Account | Define permissions for Google Cloud access |
| Grant Impersonation Rights | Allow identity pool to impersonate service account |
| Use in Workload | Exchange external tokens for Google Cloud tokens |

## Additional Resources

- [YouTube: What is Workload Identity Federation?](https://www.youtube.com/watch?v=4vajaXzHN08)
- [Google Cloud IAM Best Practices for Workload Identity Federation](https://cloud.google.com/iam/docs/best-practices-for-using-workload-identity-federation)
- [Google Cloud Blog: Workload Identity Federation Explained](https://blog.montrealanalytics.com/authenticating-a-service-identity-into-google-cloud-with-workload-identity-federation-adc72327daee)
- [Google Cloud Docs: Workload Identity Federation for GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [GitHub Actions Example Using Workload Identity Federation](https://github.com/google-github-actions/auth)