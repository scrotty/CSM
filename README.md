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
def myStatusInJson = toJsonString(
   new StatusMessage(component: new MajorComponent(componentName: SIA)))
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
(! indicates a mandatory and non-null field)

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
As previously alluded to, Status Messages have four required fields: identity, message type, status, and timestamp (both a epoch long and a ISO-8601 UTC). Let's look at these fields a little closer.

##### Status Message : _Component_ Field (Required)
The Component field of the Status Message acts as an identity. There are three types of Component: Major, Major External, and General. All Video Services Applications should have a Major Component type defined for it. For example, SIA has MajorComponentType.SIA. An enum defines all the possible values. You should check and ensure your component is defined. If not, make sure it gets added! It's important you refer to your own application and other Video Services applications by their MajorComponentType. That way message flows can be properly correlated.

The Major _External_ Component type is similar to the previous component. The difference is that it refers to common applications (sometimes vendor applications) that are not part of the Video Services stable. These include applications like EDW, DSB, DIGITALSMITHS, etc. Check through that enum to see what's available. If you think a common external application should be included make sure to propose that.

The final Component type is _General_. It is designed to be a catchall for anything that isn't named in MajorComponentType or MajorExternalComponentType. It could be a vendor application that few if any other Video Services applications interface with. It might also be sub-components of your own Major component. Maybe you wish to create a StatusMessage about your application's persistence system. You could create a General Component with the component name "Persistence" - or _whatever_ provides meaning to you. Remember, the CSM is purposefully designed to allow the implementor to define meaning how they see fit. 

All component types optionally allow for the inclusion of further means of identification. They are named: secondaryId, tertiaryId, and quaternaryId. All are simple string fields and can be set to _anything_ you might find useful - or not used at all! For example, I might create the following MajorComponent for mADM:
```
Component: {
    componentName: MajorComponentType.MADM,
    secondaryId: "Division: Syracuse",
    tertiaryId: "Server: 02",
    quaternaryId: "Cluster-Instance: 04"
}
```
But I could have just as easily decided on a different scheme to record the same information:
```
Component: {
    componentName: MajorComponentType.MADM,
    secondaryId: "SYR/2/4"
}
```
Either (and many other possibilities) are perfectly valid options for how to identify your component(s) in CSM. Although notice that the component name of MADM (as a MajorComponentType) is included in both examples. The extra id fields can be used however you wish, but the main identification field, componentName, needs to be standard.

While the lack of strict direction in component identification gives you a lot of flexibility it will also force a lot of special coordination on the system(s) that will ultimately parse and process those messages. But in the end it was decided that having a minimal but flexible CSM format would allow teams to better express whatever was needed for any situation.

##### Status Message : _StatusMessageType_ Field (Required)
To assist the parsing and processing system - and even human readers - each Status Message includes a message type. The default message type is called simply _Component_. That means the Status Message is referring to _something_ interesting about the component itself (as identified in the Component field). But you can also choose from other message types including:
- _External Dependency_ - Indicates this Status Message branch (read about Status Message nesting later in this document) contains information primarily about an external entity your application relies on.
- _Dataflow Inbound_ - Indicates this status message branch contains information about data provided to your application from another entity. This entity may be another Video Services application, an external application, or something else entirely. Hopefully you'll include identification information!
- _Dataflow Outbound_ - Indicates this status message branch contains information about data _your_ application provides to another entity.
- _Cluster Node_ - Indicated this status message branch contains information about your application's cluster and cluster nodes.

As is the theme of the CSM, what message types you decide to use is up to you.

##### Status Message : _Status_ Field (Required)
It wouldn't be much of a status message without a status field. The status enumeration provides four states:
- _OK_ - This is the default state. It indicates that this branch of the Status Message nesting is operating as expected. You may decide to include more information about OK states (there are structures we have yet to discuss that are designed for that purpose) or simply decide a status of "OK" is informative enough. It's up to you.
- _WARN_ - This status indicates _something_ is amiss but in your estimation it's not too big a deal. Maybe it a known transient state that you want the watchers to be aware of but not overly concerned about.
- _ERROR_ - This status indicates a meaningful aspect of your application or its function is behaving incorrectly. You'll be able to use other CSM constructs to detail what is wrong and what might be done to resolve the issue.
- _CRITICAL_ - This status indicates a critical piece (or pieces) or your application is not operating correctly. You will almost certainly wish to include additional information that describes the extent of the failures, what might be done to resolve, and even if someone should be woken up to handle it! 

Status Messages also support additional optional fields that make them especially expressive. 
