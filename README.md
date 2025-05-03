# Document of our Client/Server Model

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

We implement servers using GitHub Actions.
GitHub Actions is much easier to implement and maintain servers than other tools such as AWS Lambda, Google Cloud Function, k8s, and so on.

## How To Trigger Server workflows

There are two ways to trigger server workflows:

1. `labels:created`
1. `workflow_run:complete`

### 1. `labels:created`

![client-server-model drawio](https://github.com/user-attachments/assets/fb85fd55-66a6-47b1-8b21-90ea0eb7102b)

Client workflows create GitHub Issue labels, then server workflows are triggered by `labels:created` event.
By separating server repositories and workflows from client repositories and workflows, you can protect server workflows.

Generally, `repository_dispatch` event is used to trigger workflows by GitHub API.
But `repository_dispatch` requires `contents:write` permission, which is too strong.
It's undesirable to grant the `contents:write` permission to client workflows.

Instead, we use `labels:created` event because it requires only `issues:write` permission, which is hard to abuse.
To create labels in other repositories, GitHub Actions tokens are unavailable.
So clients require GitHub Apps with `issues:write` permissions.

`labels:created` event can pass small parameters to server workflows via label name and description.
But if you need to pass larger parameters, you should use GitHub Actions Artifacts.

### 2. `workflow_run:complete`

`labels:created` requires `issues:write` permission, but `pull_request` workflows don't have the write permission if pull request come from fork repositories.
In that case, `workflow_run:complete` event is useful as it doesn't need write permissions.

`workflow_run:complete` has some drawback compared with `labels:created`:

- You need to create server workflows per client repository
  - It's bothersome to maintain server workflows
  - It's hard to manage credentials if necessary
- `workflow_run` workflows are triggered even if they are unnecessary
- Client workflows need to use GitHub Actions Artifacts or something to pass parameters to `workflow_run`.

## Limitation

Implementing servers by GitHub Actions has some limitation:

- When client workflows trigger server workflows, client workflows can't receive response from server workflows
