# NGINX for Azure Deployment Action  

This action supports managing the configuration of an [NGINX for Azure](https://docs.nginx.com/nginx-for-azure/quickstart/overview/) deployment in a GitHub repository. It enables continuous deployment through GitHub workflows to automatically update the NGINX for Azure deployment when changes are made to the NGINX configuration files stored in the respository.

## Connecting to Azure

This action leverages the [Azure Login](https://github.com/marketplace/actions/azure-login) action for authenticating with Azure and performing update to an NGINX for Azure deployment. Two different ways of authentication are supported:
- [Service principal with secrets](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-a-service-principal-secret)
- [OpenID Connect](https://docs.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#use-the-azure-login-action-with-openid-connect) (OIDC) with an Azure service principal using a Federated Identity Credential

### Sample workflow that authenticates with Azure using an Azure service principal with a secret

```yaml
# File: .github/workflows/nginxForAzureDeploy.yml

name: Sync the NGINX configuration from the GitHub repository to an NGINX for Azure deployment
on:
  push:
    branches:
      - main
    paths:
      - config/**

jobs:
  Deploy-NGINX-Configuration:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout repository'
      uses: actions/checkout@v2

    - name: 'Run Azure Login using an Azure service principal with a secret'
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: 'Sync the NGINX configuration from the GitHub repository to the NGINX for Azure deployment'
      uses: nginxinc/nginx-for-azure-deploy-action@v0.1.0
      with:
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resource-group-name: ${{ secrets.AZURE_RESOURCE_GROUP_NAME }}
        nginx-deployment-name: ${{ secrets.NGINX_DEPLOYMENT_NAME }}
        nginx-config-directory-path: config/
        nginx-root-config-file: nginx.conf
        transformed-nginx-config-directory-path: /etc/nginx/
```

### Sample workflow that authenticates with Azure using OIDC

```yaml
# File: .github/workflows/nginxForAzureDeploy.yml

name: Sync the NGINX configuration from the GitHub repository to an NGINX for Azure deployment
on:
  push:
    branches:
      - main
    paths:
      - config/**

permissions:
      id-token: write
      contents: read

jobs:
  Deploy-NGINX-Configuration:
    runs-on: ubuntu-latest
    steps:
    - name: 'Checkout repository'
      uses: actions/checkout@v2

    - name: 'Run Azure Login using OIDC'
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: 'Sync the NGINX configuration from the GitHub repository to the NGINX for Azure deployment'
      uses: nginxinc/nginx-for-azure-deploy-action@v0.1.0
      with:
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resource-group-name: ${{ secrets.AZURE_RESOURCE_GROUP_NAME }}
        nginx-deployment-name: ${{ secrets.NGINX_DEPLOYMENT_NAME }}
        nginx-config-directory-path: config/
        nginx-root-config-file: nginx.conf
        transformed-nginx-config-directory-path: /etc/nginx/
```
## Handling NGINX configuration file paths

To facilitate the migration of the existing NGINX configuration, NGINX for Azure supports multiple-files configuration with each file uniquely identified by a file path, just like how NGINX configuration files are created and used in a self-hosting machine. An NGINX configuration file can include another file using the [include directive](https://docs.nginx.com/nginx/admin-guide/basic-functionality/managing-configuration-files/). The file path used in an `include` directive can either be an absolute path or a relative path to the [prefix path](https://www.nginx.com/resources/wiki/start/topics/tutorials/installoptions/).

The following example shows two NGINX configuration files inside the `/etc/nginx` directory on disk are copied and stored in a GitHub respository under its `config` directory.

| File path on disk                    | File path in the respository      |
|--------------------------------------|-----------------------------------|
| /etc/nginx/nginx.conf                | /config/nginx.conf                |
| /etc/nginx/sites-enabled/mysite.conf | /config/sites-enabled/mysite.conf |

To use this action to sync the configuration files from this example, the directory path relative to the GitHub repository root `config/` is set to the action's input `nginx-config-directory-path` for the action to find and package the configuration files. The root file `nginx.conf` is set to the input `nginx-root-config-file`.

```yaml
    - name: 'Sync the NGINX configuration from the GitHub repository to the NGINX for Azure deployment'
      uses: nginxinc/nginx-for-azure-deploy-action@v0.1.0
      with:
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resource-group-name: ${{ secrets.AZURE_RESOURCE_GROUP_NAME }}
        nginx-deployment-name: ${{ secrets.NGINX_DEPLOYMENT_NAME }}
        nginx-config-directory-path: config/
        nginx-root-config-file: nginx.conf
```

By default, the action uses a file's relative path to `nginx-config-directory-path` in the respository as the file path in the NGINX for Azure deployment.

| File path on disk                    | File path in the respository      | File path in the NGINX for Azure deployment |
|--------------------------------------|-----------------------------------|---------------------------------------------|
| /etc/nginx/nginx.conf                | /config/nginx.conf                | nginx.conf                                  |
| /etc/nginx/sites-enabled/mysite.conf | /config/sites-enabled/mysite.conf | sites-enabled/mysite.conf                   |

The default file path handling works for the case of using relative paths in `include` directives, for example, if the root `nginx.conf` references `mysite.conf` using:

```
include sites-enabled/mysite.conf;
```

For the case of using absolute paths in `include` directives, for example, if the root `nginx.conf` references `mysite.conf` using:

```
include /etc/nginx/sites-enabled/mysite.conf;
```

The action supports an optional input `transformed-nginx-config-directory-path` to transform the absolute path of the configuration directory in the NGINX for Azure deployment. The absolute configuration directory path on disk `/etc/nginx/` can be set to `transformed-nginx-config-directory-path` as follows to ensure the configuration files using absolute paths in `include` directives work as expected in the NGINX for Azure deployment.

```yaml
    - name: 'Sync the NGINX configuration from the Git repository to the NGINX for Azure deployment'
      uses: nginxinc/nginx-for-azure-deploy-action@v0.1.0
      with:
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
        resource-group-name: ${{ secrets.AZURE_RESOURCE_GROUP_NAME }}
        nginx-deployment-name: ${{ secrets.NGINX_DEPLOYMENT_NAME }}
        nginx-config-directory-path: config/
        nginx-root-config-file: nginx.conf
        transformed-nginx-config-directory-path: /etc/nginx/
```
The transformed paths of the two configuration files in the NGINX for Azure deployment are summarized in the following table

| File path on disk                    | File path in the respository      | File path in the NGINX for Azure deployment |
|--------------------------------------|-----------------------------------|---------------------------------------------|
| /etc/nginx/nginx.conf                | /config/nginx.conf                | /etc/nginx/nginx.conf                       |
| /etc/nginx/sites-enabled/mysite.conf | /config/sites-enabled/mysite.conf | /etc/nginx/sites-enabled/mysite.conf        |
