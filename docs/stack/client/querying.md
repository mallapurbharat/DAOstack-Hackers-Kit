# Query, Observables & Subscription

The interaction with DAOstack subgraph comprises of:

  - _**Query**_: the graphQL queries sent to graphnode to fetch DAOstack data from the subgraph.
  - _**Observable**_: object representing the stream of data that can be subscribed.
  - _**Subscription**_: invokes our handler every time a new value is emitted for the observable.

The entity methods provided by `@daostack/client` for querying the subgraph, by themself do not actually send the query to the server. Instead, each methods returns an *Observable* to which we can *subscribe*.

Take a look at the following methods that return observable:
```
dao.proposals()
proposal.state()
```

Now, in order to query the server we must subscribe
```
const observable = arc.daos()
// only in the next line will a query be sent to the server
const subscription = arc.daos.subscribe((daos) => console.log(`we found ${daos.length} results`))
```

In this guide we will describe how to create & send query and subscribe to the data requested by the query.

## Why Subscription
Subscriptions will cause the server to send you an update each time the data changes and are useful for composing asynchronous and event-based programs.

By default, subscribing to an observable will do two things:

  - send a query to the server to fetching the data
  - send a subscription query to the server for update events

## How to Query and Subscribe

### Creating Query

Following are the two types of query that can be made to the subgraph using client:

#### Entity Methods
The entity methods return an Observables which encapsulate some predefined qraphQL queries to fetch Entity data from the subgraph.

    // proposals of the DAO
    Proposal.search(arc, { where: { dao: "0x123" } })

    // members of the given dao
    dao.members()

#### Raw GraphQL queries
To have more control over what gets fetched from the subgraph you can also customize the query. These queries will follow the standard [graphQL syntax](../subgraph/queries) which is used to query graph explorer directly. Though the query **must be wrapped inside the _gql_ tag**

    const gql = require('graphql-tag')   

    // titles of all proposals of the DAO
    let query = gql`query {
      proposals ( where: { dao: "0x294f999356ed03347c7a23bcbcf8d33fa41dc830" }) { title }
    }`

### Executing Query

As we noted above the query created need to be explicitly sent to the server.

#### Entity Methods
The query encapsulated by Entity Methods is automatically sent to the server on subscription as [discussed below]()


#### Raw GraphQL queries
These queries are made using the static method `arc.sendQuery`. Pass the query designed [above](#raw-graphql-queries) as the parameter to `sendQuery`.

    // Get votes id and outcome of the given Proposal
    arc.sendQuery(gql`query {
        proposal (id: "0x1245") {
          votes { id outcome }
        }}`)


### Subscribing Query
*Subscriptions* are used so that our handler gets called every time a new value is emitted for the observable stream. This is useful to keep the app data updated as the value changes.

#### Entity methods
As we saw the Entity methods do not send the query to the server but return an observable. We must subscribe as follows to send the query as well as subscribe for updates.

```
arc.daos().subscribe(() => {})
dao.state({subscribe: false}).subscribe( () => {})
```
  
#### Raw graphQL queries

For even more control over what data is being fetched and subscribed to, you can write explicit queries:

```
arc.subscribe(gql`subscribe { proposal (id: "0x1245") { votes { id outcome }})`)
new Proposal("0x1245").votes().subscribe(
  (next) => {

  }
)
```

## Optimizing Subscription and Using Cache
Since, subscriptions can be expensive, this behavior can be controlled/optimized in several ways.
  
  - [Controlling fetchPolicy](#controlling-apollo-fetchpolicy): controlling cache interaction.
  - [Subscribing to Apollo Cache](#subscribe-to-apollo-cache-changes): getting updates from Apollo cache instead of graphnode server.
  - [FetchAllData and Nested subscription](#use-fetchalldata-with-nested-subscription): by querying larger set at top level and subscribing to Apollo cache for nested queries.

### Controlling Apollo fetchPolicy
We can pass Apollo's [`fetchPolicy`](https://www.apollographql.com/docs/react/api/react-apollo/#optionsfetchpolicy) argument to control how the query interacts with the cache:

  - _**cache-first**_: default value. Read data from cache first, fetch from network otherwise.
  - _**cache-and-network**_: return data from cache but still always fetch from network. It optimizes quick response while also keep cached data updated.
  - _**network-only**_: will always make a request network and write data to cache. It optimizes for data consistency with the server.
  - _**cache-only**_: will never execute a query using your network interface and throw error if data not available in cache.
  - _**no-cache**_: always make a request using your network interface. Also, it will not write any data to the cache

```
arc.daos({}, { fetch-policy: 'cache-first'}) // the default value
arc.daos({where: {stage: "Boosted"}}, { fetch-policy: 'network-only'}) // bypass the cache
```

### Subscribe to Apollo Cache changes
The creation of a subscription can be controlled by passing the subscribe parameter.
In the following query we will not subscribe to the updates from network but will still watch changes in the Apollo cache and return updated results if the cache changes.

**Note**:

```
arc.daos({}, { subscribe: false}).subscribe(() => {})
dao.state().subscribe( () => {})
```
NOTE:
  - By default `subscribe` is set to false.
  - Apollo cache could change as a result of another smaller query which does subscribe to server changes.

### Use fetchAllData with Nested subscription
Most of the Entity methods are implemented in such a way that the queries will fetch (and subscribe to) just as much data as is needed to create the result set. For example, `dao.proposals()` will only fetch the proposal IDs. This can be controlled (in a limited way) by setting the parameter `fetchAllData` to true
```
dao.proposals({orderBy: "creationDate"}, {fetchAllData: true})
```
This is useful for cache handling, where it may be useful to have more complete control over what data is being fetched. Consider the following example, which will get the list of proposals from the dao, and then get the state for each of the proposals.
```
dao.proposals().subscribe(
  (props) => {
    for (let prop of props) {
      prop.state().subscribe(.....)
    }
  }
)
```
The problem with this pattern is that it is very expensive. The (subscription to) `dao.proposals(..)` will send a query and create a subscription, and then each of the calls to `proposal.state()` will create a new query and a separate subscription.

Consider now the following pattern:
```
dao.proposals({}, { fetchAllData: true }).subscribe(
  (props) => {
    for (let prop of props) {
      prop.state( {subscribe: false }).subscribe(.....)
    }
  }
)
```
This will resolve two inefficiencies. First of all, the `fetchAllData` in the proposals query will make it so that the
`dao.proposals` query will fetch (and subscribe to) a much larger query - in particular, it will get all state data for each of the proposals. This means that when `prop.state()` is called, it will find all the needed information in the cache (and so it will not send a new query to the server), and we can safely pass it the `subscribe: false` flag, because `dao.proposals()` already subscribes to updates for all the cached data.