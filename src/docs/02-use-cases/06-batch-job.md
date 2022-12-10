---
layout: default
title: Batch job
permalink: /docs/use-cases/batch-job
---

# Batch job

A lot of batch jobs are not pure data manipulation programs. For those, the existing big data frameworks are the best fit.
But if processing a record requires external API calls that might fail and potentially take a long time, Cadence might be preferable.

One of our internal Uber customer uses Cadence for end of month statement generation. Each statement requires calls to multiple
microservices and some statements can be really large. Cadence was chosen because it provides hard guarantees around durability of the financial data and seamlessly deals with long running operations, retries, and intermittent failures.
