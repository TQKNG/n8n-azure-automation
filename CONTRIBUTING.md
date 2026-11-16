# Contributing to n8n Azure Automation

Check existing issues and pull requests before starting a change.
Use the repository's bug or feature request forms for proposed work.
Keep pull requests focused and describe their effect on deployment, infrastructure, or operations.

## Understand the workflow

Read [README.md](README.md) for the architecture, prerequisites, and order of the setup and deployment workflows.
Do not run setup, deploy, start, or teardown workflows as part of ordinary pull request verification.
Those workflows can change Azure resources and require the intended environment and credentials.

## Verify changes

For Bicep changes, run the local validation commands in the README when the Azure CLI and Bicep CLI are available.
For scripts or container configuration, describe the checks you ran and any environment that could not be tested.
Check documentation links and examples against the current repository layout.
Never include Azure credentials, n8n encryption keys, or other secrets in source, logs, or screenshots.

## Submit a pull request

Use a conventional title such as `docs: clarify setup` or `fix: correct VM bootstrap` so the existing title check passes.
Explain the problem, affected workflow or module, and any related issue.
List verification commands and results, including checks you could not run.
Call out infrastructure changes, deployment steps, and rollback considerations for reviewers.
