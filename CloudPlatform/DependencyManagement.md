# Summary #

Cloud Platform is a relatively small team right now and we relay heavily on other platform teams to help us with work that needs to be done but is not related to the core of Cloud Platform does.

# Frontend #

Cloud Portal UI started as a copy of [Octopus Server UI](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/newportal) in 2019. This was done to speed up the development of [Hosted v2](https://docs.google.com/drawings/d/1jl9_g7y_V-PfPrLZY-CdV2cr58mW-A38ieVHn7zFFpg/edit) and to make future updates easier. The first goal has been achieved. The second one not so much and Cloud Portal UI is mostly 2/3 years old at the time of writing this document. 

The UI code is not where Cloud Platform wants to spend most of its time but at the same time it needs to be kept up to date.

In the future, Team Frontend Platform is planning to develop a platform that other teams can use to build web frontends on top of. We want the future of Cloud Portal UI to be build on top of it so the maintenance cost reduced to minium.

Until this platform is available our approach is to keep updating Cloud Portal UI dependencies to be in sync with Octopus Server UI dependencies. If a version of a dependency works for Octopus Server UI then there is a very good chance that it will also work for us.

# Backend #

Backend dependencies will be updated using the same approach as the one used for frontend dependencies.

# 3rd Party Services #

Cloud Platform is a glue that connects several 3rd party services (Azure SQL, Azure Storage Account, Azure Kubernetes Service, Better Uptime, Seq, SumoLogic, etc.). Some of them provide official libraries that wrap their APIs.  These libraries need also be kep up to date.

# Frequency #

Ideally, there would be a tool that we could point at a source (e.g. Octopus Server UI) and destination (e.g. Cloud Portal UI) and it would gradually sync dependencies over certain period of time (e.g. one a week).

Until ^ is implemented the described process will be performed manually once a quarter.
