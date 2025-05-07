# Client/Server Model for GitHub Actions

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

1. `labels:created`
1. `workflow_run:complete`
1. `workflow_run:complete` => `labels:created`

`labels:created` is useful in case you develop private repositories in teams.
On the other hand, `labels:created` requires the write permission, so it's unavailable in `pull_request` workflows triggred by pull requests from fork repositories.
Instead, `workflow_run:complete` is available in public repositories.
You can also combine `workflow_run:complete` and `labels:created`, creating labels by `workflow_run:complete` workflows.

### Why not `repository_dispatch`?

Generally, `repository_dispatch` event is used to trigger workflows by GitHub API.
But `repository_dispatch` requires `contents:write` permission, which is too strong.
So we use `labels:created` event instead.

### 1. `labels:created`

![client-server-model drawio](https://github.com/user-attachments/assets/fb85fd55-66a6-47b1-8b21-90ea0eb7102b)

Client workflows create GitHub Issue labels, then server workflows are triggered by `labels:created` event.
By separating server repositories and workflows from client repositories and workflows, you can protect server workflows.

We trigger server workflows by `labels:created` event because it requires only `issues:write` permission, which is hard to abuse.
GitHub Actions tokens can't trigger new workflows runs, so clients require GitHub Apps with `issues:write` permissions.

`labels:created` event can pass small parameters to server workflows via label name and description.
But if you need to pass larger parameters, you should use GitHub Actions Artifacts.

Label names must be unique, so when creating labels we add a random suffix to label names to make them unique.
We remove created labels immediately because we create them only for triggering server workflows and there is no reason to keep them.

#### Client GitHub App

A client GitHub App is a GitHub App to create labels to a server repository to trigger a server workflow.

- Deactivate Webhook
- Permissions:
  - `issues:write`: To create labels
  - You shouldn't grant other permissions to this app
- Installed Repositories: Install this app into the server repository and client repositories.
- Private key management: You can share the private key widely using GitHub Organizations Secrets as this app has only `issues:write` permission

### 2. `workflow_run:complete`

`labels:created` requires `issues:write` permission, but `pull_request` workflows don't have the write permission if pull request come from fork repositories.
In that case, `workflow_run:complete` event is useful as it doesn't need write permissions.

`workflow_run:complete` has some drawback compared with `labels:created`:

- You need to create server workflows per client repository
  - It's bothersome to maintain server workflows
  - It's hard to manage credentials if necessary
- `workflow_run` workflows are triggered even if they are unnecessary
- Client workflows need to use GitHub Actions Artifacts or something to pass parameters to `workflow_run`.

To solve these drawback, you can also combine `workflow_run:complete` and `labels:created`.

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
