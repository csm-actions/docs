# Client/Server Model for GitHub Actions

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/csm-actions/docs)

This document describes our Client/Server Model, making GitHub Actions secure.

## Overview

Our Client/Server Model is the architecture making GitHub Actions secure.
You can protect server workflows with strong permissions and credentials by separating them from client workflows.
You can implement and maintain the model by GitHub Actions easily without hosting and maintaining any server application.

![image](https://github.com/user-attachments/assets/e121e269-b072-45ca-b512-8346f7554175)

Both client workflows and server workflows are merely GitHub Actions Workflows.
The flow is simple:

1. A client workflow requests actions to a server workflow
1. A server workflow validates the request
1. A server workflow does requested actions

## Actions

We develop some actions based on this model:

- [Securefix Action is GitHub Actions to fix code securely](https://github.com/csm-actions/securefix-action)
- [Approve PR Action is GitHub Actions to approve pull requests securely](https://github.com/csm-actions/approve-pr-action)
- [Update Branch Actions is GitHub Actions to update pull request branches securely](https://github.com/csm-actions/update-branch-action)

## Features

- 🛡 Secure
  - You don't need to pass a GitHub App private key with strong permissions to GitHub Actions workflows on the client side
  - You don't need to allow external services to access your code
  - You can validate requests
- 😊 Easy to maintain
  - You don't need to host a server application

## Background

CI is powerful, but it's dangerous at the same time.
Generally, CI requires strong permissions and it can run any commands.
So the security is very important to prevent the permissions from being abused.

You need to protect credentials with strong permissions properly.
For instance, if you want to use a GitHub App with the `contents:write` permission in CI to fix codes automatically, you should manage the app's private key securely.

To achieve this, we introduce the Client/Server Model.
Clients are GitHub Actions Workflows.
Instead of granting strong permissions to workflows, workflows send requests to the server, and the server validates and handles them.
You grant strong permissions to servers, but you can protect servers by restricting people who can modify them.

You can implement servers using GitHub Actions.
GitHub Actions is much easier to implement and maintain servers than other tools such as AWS Lambda, Google Cloud Function, k8s, and so on.

## How To Trigger Server workflows

There are several ways to trigger server workflows:

1. `workflow_dispatch` (recommended)
1. `labels:created`
1. `workflow_run:complete`
1. `workflow_run:complete` => `workflow_dispatch` or `labels:created`

`workflow_dispatch` and `labels:created` are useful in case you develop private repositories in teams.
On the other hand, both of them require a GitHub App's private key in client workflows, which isn't available in `pull_request` workflows triggered by pull requests from fork repositories.
Instead, `workflow_run:complete` is available in public repositories.
You can also combine `workflow_run:complete` with `workflow_dispatch` or `labels:created`, triggering server workflows from `workflow_run:complete` workflows.

GitHub Actions tokens can't trigger new workflow runs, so clients require a GitHub App to trigger server workflows.
By separating server repositories and workflows from client repositories and workflows, you can protect server workflows.

### 1. `workflow_dispatch`

Client workflows run server workflows by [the workflow_dispatch API](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event).

- Parameters are passed as workflow inputs
- The API returns the server workflow run, so client workflows can output a link to it
- No resource such as a label needs to be created and deleted just to trigger server workflows

Client workflows specify a server workflow in the format `<owner>/<repo>/.github/workflows/<workflow file>@<branch or tag>`, the same format as `github.workflow_ref`.
[`@csm-actions/dispatch`](https://jsr.io/@csm-actions/dispatch) is a JSR package for client actions to run server workflows this way.

#### Client GitHub App

A client GitHub App is a GitHub App to run server workflows.

- Deactivate Webhook
- Permissions:
  - `actions:write`: To run server workflows
  - You shouldn't grant other permissions to this app
- Installed Repositories: Install this app into only the server repository.
  - You don't need to install it into client repositories, because client workflows run workflows only in the server repository.
- Private key management: You can share the private key widely using GitHub Organizations Secrets, as this app has only `actions:write` permission of the server repository

#### Risks of `actions:write`

`actions:write` of the server repository doesn't allow clients to change the server repository's code or read secrets, but it allows more than running workflows:

- Disable and enable workflows (DoS of server workflows)
- Cancel and re-run workflow runs
- Delete workflow runs and logs
- Delete artifacts and caches

As the private key of the client GitHub App is shared widely, anyone who can write to a client repository should be assumed to be able to do these.
We consider these acceptable as long as secrets are available only on the default branch of the server repository and the default branch is protected properly.

#### Run server workflows only on the default branch

Unlike `labels:created`, which always runs workflows on the default branch, `workflow_dispatch` lets clients choose any branch or tag of the server repository.
If an old branch or tag has a vulnerable server workflow, it could be run.
So server workflows must run only on the default branch:

- Server actions fail if the event is `workflow_dispatch` and `github.ref` isn't the default branch. This check runs in the workflow on the dispatched ref, so it doesn't protect a branch or tag whose workflow doesn't have it. Use the following measures, which are enforced outside workflows, too
- Restrict the deployment branches of the environment to the default branch
- Restrict the access to secrets by OIDC claims. When the job uses an environment, the `sub` claim doesn't include the ref, so customize the subject claim to include `job_workflow_ref`, for instance

`actions:write` also allows re-running server workflow runs.
A re-run uses the workflow at the same commit as the original run, and both the deployment branch policy of the environment and `job_workflow_ref` see only the branch name.
So when you fix a vulnerability of a server workflow, delete the server workflow runs from before the fix.

### 2. `labels:created`

![client-server-model drawio](https://github.com/user-attachments/assets/fb85fd55-66a6-47b1-8b21-90ea0eb7102b)

Client workflows create GitHub Issue labels, then server workflows are triggered by `labels:created` event.

It requires only `issues:write` permission, and the server workflow always runs on the default branch.
But `workflow_dispatch` is more natural to trigger workflows, passes parameters more flexibly, and returns the workflow run, so we recommend `workflow_dispatch`.

`labels:created` event can pass small parameters to server workflows via label name and description.
But if you need to pass larger parameters, you should use GitHub Actions Artifacts.

Label names must be unique, so when creating labels we add a random suffix to label names to make them unique.
We remove created labels immediately because we create them only for triggering server workflows and there is no reason to keep them.

#### Client GitHub App

- Deactivate Webhook
- Permissions:
  - `issues:write`: To create labels
  - You shouldn't grant other permissions to this app
- Installed Repositories: Install this app into only the server repository.
  - You don't need to install it into client repositories, because client workflows create labels only in the server repository.
- Private key management: You can share the private key widely using GitHub Organizations Secrets as this app has only `issues:write` permission

### 3. `workflow_run:complete`

`workflow_dispatch` and `labels:created` require a GitHub App's private key in client workflows, but `pull_request` workflows can't access secrets if pull requests come from fork repositories.
In that case, `workflow_run:complete` event is useful as it doesn't need write permissions.

`workflow_run:complete` has some drawback compared with `workflow_dispatch` and `labels:created`:

- You need to create server workflows per client repository
  - It's bothersome to maintain server workflows
  - It's hard to manage credentials if necessary
- `workflow_run` workflows are triggered even if they are unnecessary
- Client workflows need to use GitHub Actions Artifacts or something to pass parameters to `workflow_run`.

To solve these drawback, you can also combine `workflow_run:complete` with `workflow_dispatch` or `labels:created`.

### Why not `repository_dispatch`?

Generally, `repository_dispatch` event is used to trigger workflows by GitHub API.
But creating a `repository_dispatch` event requires `contents:write` permission of the server repository.
It would let clients rewrite server workflows or push branches with malicious workflows, so the model would have no meaning.
Also, the API doesn't return the workflow run.

## Protect server workflows

You must manage server workflows securely.
For instance, granting the write permission of server repositories to only system administrators.

## Secret Management

You must manage secrets used in server workflows securely.
Otherwise, this model has no meaning.
For instance, you must not share them using GitHub Organizations Secrets widely.
There are several ways to manage them securely:

- [Use GitHub Environment Secret](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment#deployment-protection-rules)
  - Restrict the branch
- Use a secret manager such as AWS Secrets Manager and [restrict the access by OIDC claims (repository, event, branch, workflow, etc)](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

### GitHub App private keys

You should manage GitHub App private keys with a key management service such as AWS KMS rather than GitHub Secrets or a secret manager.
A private key never expires, so anyone who obtains it can create access tokens indefinitely.
If you import the private key into AWS KMS and delegate only the JWT signing to KMS, workflows never read the private key, so it can't leak.
Restrict the access to the KMS key by OIDC claims as well.

You can create installation access tokens this way using [suzuki-shunsuke/create-github-app-token-aws-kms](https://github.com/suzuki-shunsuke/create-github-app-token-aws-kms).
For the setup, see [the agent skill github-app-private-key-aws-kms](https://github.com/suzuki-shunsuke/agent-skills/blob/main/skills/github-app-private-key-aws-kms/SKILL.md).

## Validation

For security, it's very important for server workflows to validate requests from clients.

## Error Notification

When any error happen in server workflows, server workflows notify it to clients. In case of pull request workflows, server workflows post comments to pull requests.

## Compared with other tools like AWS Lambda

You can implement server using tools like AWS Lambda, Google Cloud Function, k8s, and so on instead of GitHub Actions.

But if you use these tools, you need to consider SDLC of the server application.

- develop
- authentication
- deploy
- monitoring
- logging
- etc

On the other hand, GitHub Actions may be more expensive and slower than them.
And it's difficult for server workflows to return any response to client workflows.

So there are both pros and cons, but we adopt server workflows as it's easy to develop and maintain.
