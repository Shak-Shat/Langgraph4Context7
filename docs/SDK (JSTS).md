# SDK (JS/TS)



## Metadata

- **url**: https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/
- **html**: API reference
LangGraph Platform
SDK (JS/TS)

@langchain/langgraph-sdk

@langchain/langgraph-sdk¶
Classes¶
AssistantsClient
Client
CronsClient
RunsClient
StoreClient
ThreadsClient
Interfaces¶
ClientConfig
Functions¶
getApiKey

@langchain/langgraph-sdk

@langchain/langgraph-sdk / AssistantsClient

Class: AssistantsClient¶

Defined in: client.ts:270

Extends¶
BaseClient
Constructors¶
new AssistantsClient()¶

new AssistantsClient(config?): AssistantsClient

Defined in: client.ts:85

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

AssistantsClient

INHERITED FROM¶

BaseClient.constructor

Methods¶
create()¶

create(payload): Promise\<Assistant>

Defined in: client.ts:335

Create a new assistant.

PARAMETERS¶
payload¶

Payload for creating an assistant.

# assistantId?¶

string

# config?¶

Config

# graphId¶

string

# ifExists?¶

OnConflictBehavior

# metadata?¶

Metadata

# name?¶

string

RETURNS¶

Promise\<Assistant>

The created assistant.

delete()¶

delete(assistantId): Promise\<void>

Defined in: client.ts:387

Delete an assistant.

PARAMETERS¶
assistantId¶

string

ID of the assistant.

RETURNS¶

Promise\<void>

get()¶

get(assistantId): Promise\<Assistant>

Defined in: client.ts:277

Get an assistant by ID.

PARAMETERS¶
assistantId¶

string

The ID of the assistant.

RETURNS¶

Promise\<Assistant>

Assistant

getGraph()¶

getGraph(assistantId, options?): Promise\<AssistantGraph>

Defined in: client.ts:287

Get the JSON representation of the graph assigned to a runnable

PARAMETERS¶
assistantId¶

string

The ID of the assistant.

options?¶
# xray?¶

number | boolean

Whether to include subgraphs in the serialized graph representation. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included.

RETURNS¶

Promise\<AssistantGraph>

Serialized graph

getSchemas()¶

getSchemas(assistantId): Promise\<GraphSchema>

Defined in: client.ts:301

Get the state and config schema of the graph assigned to a runnable

PARAMETERS¶
assistantId¶

string

The ID of the assistant.

RETURNS¶

Promise\<GraphSchema>

Graph schema

getSubgraphs()¶

getSubgraphs(assistantId, options?): Promise\<Subgraphs>

Defined in: client.ts:312

Get the schemas of an assistant by ID.

PARAMETERS¶
assistantId¶

string

The ID of the assistant to get the schema of.

options?¶

Additional options for getting subgraphs, such as namespace or recursion extraction.

# namespace?¶

string

# recurse?¶

boolean

RETURNS¶

Promise\<Subgraphs>

The subgraphs of the assistant.

getVersions()¶

getVersions(assistantId, payload?): Promise\<AssistantVersion[]>

Defined in: client.ts:421

List all versions of an assistant.

PARAMETERS¶
assistantId¶

string

ID of the assistant.

payload?¶
# limit?¶

number

# metadata?¶

Metadata

# offset?¶

number

RETURNS¶

Promise\<AssistantVersion[]>

List of assistant versions.

search()¶

search(query?): Promise\<Assistant[]>

Defined in: client.ts:398

List assistants.

PARAMETERS¶
query?¶

Query options.

# graphId?¶

string

# limit?¶

number

# metadata?¶

Metadata

# offset?¶

number

RETURNS¶

Promise\<Assistant[]>

List of assistants.

setLatest()¶

setLatest(assistantId, version): Promise\<Assistant>

Defined in: client.ts:449

Change the version of an assistant.

PARAMETERS¶
assistantId¶

string

ID of the assistant.

version¶

number

The version to change to.

RETURNS¶

Promise\<Assistant>

The updated assistant.

update()¶

update(assistantId, payload): Promise\<Assistant>

Defined in: client.ts:362

Update an assistant.

PARAMETERS¶
assistantId¶

string

ID of the assistant.

payload¶

Payload for updating the assistant.

# config?¶

Config

# graphId?¶

string

# metadata?¶

Metadata

# name?¶

string

RETURNS¶

Promise\<Assistant>

The updated assistant.

@langchain/langgraph-sdk

@langchain/langgraph-sdk / Client

Class: Client\<TStateType, TUpdateType, TCustomEventType>¶

Defined in: client.ts:1323

Type Parameters¶

• TStateType = DefaultValues

• TUpdateType = TStateType

• TCustomEventType = unknown

Constructors¶
new Client()¶

new Client\<TStateType, TUpdateType, TCustomEventType>(config?): Client\<TStateType, TUpdateType, TCustomEventType>

Defined in: client.ts:1353

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

Client\<TStateType, TUpdateType, TCustomEventType>

Properties¶
assistants¶

assistants: AssistantsClient

Defined in: client.ts:1331

The client for interacting with assistants.

crons¶

crons: CronsClient

Defined in: client.ts:1346

The client for interacting with cron runs.

runs¶

runs: RunsClient\<TStateType, TUpdateType, TCustomEventType>

Defined in: client.ts:1341

The client for interacting with runs.

store¶

store: StoreClient

Defined in: client.ts:1351

The client for interacting with the KV store.

threads¶

threads: ThreadsClient\<TStateType, TUpdateType>

Defined in: client.ts:1336

The client for interacting with threads.

@langchain/langgraph-sdk

@langchain/langgraph-sdk / CronsClient

Class: CronsClient¶

Defined in: client.ts:175

Extends¶
BaseClient
Constructors¶
new CronsClient()¶

new CronsClient(config?): CronsClient

Defined in: client.ts:85

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

CronsClient

INHERITED FROM¶

BaseClient.constructor

Methods¶
create()¶

create(assistantId, payload?): Promise\<CronCreateResponse>

Defined in: client.ts:215

PARAMETERS¶
assistantId¶

string

Assistant ID to use for this cron job.

payload?¶

CronsCreatePayload

Payload for creating a cron job.

RETURNS¶

Promise\<CronCreateResponse>

createForThread()¶

createForThread(threadId, assistantId, payload?): Promise\<CronCreateForThreadResponse>

Defined in: client.ts:183

PARAMETERS¶
threadId¶

string

The ID of the thread.

assistantId¶

string

Assistant ID to use for this cron job.

payload?¶

CronsCreatePayload

Payload for creating a cron job.

RETURNS¶

Promise\<CronCreateForThreadResponse>

The created background run.

delete()¶

delete(cronId): Promise\<void>

Defined in: client.ts:241

PARAMETERS¶
cronId¶

string

Cron ID of Cron job to delete.

RETURNS¶

Promise\<void>

search()¶

search(query?): Promise\<Cron[]>

Defined in: client.ts:252

PARAMETERS¶
query?¶

Query options.

# assistantId?¶

string

# limit?¶

number

# offset?¶

number

# threadId?¶

string

RETURNS¶

Promise\<Cron[]>

List of crons.

@langchain/langgraph-sdk

@langchain/langgraph-sdk / RunsClient

Class: RunsClient\<TStateType, TUpdateType, TCustomEventType>¶

Defined in: client.ts:701

Extends¶
BaseClient
Type Parameters¶

• TStateType = DefaultValues

• TUpdateType = TStateType

• TCustomEventType = unknown

Constructors¶
new RunsClient()¶

new RunsClient\<TStateType, TUpdateType, TCustomEventType>(config?): RunsClient\<TStateType, TUpdateType, TCustomEventType>

Defined in: client.ts:85

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

RunsClient\<TStateType, TUpdateType, TCustomEventType>

INHERITED FROM¶

BaseClient.constructor

Methods¶
cancel()¶

cancel(threadId, runId, wait, action): Promise\<void>

Defined in: client.ts:985

Cancel a run.

PARAMETERS¶
threadId¶

string

The ID of the thread.

runId¶

string

The ID of the run.

wait¶

boolean = false

Whether to block when canceling

action¶

CancelAction = "interrupt"

Action to take when cancelling the run. Possible values are interrupt or rollback. Default is interrupt.

RETURNS¶

Promise\<void>

create()¶

create(threadId, assistantId, payload?): Promise\<Run>

Defined in: client.ts:809

Create a run.

PARAMETERS¶
threadId¶

string

The ID of the thread.

assistantId¶

string

Assistant ID to use for this run.

payload?¶

RunsCreatePayload

Payload for creating a run.

RETURNS¶

Promise\<Run>

The created run.

createBatch()¶

createBatch(payloads): Promise\<Run[]>

Defined in: client.ts:844

Create a batch of stateless background runs.

PARAMETERS¶
payloads¶

RunsCreatePayload & object[]

An array of payloads for creating runs.

RETURNS¶

Promise\<Run[]>

An array of created runs.

delete()¶

delete(threadId, runId): Promise\<void>

Defined in: client.ts:1066

Delete a run.

PARAMETERS¶
threadId¶

string

The ID of the thread.

runId¶

string

The ID of the run.

RETURNS¶

Promise\<void>

get()¶

get(threadId, runId): Promise\<Run>

Defined in: client.ts:972

Get a run by ID.

PARAMETERS¶
threadId¶

string

The ID of the thread.

runId¶

string

The ID of the run.

RETURNS¶

Promise\<Run>

The run.

join()¶

join(threadId, runId, options?): Promise\<void>

Defined in: client.ts:1007

Block until a run is done.

PARAMETERS¶
threadId¶

string

The ID of the thread.

runId¶

string

The ID of the run.

options?¶
# signal?¶

AbortSignal

RETURNS¶

Promise\<void>

joinStream()¶

joinStream(threadId, runId, options?): AsyncGenerator\<{ data: any; event: StreamEvent; }>

Defined in: client.ts:1027

Stream output from a run in real-time, until the run is done. Output is not buffered, so any output produced before this call will not be received here.

PARAMETERS¶
threadId¶

string

The ID of the thread.

runId¶

string

The ID of the run.

options?¶

AbortSignal | { cancelOnDisconnect: boolean; signal: AbortSignal; }

RETURNS¶

AsyncGenerator\<{ data: any; event: StreamEvent; }>

An async generator yielding stream parts.

list()¶

list(threadId, options?): Promise\<Run[]>

Defined in: client.ts:935

List all runs for a thread.

PARAMETERS¶
threadId¶

string

The ID of the thread.

options?¶

Filtering and pagination options.

# limit?¶

number

Maximum number of runs to return. Defaults to 10

# offset?¶

number

Offset to start from. Defaults to 0.

# status?¶

RunStatus

Status of the run to filter by.

RETURNS¶

Promise\<Run[]>

List of runs.

stream()¶

Create a run and stream the results.

PARAM¶

The ID of the thread.

PARAM¶

Assistant ID to use for this run.

PARAM¶

Payload for creating a run.

CALL SIGNATURE¶

stream\<TStreamMode, TSubgraphs>(threadId, assistantId, payload?): TypedAsyncGenerator\<TStreamMode, TSubgraphs, TStateType, TUpdateType, TCustomEventType>

Defined in: client.ts:706

Type Parameters¶

• TStreamMode extends StreamMode | StreamMode[] = StreamMode

• TSubgraphs extends boolean = false

Parameters¶
# threadId¶

null

# assistantId¶

string

# payload?¶

Omit\<RunsStreamPayload\<TStreamMode, TSubgraphs>, "multitaskStrategy" | "onCompletion">

Returns¶

TypedAsyncGenerator\<TStreamMode, TSubgraphs, TStateType, TUpdateType, TCustomEventType>

CALL SIGNATURE¶

stream\<TStreamMode, TSubgraphs>(threadId, assistantId, payload?): TypedAsyncGenerator\<TStreamMode, TSubgraphs, TStateType, TUpdateType, TCustomEventType>

Defined in: client.ts:724

Type Parameters¶

• TStreamMode extends StreamMode | StreamMode[] = StreamMode

• TSubgraphs extends boolean = false

Parameters¶
# threadId¶

string

# assistantId¶

string

# payload?¶

RunsStreamPayload\<TStreamMode, TSubgraphs>

Returns¶

TypedAsyncGenerator\<TStreamMode, TSubgraphs, TStateType, TUpdateType, TCustomEventType>

wait()¶

Create a run and wait for it to complete.

PARAM¶

The ID of the thread.

PARAM¶

Assistant ID to use for this run.

PARAM¶

Payload for creating a run.

CALL SIGNATURE¶

wait(threadId, assistantId, payload?): Promise\<DefaultValues>

Defined in: client.ts:861

Parameters¶
# threadId¶

null

# assistantId¶

string

# payload?¶

Omit\<RunsWaitPayload, "multitaskStrategy" | "onCompletion">

Returns¶

Promise\<DefaultValues>

CALL SIGNATURE¶

wait(threadId, assistantId, payload?): Promise\<DefaultValues>

Defined in: client.ts:867

Parameters¶
# threadId¶

string

# assistantId¶

string

# payload?¶

RunsWaitPayload

Returns¶

Promise\<DefaultValues>

@langchain/langgraph-sdk

@langchain/langgraph-sdk / StoreClient

Class: StoreClient¶

Defined in: client.ts:1084

Extends¶
BaseClient
Constructors¶
new StoreClient()¶

new StoreClient(config?): StoreClient

Defined in: client.ts:85

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

StoreClient

INHERITED FROM¶

BaseClient.constructor

Methods¶
deleteItem()¶

deleteItem(namespace, key): Promise\<void>

Defined in: client.ts:1205

Delete an item.

PARAMETERS¶
namespace¶

string[]

A list of strings representing the namespace path.

key¶

string

The unique identifier for the item.

RETURNS¶

Promise\<void>

Promise

getItem()¶

getItem(namespace, key, options?): Promise\<null | Item>

Defined in: client.ts:1161

Retrieve a single item.

PARAMETERS¶
namespace¶

string[]

A list of strings representing the namespace path.

key¶

string

The unique identifier for the item.

options?¶
# refreshTtl?¶

null | boolean

Whether to refresh the TTL on this read operation. If null, uses the store's default behavior.

RETURNS¶

Promise\<null | Item>

Promise

EXAMPLE¶
const item = await client.store.getItem(

  ["documents", "user123"],

  "item456",

  { refreshTtl: true }

);

console.log(item);

// {

//   namespace: ["documents", "user123"],

//   key: "item456",

//   value: { title: "My Document", content: "Hello World" },

//   createdAt: "2024-07-30T12:00:00Z",

//   updatedAt: "2024-07-30T12:00:00Z"

// }

listNamespaces()¶

listNamespaces(options?): Promise\<ListNamespaceResponse>

Defined in: client.ts:1301

List namespaces with optional match conditions.

PARAMETERS¶
options?¶
# limit?¶

number

Maximum number of namespaces to return (default is 100).

# maxDepth?¶

number

Optional integer specifying the maximum depth of namespaces to return.

# offset?¶

number

Number of namespaces to skip before returning results (default is 0).

# prefix?¶

string[]

Optional list of strings representing the prefix to filter namespaces.

# suffix?¶

string[]

Optional list of strings representing the suffix to filter namespaces.

RETURNS¶

Promise\<ListNamespaceResponse>

Promise

putItem()¶

putItem(namespace, key, value, options?): Promise\<void>

Defined in: client.ts:1105

Store or update an item.

PARAMETERS¶
namespace¶

string[]

A list of strings representing the namespace path.

key¶

string

The unique identifier for the item within the namespace.

value¶

Record\<string, any>

A dictionary containing the item's data.

options?¶
# index?¶

null | false | string[]

Controls search indexing - null (use defaults), false (disable), or list of field paths to index.

# ttl?¶

null | number

Optional time-to-live in minutes for the item, or null for no expiration.

RETURNS¶

Promise\<void>

Promise

EXAMPLE¶
await client.store.putItem(

  ["documents", "user123"],

  "item456",

  { title: "My Document", content: "Hello World" },

  { ttl: 60 } // expires in 60 minutes

);

searchItems()¶

searchItems(namespacePrefix, options?): Promise\<SearchItemsResponse>

Defined in: client.ts:1256

Search for items within a namespace prefix.

PARAMETERS¶
namespacePrefix¶

string[]

List of strings representing the namespace prefix.

options?¶
# filter?¶

Record\<string, any>

Optional dictionary of key-value pairs to filter results.

# limit?¶

number

Maximum number of items to return (default is 10).

# offset?¶

number

Number of items to skip before returning results (default is 0).

# query?¶

string

Optional search query.

# refreshTtl?¶

null | boolean

Whether to refresh the TTL on items returned by this search. If null, uses the store's default behavior.

RETURNS¶

Promise\<SearchItemsResponse>

Promise

EXAMPLE¶
const results = await client.store.searchItems(

  ["documents"],

  {

    filter: { author: "John Doe" },

    limit: 5,

    refreshTtl: true

  }

);

console.log(results);

// {

//   items: [

//     {

//       namespace: ["documents", "user123"],

//       key: "item789",

//       value: { title: "Another Document", author: "John Doe" },

//       createdAt: "2024-07-30T12:00:00Z",

//       updatedAt: "2024-07-30T12:00:00Z"

//     },

//     // ... additional items ...

//   ]

// }


@langchain/langgraph-sdk

@langchain/langgraph-sdk / ThreadsClient

Class: ThreadsClient\<TStateType, TUpdateType>¶

Defined in: client.ts:457

Extends¶
BaseClient
Type Parameters¶

• TStateType = DefaultValues

• TUpdateType = TStateType

Constructors¶
new ThreadsClient()¶

new ThreadsClient\<TStateType, TUpdateType>(config?): ThreadsClient\<TStateType, TUpdateType>

Defined in: client.ts:85

PARAMETERS¶
config?¶

ClientConfig

RETURNS¶

ThreadsClient\<TStateType, TUpdateType>

INHERITED FROM¶

BaseClient.constructor

Methods¶
copy()¶

copy(threadId): Promise\<Thread\<TStateType>>

Defined in: client.ts:502

Copy an existing thread

PARAMETERS¶
threadId¶

string

ID of the thread to be copied

RETURNS¶

Promise\<Thread\<TStateType>>

Newly copied thread

create()¶

create(payload?): Promise\<Thread\<TStateType>>

Defined in: client.ts:479

Create a new thread.

PARAMETERS¶
payload?¶

Payload for creating a thread.

# ifExists?¶

OnConflictBehavior

# metadata?¶

Metadata

Metadata for the thread.

# threadId?¶

string

RETURNS¶

Promise\<Thread\<TStateType>>

The created thread.

delete()¶

delete(threadId): Promise\<void>

Defined in: client.ts:535

Delete a thread.

PARAMETERS¶
threadId¶

string

ID of the thread.

RETURNS¶

Promise\<void>

get()¶

get\<ValuesType>(threadId): Promise\<Thread\<ValuesType>>

Defined in: client.ts:467

Get a thread by ID.

TYPE PARAMETERS¶

• ValuesType = TStateType

PARAMETERS¶
threadId¶

string

ID of the thread.

RETURNS¶

Promise\<Thread\<ValuesType>>

The thread.

getHistory()¶

getHistory\<ValuesType>(threadId, options?): Promise\<ThreadState\<ValuesType>[]>

Defined in: client.ts:677

Get all past states for a thread.

TYPE PARAMETERS¶

• ValuesType = TStateType

PARAMETERS¶
threadId¶

string

ID of the thread.

options?¶

Additional options.

# before?¶

Config

# checkpoint?¶

Partial\<Omit\<Checkpoint, "thread_id">>

# limit?¶

number

# metadata?¶

Metadata

RETURNS¶

Promise\<ThreadState\<ValuesType>[]>

List of thread states.

getState()¶

getState\<ValuesType>(threadId, checkpoint?, options?): Promise\<ThreadState\<ValuesType>>

Defined in: client.ts:584

Get state for a thread.

TYPE PARAMETERS¶

• ValuesType = TStateType

PARAMETERS¶
threadId¶

string

ID of the thread.

checkpoint?¶

string | Checkpoint

options?¶
# subgraphs?¶

boolean

RETURNS¶

Promise\<ThreadState\<ValuesType>>

Thread state.

patchState()¶

patchState(threadIdOrConfig, metadata): Promise\<void>

Defined in: client.ts:647

Patch the metadata of a thread.

PARAMETERS¶
threadIdOrConfig¶

Thread ID or config to patch the state of.

string | Config

metadata¶

Metadata

Metadata to patch the state with.

RETURNS¶

Promise\<void>

search()¶

search\<ValuesType>(query?): Promise\<Thread\<ValuesType>[]>

Defined in: client.ts:547

List threads

TYPE PARAMETERS¶

• ValuesType = TStateType

PARAMETERS¶
query?¶

Query options

# limit?¶

number

Maximum number of threads to return. Defaults to 10

# metadata?¶

Metadata

Metadata to filter threads by.

# offset?¶

number

Offset to start from.

# status?¶

ThreadStatus

Thread status to filter on. Must be one of 'idle', 'busy', 'interrupted' or 'error'.

RETURNS¶

Promise\<Thread\<ValuesType>[]>

List of threads

update()¶

update(threadId, payload?): Promise\<Thread\<DefaultValues>>

Defined in: client.ts:515

Update a thread.

PARAMETERS¶
threadId¶

string

ID of the thread.

payload?¶

Payload for updating the thread.

# metadata?¶

Metadata

Metadata for the thread.

RETURNS¶

Promise\<Thread\<DefaultValues>>

The updated thread.

updateState()¶

updateState\<ValuesType>(threadId, options): Promise\<Pick\<Config, "configurable">>

Defined in: client.ts:618

Add state to a thread.

TYPE PARAMETERS¶

• ValuesType = TUpdateType

PARAMETERS¶
threadId¶

string

The ID of the thread.

options¶
# asNode?¶

string

# checkpoint?¶

Checkpoint

# checkpointId?¶

string

# values¶

ValuesType

RETURNS¶

Promise\<Pick\<Config, "configurable">>

@langchain/langgraph-sdk

@langchain/langgraph-sdk / getApiKey

Function: getApiKey()¶

getApiKey(apiKey?): undefined | string

Defined in: client.ts:50

Get the API key from the environment. Precedence: 1. explicit argument 2. LANGGRAPH_API_KEY 3. LANGSMITH_API_KEY 4. LANGCHAIN_API_KEY

Parameters¶
apiKey?¶

string

Optional API key provided as an argument

Returns¶

undefined | string

The API key if found, otherwise undefined

@langchain/langgraph-sdk

@langchain/langgraph-sdk / ClientConfig

Interface: ClientConfig¶

Defined in: client.ts:68

Properties¶
apiKey?¶

optional apiKey: string

Defined in: client.ts:70

apiUrl?¶

optional apiUrl: string

Defined in: client.ts:69

callerOptions?¶

optional callerOptions: AsyncCallerParams

Defined in: client.ts:71

defaultHeaders?¶

optional defaultHeaders: Record\<string, undefined | null | string>

Defined in: client.ts:73

timeoutMs?¶

optional timeoutMs: number

Defined in: client.ts:72

langchain/langgraph-sdk

langchain/langgraph-sdk/react¶
Type Aliases¶
MessageMetadata
Functions¶
useStream

@langchain/langgraph-sdk

@langchain/langgraph-sdk / useStream

Function: useStream()¶

useStream\<StateType, Bag>(options): UseStream\<StateType, Bag>

Defined in: stream.tsx:586

Type Parameters¶

• StateType extends Record\<string, unknown> = Record\<string, unknown>

• Bag extends object = BagTemplate

Parameters¶
options¶

UseStreamOptions\<StateType, Bag>

Returns¶

UseStream\<StateType, Bag>

@langchain/langgraph-sdk

@langchain/langgraph-sdk / MessageMetadata

Type Alias: MessageMetadata\<StateType>¶

MessageMetadata\<StateType>: object

Defined in: stream.tsx:169

Type Parameters¶

• StateType extends Record\<string, unknown>

Type declaration¶
branch¶

branch: string | undefined

The branch of the message.

branchOptions¶

branchOptions: string[] | undefined

The list of branches this message is part of. This is useful for displaying branching controls.

firstSeenState¶

firstSeenState: ThreadState\<StateType> | undefined

The first thread state the message was seen in.

messageId¶

messageId: string

The ID of the message used.

Comments
