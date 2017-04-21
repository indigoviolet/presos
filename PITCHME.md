---

# Perf (mostly SQL)

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

+++

![perf-stack](diagrams/perf-stack.png)

---

## The optimization loop

+++

![opt-loop](diagrams/opt-loop.png)

---

## Profilers

+++

![profilers](diagrams/profilers.png)

---

## NewRelic

+++

### Apdex

* Response time setting to record "traces"
* satisfying < tolerable (apdex_t) < frustrating (apdex_f = 4 * apdex_t)
* Global setting of apdex_t = 300ms
* Can be changed on per-transaction level (for key transactions)

+++

### Custom instrumentation

Set or modify transaction names:

```ruby
  NewRelic::Agent.set_transaction_name("#{transaction_name} - empty")
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

---
