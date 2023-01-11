---
sidebar_position: 21
title: The schema file
description: >-
  The schema file defines the target data model.
---

# The schema file

The file `schema.graphql` defines the target db schema via the TypeORM entity classes generated by [`squid-typeorm-typegen`](https://github.com/subsquid/squid-sdk/tree/master/typeorm/typeorm-codegen) in `src/model/generated`. To generate the entity classes, run

```bash
npx squid-typeorm-codegen
npm run build
```


The schema file also defines the GraphQL API of the [OpenReader](https://github.com/subsquid/squid/tree/master/openreader) server to automatically present the data with a rich GraphQL API. See [GraphQL API](/graphql-api) section for more details on how to model the API with the schema. 