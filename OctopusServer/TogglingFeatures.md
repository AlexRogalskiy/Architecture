# Toggling Feature Availability
Sometimes we don’t want a feature to be available to all customers, immediately upon deployment of the code. We may want to slowly trial an idea with some early alpha/beta-testers, or we might want a kill-switch to be able to react quickly in a way that doesn’t require redeployment.

For this, we have a range of different types of feature flags and toggles that control the availability of our features.

## User-facing Configuration Settings
These are visible to customers in the Server Portal, under Configuration > Settings.

### How to decide?

### Example implementation

## Hidden Configuration Settings
These don’t show up under Configuration > Settings, but can be manually navigated to if you know the right URI slug, eg:

https://<octopus-server>/app#/<space-id>/configuration/settings/**behavioural-telemetry**

### How to decide?
The same as User-facing Configuration Settings, but when you only want Octopus people to change them

### Example implementation
https://github.com/OctopusDeploy/OctopusDeploy/pull/9205

Specifically, add the `IHiddenConfigurationSettings` interface to your `ConfigurationSettings<,>` implementation.

## FeatureToggles
These are lightweight, environment-variable-only toggles. 

They are intended to be used to control things that happen in the startup and bootstrapping of Octopus Server, for cases where we don’t yet have a DB or IoC Container available. 

They should be short-lived, and proactively removed after their rollout has been proven and stabilised.

### How to decide?
Use these if:
* You only need a simple enabled/disabled switch
* You need to toggle something at the infrastructure-level of Server
* You only need to enable the toggle for a small number of instances
* You intend the toggle to be short-lived (eg < 2 months)

Do not use these if:
* Your toggle will be long-lived or permanent
* You want end-users to be able to see/control feature availability
* Your toggle requires additional related settings (eg a modifiable endpoint, threshold values etc) to function

### Example implementation
https://github.com/OctopusDeploy/OctopusDeploy/pull/10702

## The Konami Code
?

### How to decide?

### Example implementation
