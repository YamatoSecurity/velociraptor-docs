---
title: Server.Hunts.List
hidden: true
tags: [Server Artifact]
---

List Hunts currently scheduled on the server.


```yaml
name: Server.Hunts.List
description: |
  List Hunts currently scheduled on the server.

type: SERVER

sources:
  - query: |
      SELECT hunt_id,
             timestamp(epoch=create_time) as Created,
             join(array=start_request.artifacts, sep=",") as Artifact,
             state
      FROM hunts()

```
