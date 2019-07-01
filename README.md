# Opinionated Terraform Runner for Azure DevOps

This is an opinionated [terraform](https://terraform.io) wrapper that eases integration with [Azure DevOps](https://dev.azure.com/) pipelines. The basic idea is that you can use the `run_tf.sh` to execute terraform deployments from a straight forward [Azure CLI Task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli?view=azure-devops) in Azure DevOps - no additional third party plugin required.


## Installation - How to integrate with your Terraform Project? 

Download the latest release of `run_tf.sh` from the release section.

Just follow this convention to structure your Terraform deployment(s):
```txt
+ my-deployment
      - main.tf
      - variables.tf
      - ...
- run_tf.sh              <-- drop script to level
````

If you want to split your deployment into separate steps, structure it like this:

```txt
+ 01-part1
      - main.tf
      - backend.tf       <-- important: configure your backend, see below!
      - variables.tf
      - ...
+ 02-part2               <-- optional split deployments into multiple steps
      - main.tf
      - variables.tf
      - ...
- run_tf.sh
```

In simple terms, put the `run_tf.sh` script top level in your directory. The script will automatically detect terrafrom folders on the second level and will run `terraform plan` / `terraform apply` etc. from there. If you have separate terraform deployments, the script will execute them in natural sorting order. In this case best use a prefix like `01-` and `02-` to clearly state that.

**Here is a repo with full sample: [https://github.com/olohmann/](https://github.com/olohmann/terraform-azuredevops-reference) **

## Repo Contents

```txt
.editorconfig           See https://editorconfig.org/
CHANGELOG.md            Tracking the changelog.
LICENSE                 License information on this project.
README.md               This README.
run_tf.sh               The wrapper script. You only need this file!
```

## run_tf.sh Usage

```txt
Usage: run_tf.sh [-e <environment_name>] [-v] [-f] [-p] [-h]

Options
-e <environment_name>    Defines an environment name that will be activated
                         as a terraform workspace, e.g. 'dev', 'qa' or 'prod'.
                         Default is terraform's 'default'.
-v                       Validate: perform a terraform validation run.
-f                       Force: Defaults all interaction to yes.
-p                       Print env.
-d                       Download minimal version of terraform client.
-h                       Help: Print this dialog and exit.

You can provide terraform params via passing 'TF_VAR_' prefixed environment vars.
For example:
export TF_VAR_location="northeurope"
Will pass an according variable to all terraform invocations.
```

## Terraform State

Being opinionated, the script will automatically create an Azure Storage Account in predefined way to store the [Terraform remote state](https://www.terraform.io/docs/backends/types/azurerm.html) securely and automation-friendly in an Azure Storage Account.

The *State Storage Account* will be created in the same subscription as the actual deployment. This is usually a sane default, as separation of environments via subscription will then also safely separate the Terraform state storage by environment. Please remember, that the Terraform state is extremely sensible information. You have to treat it as an administrative resource with the same security policy as the actual application deployment.

You can provide the following environment variables to customize the naming of state storage:

```sh
export __TF_backend_resource_group_name="MySpecialName_RG"
export __TF_backend_location="NorthEurope"
export __TF_backend_storage_account_name="s98si89p"
export __TF_backend_storage_container_name="tf-state"

# comma separated list of IPs and/or CIDRs 
export __TF_backend_network_access_rules="23.92.28.29,126.20.2.0/24"
```

> **Please Note:** If no customization is applied, sane defaults will be chosen and a resource group and a storage account will be created accordingly.


**Important: Configure the State Backend in Your Code.** In your own terraform (sub-)deployments you need to provide the backend config to actually use the Azure Remote State. Best convention is to put a `backend.tf` in place with a unique key per (sub-)deployment. For example:

```hcl
terraform {
  backend "azurerm" {
    key = "my-deployment.tfstate"
  }
}
```

## Integration into Azure DevOps

### Build Pipeline for Validation (Optional)

Integration into the build pipeline for validation and artefact passing is simple. You just need to hook up the `run_tf.sh` script with the parameters `-v` for validation only and `-f` for non-interactive.The `-d` flag will ensure that the right terraform version will be downloaded on the build agent.

Here is a yaml dump of such a pipeline:

```yaml
steps:
- task: AzureCLI@1
  displayName: Validate
  inputs:
    azureSubscription: '<Azure Subscription Reference>'
    scriptPath: 'run_tf.sh'
    arguments: '-v -f -d'
    addSpnToEnvironment: true
```

> **Please note**: It is important to use the Azure CLI task *and* selecting to pass the SPN details in the actual task (`addSpnToEnvironment: true`). This allows the `run_tf.sh` script to automatically detect that it is being used in combination with a service principal in an Azure CLI pipeline task.

In addition, you can add as many `TF_VAR_`-prefixed variables as required to pass parameters to the terraform deployment. To override the location value, for example, you can pass a `TF_VAR_location` variable:

```yaml
variables:
  TF_VAR_location: 'North Europe'
```

### Release Pipeline

As long as Azure DevOps is not supporting direct YAML integration, you have to setup the environment configuration manually or use the other import options in Azure DevOps.
However, using `run_tf.sh`, the setup is straight and simple. Just use the Azure CLI task as in the optional build pipeline and make sure to set the `addSpnToEnvironment` to true. Only the arguments are different: `-e` for passing the environment name (like dev, qa, or prod) and again `-f` for non-interactive and `-d` to download.

```yaml
steps:
- task: AzureCLI@1
  displayName: 'Terraform Apply'
  inputs:
    azureSubscription: '<Azure Subscription Reference>'
    scriptPath: '$(System.DefaultWorkingDirectory)/_terraform-reference/run_tf.sh'
    arguments: '-e $(Release.EnvironmentName) -f -d'
    addSpnToEnvironment: true
    workingDirectory: '$(System.DefaultWorkingDirectory)/_terraform-reference'
```
