# Octopus Core + Server API Philosophy

# Executive summary

* Octopus uses a command/request pattern for interactions with its API.
* Octopus has numerous pre-packaged API clients for different languages.
* API contract libraries are pluggable and can be mixed and matched at will.

# Princples

## The payload is the contract.

The command or request payload received by the API surface (there is no other kind of payload) must contain the entirety of the information required in order to perform the desired action.

Payloads must be transport-independent and not rely on any additional contextual information provided by any particular transport. For example, the ambient knowledge of a "space context" provided via an HTTP route has no meaning when that same command payload must also be receivable via a web socket or a direct call to `mediator.Do(...)`.

## The transport is immaterial.

The transport layer's responsibility is only to ensure that the contract DTO is assembled correctly and then to dispatch that contract to its appropriate handler. The last is done solely via the `IMediator`.

An MVC controller action method might look something like this:

```csharp
public async Task<DoFooResponse> DoFoo(DoFooCommand command, CancellationToken cancellationToken)
{
    var response = await mediator.Do(command, cancellationToken);
    return response;
}
```

An equivalent service message handler might look something like this:

```csharp
foreach (var command in result.OutputActions)
{
    await mediator.Do(command, cancellationToken);
}
```

In both cases, the `DoFooCommand` would be dispatched to the same handler, where exactly the same business logic would be applied. E.g.

```csharp
public class DoFooCommandHandler : IHandleCommand<DoFooCommand, DoFooResponse>
{
    // ...

    public async Task<DoFooResponse>(DoFooCommand command, CancellationToken cancellationToken)
    {
        throw new NotImplementedException();
    }
}
```

## The service which provides the contract defines the contract.

Message contracts (command, request, event and resource DTOs) define the capabilities provided by a service, and are dictated by that service. They are not some nebulous thing without an owner.

A consumer can define whatever contract it likes, but having a consumer define a contract to “Make me a coffee” and attempting to impose that contract upon an industrial press or a washing machine is ridiculous.

Our position is that the provider of a service is responsible for defining the contract for that service, and is responsible for honouring that contract.

# Contract definitions

With our push towards modularity in its various guises, this means that DTO definitions are likely to be managed by different teams within Octopus, and this is to be celebrated rather than resisted.
* We expect all message contracts to have a clear owner.
* The provider of a service is the owner of its message contract.
* The provider of a service is responsible for honouring that contract.
* Contract definitions live close to the code providing that service (i.e. generally the same Git repository).
* Contract definitions are exposed via Swagger/OpenAPI, JSON Schema and pre-packaged client libraries.

The base contracts of `ICommand<TCommand, TResponse>`, `IRequest<TRequest, TResponse>` etc. are defined in the `OctopusDeploy` repository in the `Octopus.Core.MessageContracts.Base` project.

For Octopus Deploy's core capabilities, contracts are defined in C# in the `OctopusDeploy` repository, in the `Octopus.Server.MessageContracts` project. They are published:
1. directly as a NuGet package as `Octopus.Server.MessageContracts.<version>.nupkg`;
1. as JSON Schema objects to the Octopus schema registry;
1. (via the schema registry) as
    - `com.octopus.server.messagecontracts.<version>.jar` for Java/Kotlin/JVM languages;
    - `octopus-server-messagecontracts-<version>.tar.gz` for Python;
    - and so on.

# API versioning and backwards compatibility

**We do not break backwards compatibility.**

**We DO NOT break backwards compatibility.**

**WE DO NOT BREAK BACKWARDS COMPATIBILITY.**

Before introducing any discussion about API versioning, it is important to understand that our loyal customers are our greatest promoters. They have built many of their own solutions on the top of our existing API surface. We respect and appreciate that, and do not intend to break their solutions or harm that loyalty.

That having been said, there exists a real need in many cases to make breaking changes to API operations in order to accommodate new features, security fixes etc., and we must have a way to achieve this.

Historically, we have attempted to keep the Octopus API entirely backwardly-compatible - and with quite a degree of success, but at significant cost. We have also not given our customers any indication that we plan to deprecate our existing API surface, and to do so now would be a rude surprise to many of them. We will, of course, eventually need to deprecate and remove some payloads, but we will 1) collect metrics on their use (where we can) to identify when they are disused; 2) telegraph our intentions quite some time in advance; and 3) give our customers a high-quality and low-friction upgrade path

## API versioning

**We version individual operations and resource types rather than the entire API surface.**

In other words, we do this:

```
POST /api/spaces/Space-1/rename-space/v1    # API version suffix (at the end) allows us to version just this operation.
```

rather than this:

```
POST /api/v1/spaces/Space-1/rename-space    # API version prefix (at the beginning) requires us to version the entire API surface.
```

This gives us numerous benefits, including:
* No "big bang" version upgrade just because we need to make a breaking change to a single operation.
* We can easily deprecate a single operation.
* We can easily _upgrade_ a single operation.
* Each team within Octopus can iterate independently and not have to coordinate breaking API changes.
* Extension authors are not obliged to version anything in lock-step with us.

All DTOs forming part of the Octopus API surface must be versioned via a version suffix. This includes commands (e.g. `CreateProjectCommandV1`), requests (e.g. `GetSpaceRequestV1`), events (e.g. `TeamCreatedEventV1`) and resources (e.g. `ProjectResourceV1`). Any contract without a version number suffix is declared to be V1 of that contract, and subsequent numbering starts at V2.

### Forwarding old commands/requests to new handlers

When we introduce a breaking change, we do not want multiple forks of the same business logic running. In order to achieve this, we forward any old commands/requests to the new handlers.

E.g. consider a `GetDeploymentRequestV1` request. Its relevant controller and handler would initially look something like this:

```csharp
[ApiController]
public class GetDeploymentController : ControllerBase
{
    [HttpGet]
    [Route("api/spaces/{spaceId}/projects/{projectId}/releases/{releaseId}/deployments/{deploymentId}/v1")]
    [Route("api/spaces/{spaceId}/projects/{projectId}/git-ref/{gitRef}/releases/{releaseId}/deployments/{deploymentId}/v1")]
    public async Task<GetDeploymentResponseV1> GetDeployment([FromRoute] GetDeploymentRequestV1 request, CancellationToken cancellationToken)
    {
        var response = await mediator.Request(request, cancellationToken);
        return response;
    }
}

class GetDeploymentRequestV1Handler: IHandleRequest<GetDeploymentRequestV1, GetDeploymentResponseV1>
{
    // ...

    public async Task<GetDeploymentResponseV1> Handle(GetDeploymentRequestV1 request, CancellationToken cancellationToken)
    {
        var deployment = await deploymentDocumentStore.Get(request.DeploymentId, cancellationToken);
        var deploymentResource = await deploymentResourceMapper.Map(deployment, cancellationToken);
        return new GetDeploymentResponseV1(deploymentResource);
    }
}
```

When deprecating this in favour of `GetDeploymentRequestV2`, we would do the following:
* Mark `GetDeploymentRequestV1` as `[Obsolete]`. (MVC filters will take care of returning warnings in HTTP headers.)
* Leave the existing `GetDeployment` method on the `GetDeploymentController` exactly as is.
* Add a new `GetDeployment` method to the `GetDeploymentController` accepting a `GetDeploymentRequestV2Handler` payload.
* Lift the logic currently in the `GetDeploymentRequestV1Handler` into a `GetDeploymentRequestV2Handler` handler and modify it as appropriate.
* Implement an adapter in the `GetDeploymentRequestV1Handler` to create a `GetDeploymentRequestV2` request and dispatch it accordingly.

```csharp
[ApiController]
public class GetDeploymentController : ControllerBase
{
    [HttpGet]
    [Route("api/spaces/{spaceId}/projects/{projectId}/releases/{releaseId}/deployments/{deploymentId}/v1")]
    [Route("api/spaces/{spaceId}/projects/{projectId}/git-ref/{gitRef}/releases/{releaseId}/deployments/{deploymentId}/v1")]
    public async Task<GetDeploymentResponseV1> GetDeployment([FromRoute] GetDeploymentRequestV1 request, CancellationToken cancellationToken)
    {
        var response = await mediator.Request(request, cancellationToken);
        return response;
    }

    [HttpGet]
    [Route("api/spaces/{spaceId}/projects/{projectId}/releases/{releaseId}/deployments/{deploymentId}/v2")]
    [Route("api/spaces/{spaceId}/projects/{projectId}/git-ref/{gitRef}/releases/{releaseId}/deployments/{deploymentId}/v2")]
    public async Task<GetDeploymentResponseV2> GetDeployment([FromRoute] GetDeploymentRequestV2 request, CancellationToken cancellationToken)
    {
        var response = await mediator.Request(request, cancellationToken);
        return response;
    }
}

class GetDeploymentRequestV1Handler: IHandleRequest<GetDeploymentRequestV1, GetDeploymentResponseV1>
{
    // ...

    public async Task<GetDeploymentResponseV1> Handle(GetDeploymentRequestV1 request, CancellationToken cancellationToken)
    {
        // We are now just an adaptor over the V2 API call.

        // Marshal whatever we need to in order to map from the V1 request to the V2 request.
        var spaceId = ambientSpaceContext.SpaceId;  // This comes from ambiend context in V1.
        var projectId = //TODO look up the deployment's relevant project ID so that we can include it in the V2 payload.
        var requestV2 = new GetDeploymentRequestV2(spaceId, projectId, request.DeploymentId);

        // Dispatch the request to the new handler.
        var responseV2 = await mediator.Request(requestV2, cancellationToken);

        // Map the V2 response back onto the V1 response.
        var response = new GetDeploymentResponseV1(responseV2.Deployment);

        return response;
    }
}

class GetDeploymentRequestV2Handler: IHandleRequest<GetDeploymentRequestV2, GetDeploymentResponseV2>
{
    // ...

    public async Task<GetDeploymentResponseV2> Handle(GetDeploymentRequestV2 request, CancellationToken cancellationToken)
    {
        var id = new DeploymentIdentifier(request.SpaceId, request.ProjectId, request.DeploymentId);
        var deployment = await deploymentDocumentStore.Get(id, cancellationToken);
        var deploymentResource = await deploymentResourceMapper.Map(deployment, cancellationToken);
        return new GetDeploymentResponseV2(deploymentResource);
    }
}
```

We recognise that with umpteen deprecated versions of an operation all chaining together, the performance is likely to take a small hit for callers of the older operation versions. We are comfortable with this as the expectation is that 1) there will be good reasons to upgrade to a newer (and faster) version of the operation; and 2) we will have provided a low-friction upgrade path.

### Use the `[Obsolete]` attribute, Luke.

All versions of a DTO except the most current version must be annotated with an `[Obsolete]` attribute.

### Must we make new features available via old API versions?

This is not required.

Octopus is an evolving product with an ever-increasing number of features. Not all of those features must necessarily be back-ported to all versions of the API.

Ease of migration is critically important. We must give our customers (and ourselves) incremental ways to move between API versions. “Big bang” changes are expressly off the table, as are lock-step releases.

## Backwards compatibility

### How do we avoid breaking backwards compatibility?

No matter how our payload types are initially authored, the source of truth is the JSON Schema representations of those types.

In our unit test suites, we apply a convention test to all of our DTO contracts which converts those contracts to JSON Schema documents and then asks our internal schema registry if any of those changes are breaking changes. If there exists a breaking change, the build will fail - both on the local, development workstation (giving fast feedback to the engineer) and on the build agents (thus preventing a breaking change from rolling out the door).

Whenever a build successfully completes on `main`, our schema registry is updated to contain the latest versions of all of our DTO definitions, with all subsequent test runs now using the updated versions of those schema documents.

### What about HTTP routes?

HTTP routes are covered by assent tests. They are not the responsibility of the schema registry - remember, **the payload is the contract**.

# Transport-specific concerns
## HTTP

### HTTP routes form part of the HTTP contract surface.

Although Octopus Deploy has historically provided a hypermedia-shaped `Links` collection on its resource types, this has led to a misguided belief that HTTP routes can be changed at will as long as the `Links` collection is kept up to date. This position is at odds with API clients generated from the OpenAPI specification (which we publish and support). In this light, we must view HTTP routes as part of the contract which must be honoured by the HTTP API surface.

We intend to continue providing a `Links` collection on at least our v1 resource types, but do not intend to support this pattern going forward.

### HTTP route versioning

HTTP routes are versioned in a similar manner to the contract DTOs they accept, with the HTTP route ending with the same number suffix. E.g.

```csharp
[Route("api/spaces/{spaceId}/projects/{projectId}/releases/{releaseId}/deployments/{deploymentId}/v2")]
public async Task<GetDeploymentResponseV2> GetDeployment(GetDeploymentRequestV2 request, CancellationToken cancellationToken)
{
    var response = await mediator.Request(request, cancellationToken);
    return response;
}
```

### HTTP route structure

HTTP routes are fully qualified. **This means that there is no implicit space, project or any other context.**
* If a project exists within a space then any request to fetch that project must contain a space identifier.
* If a channel exists within a Git repository owned by a project in a space, then the request payload must include a space identifier, project identifier, Git reference and channel identifier.

HTTP route tokens are prefixed by a constant token indicating what comes next.

```
BAD:   /api/Spaces-1/Projects-1/Channels-1         // Is Channels-1 a Git ref or a channel ID?
BAD:   /api/Spaces-1/Projects-1/Channels-1         // Is Channels-1 even a channel ID? It could be a runbook named `Channels-1`.
BAD:   /api/Spaces-1/Projects-1/main/Channels-1    // Is main a Git tag? Or a branch? What if it's both?

GOOD:  /api/spaces/Spaces-1/projects/Projects-1/channels/Channels-1     // This is definitely referring to a channel.
GOOD:  /api/spaces/Spaces-1/projects/Projects-1/runbooks/Channels-1     // This is definitely referring to a runbook.
GOOD:  /api/spaces/Spaces-1/projects/Projects-1/git-ref/refs%2fheads%2fmain/channels/Channels-1 // This is clearly a channel within a Git repository.
```

**Git references in routes (and all request/command payloads) are always fully qualified. _We do not guess._** Although silly, it is entirely valid to have a Git commit with the hash `180cbd4c148636d5f9141017774c61401b906c9d` along with a `refs/tags/180cbd4c148636d5f9141017774c61401b906c9d` tag and a `refs/heads/180cbd4c148636d5f9141017774c61401b906c9d` branch. We need to avoid this ambiguity.

### HTTP route namespaces

There are a number of reserved top-level HTTP route namespaces.

```
/app            // Reserved for the Octopus portal (web UI).
/api            // Reserved for Octopus Server core components.
/extensions     // Reserved for dynamically-loaded extensions.
```

When an extension is registered, its HTTP routes (advertised via MVC controller `[Route("...")]` annotations) will be asserted to start with its extension name. E.g. when an extension named Octopus.Extensions.Slack is loaded, it will have the namespace `/extensions/octopus.extensions.slack` reserved for it. If it attempts to register an MVC route not starting with that prefix then the extension will not be permitted to load.

The extension namespaces are fully qualified in order to cater for third-party extension authors. For example, let's say that [Wide World Importers](https://docs.microsoft.com/en-us/sql/samples/wide-world-importers-what-is?view=sql-server-ver15), a wholesale novelty goods importer and distributor, created a custom extension to provision a white-labeled version of its web store on demand for its distribution partners. It might publish packages along these lines:

```
WideWorldImporters.Octopus.WhiteLabelWebStore.nupkg
WideWorldImporters.Octopus.WhiteLabelWebStore.MessageContracts.nupkg
WideWorldImporters.Octopus.WhiteLabelWebStore.MessageContracts.HttpRoutes.nupkg
```

The MVC controllers within `WideWorldImporters.Octopus.WhiteLabelWebStore` would be constrained to the HTTP route namespace of `/extensions/wideworldimporters.octopus.whitelabelwebstore`.

If one of their distribution partners were to write code like this:

```PowerShell
Install-Package WideWorldImporters.Octopus.WhiteLabelWebStore.MessageContracts.HttpRoutes
```

```csharp
var command = new ProvisionWhiteLabelWebStoreCommandV3(...);
var response = await client.Do(command, cancellationToken);
```

then the resultant HTTP request would look something like this:

```
POST /extensions/wideworldimporters.octopus.whitelabelwebstore/provision-white-label-web-store/v3
Host: wideworldimporters.octopus.app

{...}
```

## SignalR

SignalR method names are simply the name of the command or request DTO.

E.g. to make a `CreateProjectCommandV1` method call, the SignalR method name would simply be `CreateProjectCommandV1`. This will be dispatched to the `CreateProjectCommandV1Handler` via the mediator, after which exactly the same business logic will be applied regardless of the transport.

While this does give the possibility of a naming collision between core and extensions, the saving in verbosity is worth that trade. In the event that an extension attempts to override an in-built operation, that extension will not be permitted to load. In the case where one extension attempts to override another extension's operation, the second/subsequent extension will not be permitted to load.

## Octopus Service Messages

Service message names are the same name as the command. To further the above example, creating and deleting an ephemeral environment via a service message would look like this:

```
##octopus[CreateEnvironmentCommandV1 spaceId='Spaces-1' name='My ephemeral environment' slug='ephemeral-environment-123']
```

E.g.

```bash
#!/bin/bash
set -u

echo "##octopus[CreateEnvironmentCommandV1 spaceId='Spaces-1' name='My ephemeral environment' slug='ephemeral-environment-$TASK_ID']"
npm test
echo "##octopus[DeleteEnvironmentCommandV1 spaceId='Spaces-1' slug='ephemeral-environment-$TASK_ID']"
```

which would, again, be dispatched to the exact same `CreateEnvironmentCommandV1Handler` and `DeleteEnvironmentCommandV1Handler` respectively as all of the others.

Naming collisions are addressed as per SignalR.

# API client libraries

We publish a number of client libraries for various languages. In general, those libraries come in three shapes:

1. A contract library, _which knows nothing about transports_.
1. A contract library, _which provides only transport-specific annotations for contracts_.
1. A transport-specific library, _which knows nothing about specific contracts_.

In the C# ecosystem, this looks like:

1. `Octopus.Server.MessageContracts.nupkg` (Contracts only)
1. `Octopus.Server.MessageContracts.HttpRoutes.nupkg` (HTTP routes only)
1. `Octopus.Client.nupkg` (HTTP-aware client; does not know about the other two packages.)

with equivalents in Java being:
1. `com.octopus.server.messagecontracts.jar` (Contracts only)
1. `com.octopus.server.messagecontracts.httproutes.jar` (HTTP routes only)
1. `com.octopus.client.jar` (HTTP-aware client; does not know about the other two packages.)

and so on.

Importantly, `Octopus.Client` does not know about either `Octopus.Server.MessageContracts.HttpRoutes` or `Octopus.Server.MessageContracts` - it just knows about the generic concepts of `ICommand<,>`, `IRequest<,>` etc. It also knows about the `IHttpRouteFor<T>` interface, where `T` is either a request or a command. In other words, the client is simply given a payload and can figure out where to send it and which HTTP verb to use.

A sample use case might look like this:

```csharp
var request = new GetDeploymentRequestV4(...);
var response = await client.Request(request, cancellationToken);    // Use client.Request for requests and client.Do for commands.

Console.WriteLine($"Deployment {response.Deployment.Id} state is {response.Deployment.State}.");
```

with the `GetDeploymentRequestV4` class like this:

```csharp
public class GetDeploymentRequestV4 : IHaveSpaceId, IHaveProjectId, IHaveGitRef, IHaveReleaseId, IHaveDeploymentId
{
    [Required]
    public string SpaceId { get; set; } = null!;

    [Required]
    public string ProjectId { get; set; } = null!;

    [Required]
    public string ReleaseId { get; set; } = null!;

    [Required]
    public string DeploymentId { get; set; } = null!;

    [Optional]
    public string? GitRef { get; set; }
}
```

and the HTTP routing information provided like this:

```csharp
public class GetDeploymentRequestV4HttpRoute : IHttpRouteFor<GetDeploymentRequestV4>
{
    [HttpRouteTemplate]
    public const string RouteTemplate = "api/spaces/{spaceId}/projects/{projectId}/releases/{releaseId}/deployments/{deploymentId}/v4";

    [HttpRouteTemplate]
    public const string RouteTemplateWithGitRef = "api/spaces/{spaceId}/projects/{projectId}/git-ref/{gitRef}/releases/{releaseId}/deployments/{deploymentId}/v4";

    public HttpMethod HttpMethod { get; } = HttpMethod.Get;
}
```

The key advantage of this approach to pluggable API contracts is that consumers of the API client can pick and choose additional API contracts to add for their particular use case. To this point, we have only discussed core Octopus Deploy contracts, but extensions can also advertise their contracts in the same manner.

For example, consider a Slack notification extension. This clearly isn't a core capability, but it would nonetheless be very useful to be able to configure and control it via the same HTTP API surface.

With packages like this:

```
Octopus.Extensions.Slack.nupkg                              // The extension dynamically loaded into Octopus.
Octopus.Extensions.Slack.MessageContracts.nupkg             // Pure payload message contract definitions.
Octopus.Extensions.Slack.MessageContracts.HttpRoutes.nupkg  // HTTP routes for the payload contracts.
```

a caller could do something simple like this:

```PowerShell
Install-Package Octopus.Extensions.Slack.MessageContracts.HttpRoutes
```

```csharp
var command = new ConfigureSlackNotificationCommandV1(...);
await client.Do(command, cancellationToken);
```

and the same `Octopus.Client` library would be able to work out how to deliver the payload to the correct HTTP endpoint using the correct verb.

Likewise, a deployment Bash/PowerShell script step would be able to send a service message like this:

```
##octopus[SendSlackNotificationCommandV1 channel='#team-core-platform' message='All your build are belong to us! Ha ha ha!']
```

and have that command routed to the appropriate handler within the `Octopus.Extensions.Slack` extension.
