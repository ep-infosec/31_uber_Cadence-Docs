---
layout: default
title: Polling
permalink: /docs/use-cases/polling
---

# Polling

Polling is executing a periodic action checking for a state change. Examples are pinging a host, calling a REST API, or listing an Amazon S3 bucket for newly uploaded files.

Cadence support for long running :activity:activities: and unlimited retries makes it a good fit.

Some real-world use cases:

* Network, host and service monitoring
* Processing files uploaded to FTP or S3
* [Cadence Polling Cookbook by Instaclustr: Polling an external API for a specific resource to become available: ](https://info.instaclustr.com/rs/620-JHM-287/images/Cadence_Cookbook.pdf)
