---

# Perf, SQL, etc.

### Lunch & Learn Apr 20, 2017

##### Venky Iyer

---

## Overview

+++

 * Identifying perf problems
 * Montage: SQL operations/internals
 * What affects SQL query performance
 * EXPLAIN output
 * Redshift SQL & Hive SQL

<!-- Caveats about I am not an expert -->

---

## Where in the stack is the perf bottleneck?

![perf-stack](diagrams/perf-stack.png)

---

## The optimization loop


![opt-loop](diagrams/opt-loop.png)

---

## Profilers


![profilers](diagrams/profilers.png)

---

🐻

---

## NewRelic

+++

### Apdex = % satisfactory response times

* satisfying < tolerable (T) < frustrating (4T)
* Global setting of T = 300ms
* Response time setting to record "traces" >= 4T
* Can be changed on per-transaction level (for key transactions)

+++

### Custom instrumentation

Set or modify transaction names:

```ruby
  def contacts_results(search_term, tags, page_number: 0, paginate: true)
    transaction_name = NewRelic::Agent.get_transaction_name
    if search_term.blank?
      NewRelic::Agent.set_transaction_name("#{transaction_name} - empty")
    elsif UserConfig.value_for_user_id_and_key(@user.id, 'experimental-search').to_b
      NewRelic::Agent.set_transaction_name("#{transaction_name} - experimental")
    elsif search_term.split(/\s+/).length > 3
      NewRelic::Agent.set_transaction_name("#{transaction_name} - long")
    end
```
+++

Measure a block:

```ruby
   include ::NewRelic::Agent::MethodTracer

   self.trace_execution_scoped(['MessageThreadTagsController/log_work_action']) do
     log_work_action(:message_thread_tagged, message_thread_tag, client_created_at: nil, thread_id: params[:message_thread_id].to_i)
   end
```

+++

Split out a method:

```ruby
add_method_tracer :contacts_results, 'ClientSearch/contacts_results'
```

+++

[notes]

* transactions
* traces
* database
* dashboards
* throughput v. slow queries
* percentiles

---

## Profiling Ruby

+++

`!debug`

* Turn log level to DEBUG (shows Elasticsearch queries)
* Pipe the ActiveRecord query logs to STDOUT
* shows backtraces etc

[demo]

---

[`.pryrc`](https://github.com/finventures/fin-core-beta/blob/master/.pryrc)

Custom commands:

* debugger
* `dp` | `ep`
* backtrace
* autocompleter
* prompt

---

`Benchmark`

```ruby

rails> Benchmark.bm do |x|
*   x.report { User.find(2) }
* end
  user     system      total        real
  0.020000   0.010000   0.030000 (  0.031872)


rails> Benchmark.realtime { User.find 2 }
=> 0.004197215981548652

```

+++

`RubyProf`


```ruby

> result = RubyProf.profile { AgentState.event_loop }
> printer = RubyProf::CallStackPrinter.new(result)
> printer.print(open("./profile2.html", "w"))

$> open profile2.html

```

---

🌝

---

# SQL

* Declarative (v. imperative). cf. `select, map, group_by` v. `for` loops
* __Structured English Query Language__ (`SEQUEL`) -> `SQL`
* Ingres(s) (Berkeley) -> Postgres (1980's)

+++

[EF Codd, a Relational Algebra](http://www.morganslibrary.net/files/codd-1970.pdf)

### Relation

<blockquote>
Given sets X1 , S1, S2, . . , S, (not necessarily
distinct), R is a relation on these n sets if it is a set of ntuples
each of which has its first element from S1, its
second element from Sz , and so on.
</blockquote>


+++

## What is ACID-compliance?

* Atomicity: Transaction that rolls back completely or commits completely
* Consistency: Transaction rolls back on trigger failure or validation violation
* Isolation: Concurrent transactions
* Durability: Write-ahead log

---

### MVCC

* Every transaction has an id (`xid`)
* Every row can have multiple versions, each has an `(xmin, xmax)`
* Rows are only visible for read to `xid IN [xmin, xmax]`
* `INSERT`: create version with `xmin = xid`
* `DELETE`: create version with `xmax = xid`
* `COMMIT`: set `committed[xid] = true`
* Always read latest visible committed version

+++

## Rollbacks

* Can leave versions lying around, bloating the db and indexes
* `VACUUM` cleans these up
* Postgres has `auto_vacuum`. Tuning frequency of vacuuming (& analyzing) is a
  big deal for perf

+++

## xid Wraparound

* `xid` is 32-bit integer (2^32 = 4 billion)
* If it wraps around -> 0, all rows have `xmin` > `xid` and become visible 💥
* Need to `VACUUM FREEZE` to set `xmin` to sentinel value

---

## Write-ahead logs

* Every `COMMIT`, write a binary log of the changes made to data
* Regularly __checkpoint__, ie. flush to durable storage

1. Crash recovery: find last checkpoint and replay
2. Replication: ship binary log over and replay

+++

## Log-table duality

<blockquote>
Logs are tables in motion -- Jay Kreps
</blockquote>

cf. Streams (Kafka/Kinesis etc.)

---

🐡

---

### `ActiveRecord`, `arel` etc

(Pat Shaughnessy)

![arel-1](diagrams/arel-1.png)

+++

![arel-2](diagrams/arel-2.png)

+++

![arel-3](diagrams/arel-3.png)

---

👾

---

### The Database

+++

![libpq](diagrams/libpq.png)

+++

### Parser → Rewriter → Optimizer → Cost analysis → Execute

+++

![parser](diagrams/parser.png)

+++

### Rewriter

Replace Views with underlying query tree

+++

### Optimizer

* How to scan tables
  * e.g. Sequential, Index
* How to join tables (and in what order)
  * e.g. Hash, Merge
* Inlining, predicate pushdown etc

📌

+++

![plan](diagrams/query-plan.png)

+++

### Cost analysis across many possible plans

`postgresql.conf`

```
seq_page_cost = 1.0                    # Read 8K Page from disk sequentially
random_page_cost = 4.0                 # Random I/O
cpu_tuple_cost = 0.01                  # Process a row
cpu_index_tuple_cost = 0.005           # Process an index entry
cpu_operator_cost = 0.0025             # Perform an operation

```

Goal: Find a good plan fast enough so that it actually matters

+++

### Statistics

* Estimate # of rows returned by query (each step in plan)
* `ANALYZE` updates statistics
* `auto_vacuum` runs `ANALYZE`
  * `n_distinct`,
  * `most_common_vals` (and their frequencies),
  * `histogram_bounds` (distribution of values)

👎  Assumes independence of columns

+++

### Executor

* stream-processing style dependency graph
* Parent node _pulls_ on its children
* i.e. time to first row matters, as does generating all rows

---

🥑

---

## Tables and Indexes

![index-table](diagrams/index-table.png)

---

## Reading a table

* Cost depends on how much data you're reading
* Index reads are random and expensive (but you can read only what you want)
* Sequential reads are cheap (but you have to read all of it)

+++

## Sequential scan

![seq-scan](diagrams/seq-scan.png)

+++

## Index scan

![index-scan](diagrams/index-scan.png)

🍩  Index-_only_ scans: when **all** `SELECT` columns are indexed

+++

## Bitmap Index Scan + Bitmap Heap Scan

![bitmap-scan](diagrams/bitmap-scan.png)


---
