## Purchasing Domain ##

### CloudSubscriptionStartedV1Event ###

| Property | Type        | Description | 
| ---------| ----------- | ----------- |
| CloudSubscriptionId   | string(200)  ||
| DnsPrefix   | string(4-100)  ||
| OctopusIdClientId  | string(50)  ||
| OctopusIdClientSecret  | string(200)  |Base64 encrypted value|
| RegionName   | string(50)  ||
| RetentionLimitInDays   | int?  ||
| TaskCap   | int(5\|10\|20\|40\|80\|160)  ||
| StorageLimitInGb   | int(1000)  ||
| Serial   | string(200)  ||
| UseBaselineResources  | bool  ||



## Cloud Platform Domain ##

? Would `Hosting` be a better name?

## Release Management Domain ##