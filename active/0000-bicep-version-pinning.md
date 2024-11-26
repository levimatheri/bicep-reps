---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: levimatheri (Levi Muriuki)
Start Date: 2024-11-20
Feature Status: Private Preview
Bicep Issue Number(s): [#12290](https://github.com/Azure/bicep/issues/12290), [#8978](https://github.com/Azure/bicep/issues/8978)
---

# Bicep version pinning

## Summary

This document proposes a feature that enables users to pin specific versions of Bicep for their deployments.

## Terms and definitions
### Definitions
- **AzCLI:** Azure Command-Line Interface, a set of commands used to create and manage Azure resources.
- **AzPwsh:** Azure PowerShell, a set of cmdlets for managing Azure resources directly from the PowerShell command line.
- **ADO:** Azure DevOps, a set of development tools for software development teams.
- **Ev2:** A deployment system used internally at Microsoft for deploying services.
- **bicepconfig.json:** A configuration file used to specify settings for the Bicep compiler.
- **Registry Module:** A reusable component or package that can be published and shared within a registry.
- **Entrypoint Bicep File:** The main Bicep file that serves as the starting point for a deployment.
- **Compilation:** The process of converting Bicep code into an Azure Resource Manager (ARM) template.

## Motivation

Currently, several tools allow the use of Bicep without pinning to a specific version, which is a common practice in CI pipelines. These tools include:
- AzCLI
- AzPwsh
- ADO task (AzureResourceManagerTemplateDeployment@3)
- GitHub Actions (azure/arm-deploy@v1)
- ADO/GitHub Actions base image

This approach poses a significant risk because when a new Bicep version is released, users are automatically upgraded to the latest version without their explicit consent. This can lead to unexpected compilation issues, especially if the new version introduces breaking changes.

Implementing Bicep version pinning would ensure reproducible Bicep compilations across all deployment workflows, whether executed locally or within CI/CD pipelines. This feature would enhance stability and predictability, providing users with greater control over their deployment environments. Additionally, it would be crucial for enabling the use of Bicep natively in Ev2 to ensure safe deployment practices.

## Detailed design
### Client side changes
#### 1a. Add new property in bicepconfig.json

The following example shows the proposed `version` property syntax to be added in the _bicepconfig.json_ file:

```json5
{
    "bicep": { // section for bicep compiler options
        "version": "0.31.92" // format: [0-9]+.[0-9]+.[0-9]+ 
    }
}
```

The resolution of the appropriate Bicep version will be determined using the closest `bicepconfig.json` to the entrypoint Bicep file merged with the default configuration. 

#### 1b. Cross-module compilation

When dealing with cross-module compilation, each module should be able to specify its own Bicep version constraint in its respective `bicepconfig.json` file. This allows for greater flexibility and ensures that each module can be compiled with the version it was designed for. This behavior would be especially helpful for when we support version ranges, i.e. a module author can specify a minimum version for compatibility in case the module uses functions or behaviors that were introduced after the minimum version.


#### 2. Update Bicep CLI
The Bicep CLI should be updated to check for the version constraint in the configuration.

If no `version` constraint is specified anywhere in the `bicepconfig.json` resolution chain, the compilation should fallback to using the currently installed version.

If the resolved `version` constraint does not match the locally-installed Bicep version, the compilation should fail with a clear message indicating the constraint violation. Otherwise, the compilation should continue normally using the resolved version.

#### 3. Update build tools
The tools highlighted in the [Motivation section](#motivation) section should be updated to parse the `bicepconfig.json` (also following the same mechanism of using the closest `bicepconfig.json` file to the entrypoint Bicep file merged with the default configuration), and install the specified Bicep binary version. The following validations should be handled:
- If no `version` constraint is specified anywhere in the resolution chain, the tooling should fallback to downloading and installing the latest Bicep version.
- If a specific `version` constraint is resolved, the tooling should download the specified Bicep version.

### Server side changes (if applicable)
N/A

### Microsoft.Resources/deployments API changes (if applicable)
N/A


## Drawbacks
- **Tooling Updates:** The various tools highlighted in the [Motivation section](#motivation) would need to be updated to support and correctly interpret the `version` property which introduces additional implementation costs. Additionally, any future changes to the mechanism of resolving Bicep configurations (such as support configuration merge semantics) and/or introducing wildcard semantics to the `version` property will require careful consideration to ensure we don't break the tooling.
- **Limited flexibility:** This current proposal does not support wildcard or operator semantics for versioning, which limits flexibility in specifying version ranges or minimum versions.
- **Degraded visibility into registry module version constraints:** When inspecting published module sources, it wouldn't be obvious what version was used to compile the module.

## Alternatives
- **Specify Bicep version in `.bicep` files:** This might look something like the following:
    ```bicep
    bicep.version=0.31.92

    resource test 'Test.RP' = {...}
    ```
    This approach provides more granular control over the Bicep version used for each individual file, which makes it clear which version is required for a specific file, which could be useful in scenarios where different files need to be compiled with different versions. It would also be easier to discern what version was used to compile a published module by inspecting the source.
    
    However, the downside with this approach is if usage of the same Bicep version for multiple is desired, it would require specifying the `version` in each file which can be a maintenance overhead.


## Rollout plan

This proposal only involves client-side changes. We will have it behind an experimental feature-flag in order to allow us to collect feedback before releasing it generally.

## Unresolved questions
1. How should we handle future updates to the version syntax in terms of updating the tooling, e.g. when we add version range syntax? As it is, the logic for parsing the `bicepconfig.json` would be scattered around multiple tools. Is there a way we can centralize this logic?
1. Technically, the Bicep versions begin with a `v`. Do we need to include this prefix as pattern of the constraint format?

## Out of scope
### Wildcard and operator semantics for Bicep versions in `bicepconfig.json`

At this time, this proposal does not include support for wildcard semantics (such as `version: "0.31.*"`) or operator syntax (such as `version: ">= 0.20.0`) for Bicep versions.
