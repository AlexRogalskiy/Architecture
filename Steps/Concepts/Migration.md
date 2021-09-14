# Index

- [Overview](#overview)
  - [Tl;Dr](#tldr)
- [Constraints](#constraints)
- [Authoring Migrations](#authoring-migrations)
  - [Required Fields](#required-fields)
  - [Bound Values](#bound-values)
  - [Opaque/Server Types](#opaqueserver-types)
- [Migration Process](#migration-process)
  - [Multi-version Migrations](#multi-version-migrations)
  - [Minor/Patch Versions](#minorpatch-versions)
- [Step Package Repository Design](#step-package-repository-design)
  - [Recommendations](#recommendations)
    - [Tooling](#tooling)
  - [Common Code](#common-code)
- [Alternative Approaches](#alternative-approaches)
  - [Only latest major version of step in repo](#only-latest-major-version-of-step-in-repo)

# Overview

As steps and deployment targets that are created in step packages evolve over time, there will be a need for changes to a step package that result in a new major version. Reasons for these changes might include:

- Breaking changes to the [_inputs_](./InputsAndOutpus.md) of a step e.g. a new required field, or a new shape of inputs.
- A new version of some dependent tool or cloud service that is incompatible with an existing version, even though the inputs are the same. An example of this might be a tool like Terraform where the inputs to a step may not change but the templating language has changed.

Changing from one major version of a step to another will be an opt-in process for the end user configuring a step within a deployment process. As part of this change of versions, the inputs may require changes to get to a new valid shape. This process of updating the saved inputs is termed _migration_.

Additional complexity in migration arises when a user is attempting to update their step across multiple versions e.g. from v1 to v3. In this case there could be multiple sets of new required fields and transformations that need to be considered.

## Tl;Dr

- Step authors will provide migrations inside the step package to upgrade inputs for major versions
- Migrations will take current inputs and return migrated inputs
- Users will need to review migrated inputs and save the step/target after fixing any issues
- Multi-version migrations will run all relevant migrations and then return fully migrated inputs to the end user to be saved
- Step authors will be expected to provide required fields and handle bound values
- Server types (like `PackageReference`) that are new in a given version will be handled specifically and will not need to be provided by the step author in a migration
- We will use `pnpm` in step package repositories to allow multiple versions of steps to co-exist within a package and be versioned independently

# Constraints

The following constraints will apply when considering migrations:

- It should be impossible for Octopus Server to have a set of steps or packages that result in "impossible scenarios". Some examples of impossible scenarios:
  - Missing migrations: Server has v1 and v3 of a given step as a package installed didn't contain v2 and therefore there is no way to migrate a step from v1 -> v3.
  - Missing packages: Deployment processes or deployment targets exist within Server that reference versions of packages that cannot be resolved e.g a deployment process references v2 of a step but that version doesn't exist in Server anymore.
- Creating a new major version of a step should be a simple and safe process for a step author
- Migrations must be testable

# Authoring Migrations

In order to migrate inputs between versions, a step author will be expected to provide a migration function as part of the step/deployment target in the step package which will take in inputs from the previous version and will be expected to return the migrated inputs matching the current version.

In order to satisfy "No impossible scenarios" constraint mentioned above, all _major_ versions of a step/target must always be contained within a step package repository (and published step package), and therefore each migration function will only need to consider the inputs from the previous version. All versions of a step/target above v1 will require a migration, even if it is just returning the inputs unmodified (in the case of a migration where nothing has changed).

These migration functions will be stored by convention in a `migration.ts` file in the same way that the `ui`, `validation` and `executor` are stored.

## Required Fields

Some inputs from a new version of a step will likely have fields that are required that are new and cannot be derived from the previous inputs. The migration process will require that the step author supply a value for these fields, in the same way that these fields are expected to be provided in the `createInitialInputs` function of the UI definition which is used when a new step is added. This doesn't mean that the value that is supplied as the migration default has to be a valid value, for example a new `port` field might have a migration value set to 0 as there is no way to supply a more sensible default however the valid range is >0. In this case before the user can successfully save the resultant step after migration they will need to set a valid value.

In the case of multi-version migrations the implication for the step author of required fields that may not be valid is that the types of the inputs to a migration function will be correct but no assumption can be made about the validity of data. This may cause some complexity in subsequent migrations after the version that introduced the field, the step author will be expected to navigate this complexity as part of their migration scripts. This is so that the end user does not need to fill in require fields at each step of a multi-version migration.

## Bound Values

Saved inputs to a step may contain bound values. When validation functions are run these are ignored (at execution time variables are resolved and then validation is run) however for migrations these values will not be ignored. The step author will need to handle any complexities that might arise from bound values. If copying data directly from one property to another (e.g. if change shape) this is probably not much of a concern, however there may be situations where data is being parsed in order to be manipulated. Functions will be provided for step authors to detect whether a given value is a bound value to put in place any desired logic.

## Opaque/Server Types

There are certain types of inputs that are opaque, where the underlying shape at configuration and execution time are owned by Octopus Server. Examples of this include sensitive values, package/container image references and accounts. When properties of these types are part of inputs passed in to a migration function these will not be typed as the configuration/execution types.

When new properties are added to the inputs during migration to a new version that are of one of these opaque types, the step author will not be expected to supply a value for these. These properties will be marked as optional in the output type automatically by the migration api, however the property in the inputs that get passed to the `ui` and `executor` will still be defined as required. See [Bounded Contexts](./InputsAndOutpus.md#bounded-contexts) for more detail on this approach.

The migration functionality within Octopus Server will instantiate an appropriate value for these properties and input may be required from the user e.g. need to pick a package reference.

# Migration Process

The migration process will be initiated by the end user when configuring a step that has an available upgrade to a newer major version. The migration process will be run by Octopus Server, running the relevant migration functions outlined in [Authoring Migrations](#authoring-migrations) and upon completion the migrated inputs will be set as the inputs to the step and the updated UI shown to the user. The step will not be automatically saved, instead the end user performing the migration will be expected to review and adjust the inputs as they see fit and then save the updated deployment process, which will include changing the step to the new version.

## Multi-version Migrations

In the case of migrating across multiple major versions each migration function will be run in order as part of the process and the output of one migration function would become the inputs to the next migration function. Only once all relevant migrations have been run would the results be returned to the user for saving. For example upgrading from v1 to v3 would involve:

- Running the v1 -> v2 migration with the current inputs
- Running the v2 -> v3 migration with output of the v2 migration
- Returning the results of the v3 migration and setting these as the inputs of the step/target before user review and save

## Minor/Patch Versions

At this stage migrations will not triggered for updates to minor versions of steps, any additive properties that are optional will need to be handled appropriately by the step author within the `executor` of the step.

# Step Package Repository Design

When a new major version of a step/target is developed, previous versions of the step will likely still be in use within existing Octopus installations and will need to be maintained. At some point in the future certain older versions of a step might be deprecated and removed, the timing of this might differ across steps and may also be impacted by [compatibility surfaces](./Version.md#compatibility) with Octopus Server.

The repository for step packages needs to be designed to meet these requirements.

## Recommendations

It is recommended that multiple versions of steps/targets be stored side-by-side in a monorepo for the lifetime of versions. This will allow for maintenance of older version as well as publication of multiple versions of a given step/target within the same package for consumption in Octopus Server.

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

To create a new version of a step in the step package repository:

- Create a new folder representing the new version of the step/target
- Copy any relevant contents from the previous step e.g. inputs, ui, validations, executor
- Adjust contents as required. Particularly adding a version number to the inputs will aid in readability for migration functions etc. e.g. use naming like `type MyStepInputsV1` and `type MyStepInputsV2`
- Add a migration function to migrate from the previous version of the step/target to the new version

### Tooling

It is recommended that the following tooling be used to assist with dependency management and versioning within a step package repository:

- [pnpm](https://pnpm.io/)
- [changesets](https://github.com/atlassian/changesets)

## Common Code

When introducing a new version of a step, there still may be common code that is compatible across multiple versions of that step, this code could be moved to common packages as the step author sees fit. This could theoretically also be extended to input types/migrations if desired.

# Alternative Approaches

There are some alternative approaches to migration and repository design that have been considered, these are captured here for context.

## Only latest major version of step in repo

Instead of enforcing that all major versions of a step exist in the repo and published package, this approach would only see the latest major version of any given step/target being stored in the repository and published package. Git branches would be used to track previous versions that need to be supported e.g. `release/v1`.

A set of migration functions would need to be maintained to update previous major versions to the current version, in a sequential manner similar to DbUp migrations e.g. v1 -> v2, v2 -> v3. Tooling would need to enforce this to ensure that steps can be migrated safely.

Patching fixes to a previous release would involve committing to release branch then merging forward as required.

Benefits:

- Don't necessarily need monorepo tooling such as pnpm or changesets. Changesets would still be useful for versioning and release note management though.
- Less code to have in the repo at any given time.
- Published step packages would be smaller as they would only contain the latest major version, not all major versions. This would only really be at acquisition time, at execution time only the specific version required is sent to the worker to be executed.

Downsides:

- Independently versioning multiple steps/targets becomes more complex. For example if there is a v2 of a step but other steps are still on v1 how does a branching strategy work?
- Depending on which published versions of a step have been installed, Server needs to be more aware that certain versions may not be available for operations like migration. This may not be a big deal, but adds additional complexity to the handling of operations in Server.
