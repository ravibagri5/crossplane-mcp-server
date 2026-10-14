---
title: "Order a Database in Plain English: Crossplane Platform APIs, Now AI-Ready"
published: true
description: "Crossplane turns infrastructure into platform APIs. An MCP server makes them usable from any AI assistant, so developers can order and troubleshoot infrastructure in plain English, without writing YAML or learning Crossplane."
tags: crossplane, platformengineering, ai, mcp
layout: article
permalink: /platform-apis-ai-crossplane-mcp/
date: 2026-10-14
---

A developer needs a Postgres database for a new service.

On a Crossplane platform, the capability already exists. The platform team has published a database API, wired it to a Composition, and decided what "small" means. But to use it, the developer still has to find the right kind, read an XRD schema, write YAML, apply it, and then interpret `Ready` and `Synced` conditions when something does not come up.

The platform API is there. The experience of using it still assumes Crossplane knowledge, and a fair amount of YAML.

What if the developer could simply ask, in plain English?

> Create a small Postgres database called orders-db.

And when it is not ready ten minutes later:

> Why isn't orders-db ready?

That is what [crossplane-mcp-server](https://github.com/ravibagri5/crossplane-mcp-server) is for. It turns a Crossplane control plane into a set of AI-ready tools, so the platform APIs you already built can be discovered, used and troubleshot from any MCP-compatible assistant or agent.

## Platform APIs are the product, access is the bottleneck

Crossplane's real achievement is not provisioning cloud resources. Plenty of tools do that. It is letting platform teams expose infrastructure as **APIs they own**: a `PostgreSQLInstance` or an `App` that hides provider details behind a contract the platform team controls.

That contract is exactly what you want application teams to consume. But today, consuming it usually means one of:

- Learning enough Crossplane and Kubernetes to write and debug manifests.
- Waiting for a portal, CLI or form that the platform team has to build and maintain.
- Opening a ticket and waiting for the platform team.

Each of those puts the platform team back in the loop for work the API was meant to make self-service.

## MCP: one interface for every AI client

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) is an open standard that lets AI applications discover and call tools exposed by a server. Write the integration once, and every MCP-compatible client can use it: Claude Desktop and Claude Code, Cursor, Windsurf, VS Code, Goose, and agent frameworks that speak MCP.

`crossplane-mcp-server` is that integration for Crossplane. It is a standard MCP server, configured like any other, that exposes your control plane as tools an assistant can reason with:

| A person asks | The server provides |
| --- | --- |
| "What can I order from this platform?" | The platform APIs the control plane offers, from its XRDs |
| "What do I need to give you for a database?" | The fields that API accepts, with an example manifest |
| "Create a small Postgres database called orders-db." | A claim built from the API's own schema, with a dry run first |
| "Is it ready yet? Why not?" | The composition tree, the deepest failing resource and the provider's real error |
| "Is anything broken in production?" | Control plane health, unhealthy resources and package status |
| "What would deleting this destroy?" | The blast radius, before anyone acts |

Nobody in that table named a Kubernetes kind. The server finds the platform API, reads its schema and fills it in. The developer works with the capability, not the implementation.

The server provides tools and live data; the MCP client provides the language model and the conversation. The assistant answers from what the control plane actually returns, not from what a model guesses.

## Ordering a database in plain English

Here is the flow against the project's demo control plane. The developer asks for a database, in their own words, and wants to see it first:

> Create a small Postgres database called orders-db. Show me the manifest before you create anything.

The assistant calls `crossplane_database_create` with a dry run. The server finds the database API this control plane offers, reads the schema its XRD declares, and returns exactly what would be submitted:

```text
crossplane_database_create(name="orders-db", size="small", dryRun=true)

apiVersion: demo.crossplane.io/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: orders-db
  namespace: default
spec:
  parameters:
    engine: postgresql
    size: small
    version: "16"
```

The developer approves, the claim is created, and Crossplane does what it always does: reconciles it through the Composition the platform team wrote. The platform team still decides what a "small Postgres database" is. The developer never wrote a line of YAML or looked up a kind.

## Why isn't my database ready?

Creating is not the same as Ready. When something fails, a generic Kubernetes tool can return objects, but it does not know how a claim leads to a composite, how a composite leads to managed resources, or that the useful error is usually in an event three levels down.

This server does. Ask why a claim is not ready and it walks the tree to the deepest failure:

```text
crossplane_diagnose(kind="PostgreSQLInstance", name="app-db")

PostgreSQLInstance app-db is NOT READY.

Verdict: Instance/app-db-rds is the deepest failure: InvalidParameterValue:
the instance class db.t2.mega does not exist

Control plane checks:
CHECK         STATE   DETAIL
composition   OK      Composition "postgres-aws" exists.
providers     OK      all 3 provider(s) are Installed and Healthy
```

That is the difference between "something is wrong" and "fix the instance class in your Composition". The same Crossplane knowledge powers the rest of the toolset: drift detection between declared and observed state, finding what is holding up a stuck deletion, validating or rendering a Composition before applying it, and checking provider and function health.

## Built so platform teams can say yes

Giving an AI assistant access to infrastructure is a trust decision. The server is designed so that decision can be made incrementally:

- **Read-only by default.** Out of the box, the create and update tools are not registered at all, so a client cannot see or call them. Point it at production and it can explain everything and change nothing.
- **Writes are opt-in.** Start the server with `--read-only=false` to let teams order through your platform APIs, ideally starting with a development control plane.
- **Nothing deletes.** There is no delete tool and no delete call in the codebase. Ask it to delete a database and it can show the impact; a person runs the deletion.
- **Platform APIs only.** Writes are limited to Crossplane kinds. The server refuses Secrets, Deployments and RBAC, no matter how it is asked.
- **Reviewable writes.** Every write tool supports `dryRun`, writes are server-side applies under a named field manager, and created objects are labelled for easy discovery.
- **RBAC still decides.** The example [`rbac-write.yaml`](https://github.com/ravibagri5/crossplane-mcp-server/blob/main/deploy/rbac-write.yaml) grants `create`, `update` and `patch` on named platform API groups, with no `delete` verb.

A sensible rollout: connect read-only, let engineers troubleshoot with it, then enable ordering where the platform API and permissions are ready for self-service.

## Works wherever your AI works

Because it is a standard MCP server, the same tools can reach developers in their editor, in a desktop assistant, or inside an AI-driven automation. A chat surface such as Slack can use it too, through an assistant or bridge that supports MCP. The project itself does not ship a Slack bot; MCP is the connection point, and the host application owns the chat experience and user identity.

For platform teams, that means maintaining one Crossplane integration instead of building a separate one for every assistant, portal and bot your organization adopts.

## Get started in minutes

Install it, see what it exposes, and check it can reach your control plane:

```sh
go install github.com/ravibagri5/crossplane-mcp-server/cmd/crossplane-mcp-server@latest

crossplane-mcp-server tools
crossplane-mcp-server call crossplane_status
```

Installing from source needs Go 1.26 or newer; pre-built binaries and a container image are on the [releases page](https://github.com/ravibagri5/crossplane-mcp-server/releases). It is also listed in the official MCP Registry and on Smithery.

Then add it to your MCP client like any other server. For VS Code, in `.vscode/mcp.json`:

```json
{
  "servers": {
    "crossplane": {
      "type": "stdio",
      "command": "crossplane-mcp-server",
      "args": ["--clusters", "staging,production"]
    }
  }
}
```

`--clusters` names the control planes the assistant may reach. Add `--read-only=false` to `args` on a development control plane when you want to order as well as inspect.

No Crossplane platform handy? The [demo control plane](https://github.com/ravibagri5/crossplane-mcp-server/tree/main/examples/demo) runs on kind with provider-nop, so you can order fake databases and workloads, watch them reconcile, and try every tool without a cloud account.

## The point

Crossplane already lets platform teams turn infrastructure into products. MCP makes those products reachable from the AI tools developers are adopting anyway. Put the two together and a platform API stops being something you need Crossplane expertise and YAML to use, and becomes something anyone on the team can ask for in plain English, inspect and understand, within the boundaries the platform team set.

Try [crossplane-mcp-server](https://github.com/ravibagri5/crossplane-mcp-server), run the demo, and tell me which platform API you would put in front of your developers first.
