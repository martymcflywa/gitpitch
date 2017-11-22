# BI event store replication and deployment pipeline

---

## Event store replication

---

#### Where does our read model data come from?

- Cloud event store
- OnPrem event store
- Flat file cache event store

---

## Cloud event store challenges

- It's prod
- Header stored as SQL, message as JSON Azure blob
- Users are generating events on it
- Hit 500M events a week or two ago

---

- We want to minimize load on this resource
- It's on Azure
- Expensive/slow to rebuild read models from event zero
- But it's replicated to **OnPrem** SQL server!

---

## OnPrem event store challenges

- Still relied on by a lot of people/systems in different teams
- Still want to minimize load on this resource during rebuilds
- Even though its local (in the building), it's still stored on an SQL server somewhere
    - Not on my machine
- Throughput still considered too slow for BI read model rebuilds

---

## Solution: Flat file cache 

- See [`CCPF.EventStore.MessagePackReplicator`](https://github.com/CashConverters/CCPF.EventStore.MessagePackReplicator)
- Was initially part of `DashboardReporting` repo but is now separated

---

- Three main components
    1. CCPF.EventStore.MessagePackReplicator
    2. CCPF.EventStore.MessagePackReader.Contractless
    3. CCPF.EventStore.MessagePackReader.Mercury

---

## `MessagePackReplicator`

- net461 executable
- Replicates an event store to a binary flat file format
- Currently points to OnPrem
    - No impact to prod cloud event store
    - Only one point of contact to OnPrem rather than all BI team members querying it during dev rebuilds

---

- Flat files written to local disk on host where read models are populated
    - Goal is fast read model population
    - Removes network bottleneck

---

### Replicator overview

- Queries event source, retrieves pages of event headers/messages
- Pages serialized to LZ4 MessagePack binary
- Binary is written to files in target path
- Rinse/repeat
- If process falls over, it will continue from last sequence id written to file

---

## This sounds cool, how do I use it?

![shutupandtakemymoney](http://i0.kym-cdn.com/entries/icons/mobile/000/005/574/takemymoney.jpg)

---

- Replicator currently deployed for testing at:
    - `AUPERNVME C:\`
- Generated cache files at:
    - `AUPERNVME C:\CCPF.EventStoreCache\Lz4MsgPackContent`
- Can kick off replication by running this script:
    - `AUPERNVME C:\MessagePackReplicator.ps1`
- Will eventually move to AUPERNVME D:\ drive though

---

### How do I read the cache files?

- CCPF.EventStore.MessagePackReader.Mercury
- CCPF.EventStore.MessagePackReader.Contractless
- Available in Cash Converters nuget feed
- Mercury reader has dependency on `Mercury.Contracts`
    - Can deserialize to Mercury Contract types
    - Uses Mercury Date type

---

- Have to write your own proxy types when using Contractless
    - Advantage of picking/choosing which fields you want

---

### Reader usage

``` csharp
using CCPF.EventStore.MessagePackReader.Contractless; // or .Mercury

var reader = new EventStoreReader(pathToFlatFiles);
var events = reader
    .GetEvents(startSequenceId)
    .Select(rawEvent => rawEvent.GetContent<EventProxy>());
```

- Up to you to implement filtering/paging

---

- New up an EventStoreReader and pass in the path where the cache files live
- `.GetEvents()` starts reading from the sequence Id + 1 you pass in
- `.GetContent<T>()` deserializes MessagePack to type `T`
- Can replace `<EventProxy>` with `<SomeMercuryEvent>` if using Mercury variant
- Returns `IEnumerable<T>`

---

## Deployment pipeline

---

## Github PR / AppVeyor

- AppVeyor environment is set up, listening for PR events on the repo
- When PR created, AppVeyor executes a build script
    - Referenced in `appveyor.yml`
    - Using Cake script

---

## Octopus deploy

- `appveyor.yml` also references a deploy script
- Hooks into Octopus deploy API
- Pushes artifacts to Octopus deploy nuget feed
- Multiple environments
    - Dev
    - Preview
    - Production

---

### Octopus deploy steps

- PR deployments end up on dev first
- Deployment process composed of multiple steps, example
    - Refresh event store cache
    - Upgrade the database
    - Populate the read model
    - ...

---

- Any failed step will halt the process and generate notifications
- Successful deployments stay in dev until...

---

### Promotion

- DevBox with tester
- If passes DevBox, dev deployment promoted to preview environment
- Tester validates preview deployment in PowerBI
- If greenlit and passes tests, promoted to production

---

### Scheduled tasks

- Preview and production environments are redeployed every hour
- Kicks off cache replication, read model population and cube updates
- Loan Book exception
    - Only working with daily totals so refresh only occurs once a day after midnight
    - Has separate deployment stream