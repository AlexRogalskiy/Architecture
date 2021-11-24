# Logging

System logging in Octopus Server is via the [Serilog](https://serilog.net/) `ILogger` interface. Full structured logging is supported.

To get started, inject an `ILogger` via dependency injection. Be careful to inject the Serilog interface and not others. Accessing the logger instance directly via `Serilog.Log.Logger` is highly discouraged and should only be used in extenuating circumstances. 

## FAQ

### What happend to ISystemLog?

`ISystemLog` is now obsolete in favour of Serilog as the first-class system logging implementation in Octopus Server.

### What about Task logs? ITaskLog?

Task logging remains unchanged and continues to operate as usual. Continue to use `ITaskLog` for task logging purposes.

### Why not Microsoft.Extensions.Logging?

Diversity in the .NET ecosystem is valued highly at Octopus Deploy. Supporting quality libraries like Serilog and its extensive ecosystem is valuable for the continuing health of .NET generally. Serilog also affords the ability to contribute fixes and enhancements that deliver value to everyone much faster.