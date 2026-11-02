# Weik.io GitOps example

This repository has the examples that the [Weik.io](https://weik.io) docs, the University videos and the live demo use. Fork it to run them on your own Weik.io installation from GitHub Actions.

- `hello` is the integration that `weikio init hello` writes. A timer starts it every 5 seconds and it logs a line. [Your first integration](https://weik.io/docs/start/first-integration) runs it with `weikio run ./hello`.
- `finance` is a project with an example of each kind: Camel integrations, a transfer, an API and a channel. Its folders are laid out for GitOps, and the workflows in `.github/workflows/` plan and apply them from GitHub Actions.

Weik.io doesn't read this repository. The workflows run the `weikio` CLI in GitHub Actions, and the CLI sends each change to your installation. A fork points at your installation through the address in its workflows and an API key in its GitHub environments.

## What each folder contains

| Folder | Contents |
| --- | --- |
| `hello/` | The route `hello.camel.yaml`, the message it logs in `application.properties`, and `integration.yaml` with commented examples of what you can add later |
| `finance/project.yaml` | The project `finance`, which a platform administrator creates once |
| `finance/variables/` | The project's default values: the ERP address `erp-url` and the connections `supplier-ftp` and `invoice-archive` |
| `finance/erp-invoice-export/` | A Camel integration that sends each new invoice to the ERP over HTTP, counts it in the custom metric `invoices_processed` and publishes an event to `invoice-created` |
| `finance/transfers/` | `supplier-invoices.yaml`, a transfer that moves supplier invoice PDFs from an FTP server to Azure Blob Storage every morning at 05:00 UTC |
| `finance/apis/invoices/` | The API `finance/invoices` with version `v1`, which passes requests on to the ERP's invoice API |
| `finance/invoice-created/` | The channel `invoice-created`, and `invoice-archiver`, an integration that writes each event it receives from the channel to a folder |
| `finance/devtest/` | The deployment that runs `erp-invoice-export` in devtest |
| `finance/production/` | What production has of its own: the deployment of `erp-invoice-export`, the subscription `partner-x` to the invoice API and the production ERP address |
| `.github/workflows/` | `plan.yaml` shows the plan on each pull request that changes `finance/`. `devtest.yaml` and `production.yaml` apply the folders after a merge to `main`, and production waits for its required reviewers |

`project.yaml` and `variables/` belong to a platform administrator, because they change every environment at once. The workflows apply everything else.

## Run the examples on your installation

You need a Weik.io installation and a platform administrator's sign-in. To try Weik.io on your own machine first, start with [Try Weik.io on your machine](https://weik.io/docs/start/try-on-your-machine).

1. Fork this repository on GitHub and clone your fork.

2. Install the CLI and sign in to your installation:

   ```sh
   curl -fsSL https://weik.io/install.sh | sh
   weikio login https://weikio.example.com
   ```

   On Windows, or to install one release, see [Install the CLI](https://weik.io/docs/start/install-the-cli).

3. As a platform administrator, create the project and its default values:

   ```sh
   weikio apply -f ./finance/project.yaml -f ./finance/variables
   ```

4. Set the secrets the examples read. Each command asks for the value with hidden input:

   ```sh
   weikio secret set finance/erp-api-key --env devtest
   weikio secret set finance/supplier-ftp-password --env devtest
   weikio secret set finance/invoice-archive-key --env devtest
   weikio secret set finance/erp-api-key --env production
   ```

5. Apply the folders that every environment shares and the devtest folder. This publishes a version of each integration and starts `erp-invoice-export` in devtest:

   ```sh
   weikio apply --env devtest --prune finance \
     -f ./finance/erp-invoice-export -f ./finance/transfers -f ./finance/apis \
     -f ./finance/invoice-created -f ./finance/devtest
   ```

6. Create a user for the workflows and one API key for each environment. Each key has the `deployer` role on `finance` in its own environment only. The CLI shows a key once, so copy it when it's created:

   ```sh
   weikio user create ci@example.com --display-name "Finance CI"
   weikio member add finance ci@example.com --role deployer --env devtest,production
   weikio apikey create --name finance-ci-devtest --user ci@example.com --expires 365d --grant finance/deployer@devtest
   weikio apikey create --name finance-ci-prod --user ci@example.com --expires 365d --grant finance/deployer@production
   ```

7. In your fork, change `WEIKIO_SERVER: https://weikio.example.com` in each file of `.github/workflows/` to the address of your installation.

8. In your fork's settings, add the GitHub environments `devtest`, `production-plan` and `production`. Add the devtest key to `devtest` as the secret `WEIKIO_API_KEY`, add the production key to the other two under the same name, and give `production` required reviewers.

9. GitHub doesn't run the workflows of a fork until you turn them on. Open the Actions tab of your fork and turn them on.

From then on, a pull request that changes `finance/` gets a plan, and a merge to `main` applies the change to devtest and production. Promotion is a pull request that changes the version in `finance/production/deployments/erp-invoice-export.yaml`.

## Docs

- [Keep integrations in Git](https://weik.io/docs/deliver/gitops) describes the whole loop from a pull request to production.
- [API keys](https://weik.io/docs/secure/api-keys) covers keys for pipelines and how to rotate them.
- [Secrets](https://weik.io/docs/secure/secrets) says how Weik.io keeps secret values and how an integration gets them.

## Licence

The examples are open source under the [Apache License 2.0](LICENSE), copyright Adafy Oy.
