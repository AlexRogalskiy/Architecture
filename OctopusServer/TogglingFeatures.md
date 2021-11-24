# Toggling Feature Availability
Sometimes we don’t want a feature to be available to all customers, immediately upon deployment of the code. We may want to slowly trial an idea with some early alpha/beta-testers, or we might want a kill-switch to be able to react quickly in a way that doesn’t require redeployment.

For this, we have a range of different types of feature flags and toggles that control the availability of our features.

## User-facing Configuration Settings
These are visible to customers in the Server Portal, under Configuration > Settings.

## Hidden Configuration Settings
These don’t show up under Configuration > Settings, but can be manually navigated to if you know the right URI slug, eg:

https://<octopus-server>/app#/<space-id>/configuration/settings/behavioural-telemetry

## FeatureToggles
See https://github.com/OctopusDeploy/OctopusDeploy/pull/10702 - Connect to preview for an example of use

These are lightweight, environment-variable-only toggles. They are intended to be used to control things that happen in the startup and bootstrapping of Octopus Server, for cases where we don’t yet have a DB or IoC Container available.

They should be short-lived, and proactively removed after their rollout has been proven and stabilised.

## The Konami Code
?