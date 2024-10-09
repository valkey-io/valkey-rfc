---
RFC: 9
Status: <Proposed>
---

# Valkey Triggers RFC

## Abstract
Valkey triggers are persistent procedures which are executed in response to some keyspace event. 
Today [Valkey keyspace Notifications](https://valkey.io/topics/notifications/) offer the ability to publish events to dedicated pub/sub channels. 
Valkey triggers add the ability to place extending logic on such keyspace events and to localize actions on the server side reducing the need to monitor and react to events on the application side.
We propose to extend the existing [Valkey Functions](https://valkey.io/topics/functions-intro/) with the ability to define trigger function calls on specific keyspace events.
As they are being integrated into the Valkey Functions infrastructure, Triggers are persisted as part of the user data.

## Motivation 

1. ***Localized actions.*** 
   Today different application use cases rely on [Valkey keyspace notifications](https://valkey.io/topics/notifications/) in order to manage the database. For example,       consider a case were a key is referenced in different places (like SortedSet and lists). In order to support TTL logic a cleanup needs to be made when the key is evicted to remove it from the different lists or sets. It is possible to have the application listen for eviction events and perform the cleanup, however this would require the application to rely on non-persistent subscribe connections, implement application side logic and extra network hops, which can cause the cleanup to be performed long after the key was evicted.
   For example lets take a commonly used pattern of scheduled tasks.
   In such cases the user usually registers tasks in a ZSET with the matching execution time as the score.
   Usually what is being done is setting a puller job in the application side which periodically reads all items in the set which have scores smaller than the current timestamp, and issue eval/fcall commands back to Valkey with the relevant tasks to run. 
   This pattern introduces some waste as the application needs to maintain a puller job and perform the round trip back to the application in order to execute the scheduled operations.
   With Valkey triggers this can be achieved without the need for a puller job by the user.
   In order to achieve that we can use:
   - ZSET z for holding scheduled tasks
   - Key k to manage the next task execution time

   The external application code will only have to add tasks to the ZSET **z** with score matching their required execution time.
   1. once the task is added to **z** a trigger code will be executed which will take the minimal score from the ZSET **z**, and apply the diff to the current time to the TTL of key **k**.
   2. once key **k** has expired, a trigger will be executed which will remove and execute the tasks from the zset **z** that have scores lower than the current time.

   Here is an example library code to schedule tasks to be executed after/every several seconds

   ![image](https://user-images.githubusercontent.com/88133677/225652613-aa85beee-05f1-4c5e-abe7-1d334b0e882f.png)

   So in order to use them the user can simply register operations to be triggered:

   ```
   fcall schedule 0 'server.call("PUBLISH", "scheduled", "this is a msg from the past")' "3"
   ```
   will cause the message to be published 3 seconds after the `fcall` is processed.

   ```
   fcall every 0 "3" 'redis.call("PUBLISH", "scheduled", "this is an annoying msg")'
   ```
   will cause the specified message to be published every 3 seconds.
   Note that the same concept  can be used to implement HASH members eviction!

2. ***Flexibility of extensibility.*** 
   There are some cases were application needs to extend the logic of server side operations. In some of the cases it might be problematic making the change on the application    
   side, either because of the risk to deploy the application part or since the operation is not triggered by the application (eg evictions, expirations etc...).
   For example, consider the current implementation of [Valkey keyspace notifications](https://valkey.io/topics/notifications/). The existing mechanism relies on non-persistent pub/sub notifications.
   In order to persist the notifications it is possible to write them into a sorted set. 
   Triggers can provide this ability in a very straightforward way. For example, consider the following function library:

   ![image](https://user-images.githubusercontent.com/88133677/225616988-b9bb8730-59c1-478b-ae21-ece3b8b3a617.png)

   In this implementation we will capture each keyspace event and place the relevant key name and event on a dedicated stream called
   "keyevent-notifications".

   ```
   > set a b
   OK
   > lpush mylist f
   (integer) 1
   > xread streams keyevent-notifications 0
   1) 1) "keyevent-notifications"
      2) 1) 1) "1678974201461-0"
            2) 1) "a"
               2) "new"
         2) 1) "1678974201461-1"
            2) 1) "a"
               2) "set"
         3) 1) "1678974210181-0"
            2) 1) "mylist"
               2) "new"
         4) 1) "1678974210181-1"
            2) 1) "mylist"
               2) "lpush"
   ```   
   
   [Valkey keyspace notifications](https://valkey.io/topics/notifications/) has another major disadvantage: they are not reported on the cluster-bus. While this might have good reasoning (eg avoid cluster-bus load, duplication of events etc...), this makes Valkey clients almost unable to manage key event listeners on Valkey clusters. Valkey triggers can enable the user to decide to publish keyspace events on the cluster bus. This way the trigger code can make smart decisions about publishing of the event and then use the 'PUBLISH' call in order to propagate the event to all cluster nodes.

3. ***Security.*** 
   In some cases there is a need to limit the access to the cache. Currently Valkey provides restricting functionality via [Access Control Lists](https://valkey.io/topics/acl/).
   However what if there is a need to access restricted parts of the dataset without granting real permissions to clients?
   Since a trigger can operate without explicit user request, it can potentially be operated under any defined ACL user which will enable it to operate actions on restricted parts of the dataset.   

### Can't the same be done with modules?
Modules have the ability to build logic placed on top of keyspace events. However Modules provide a much more restrictive form of logic which is much harder to implement and dynamically replace. 
By building this new logic inside the Valkey function library users will have a much simpler way to create stored procedures which are persistent, managed and replicated. 


## Design and Specification

### Triggers as part of function libraries
Valkey triggers will be implemented as an extension of the Valkey [Function Libraries](https://valkey.io/topics/functions-intro/) .
A trigger is basically a Valkey function which will be called when some specific keyspace event is issued.
In order to register a specific function as trigger, one will simply have to use the new **"register_trigger"**  API which will be available for the different library engines. 
For example for the (currently only) LUA engine the registration code will look like:

![image](https://user-images.githubusercontent.com/88133677/225090299-1cf05508-75e6-43dd-a90f-05ca91dd3a6f.png)

### Register trigger with named arguments form
 In order to provide some extra arguments to the trigger, the new API will also support the named arguments form. 
 For example:

![image](https://user-images.githubusercontent.com/88133677/225531522-d3f66a00-bd90-490b-98c8-11b62c3f3eba.png)

### The Trigger code block
As stated before, a trigger is basically a specific engine function.
The function synopsis will be identical for all triggers which will get the following parameters:

**a.     key -** the key for which the event was triggered

**b.     event -** a string specifying the specific event (ie. “expire ”, “set”, “move”, “store” etc...)

**c.     db -** the id of the db context for which the event is triggered (note it is not necessarily the current db scope the trigger is executed in, ie. MOVE event).

For LUA engine based functions, the key will be supplied as the first argument in the keys array (i.e. KEYS[1]).
The **event** and **db** will be supplied in the ARGS array.  (event will be ARGS[1] and db will be ARGS[2]).


### Triggering Events

Valkey already has a feature to report different [keyspace notifications](https://valkey.io/topics/notifications/) which currently provides the ability to publish keyspace events to registered subscribers. The implementation should introduce the ability to set a trigger function to run on **ANY** event which is defined for [keyspace notifications](https://valkey.io/topics/notifications/).

### Triggers configuration
By design Triggers will not require ANY configuration in order to work. The Triggers will be part of the loadable function libraries and should be able to run on the target events even if the user **DID NOT** explicitly configure keyspace-notifications.

### Atomicity and triggers

All keyspace events are triggered immediately after the operation which triggered them. If we allow the triggers to be executed when the keyspace event has been triggered, 
It means allowing a write call performed in the middle of operating another command. Another (technical) issue is related to nested scripts. Currently there is a [protection](https://github.com/valkey-io/valkey/blame/unstable/src/script_lua.c#L891) in Valkey code preventing to execute lua script from a scripts. 

There are 2 design options:

1. Post execution of triggers. Much like what was provided for modules with `VM_AddPostNotificationJob`, which was introduced to allow modules to
perform write operations on keyspace events as part of postExecutionUnitOperations. It is possible to follow the same logic as modules and share the same notification mechanism ie. to execute the trigger during the unit operation completion.
This will help prevent breaking atomicity of operations and better follows the way applications are currently reacting to keyspace events. The downside of this option is that since there is no 
grantee about the state of the data after the operation is completed. 

2. Allow nested script calls for trigger scripts. This suggestion will require some handling of nested lua run contexts, but during PoC was tested to work.
   
### recursive trigger flow or "should trigger trigger trigger?"

We do not want to reach a nested call of triggers and a potential endless loops. According to this design a trigger execution as a result of some keyspace event is a terminal state.
we could keep some recursion depth counter that will stop triggering once it reaches some depth, but that would cause some unexpected behavior for the library developers, and difficulty debugging cases where triggers where not executed.
Another important aspect to consider is if modules based keyspace events calbacks should be triggered from trigger actions and vice versa.
In the scope of this document I will assume that the trigger based actions **WILL NOT** cause matching module calbacks to be called and that module actions performed during it's callback operation **WILL NOT** cause triggering triggers, but each will be executed only in 1 level of nesting.

So for example lets say I have a module callback **cb** set on some keyspace event **e1** and a trigger action **t** registered on some event **e2**.
when the event **e1** is issued the **cb** will be executed and will cause event **e2** which will **NOT** trigger **t**.
Also in case **e2** is issued the **t** is being executed and cause event **e1** which will **NOT** trigger **cb**.

### Scope of a trigger

Whenever some event triggers an action, we will need this action to be processed in the scope of a specific database and ACL user.

#### Database context
Since Valkey triggers are built on keyspace events, the DB scope will be the same as the keyspace the event was issued on.

#### ACL User scope
By default the scope of the user will by the superuser which will not be limited by any authentication restrictions. 
This makes sense as the triggers might be needed in order to operate a managed operation using some commands which should be otherwise restricted. 
However in some cases the user might want the trigger to operate under some ACL restrictions. 
The problem is that triggers might not operate in the scope of a specific external client (eg expiration events), so we suggest during trigger registration it will be possible to specify 
the name of the ACL user to attach to the trigger run context.

### New Commands
1. `CLIENT TRIGGERS [ON|OFF]` new subcommand. Sometimes it will be required to mute trigger execution in some context (say administrative fix to the dataset) 
    In order to support it the specific client will be able to mute triggers execution.
    Response will always be "O.K".  
2. The `FUNCTION LIST` command output will now include a new part named "triggers".
Much like functions the triggers part will describe a per-trigger information:

```
"library_name"
  "mylib"
  "engine"
  "LUA"
  "functions"
         1) "name"
         2) "myfunc"
         3) "description"
         4) <some description>
         5) "flags"
         6) <relevant flags>
   "triggers"
          1) "name"
          2) "trigger_name"
          3) "event"
          4) "__keyspace@0__:*"
          7) "calls"
          8) "<the total number of times the trigger was called>"
          9) "errors"
         10) "<the total number of errors issued during all executions of this trigger>"
         11) "total_duration"
         12) <the aggregated total duration time of all executions of this trigger>
``` 

### Benchmarking (Optional)

TBD 

### Testing 
TBD

### Observability 

#### general statistics
The "Stats" info section will be added with the following statistics:
- **total_trigger_calls -** the number of general trigger calls. this will not be a teardown list of calls per trigger function (this can be taken from the function list command as will be explained later)
- **total_trigger_errors -** The total number of errors during trigger execution (an error is counted for each time the afterErrorReply is called).
- **total_trigger_duration -** The total time spent running trigger related code.

### Per trigger statistics
The same global statistics will be available on a per-trigger resolution via the `FUNCTION LIST` command:
"calls" -  the total number of times the trigger has called
"errors" - the total number of errors issued during this trigger run
"total_duration" - the aggregated total duration time this trigger run

*** Open question ***
should `CONFIG RESETSTAT` also reset the triggers statistics, or should that only be done when functions lib is reloaded?

### Debug mechanism 

When Valkey functions and scripts are executed via FCALL/EVAL, it is possible to report the errors back to the calling client.
Triggers, however, are silently executing which makes it hard for the user to understand why the trigger did not work and what errors occurred during execution of the trigger.
The suggestion here is to enable reporting error msg on a predefined pub/sub channel:
1. All error msgs will be intercepted as deferred errors (much like modules do). 
2. After trigger completes run, in case errors were issues, it will update the stats with the num,ber of errors.
3. The trigger will report the errors to a dedicated channel in the following structure:
   `__triggers@errors__:<trigger-name>`
