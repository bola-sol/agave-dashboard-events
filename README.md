# Events the dashboard needs

The dashboard is a web page for one validator. This document lists the events it needs from agave's event system: the `agave-event-system` crate, which [anza-xyz/agave#16054](https://github.com/anza-xyz/agave/pull/16054) adds to the validator. It assumes the dashboard runs as its own process and reads only events and the host. Each dashboard branch's `dashboard/INPUTS.md` lists what that build reads today, with each exact field.

Each event has a priority:

- **1**: replaces a metrics datapoint, the bank notification channel, or a parse of ledger bytes. These are the fragile inputs today. Most are static events where the validator already reports.
- **2**: state that a separate process cannot read itself.
- **3**: new. The validator does not expose it today.

The **In process** column is for a dashboard that runs inside the validator. **Needed** means it needs the event too. Otherwise the column names the direct read it can keep.

## Rules for every stream

- Include the slot where there is one.
- Include a time: monotonic nanoseconds for spans on this node, and wall clock where the table says so, to compare with other nodes.
- Send counters as running totals since the validator started, not counts since the last event. Then a dropped event loses detail, not totals.
- Bound every event. Give strings and lists a maximum size.
- For state, send the full value at start, on each change, and at a fixed interval. A subscriber that starts late then gets it without waiting for a change. So does one that reconnects after a validator restart clears the event directory.
- Where an event replaces a datapoint, the datapoint's fields are enough. INPUTS.md lists them.
- Put the dashboard's streams under few name prefixes, so that one short filter enables them.

## Events

### Slots

| Event | Fields | When | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- | --- |
| Block frozen | slot, parent slot, leader, transactions, non-vote transactions, failed transactions, entries, block cost, block cost limit, account cost limit, total fees, priority fees, tips (jito), blockhash; counts for this block only | replay freezes a bank | Static | the bank notification channel, the reads under "Frozen banks" in INPUTS.md, and the working bank's counts for TPS | 1 | Needed |
| Slot replayed | fields of `replay-slot-stats` | replay completes a slot | Static | `replay-slot-stats` | 1 | Needed |
| Block full | fields of `shred_insert_is_full`, wall clock ns | a slot's last shred is inserted | Static | `shred_insert_is_full` | 1 | Needed |
| Turbine layers | fields of `retransmit-stage-slot-stats` | each slot | Static | `retransmit-stage-slot-stats` | 1 | Needed |
| First shred | slot, wall clock ms | a slot's first shred arrives | Static | `Blockstore::meta`, `Blockstore::highest_slot` | 2 | `Blockstore::meta` |
| Slot status | slot, status (optimistically confirmed, rooted, finalized, skipped), monotonic ns | each change | Static | `BlockCommitmentCache`, `get_latest_optimistic_slots`, `Validator::highest_finalized`, `BankForks::root_bank` | 2 | those reads |
| Our leader slots this epoch | epoch, leader slots passed, produced, skipped | each root, and at an interval | Static, state | `Bank::get_slot_history` | 2 | `Bank::get_slot_history` |
| Shred arrival | slot, shred index, repaired or not, monotonic ns | each shred | Static | nothing: a new shred timeline | 3 | Needed |

### Our blocks

| Event | Fields | When | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- | --- |
| Leader slot timings | fields of `slot-metrics`, `leader-slot-start-to-cleared-elapsed-ms`, `broadcast-transmit-shreds-stats`, `broadcast-process-shreds-stats`, `banking_stage_scheduler_slot_timing` | each of our slots | Static | those datapoints | 1 | Needed |
| Scheduler counts | fields of `banking_stage_scheduler_slot_counts`, scheduler id | each of our slots | Static | `banking_stage_scheduler_slot_counts` | 1 | Needed |
| Worker timing | fields of `banking_stage_worker_timing`, worker id, **slot** | each report | Static | `banking_stage_worker_timing`; the slot is new | 1 | Needed |
| Vote worker timing | fields of `banking_stage-leader_slot_vote_execute_and_commit_timings` | each of our slots | Static | that datapoint | 1 | Needed |
| Block cost | fields of `cost_tracker_stats` | each of our slots | Static | `cost_tracker_stats` | 1 | Needed |
| Bundles (jito) | fields of `bundle_stage-stats` | each of our slots | Static | `bundle_stage-stats` | 1 | Needed |
| Message versions | slot, non-vote transactions that are legacy, v0, v1 | each of our slots, once full | Static | `Blockstore::get_slot_entries` | 2 | `Blockstore::get_slot_entries` |

### Votes and certificates (alpenglow)

| Event | Fields | When | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- | --- |
| Vote timing | fields of `event_handler_slot_tracking` | as votor reports | Static | `event_handler_slot_tracking` | 1 | Needed |
| Reward certificate | rewarded slot, writer slot, notarize or skip, ranks paid (bitmap), notarize count, skip count, block producer time (wall clock ns) | each block footer replayed | Bounded: a bitmap of the admitted set | the parse of block footers (`get_slot_components_with_shred_info`) | 1 | the footer parse, which reads ledger bytes |
| Finalization certificate | finalized slot, ranks that voted (bitmap) | each certificate | Bounded | `Bank::vote_accounts` `last_voted_slot` at freeze | 2 | `Bank::vote_accounts` |
| Vote latency | fields of `consensus_vote_metrics`, vote account | once per epoch per vote account | Static | `consensus_vote_metrics` | 1 | Needed |

### Counters

All of these are running totals, sent each second (gossip every two seconds).

| Event | Fields | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- |
| TPU ingest | fields of `quic_streamer_tpu`, `quic_streamer_tpu_forwards`, `quic_streamer_tpu_vote`, `tpu-verifier`, `banking_stage_scheduler_counts` (with its `id`), `banking_stage_worker_counts`, `banking_stage_worker_error_metrics`, `bundle_stage-loop_stats` (jito) | Static | those datapoints | 1 | Needed |
| Receivers and senders | fields of `shred_fetch_receiver`, `shred_fetch_repair_receiver`, `gossip_receiver`, `tpu_vote_receiver`, `votor_datagram_server`, the `Gossip` and `Repair` senders, `retransmit-stage` | Static | those datapoints | 1 | Needed |
| Accounts and caches | fields of `accounts_db_store_timings`, `accounts_db_load_accounts`, `accounts_db-stores`, `accounts_db-flush_accounts_cache`, `loaded-programs-cache-stats` | Static | those datapoints | 1 | Needed |
| Gossip | fields of `cluster_info_stats` to `cluster_info_stats5`, `cluster_info_crds_stats`, `cluster_info_crds_stats_fails` | Static | those datapoints | 1 | Needed |

### Peers, stake and epoch

| Event | Fields | When | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- | --- |
| Peer | identity, shred version, client, version, gossip, TVU, TPU, TPU QUIC, RPC and votor addresses, start time, last heard (wall clock), full and incremental snapshot slots, lowest slot | on change; all peers at an interval | Per peer, bounded | `ClusterInfo::all_peers`, `gossip.crds` | 2 | `ClusterInfo::all_peers` |
| Vote account | vote key, identity, activated stake, commission, credits this epoch, BLS key set, last voted slot | each epoch and each change of stake or commission; last voted slot and all accounts at an interval | Per account | `Bank::vote_accounts` | 2 | `Bank::vote_accounts` |
| Admitted set (alpenglow) | epoch, rank, identity, vote key, stake; for this node: seat this epoch, seat next epoch, ticket shortfall | each epoch, for this epoch and the next; all ranks at an interval | Per rank | `Bank::get_rank_map`, `get_vat_health_for_next_epoch` | 2 | those reads |
| Epoch | epoch, first slot, slots in epoch, ns per slot, cluster type, consensus (tower or alpenglow), lamports per signature, admission ticket minimum, rent-exempt minimum for a vote account | at start, each epoch, and at an interval | Static, state | `Bank::epoch_schedule`, `ns_per_slot_at_slot`, `cluster_type`, `migration_status`, the fee and ticket reads | 2 | those reads |
| Slot clock | slot, block height, clock unix timestamp | each root | Static | `Bank::block_height`, `Bank::clock` | 2 | those reads |
| Leader schedule | epoch, the leader of each four-slot turn | each epoch, for this epoch and the next | Large: about 108,000 turns in a 432,000-slot epoch | `LeaderScheduleCache` | 2 | `LeaderScheduleCache` |
| Validator info | identity, name, icon URL, parsed in the validator | on change; all validators at an interval, which needs the config program's account index | Per validator, bounded: under 700 bytes, as validator info is at most 576 bytes | the config program's accounts | 2 | `get_filtered_indexed_accounts`, `get_program_accounts_modified_since_parent` |

The leader schedule is too large for one event. Send it one turn at a time, or send the epoch's node stakes so that the dashboard calculates it.

### Node and start-up

| Event | Fields | When | Size | Replaces | P | In process |
| --- | --- | --- | --- | --- | --- | --- |
| Node | identity, vote account, client, version, commit, shred version, our ports, process id, snapshot archive directories and intervals | at start and at an interval | Bounded, state | `Version::this_build`, `my_contact_info`, `SnapshotConfig` | 2 | those reads |
| Start progress | phase, detail, fraction, time in phase, phases taken | each change, and at an interval | Bounded, state | `ValidatorStartProgress` | 2 | `ValidatorStartProgress` |
| Supermajority wait | per validator: identity, stake, version, seen in gossip; totals: fields of `wfsm_gossip` | each check during the wait | Per validator | the gossip-ready channel, `ClusterInfo::tvu_peers`, `wfsm_gossip` | 2 | the gossip-ready channel |
| XDP setup | fields of `xdp-network-config` | at start, and at an interval | Bounded, state | `xdp-network-config` | 1 | Needed |
| Snapshot written | slot, full or incremental, start and end time | each archive | Static | a scan of the archive directories | 3 | the directories |

### New events that add panels

Each of these is priority 3. The dashboard shows none of them today.

- Shred arrival, in the table above: a shred timeline for each block.
- The TPU counters with a slot, read when the slot ends: the TPU path for each slot, not each leader turn.
- A record for each transaction in our blocks: signature, fee, compute, status and execution time. The record is bounded, and does not need the transaction itself.
- The reason for each skipped slot.
- The costliest accounts of each of our blocks, as a bounded list.
- Compute for each of our blocks, split into votes, bundles and the rest.
- Delivered packet counts for the serve repair, ancestor hashes and block id repair receivers. anza-xyz/agave#15157 adds the first.
- Counters for failed QUIC accepts and refused connections.
- Address lookup failures, counted apart from malformed transactions.
- Gossip traffic for each peer, and bytes for each gossip message type.
- Replay time for each program, without trace logging.
- Separate worker ids for the bundle stage and BAM.

## What the dashboard reads itself

- The host: CPU, memory, disks, network interfaces and sockets, from `/proc`, `/sys` and `statvfs`. INPUTS.md lists them.
- The validator's memory and threads: `/proc/<pid>/status` and `/proc/<pid>/task`, with the process id from the Node event.
- Snapshot archives: the directories in the Node event.
- The event directory: `<ledger>/events`. The operator gives the dashboard the ledger path.
- Nothing for the pages' own requests. The dashboard answers them from what it keeps.

## Limits of the first version

- Raw account and transaction data. The dashboard needs neither. The validator can parse what the dashboard uses into bounded events: validator info is at most 576 bytes on chain, and a transaction record needs a signature and a few numbers.
- A large value in one event. Each queue cell is the size of the largest event. So the leader schedule, the vote accounts and the peers go one item at a time. The stream's capacity must hold a full set, or the publisher spreads the set over time.
- Start-up before the event system exists. agave#16054 starts it after the ledger loads, so searching for RPC, downloading a snapshot and loading the ledger send no events.
- State for a late subscriber. A subscriber starts at the newest event, and agave#16054 clears the event directory at each validator start. State streams must resend.
- A slow subscriber. When any subscriber falls a full queue behind, the publisher rejects new events for every subscriber. Subscribers see the loss only in each lane's rejected count.
- A subscriber that exits without dropping its handle. It keeps its place, so the stream rejects every event for every subscriber. Only a validator restart, or turning the stream off and on, recreates it.

## If the dashboard runs inside the validator

- It needs only the events marked Needed. It keeps the direct reads that the column names.
- State streams need no resends, because the direct reads give the current state.
- `/proc/self` is the validator, so it needs no process id.
- Validator names, the leader schedule and the early start-up phases stay available.
- Without the bank notification channel, it refreshes validator names with a periodic scan of the config program's account index.
- Upstream, it still needs the run command's flags and start-up code, and three public fields on `Validator`: `block_commitment_cache`, `leader_schedule_cache` and `highest_finalized`. The events remove the metrics observer, the bank notification relay and the gossip-ready channel.

## What each dashboard build reads today

Each branch's INPUTS.md lists the inputs that build reads, with each exact field.

| Validator | Branch | INPUTS.md |
| --- | --- | --- |
| Agave 4.3 | `dashboard/v4.3.1` | [bola-sol/agave](https://github.com/bola-sol/agave/blob/dashboard/v4.3.1/dashboard/INPUTS.md) |
| Agave 4.4 | `dashboard/v4.4.0-beta.1` | [bola-sol/agave](https://github.com/bola-sol/agave/blob/dashboard/v4.4.0-beta.1/dashboard/INPUTS.md) |
| Jito 4.3 | `dashboard/v4.3.1-jito` | [bola-sol/jito-solana](https://github.com/bola-sol/jito-solana/blob/dashboard/v4.3.1-jito/dashboard/INPUTS.md) |
| Jito 4.4 | `dashboard/v4.4.0-beta.0-jito` | [bola-sol/jito-solana](https://github.com/bola-sol/jito-solana/blob/dashboard/v4.4.0-beta.0-jito/dashboard/INPUTS.md) |
