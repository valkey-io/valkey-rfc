### UML code for log compaction

```
@startuml
participant "Primary" as P
database "Primary Log" as PL
database "Snapshot File" as SF
participant "Replica 1" as R1
participant "Replica 2" as R2

== Snapshot Creation ==
P -> P: Serialize in-memory state
P -> SF: Write snapshot (state, index, term, metadata)
note right of SF: Includes lastIncludedIndex\nand cluster metadata

== Log Truncation ==
P -> PL: Truncate log up to lastIncludedIndex
note over PL: Retain only recent entries\nfor future replication

== Snapshot Advertisement ==
P -> R1: AppendEntries / InstallSnapshot\n(advertise snapshot index)
P -> R2: AppendEntries / InstallSnapshot\n(advertise snapshot index)
R1 --> P: Acknowledge
R2 --> P: Acknowledge
note over P,R2: Followers can request snapshot\nif behind compaction point
@enduml

```

### UML code for bootstrap

```uml
@startuml
participant "NodeA" as A
participant "NodeB" as B
participant "NodeC" as C
database "NodeA Log" as AL
database "NodeB Log" as BL
database "NodeC Log" as CL

note over A,C: All nodes load shard-nodes from config
A -> A: shard-nodes = {A,B,C}
B -> B: shard-nodes = {A,B,C}
C -> C: shard-nodes = {A,B,C}

A -> B: Connect
B --> A: ACK
A -> C: Connect
C --> A: ACK
B -> C: Connect
C --> B: ACK

note over A,C: All nodes know each other

note over A,C: All nodes start as followers
A -> A: Start as Follower (term=0)
B -> B: Start as Follower (term=0)
C -> C: Start as Follower (term=0)

note over A: Election Timeout
A -> A: Election timeout
A -> A: Become Candidate (term=1)\nVote for self

note over A: Request Votes
A -> B: RequestVote(term=1)
B --> A: VoteGranted
A -> C: RequestVote(term=1)
C --> A: VoteGranted


A -> A: Become Leader

note over A,C
Leader = NodeA
Followers = NodeB, NodeC
end note

A -> A: Now acts as Primary (P)
B -> B: Now acts as Replica 1 (R1)
C -> C: Now acts as Replica 2 (R2)

@enduml
```

### UML code for Node Addition

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
participant "New Replica" as NR
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L
database "New Replica Log" as NRL

== Node Startup ==
NR -> NR: Start as Follower (term=0)\nNo log, no leader
NR -> P: CLUSTER ADDNODE <ip> <port>

note over P,NR
Primary receives join request
Validates version, role, and cluster membership
end note

== Phase 1: Add as Learner (Non-Voting Member) ==
P -> PL: Propose membership change\n(+NR as learner)
P -> R1: Replicate config change
P -> R2: Replicate config change
R1L <- R1: Write config-change entry
R2L <- R2: Write config-change entry
R1 --> P: Ack
R2 --> P: Ack
P -> PL: Commit new joint configuration
note over P,NR
New Replica added as learner (non-voting)
Cluster = {P,R1,R2,NR(learner)}
end note

== Phase 2: State Catch-up ==
P -> NR: Send snapshot / log segments
NRL <- NR: Apply snapshot and incremental logs
P -> NR: Continue AppendEntries until caught up
NR --> P: CaughtUpAck
note over NR: Learner log state aligned with leader

== Phase 3: Promote to Full Member ==
P -> PL: Propose membership change\n(promote NR to voter)
P -> R1: Replicate config change
P -> R2: Replicate config change
P -> NR: Replicate config change
R1L <- R1: Write config-change entry
R2L <- R2: Write config-change entry
NRL <- NR: Write config-change entry
R1 --> P: Ack
R2 --> P: Ack
NR --> P: Ack
P -> PL: Commit final configuration
note over P,NR
NR promoted to full voting member
Cluster = {P,R1,R2,NR}
end note

== Normal operation continues ==

@enduml
```

### UML code for Replica Removal

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
participant "Replica 3 (to be removed)" as R3
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L
database "Replica 3 Log" as R3L

== Removal Request ==
Client -> P: CLUSTER REMOVENODE R3
note over P: Validate request (node exists, not leader, quorum available)

== Phase 1: Propose Joint Configuration ==
P -> PL: Propose config change\n(old + new minus R3)
P -> R1: Replicate config-change entry
P -> R2: Replicate config-change entry
R1L <- R1: Write config-change entry
R2L <- R2: Write config-change entry
R1 --> P: Ack
R2 --> P: Ack
P -> PL: Commit new configuration
note over P,R2: Cluster committed config excluding R3

== Phase 2: Notify and Detach ==
P -> R3: CLUSTER FORGET <self>\n(stop replication)
R3 -> R3L: Flush and close Raft session
R3 -> R3: Transition to standalone (non-member)
note over R3: Node R3 removed from cluster\nand can be safely shut down

== Phase 3: Continue normal operations ==
@enduml
```
### UML code for Primary Removal

```
@startuml
actor Client
participant "Primary (self-removal)" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L

== Self-Removal Request ==
Client -> P: CLUSTER REMOVENODE P
note over P: Detected self-removal request\nand current role = Leader

== Phase 1: Controlled Step-Down ==
P -> P: Block new client writes
P -> R1: AppendEntries (final log flush)
P -> R2: AppendEntries (final log flush)
R1 --> P: Ack
R2 --> P: Ack
P -> P: Set state=follower, increment term\n(new election will start)
note over P,R2: Leader steps down voluntarily

== Phase 2: New Election ==
R1 -> R1: Start election (term+1)
R1 -> R2: RequestVote(term+1)
R2 --> R1: VoteGranted
R1 -> R1: Become new leader

== Phase 3: Membership Change ==
R1 -> R1L: Propose config change\n(remove old leader P)
R1 -> R2: Replicate config-change entry
R2L <- R2: Write config-change entry
R2 --> R1: Ack
R1 -> R1L: Commit new configuration
note over R1,R2: New leader R1 committed config\nexclud
```

### UML code for Primary Failure

```
@startuml
actor Client
participant "Primary (down)" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L

== Normal Operation ==
P -> R1: AppendEntries (heartbeat)
P -> R2: AppendEntries (heartbeat)
R1 --> P: Ack
R2 --> P: Ack
note over P,R2: Cluster stable under P as leader

== Phase 1: Failure Detection ==
P -[#red]-> X: Primary crash / network loss
R1 -> R1: Detect missing heartbeat
R2 -> R2: Detect missing heartbeat
note over R1,R2: Election timeout exceeded\nNo heartbeat received from leader

== Phase 2: Election Start ==
R1 -> R1: Increment term, state=candidate
R1 -> R2: RequestVote(term+1, lastLogIndex)
R2 -> R2L: Compare logs, vote if up-to-date
R2 --> R1: VoteGranted

== Phase 3: Leader Election ==
R1 -> R1: Votes received (majority)\nBecome new leader
note over R1,R2: New leader = R1\nTerm incremented

== Phase 4: Log Consistency Check ==
R1 -> R2: AppendEntries (no-op heartbeat)\nEnsure log alignment
R2 --> R1: Ack

== Phase 5: Cluster Recovery ==
Client -> R1: Redirected write request
R1 -> R1L: Append log entry
R1 -> R2: Replicate log entry
R2 --> R1: Ack
R1 -> R1L: Commit entry
note over R1,R2: Cluster recovered under new leader R1

== Phase 6: Old Leader Returns ==
P -> R1: AppendEntries(term=old)
R1 --> P: Reject (term stale)
note over R1,P: Old leader demoted automatically\nStale term rejected
@enduml
```

### UML code for Leader Liveness

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L

autonumber
note over P: No new writes from client

loop Heartbeat interval (e.g. 150–300 ms)
  P -> R1: Heartbeat (empty AppendEntries)
  P -> R2: Heartbeat (empty AppendEntries)
  R1 -> P: Heartbeat ACK
  R2 -> P: Heartbeat ACK
  note over R1, R2: Followers reset election timeout
end
@enduml
```

### UML code for slow replica

```
@startuml
participant "Primary" as P
participant "Replica 1 (healthy)" as R1
participant "Replica 2 (lagging)" as R2
database "Primary Log" as PL
database "Replica 2 Log" as R2L

== Normal Replication ==
P -> R1: AppendEntries [Log Index 100–101]
R1 --> P: Ack (matchIndex=101)
P -> R2: AppendEntries [Log Index 100–101]
R2 --> P: Timeout / No Ack
note over P,R2: Replica 2 falls behind due to network delay

== Incremental Catch-Up ==
P -> R2: Retry AppendEntries [Index 100]
R2 --> P: Partial Ack (matchIndex=99)
note right of P: Leader tracks R2 using nextIndex=100
P -> R2: AppendEntries [Index 100–105]
R2 --> P: Ack (matchIndex=105)
note over P,R2: Replica 2 still lags significantly

== Snapshot Installation ==
P -> R2: InstallSnapshot [State up to Index 200]
R2 -> R2L: Replace local log with snapshot state
R2 --> P: Snapshot installed (matchIndex=200)
note over P,R2: Replica 2 catches up via snapshot

== Resume Normal Operation ==
P -> R2: AppendEntries [Index 201–205]
R2 --> P: Ack
note over P,R2: All replicas now in sync
@enduml

```

### UML code for lifecycle of a write command

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L

autonumber
Client -> P: Write operation on key K
P -> P: Execute operation

note right of P
Add response to client buffer
Block the client write handler 
until replication offset reaches X
end note

P -> PL: Durably log operation
P -> P: Add to durable stream
note right of P: After certain period/operations
P -> R1: Replicate operations
P -> R2: Replicate operations
R1 -> R1L: Durably log operation
R2 -> R2L: Durably log operation
R1 -> P: Ack the durable write upto offset X
R2 -> P: Ack the durable write upto offset X
note right of P
Quorum reached
upto offset X
end note
P -> Client: Write response
P -> R1 : Ack the commit of operation upto X
R1 -> R1: Execute operation
note over P, R1: Same view across Primary and Replica 1
P -> R2 : Ack the commit of operation upto X
R2 -> R2: Execute operation
note over P, R2: All the nodes have the same view for key K
@enduml
```

### UML code for lifecycle of a read command

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2

autonumber
Client -> P: Read operation on key K
P -> P: Verify if the key is blocked for operation
alt Blocked
  P -> P: Block the client
  ... wait until unblocked ...
  P -> P: Unblock client
  P -> P: Execute the operation
  P -> Client: Write response
else Not Blocked
  P -> P: Execute the operation
  P -> Client: Write response
end
@enduml
```

### UML code for write outage

```
@startuml
actor Client
participant "Primary" as P
participant "Replica 1" as R1
participant "Replica 2" as R2
database "Primary Log" as PL
database "Replica 1 Log" as R1L
database "Replica 2 Log" as R2L

autonumber
Client -> P: Write operation on key K
P -> P: Execute operation
note right of P: Block key K
P -> PL: Durably log operation
P -> P: Add to durable stream
note right of P: After certain period/operations
P -> R1: Replicate operations
P -> R2: Replicate operations
note over R1: Replica 1 down
note over R2: Replica 2 down
P -> P: Timeout waiting for replica acknowledgments
P -> P: Detect both replicas are down
P -> P: Keep key K blocked
note right of P: Key K remains blocked until replicas recover
Client-x P: Timeout
Client-> P: Write operation on key K
P -> P: Key K is blocked
note over P: Operator involvement required for failover/recover
@enduml
```
