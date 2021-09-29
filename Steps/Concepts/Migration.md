# Index

- [Overview](#overview)
  - [Tl;Dr](#tldr)
- [Constraints](#constraints)
- [Authoring](#authoring)
  - [Required Fields](#required-fields)
  - [Bound Values](#bound-values)
  - [Opaque/Server Types](#opaqueserver-types)
- [User Experience](#user-experience)
  - [Multi-version Migrations](#multi-version-migrations)
  - [Minor/Patch Versions](#minorpatch-versions)
- [Step Package Repository Design](#step-package-repository-design)
  - [Recommendations](#recommendations)
    - [Tooling](#tooling)
  - [Common Code](#common-code)

# Overview

As step packages evolve over time, there will be a need for changes to a step package that require a major version increment.

A major version increment will be required when changes are made that require user intervention before it will be able to function correctly - this may be adding new information, changing existing information, or changing the behaviour of the step in a way that may be incompatible with the existing inputs.

Reasons for these changes might include (but are not limited to):

- Breaking changes to the [_inputs_](./InputsAndOutpus.md) of a step e.g. a new required field that cannot be defaulted, or a new shape of inputs that we cannot simply map to.
- A new version of some dependent tool or cloud service that is incompatible with an existing version, even though the inputs are the same. An example of this might be a tool like Terraform where the inputs to a step may not change but the templating language has changed.

Changing from one major version of a step to another will be an opt-in process for the user configuring the step. When changing versions, the inputs for the step may require _migration_ - changes that take the inputs from their previously valid form, to a new valid form.

Additional complexity in migration arises when a user is updating their step across multiple versions e.g. from `v1` to `v3`. In this case there could be multiple sets of new required fields and transformations that need to be considered.

## Tl;Dr

- Step authors will provide migrations inside step packages
- A migration must be provided for each major version of a step package > 1, so `v3` of a package would include the `v1 => v2` migration, and the `v2 => v3` migration.
- Migration will accept vPrevious inputs and return vNext migrated inputs
- Migrated inputs may still require user intervention before they are valid inputs
- Multi-version migrations (i.e. `v1 => v3`) will run all relevant migrations sequentially as a single logical operation
- Step authors will be expected to provide any new required fields and handle bound values within migrations
- Server types (like `PackageReference`) that are new in a given version will be handled specifically and will not need to be provided by the step author in a migration
- We will use `pnpm` in step package repositories to allow multiple versions of steps to co-exist within a package and be versioned independently

# Constraints

The following constraints will apply when considering migrations:

- It should be impossible for Octopus Server to have a set of step packages that result in "impossible scenarios". Some examples of impossible scenarios:
  - Missing migrations: Server has v1 and v3 of a given step package but not v2. Just because server is missing an intermediary version does not mean it should to be able to perform the upgrade from v1 to v3.
  - Missing packages: Deployment processes or deployment targets exist within Server that reference versions of step packages that cannot be resolved by Server
- Creating and maintaining new major versions of a step package should be a simple and safe process for a step package author
- Migrations must be testable

# Authoring

For each major step package version > 1, a step author will be expected to provide the set of migration functions required to take the inputs from `v1` to `vCurrent`. This satisfies the _no impossible scenarios_ constraint. How this is achieved is left to the step package author, as long as it fits the defined migration API shape.

All versions of a step package above v1 will require a migration, even if it is just returning the inputs unmodified in the case of a migration where nothing has changed.

These migration functions will be defined by convention in a `migration.ts` file in the same way that the `ui`, `validation` and `executor` are defined within a step package.

[Further Discussion](https://octopusdeploy.slack.com/archives/C02D9DHAE78/p1632871549200900)

## Required Fields

Some inputs from a new version of a step will likely have fields that are required that are new and cannot be derived from the previous inputs. The migration process will require that the step author supply a value for these fields, in the same way that these fields are expected to be provided in the `createInitialInputs` function of the UI definition which is used when a new step is added. This doesn't mean that the value that is supplied as the migration default has to be a valid value, for example a new `port` field might have a migration value set to 0 as there is no way to supply a more sensible default however the valid range is > 0. In this case before the user can successfully save the step after migration they will need to set a valid value.

The implications of this on multi-version migrations are that the types of the inputs to a migration function will be correct but no assumption can be made about the validity of data. This may cause some complexity in subsequent migrations after the version that introduced the field - it is up to the step author to handle this.

## Bound Values

Inputs to a step may contain bound values. When validation functions are run these are either ignored during configuration time or materialized at execution time. However for migrations bound values will not be ignored. The step author will need to handle any complexities that might arise from bound values. If copying data directly from one property to another (e.g. if change shape) this is not a concern, however there may be situations where data is being parsed in order to be manipulated. Functions will be provided for step authors to detect whether a given value is a bound value to put in place any desired logic.

## Opaque/Server Types

There are certain input types that are opaque to the migration API, where the underlying shape at configuration and execution time are owned and defined by Octopus Server. Examples of this include sensitive values, package/container image references and accounts. When properties of these types are part of inputs passed in to a migration function these will not be typed as the configuration/execution types.

When new properties are added to the inputs during migration to a new version that are of one of these opaque types, the step author will not be expected to supply a value for these. These properties will be marked as optional in the output type automatically by the migration api, however the property in the inputs that get passed to the `ui` and `executor` will still be defined as required. See [Bounded Contexts](./InputsAndOutpus.md#bounded-contexts) for more detail on this approach.

The migration functionality within Octopus Server will instantiate an appropriate value for these properties and input may be required from the user e.g. need to pick a package reference.

# User Experience

The migration process will be initiated by the end user when configuring a step that has an available newer major version.

The migration process will be run by Octopus Server, which will run the relevant migration functions outlined in [Authoring](#authoring) and upon completion return the migrated inputs, which will be displayed in an updated step UI to the user.

The step will not be automatically saved, instead the end user performing the migration will be expected to review and adjust the inputs as they see fit and then save the updated deployment process.

## Multi-version Migrations

In the case of migrating across multiple major versions each migration function will be run sequentially in order as part of the process and the output of one migration function would become the inputs to the next migration function. Only once all relevant migrations have been run would the results be returned to the user for review and saving.

For example, upgrading from v1 to v3 would involve:

- Running the v1 -> v2 migration with the current inputs
- Running the v2 -> v3 migration with output of the v2 migration
- Returning the results of the v3 migration to the user to review and save

## Minor/Patch Versions

At this stage migrations will not triggered for updates to minor versions of steps, any additive properties that are optional will need to be handled appropriately by the step author within the `executor` of the step.

# Step Package Repository Design

When a new major version of a step package is developed, previous versions will likely still be in use within existing Octopus installations and will need to be maintained. At some point in the future older versions of a step package might be deprecated and removed, the timing of this might differ across steps and may also be impacted by [compatibility](./Version.md#compatibility) with Octopus Server.

To make supporting concurrent versions of step packages simpler, we recommend a repository design that has each supported version of a step package co-existing side-by-side, rather than relying on git branches to support this co-existence.

The goal of this structure is to make it simpler to add and maintain concurrent versions of steps over time, which helps satisfy our _Creating and maintaining new major versions of a step package should be a simple and safe process for a step package author_ constraint.

## Recommendations

It is recommended that multiple versions of step packages be stored side-by-side in a monorepo for the lifetime of each version.

This structure:

- Simplifies maintaining and patching not vCurrent versions of a step package
- Makes automating dependency updates for security reasons easier (dependabot)
- Makes reasoning about the currently supported versions of step packages simpler
- Makes adding a new version straightforward

An example repository:

```
my-step-package-repo
|
|-- steps
    |
    |-- my-step-v1
        |
        |-- executor.ts
        |-- inputs.ts
        |-- ui.ts
        |-- validation.ts
        |-- ...
    |
    |-- my-step-v2
        |
        |-- executor.ts
        |-- inputs.ts
        |-- ui.ts
        |-- validation.ts
        |-- migration.ts
        |-- ...
```

To create a new version of a step in a step package repository:

- Create a new folder for the new version of the package
- Copy the contents from the previous version's folder
- Adjust contents as required. Particularly adding a version number to the inputs will aid in readability for migration functions etc. e.g. use naming like `type MyStepInputsV1` and `type MyStepInputsV2`
- Add a new migration function to the existing migration function set within `migration.ts` to migrate from the previous version of the step/target to the new version

### Tooling

It is recommended that the following tooling be used to assist with dependency management and versioning within a step package repository:

- [pnpm](https://pnpm.io/)
- [changesets](https://github.com/atlassian/changesets)

For an example of these tools in action, see the [Azure Storage Step Package](https://github.com/octopusdeploy/step-package-azurestorage)

## Common Code

When introducing a new version of a step, there still may be common code that is compatible across multiple versions of that step, this code could be moved to common packages as the step author sees fit. This could theoretically also be extended to input types/migrations if desired.
