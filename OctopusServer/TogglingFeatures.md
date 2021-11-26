# Toggling Feature Availability
Sometimes we don't want a feature to be available to all customers, immediately upon deployment of the code. We may want to slowly trial an idea with some early alpha/beta-testers, or we might want a kill-switch to be able to react quickly in a way that doesn't require redeployment.

For this, we have a range of different types of feature flags and toggles that control the availability of our features.

## Octopus Server Portal
There are two different places that coud conceivably toggle behaviour, that are exposed in the Server Portal

### Configuration > Settings
Configuration Settings are collections of values that control a set of behaviour. They allow Customers to control and configure things like Authentication mechanism (eg Octopus SSO using AAD, which requires a number of different identifiers, secrets and URLs to get working)

Each of these settings is declared by creating a new class that derives from [ConfigurationSettings<,,>](https://github.com/OctopusDeploy/ServerExtensibility/blob/8f7387ccf824972af3ac6bde8568f379bb8987c7/source/Server.Extensibility/Extensions/Infrastructure/Configuration/ConfigurationSettings.cs).

Configuration Settings are persisted to the `Configuration` table.

#### Hidden Settings
Some configuration entries are decorated with an `IHiddenConfigurationSettings` attribute, which prevents them from showing up in the menu of available Configuration settings. However, you can still navigate to them if you know the `Id` of the Configuration item. Just click into a visible Setting, and replace the final slug of the URL with the `Id` of your hidden configuration item.

This can be useful if we've got something that will eventually fulfil the criteria of Configuration Settings, but we want to do a quiet- or soft-launch of it first.

#### How to decide?
Use these if:
* The feature is intended to be optional
* The feature should be configurable by Customers
* The feature requires a collection of values for it to work

Do not use these if:
* It only requires a state of "Enabled" or "Disabled" (try Configuration > Features instead)
* You need to modify front-end behaviour (try Configuration > Features instead)

#### Example Implementation
[Octopus Deploy > Octopus.Server > Configuration > ServerFolders](https://github.com/OctopusDeploy/OctopusDeploy/blob/f4cf807e07869c1d0956fa83e6ec01af20a3ba67/source/Octopus.Server/Configuration/ServerFolders/ServerFoldersConfigurationSettings.cs)

### Configuration > Features
Features are more traditional Feature Flags, which have a name and a value of either Enabled or Disabled.

Under the hood, they are a specific Configuration Setting, however the values are a series of boolean flags. They are persisted in the `Configuration` table with the Id `features`.

These Features can control behaviour on both the front-end and the back-end.

#### Early-Access Features
EAP is a special tag we might add to features during an Early Access Preview. In EAP, we would default the feature to Disabled, but it will still be available to opt-in (with an appropriate risk warning - think of it as Beta testing)

#### Experimental Features (the Konami code)
Features in the Experimental section are only visible when one of the following is true:

* Octopus is running in development mode (aka locally)
* The [Konami code](https://en.wikipedia.org/wiki/Konami_Code) has been activated
* The feature is set to Enabled (eg through a DB script)

Type the Konami code while on the Features page to show any hidden Experimental Features for the duration of your session.

#### How to decide?
Use these if:
* You're building a larger feature that will eventually be on for everyone, but it's not ready yet
* It only requires a state of "Enabled" or "Disabled"
* You need to access its state from front-end code
* You only need to access its state on the backend once Server has successfully started

Do not use these if:
* You need configuration values other than a state of "Enabled" or "Disabled" (try Configuration > Settings instead)
* You need to access its state before/while Server is starting (try FeatureToggles instead)

#### Example implementation
[Declaring a Feature](https://github.com/OctopusDeploy/OctopusDeploy/blob/50085b8222b6f6086923233325ae769714ab6ea0/source/Octopus.Core/Model/Features/FeaturesConfiguration.cs)

[Using a Feature - Backend](https://github.com/OctopusDeploy/OctopusDeploy/blob/0893e0bfd9150d84d91d4f9bb880fe57eb6b03e9/source/Octopus.Server/Web/Api/Actions/ActionTemplates/ActionTemplatesSearchResponder.cs#L62-L65)

[Using a Feature - Frontend](https://github.com/OctopusDeploy/OctopusDeploy/blob/f6da6dba40f4b974e74efb92ee07af107f8c09e2/newportal/app/areas/configuration/components/FeaturesLayout/FeaturesLayout.tsx#L172-L174)

## FeatureToggles
These are lightweight, environment-variable-only toggles. 

They are intended to be used to control things that happen in the startup and bootstrapping of Octopus Server, for cases where we don't yet have a DB or IoC Container available. 

They should be short-lived, and proactively removed after their rollout has been proven and stabilised.

### How to decide?
Use these if:
* You only need a simple enabled/disabled switch
* You need to toggle something at the infrastructure-level of Server
* You only need to enable the toggle for a small number of instances
* You intend the toggle to be short-lived (eg < 2 months)

Do not use these if:
* Your toggle will be long-lived or permanent
* You want Customers to be able to see/control feature availability
* Your toggle requires additional related settings (eg a modifiable endpoint, threshold values etc) to function

### Example implementation
https://github.com/OctopusDeploy/OctopusDeploy/pull/10702
