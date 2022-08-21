---
sidebar_position: 1
title: Subgraph Data Introduction
---

# Balancer Subgraph Introduction

The Balancer Subgraph indexes data on the Balancer smart contracts with a GraphQL interface. It updates data in response to function calls and contract events to maintain data on the `Vault`, `Pools`, `AssetManagers` etc, to power front-end apps and integrations.

Balancer has a GraphQL API Endpoint hosted by [The Graph](https://thegraph.com/docs/about/introduction#what-the-graph-is) called a subgraph for indexing and organizing data from the Sablier smart contracts.

This subgraph can be used to query Balancer data.

Subgraph information is serviced by a decentralized group of server operators called Indexers.

## GraphQL Schema

The schema of GraphQL elements available is defined in [`/schema.graphql` ](https://github.com/balancer-labs/balancer-subgraph-v2/blob/master/schema.graphql)

## Ethereum Mainnet

[Creating an API Key Video Tutorial](https://www.youtube.com/watch?v=UrfIpm-Vlgs)

- [Explorer Page](https://thegraph.com/explorer/subgraph?id=CptFsHp6zar7kdfGYbVAqMeF1wNA1pJs6GaaJvPgeCfu&view=Overview)
- Graphql Endpoint: https://gateway.thegraph.com/api/[api-key]/subgraphs/id/GAWNgiGrA9eRce5gha9tWc7q5DPvN3fs5rSJ6tEULFNM
- [Code Repo](https://github.com/balancer-labs/)

## Helpful Links

[Querying from an Application](https://thegraph.com/docs/en/developer/querying-from-your-app/)

[Managing your API Key & Setting your indexer preferences](https://thegraph.com/docs/en/studio/managing-api-keys/)
