# Commons Status Message (CSM)

Commons Status Message (CSM) is designed to give CTG Video Services applications a standard way to report their statuses. Applications use the CSM-defined structures to create arbitrarily complex status messages and then have CSM generate a JSON version of those nested statuses which can then be passed as the response in a status query managed by the application.

As much as is possible CSM affords users the ability to report any type and amount of information and in arbitrary granularity. But a minimal valid CSM requires only three pieces of information: identity, status, and timestamp. Here is an example:

###### Example: Minimal Valid Commons Status Message
```javascript
{ 
   component: { componentType: "MAJOR", componentName: "SIA" },
   status: "OK",
   epochSecondsTimestamp: 1437064108,
   iso8601UtcTimestamp: "2015-07-16T16:28:28Z"
}
```
###### Example (Groovy): Using CSM to Construct Minimal Valid Status Message
```groovy
def myStatus = new StatusMessage(
      component: new MajorComponent(componentName: MajorComponentType.SIA),
      status: Status.OK)
def myStatusInJson = StatusMessageSerializer.toJsonString(myStatus)
```
###### Example (Java): Using CSM to Construct Minimal Valid Status Message
```java
StatusMessage myStatus = StatusMessageBuilder.builder()
   .withComponent(MajorComponentBuilder.builder()
      .withMajorComponentType(MajorComponentType.SIA)
      .build())
   .withStatus(Status.OK)
   .withEpochSecondsTimestamp(new Date().getTime() / 1000)
   .build();
String myStatusInJson = StatusMessageSerializer.toJsonString(myStatus);
```
You might have noticed that in neither code example above was both epochSecondsTimestamp and iso8601UtcTimestamp included. In the Groovy example _no_ timestamp was set and the Java example only epochSeconds was set. But in the minimal CSM JSON output example _both_ timestamp fields exist. This is not a mistake.

It was decided to include both an ISO timestamp (for human readability) and an epoch timestamp (for simpler date handling in the code). To simplify use of the CSM API, CSM will automatically create the missing value based on the value that was supplied. Or if neither are supplied it will create _both_ based on the current time.

Similarly, a StatusMessage's Status is also automatically provided and set to "OK" if previously unset. So here are the examples for constructing a minimal, valid, and ready-for-the-wire CSM in a very small amount of code:
###### Example (Groovy): Using CSM to Construct Minimal Valid Status Message, Minimally (also using static imports)
```groovy
def myStatusInJson = toJsonString(new StatusMessage(component: new MajorComponent(componentName: SIA)))
```
###### Example (Java): Using CSM to Construct Minimal Valid Status Message, Minimally (also using static imports)
```java
String myStatusInJson = toJsonString(StatusMessageBuilder.builder()
   .withComponent(MajorComponentBuilder.builder().withMajorComponentType(SIA).build()).build());
```

## A Closer Look at the CSM Objects
The root of the CSM tree is the StatusMessage. It contains the properties previously illustrated above - component, status, and timestamps - and a few more optional fields: notes and subStatuses. It also contains one more required field called statusMessageType. StatusMessageType wasn't shown in the introduction section to keep it simple. But all the fields - required and optional will be discussed in detail below.

Before we dive into StatusMessage let's look at all the CSM domain objects and their interaction with each other. The ASCII diagram below (Fig 1) is in semi-UML Class Diagram form. The exclamation points before certain fields indicate they are required to be present and non-null.
###### Figure 1: CSM Domain Objects Class Diagram
```
        +-------------------------------------+                             
        |   STATUS MESSAGE : Domain Object    | <---+                       
        |=====================================|     |                       
   +--> | ! component : Component             |     |                       
   |    | ! statusMessageType : enum          |     |                      
   |    | ! status : enum                     |  recursive nesting               
   |    | ! epochSecondsTimestamp : long      |     |                       
   |    | ! iso8601UtcTimestamp : string      |     |                       
   | +> |   notes : list<Note>                |     |                       
   | |  |   subStatuses : list<StatusMessage> +-----+                       
   | |  +-------------------------------------+                             
   | |                                                                      
   | |  +-----------------------------------+                               
   | +--+   NOTE : Domain Object            |                               
   |    |===================================|                               
   |    | ! title : string                  |                               
   |    | ! actionImportance : enum         |                               
   |    |   detail : string                 |                               
   |    |   data : string                   |                               
   |    |   dataDeserializationKey : string |                               
   |    |   actionToTake : string           |                               
   |    +-----------------------------------+                               
   |                                                                        
   |    +--------------------------------------+                            
   +----+   COMPONENT : Abstract Domain Object |                            
        |======================================|                            
        | ! componentType : enum               |                            
        |   secondaryId : string               |                            
        |   tertiaryId : string                |                            
        |   quaternaryId : string              |                            
        +-----+--------------------------------+                            
              ^                                                             
              |      +-------------------------------------+                
              |      |   GENERAL COMPONENT : Domain Object |                
              +------+=====================================|                
              |      | ! componentName : string            |                
              |      +-------------------------------------+                
              |      +---------------------------------------------+        
              |      |   MAJOR COMPONENT : Domain Object           |        
              +------+=============================================|        
              |      | ! componentName : MajorComponentType : enum |        
              |      +---------------------------------------------+        
              |      +-----------------------------------------------------+
              |      |   MAJOR EXTERNAL COMPONENT : Domain Object          |
              +------+=====================================================|
                     | ! componentName : MajorExternalComponentType : enum |
                     +-----------------------------------------------------+
```
