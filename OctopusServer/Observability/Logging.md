# Logging

System logging in Octopus Server is via the [Serilog](https://serilog.net/) `ILogger` interface. Full structured logging is supported.

To get started, inject an `ILogger` via dependency injection. Be careful to inject the Serilog interface and not others.

```csharp
using Serilog;

class MyClass {

    readonly ILogger logger;

    public MyClass(ILogger logger) {
        this.logger = logger;
    }
}
```

Accessing the logger instance directly via `Serilog.Log.Logger` is highly discouraged and should only be used in extenuating circumstances. 


## Structured Logging Style Guide

Octopus Deploy follows the recommendations of the [Serilog guidelines](https://github.com/serilog/serilog/wiki/Writing-Log-Events) with regards to style. A few essential style directions and clarifications are below.

### Property names should Pascal case

Properties should be in Pascal case for consistency and easy visual parsing. Pascal case is the same as camel case, with the first letter of the word also capitalised.

```csharp
logger.Information("This is a {MyCoolName}", myThing);
```


### Prefer explicit over implicit property names

Specific property naming helps structured log parsing and searching. For example, when referring to the name of a project, use `ProjectName` and not `Project`. Similarly, use `ProjectId` and not `Project` or `Id`. 

```csharp
logger.Information("The Id of {ProjectName} is {ProjectId}.", project.Name, project.Id);
```

In a sea of log messages, explicit is much clearer than the alternative: `The Id of {Project} is {Id}`.


### Don't quote properties in log messages

Structured logging already delineates property values from the rest of the message. There is no reason to add single (`'`) or double (`"`) quotes. Some renderers also quote string values automatically, which could result in multiple sets of quotes.

```csharp
logger.Information("This is {MyThing} in a message.", myThing)
```



## FAQ

### What happend to ISystemLog?

`ISystemLog` is now obsolete in favour of Serilog as the first-class system logging implementation in Octopus Server.

### What about Task logs? ITaskLog?

Task logging remains unchanged and continues to operate as usual. Continue to use `ITaskLog` for task logging purposes.

### Why not Microsoft.Extensions.Logging?

Diversity in the .NET ecosystem is valued highly at Octopus Deploy. Supporting quality libraries like Serilog and its extensive ecosystem is valuable for the continuing health of .NET generally. Serilog also affords the ability to contribute fixes and enhancements that deliver value to everyone much faster.