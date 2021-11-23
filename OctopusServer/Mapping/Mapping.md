# Mapping

Mapping in Octopus Server is achieved via two key interfaces: `IMapToNew<TSource, TTarget>` and `IMapToExisting<TSource, TTarget>`.

* `IMapToNew<TSource, TTarget>` takes an instance of `TSource` and returns a new instance of `TTarget`.
* `IMapToExisting<TSource, TTarget>` takes instances of both `TSource` and `TTarget` and maps the former onto the latter.

Both of these interfaces are intentionally async-only.

When creating a mapper, please name it `MapFrom<SourceType>To<TargetType>`, e.g. `MapFromFooToFooResource` or `MapFromCreateDeploymentCommandV1ToCreateDeploymentCommandV2`. Some mappers will implement both `IMapToNew<,>` and `IMapToExisting<,>` and this is generally fine.

Please be cautious about mapping between more than a single `TSource, TTarget` pairing in the same mapper.

## FAQ

### What about convention-based mapping? Isn't that faster and easier?

Convention-based mapping makes the assumption that the source and target are very similar. This is generally not a good assumption as it misses the point of mapping. It is also fragile and prone to breaking in unexpected ways. It is entirely reasonable for a developer in the core of the system to rename a property of something internal, and this renaming should not result in a runtime mapping failure.

Reflection-based mapping is unarguably slower. Dynamic, reflection-based code emission can generally perform on par with hand-written code. The price for either convention-based approach, however, is developer confusion and fragility.

Yes, we have heard of AutoMapper. We do not intend to use AutoMapper.

### Can't I just POST a resource to Server and automatically map all the properties?

In API v1 POSTing a `FooResource` DTO to the server and mapping all the properties was the way things were done. Our intention is to support that from a backward-compatibility perspective for as long as required, although we will wrap the resources in an `UpdateFooCommandV1` wrapper.

In API v2+ we expect to have similar `UpdateFooCommandV2+` commands, although their surface area will only include things which are actually updatable (e.g. no `Links` collection).

In both cases, the expectation is that we will not directly map onto the target entity's properties - our implementation of `IMapToExisting<UpdateFooCommandV1, Foo>` will call methods on `Foo` and delegate the actual setting of the `Foo`'s properties to the entity itself. This gives the entity the ability to decline the change if it would result in its being placed into an invalid state.

### The entity I want to map onto has a property with a protected setter. Can I use reflection to bypass this?

Nope. Please don't do this.

Entities' properties should _all_ have protected setters; the principle being that the entity itself is responsible for maintaining itself in a valid state. Using reflection to set an entity's properties (other than purely when hydrating it from a persistence store) violates that principle and is very strongly discouraged.

### What role does FluentValidation have in mapping onto entites?

Effectively nil. **The mapper is not responsible for validating either the source or the target object.**

The DTO should be valid before arriving at the mapper (and should be annotated appropriately to ensure this).

If mapping onto an entity, the entity itself is responsible for refusing to be put into an invalid state. Bolt-on validation is undesirable here.

## History

Mapping within Octopus Server has historically been closely coupled to the concepts of a `Model` and a `Resource`. This is a coupling which we need to break in order to move forward.

You will find quite a number of locations in the codebase where this still the case. Please do not perpetuate this - the interfaces we would like to encourage everyone to use are simply `IMapToNew<TSource, TTarget>` and `IMapToExisting<TSource, TTarget>`.

As Octopus Server moves progressively further away from a RESTful API, mapping will become correspondingly less symmetrical. Mapping from a domain entity to a DTO will generally be a "copy the relevant properties" affair, whereas mapping from a command (or a legacy `POST <resourceDto>`) to a domain entity will be a series of method calls onto that entity, giving that entity the discretion about whether to accept the caller's changes.
