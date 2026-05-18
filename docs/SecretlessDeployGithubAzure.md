---
layout: default
title: "Secretless Deploy: GitHub Actions with OIDC and Azure Container Apps"
parent: Articles
permalink: secretless-deploy-github-azure-container
---

<p align="center">
<img src="/assets/images/secretless-cover-img.avif" alt="Cover OIDC Article" width="800" style="border-radius: 8px;"/>
</p>

How many static secrets exist in your company's repos right now? If you're not sure, this article is for you.

There's a way to deploy to Azure without storing any passwords in your pipeline — and the best part: using tools you probably already have.

---

I recently spoke at the Azure User Groups event at a University — a special place for me, as I spent some years there as a student. It was a great opportunity to share a topic that is central to my DevOps team's day-to-day: eliminating static passwords from CI/CD pipelines.

## The Problem with Static Passwords 🛑

Managing secrets for automation brings real challenges:

- **Lack of scalability:** Imagine manually rotating passwords across more than 100 repositories. The operational cost grows out of control.
- **Leakage risk:** Long-lived credentials are vulnerabilities waiting to be exploited. A single compromised secret can bring down an entire infrastructure.
- **Maintenance cost:** Periodic rotation policies consume engineering time that could go toward delivering value.

## The Solution: OpenID Connect (OIDC) 🔐

[OpenID Connect](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc) is an authentication protocol built on top of OAuth 2.0. It verifies identities via JSON Web Tokens (JWT) — the same mechanism behind "Sign in with Google."

In the context of GitHub Actions, OIDC completely eliminates the need for passwords.

The flow works like this:

<p align="center">
<img src="/assets/images/secretless-oidc-flow.avif" alt="Cover OIDC Article" width="800" style="border-radius: 8px;"/>
</p>

The only values we need to configure in the repository are identifiers, not passwords:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

Even if these values become public within the organization, an attacker cannot use them — authentication requires being on the exact branch and repository configured in the trust relationship.

## What is Azure Container Apps?

[Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) is a managed serverless service for running containers without needing to configure Kubernetes directly. It handles scalability, load balancing, and updates — you just push the image. It's the final destination of our pipeline.

## How it Works in Practice

To demonstrate this end-to-end, I built a complete lab that was presented at the talk. The repository is divided into three parts:

1. **Workflow** (`github-actions-deploy`): GitHub Actions pipeline with build and deploy configured via OIDC.
2. **Application** (`static-app`): A simple React frontend, used to test the Docker image build, push, and execution in the Container App.
3. **Infrastructure as Code** (`infra/terraform`): All provisioning via Terraform (and Bicep in the live demo):
   - Resource Group, Log Analytics Workspace, and Azure Container Registry (ACR)
   - Azure Container App and its Environment
   - Role Assignments for the Container App to pull the image from ACR
   - `github_actions_oidc.tf`: provisions the Service Principal and federated credentials — the heart of passwordless authentication

## Want to Try It Yourself?

All the code is available on GitHub. Clone, test, and adapt it to your needs:
[Access the repository](https://github.com/matias-defrederike/azure-user-groups)

Implementing OIDC is not just an improvement — it's the current standard. A practice I adopt daily at EBANX across all clouds, without exception.

How many static secrets exist in your repos right now?

If you need help or found anything wrong here, please open an issue and I can fix it.
