# Index

- [Overview](#overview)
  - [Tl;Dr:](#tldr)
  - [Examples](#examples)
- [Versioning](#versioning)
- [Compatibility](#compatibility)
- [Developer Experience](#developer-experience)
- [Deployment Target Versioning Limitations](#deployment-target-versioning-limitations)

# Overview

> This section is detailing our intentions for versioning and compatibility for Step Packages and Octopus Server, but may change as we implement them.

Versioning and Compatibility are orthogonal, but related concerns within the Step Package ecosystem.

_Versioning_ of a Step Package is about enforcing _internal consistency_ of Step Package components as they are used within Octopus Server - making sure we use the right Step Package components to edit, validate, and execute a step after it has been created, and defining useful tolerances within version ranges that allow us to more easily ship bug fixes and small changes to steps without requiring user intervention to upgrade steps in existing processes.

_Compatibility_ refers to the compatibility of various Step Package components with Server components. There are many compatibility surfaces, which we will explore below.

## Tl;Dr:

- We care about **versions** of Step Packages, and the **compatibility** of their components with Server components.
- Compatibility surfaces include: manifest + conventions, UI API, input processing (both server and UI), validation, execution, and migrations.
- We would like to embed version information for all compatibility surfaces into metadata.json
- This information can be made available to the "other side" of the compatibility surfaces in a variety of ways.
- Step packages will implement [migrations](./Migration.md) to support their versions changing over time.
- We will use shims to support the versions of compatibility surfaces changing over time.
- Server will be able to determine what Step Packages it might support.

## Examples

Different versions of Step Packages. For example, we ship `step-package-azurestorage.1.0.0`. We then make a change to the set of inputs such that it is no longer runtime compatible with existing steps that have been configured in deployment and runbook process, and ship a new version, `step-package-azurestorage.2.0.0`. We need to migrate the set of inputs stored against those steps in our deployment and runbook processes, so that the they can be configured and executed using `step-package-azurestorage.2.0.0`.

We have multiple versions of the same Step Package that we want to expose to users, so that users can pick which one they want to use. For example, we might have `terraform-1.13.0` which only works with the `0.13.x` versions of terraform. If a user wants to target terraform version `0.14.x`, they would need to upgrade their step to `terraform-1.14.0`. We could also implement this through configuration alone, so that there is a single step that supports multiple versions of terraform with a version selector at the top.

We have a step like `step-package-azurestorage.1.0.0` which is compiled against `step-ui-api.1.0.0`. We then make a breaking change to the `step-ui-api` and release a new version `step-ui-api.2.0.0` and update our framework in portal as well, so that `step-package-azurestorage.1.0.0` is no longer directly compatible with portal. We can either upgrade the `step-ui-api` dependency and release a new version of our step (say `step-package-azurestorage.2.0.0`) to make it compatible and deprecate the old step somehow, or we could have a compatibility shim in portal so that portal can support steps compiled against both `step-ui-api.1.0.0` and `step-ui-api.2.0.0`.

# Versioning

A Step Package is a versioned component. It's version is recorded within the `Version` field of its `metadata.json` file. The version will follow [SemVer 2.0.0](https://semver.org/)-like versioning semantics.

Over time, we may make changes to a Step Package, and increase the version number accordingly.

Why _semver-like_? Patch and minor increment semantics will be exactly the same, but major increments are slightly different. For Step Packages, a major version increment will be required it changes in a way that requires user intervention for it to function correctly - this may be adding new information, or changing existing information.

**Minor Version Increments:** Within Server, we will automatically retrieve the _latest version_ of the same major version a step was created with when working within any of the steps components - UI, validator, executor, etc. This means for all backwards-compatible feature additions and bug fixes, users will recieve these benefits immediately in their existing processes, without having to update anything.

**Major Version Increments:** For major version increments, a user will need to opt-in to them. There will be a visual indication presented on the step UI if a newer version is available. If it is, and the user opts into the next version available, the step will be upgraded to the newer version

For each major version upgrade, a [migration](./Migration.md) will need to be provided by the Step Package in question to migrate the persisted step inputs from `vCurrent` to `vSelected`. This may require running a number of migration functions sequentially to upgrade the step to the required version.

[Further Discussion](https://docs.google.com/document/d/1RB4PzPpbtMJBqEHxQCPD2qGMNUCGAJ8KoXEOqXC9yAA/edit#heading=h.1sahu1il44s6)

[Migration Concepts](./Migration.md)

# Compatibility

There are many compatibility surfaces within the Step Package ecosystem. Making sure these surfaces have explicit versioning in-place allows us to make changes of them over time, and make deliberate decisions about their compatibility as they evolve.

**Step Package Version**
Step packages have a version when they are built. When a step is added to a process, we care about this version, as it will dicatate how we edit and execute the step ongoing. We will eventually want to upgrade steps to newer version of the same step.

The version for the Step Package is included in its file name.

**Step Package Manifest Schema Version**
The component in Server that loads Step Packages cares about the the conventions adhered to within the Step Package, and the structure of the Step Package manifest file. This is an API surface between Server and Step Packages, and is also relied upon by the [Step Package CLI](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Components/StepPackageCLI.md).

The JSON schema for the Step Package manifest will set a version that represents the current set of conventions and structure of the manifest.

**Step UI API Version**
The Step Package UI Framework cares about versions. When we load a `ui.ts` from a Step Package, it either needs to work with the current version of the UI Framework embedded in server, or be put through compatibility shims so that it will work with vLatest of the framework and Step UI API.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Step Package Input Schema Version**
When a step is being edited via the UI framework, or is going through the execution pipeline, some components will inspect the step input Json Schema, looking for certain markers. These markers are encoded within the Step Package CLI.

The Step Package input schema will contain a version number that indicates what version of the Input Schema is currently in use. This version is owned by the Step Package CLI, and embedded in the Step Package input schema when it is generated at build time.

**Step Package Validation API Version**
When we want to persist a set of inputs for a step, Server needs to be able to run the validator defined by the step to ensure the inputs are correct and valid. If the Step Package Validation API surface changes, Server will need to interact with Step Packages that use older Validation API versions through compatibility shims.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Step Package Execution API Version**
When we want to execute a step, Calamari needs to provide a set of arguments to our Step Package Bootstrapper, which then works with these inputs to decrypt files, establish the OctopusContext ingest inputs, and provide them all to the Executor.

We will always assume Calamari uses `vLatest` of the Step Bootstrapper. The Step Bootstrapper is the place where compatibility shims might be used should a step be built against Execution framework `vPrevious`.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Step Package Migration API Version**
When we want to migrate a set of inputs for a step from vCurrent to vSelected, Server needs to be able to run the migration functions defined by the target vSelected step to apply the appropriate changes to the vCurrent inputs. If the Step Package Migration API surface changes, Server will need to interact with Step Packages that use older Migration API versions through compatibility shims.

This version will be embedded within the Step Package manifest by the Step Package CLI at build time.

**Overall Server support for a given Step API Version**
We need some mechanism so that Server can make decisions about what Step Packages to expose to users, based on what versions it knows it can support.

TBA: how we will accomplish this.

# Developer Experience

To ensure a simple experience for Step Package developers, we will produce a _meta-package_ that they can install via `npm i --save-dev @octopus/step-api`. We don't want developers to have to worry about what version of individual Step Package dependencies they should use.

The meta-package will include all of the Step Package APIs required to build a Step Package.

We do not need all of these APIs in Server, so we will produce separate API packages that server can use, to minimise the need to change Server if and when they change: `npm i --save-dev @octopus/step-ui-api`

To make it simpler to build both the individual packages and the meta-package, we will use a _monorepo_ for these API codebases (see the [step-api repository](https://github.com/octopusdeploy/step-api)).

# Deployment Target Versioning Limitations

Deployment targets will not be allowed to change beyond a `1.x` version. This means they cannot, after being shipped, undergo a major version update. We can however add new optional fields, and ship bug fixes for them (minor and patch updates respectively).

The reasons for this decision are:

- There is a _hard problem_ to solve in step-to-target compatibility. If we upgrade a step to a new version in a deployment process, and it requires a new version of its associated deployment target - how would we handle that migration? We cannot deterministically evaluate which targets might need upgrading. We do not want to burden users with this decision either.
- Deployment targets should remain _very stable_ after their initial definition. The cloud APIs they model tend to be extremely long-lived, with cloud providers going to great lengths to satisfy backwards compatibility requirements. They should not need to change based on external changes.
- The future for deployment targets is likely to be _Reflected Targets_ - targets that are discovered from cloud environments, not registered with server. For these cases, the compatibility surface between steps and targets is shifted to a contract that would sit between the step and its target cloud environment, not between the step and its Octopus deployment target.

For these reasons this limitation is considered reasonable, and future deployment target development in Reflected Targets will attempt to tackle the problem of step <> target compatibility. 