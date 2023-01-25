---
sidebar_position: 1
title: Subgraph Data Introduction
---

# Balancer Gauges Subgraph Introduction

The Balancer Subgraph indexes data on the Balancer smart contracts with a GraphQL interface. It updates data in response to function calls and contract events to maintain data on the `Vault`, `Pools`, `AssetManagers` etc, to power front-end apps and integrations.

Balancer has a GraphQL API Endpoint hosted by [The Graph](https://thegraph.com/docs/about/introduction#what-the-graph-is) called a subgraph for indexing and organizing data from the Sablier smart contracts.

This subgraph can be used to query Balancer data.

Subgraph information is serviced by a decentralized group of server operators called Indexers.

## GraphQL Schema

The schema of GraphQL elements available is defined in [`/schema.graphql` ](https://github.com/balancer-labs/balancer-subgraph-v2/blob/master/schema.graphql)

## Ethereum Mainnet


- [Explorer Page](https://thegraph.com/hosted-service/subgraph/balancer-labs/balancer-gauges-beta)
- Graphql Endpoint: https://api.thegraph.com/subgraphs/name/balancer-labs/balancer-gauges-beta
- [Code Repo](https://github.com/balancer-labs/gauges-subgraph)

## Helpful Links

[Querying from an Application](https://thegraph.com/docs/en/developer/querying-from-your-app/)

[Managing your API Key & Setting your indexer preferences](https://thegraph.com/docs/en/studio/managing-api-keys/)

[Creating an API Key Video Tutorial](https://www.youtube.com/watch?v=UrfIpm-Vlgs)