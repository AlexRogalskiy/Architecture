# Names And Ids
...Or what entities do we map between names and Ids for VCS??


## Version Controlled Entities

### Deployment Settings
📛 `DeploymentSettings.VersioningStrategy.DonorPackage.Step`

```hcl
versioning_strategy {

    donor_package {
        package = "putty"
        step = "Hello world (using PowerShell)"
    }
}
```


### Deployment Process
📛There a a [bunch of special variables](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Features/DeploymentProcesses/SpecialVariableSchema.cs#L13) that map names back to Ids. This is used in the `NameIdResolver.cs` when [mapping between tinytypes](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Features/DeploymentProcesses/NameIdResolver.cs#L157). In this case the properties have an _implied_ tiny type type via the `SpecialVariableSchema` lookup.
```c#
new SpecialVariableSchema(SpecialVariables.Action.Package.FeedId, "Feed", typeof(FeedIdOrName), SpecialVariableType.String),
new SpecialVariableSchema(SpecialVariables.Action.Package.NuGetFeedId, "Feed", typeof(FeedIdOrName), SpecialVariableType.String),
new SpecialVariableSchema(SpecialVariables.Action.DeployRelease.ProjectId, "Project", typeof(ProjectIdOrName), SpecialVariableType.String),
new SpecialVariableSchema(SpecialVariables.Action.Azure.AccountId, "Account", typeof(AccountIdOrName), SpecialVariableType.String),

// Email
new SpecialVariableSchema(SpecialVariables.Action.Email.ToTeamIds, "Team", typeof(TeamIdOrName), SpecialVariableType.CsvString),
new SpecialVariableSchema(SpecialVariables.Action.Email.CCTeamIds, "Team", typeof(TeamIdOrName), SpecialVariableType.CsvString),
new SpecialVariableSchema(SpecialVariables.Action.Email.BccTeamIds, "Team", typeof(TeamIdOrName), SpecialVariableType.CsvString),

//Manual
  new SpecialVariableSchema(SpecialVariables.Action.Manual.ResponsibleTeamIds, "Team", typeof(TeamIdOrName), SpecialVariableType.CsvString),
```
e.g.
```hcl
step "Manual Intervention Required" {

    action {
        action_type = "Octopus.Manual"
        properties = {
            Octopus.Action.Manual.ResponsibleTeamIds = "Everyone,Octopus Administrators"
        }
```


📛 Channel Names
```hcl

step "Hello world (using PowerShell)" {
    action {
        channels = ["Default", "Second Channal!!!"]
```

📛 Environment Names
```hcl

step "Hello world (using PowerShell)" {
    action {
        environments = ["Dev", "Staging"]
```
        

📛 Packages - Packege feeds are referred to via the ID.
```hcl
packages "putty" {
    acquisition_location = "Server"
    feed = "Octopus Server (built-in)"
    package_id = "putty"
```

If the feed type is a "deploy a project" step then the package itself is also the project name. This process takes place in the [OCL serialization layer](https://github.com/OctopusDeploy/OctopusDeploy/blob/e641389b25df8c8c6c684149426eb82167075b26/source/Octopus.Core/Serialization/Ocl/OclConverters/DeploymentActionOclConverter.cs#L161).
```hcl
step "Deploy a Release" {

    action {
        action_type = "Octopus.DeployRelease"
        properties = {
            Octopus.Action.DeployRelease.DeploymentCondition = "Always"
            Octopus.Action.DeployRelease.ProjectId = "BaseProject"
        }
    }
```

### Database Entities
Variables

Releases

Triggers


## Conversion Strategies and Locations

### `NameIdResolver`
* `SnapshotSnapshotter` - When snapshotting a `DeploymentProcess`, `VariableSet` (or in future `RunbookProcess`) Them the [Names are mapped to Ids](https://github.com/OctopusDeploy/OctopusDeploy/blob/c6b4737add2dc8e740e8f658122d6e3263744f82/source/Octopus.Core/Features/SnapshotSnapshotter.cs#L141). This is passed in from various controllers around creating or modifying snapshots.
* `WarningGuidanceNameIdResolver` - (via `ProcessWarningFinder` and `RunbookProcessWarningFinder`). This is used via the `ValidateDeploymentProcessController` endpoint that "validates" a deployment process [when it has been loaded](https://github.com/OctopusDeploy/OctopusDeploy/blob/27651b587bcb62455fe4f47e46368a77af0edd26/newportal/app/areas/projects/components/Process/ProcessStepsLayout.tsx#L259).
* `ProjectToVersionControlConverter` - When [converting a deployment process](https://github.com/OctopusDeploy/OctopusDeploy/blob/a033f6b5703f1aed50fa69131fa7eed2cef50747/source/Octopus.Server/Web/Api/Actions/Projects/ProjectToVersionControlConverter.cs#L170) to vcs we convert the Ids to Names.
