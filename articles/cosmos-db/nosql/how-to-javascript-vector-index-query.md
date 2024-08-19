---
title: Index and query vector data in JavaScript
titleSuffix: Azure Cosmos DB for NoSQL
description: Add vector data Azure Cosmos DB for NoSQL and then query the data efficiently in your JavaScript application
author: jcodella
ms.author: jacodel
ms.service: azure-cosmos-db
ms.subservice: nosql
ms.topic: how-to
ms.date: 08/08/2024
ms.custom: query-reference, build-2024, devx-track-js
---

# Index and query vectors in Azure Cosmos DB for NoSQL in JavaScript

[!INCLUDE[NoSQL](../includes/appliesto-nosql.md)]

The Azure Cosmos DB for NoSQL vector search feature is in preview. Before you use this feature, you must first register for the preview. This article covers the following steps:

1. Registering for the preview of Vector Search in Azure Cosmos DB for NoSQL

1. Setting up the Azure Cosmos DB container for vector search

1. Authoring vector embedding policy

1. Adding vector indexes to the container indexing policy

1. Creating a container with vector indexes and vector embedding policy

1. Performing a vector search on the stored data

This guide walks through the process of creating vector data, indexing the data, and then querying the data in a container.

## Prerequisites

- An existing Azure Cosmos DB for NoSQL account.
  - If you don't have an Azure subscription, [Try Azure Cosmos DB for NoSQL free](https://cosmos.azure.com/try/).
  - If you have an existing Azure subscription, [create a new Azure Cosmos DB for NoSQL account](how-to-create-account.md).
- Latest version of the Azure Cosmos DB [JavaScript](sdk-nodejs.md) SDK (Version 4.1.0 or later)

## Register for the preview

Vector search for Azure Cosmos DB for NoSQL requires preview feature registration. Follow the below steps to register:

1. Navigate to your Azure Cosmos DB for NoSQL resource page.

1. Select the "Features" pane under the "Settings" menu item.

1. Select for "Vector Search in Azure Cosmos DB for NoSQL."

1. Read the description of the feature to confirm you want to enroll in the preview.

1. Select "Enable" to enroll in the preview.

    > [!NOTE]
    > The registration request will be autoapproved, however it may take several minutes to take effect.

## Understand the steps involved in vector search

The following steps assume that you know how to [setup a Cosmos DB NoSQL account and create a database](quickstart-portal.md). The vector search feature is currently only supported on new containers, not existing container. You need to create a new container and then specify the container-level vector embedding policy and the vector indexing policy at the time of creation.

Let’s take an example of creating a database for an internet-based bookstore and you're storing Title, Author, ISBN, and Description for each book. We also define two properties to contain vector embeddings. The first is the "contentVector" property, which contains [text embeddings](/azure/ai-services/openai/concepts/models#embeddings ) generated from the text content of the book (for example, concatenating the "title" "author" "isbn" and "description" properties before creating the embedding). The second is "coverImageVector," which is generated from [images of the book’s cover](/azure/ai-services/computer-vision/concept-image-retrieval).

1. Create and store vector embeddings for the fields on which you want to perform vector search.
2. Specify the vector embedding paths in the vector embedding policy.
3. Include any desired vector indexes in the indexing policy for the container.

For subsequent sections of this article, we consider this structure for the items stored in our container:

```json
{
"title": "book-title",
"author": "book-author",
"isbn": "book-isbn",
"description": "book-description",
"contentVector": [2, -1, 4, 3, 5, -2, 5, -7, 3, 1],
"coverImageVector": [0.33, -0.52, 0.45, -0.67, 0.89, -0.34, 0.86, -0.78]
}
```

## Create a vector embedding policy for your container

Next, you need to define a container vector policy. This policy provides information that is used to inform the Azure Cosmos DB query engine how to handle vector properties in the VectorDistance system functions. This policy also informs the vector indexing policy of necessary information, should you choose to specify one.

The following information is included in the contained vector policy:

| | Description |
| --- | --- |
| **`path`** | The property path that contains vectors |
| **`datatype`** | The type of the elements of the vector (default `Float32`) |
| **`dimensions`** | The length of each vector in the path (default `1536`) |
| **`distanceFunction`** | The metric used to compute distance/similarity (default `Cosine`) |

For our example with book details, the vector policy can look like the example JSON:

```javascript
const vectorEmbeddingPolicy: VectorEmbeddingPolicy = {
      vectorEmbeddings: [
        {
          path: "/coverImageVector",
          dataType: "float32",
          dimensions: 8,
          distanceFunction: "dotproduct",
        },
        {
          path: "contentVector",
          dataType: "float32",
          dimensions: 10,
          distanceFunction: "cosine",
        },
      ],
    };
```

## Create a vector index in the indexing policy

Once the vector embedding paths are decided, vector indexes need to be added to the indexing policy. You must apply the vector policy during the time of container creation and it can’t be modified later. For this example, the indexing policy would look like this:

```javascript
const indexingPolicy: IndexingPolicy = {
  vectorIndexes: [
    { path: "/coverImageVector", type: "quantizedFlat" },
    { path: "/contentVector", type: "diskANN" },
  ],
    inlcludedPaths: [
      {
        path: "/*",
      },
    ],
    excludedPaths: [
      {
        path: "/coverImageVector/*",
      },
      {
        path: "/contentVector/*",
      },
    ]
};
```

Now create your container as usual.

```javascript
const containerName = "vector embedding container";
    // create container
    const { resource: containerdef } = await database.containers.createIfNotExists({
      id: containerName,
      vectorEmbeddingPolicy: vectorEmbeddingPolicy,
      indexingPolicy: indexingPolicy,
    });
```

> [!IMPORTANT]
> Currently vector search in Azure Cosmos DB for NoSQL is supported on new containers only. You need to set both the container vector policy and any vector indexing policy during the time of container creation as it can’t be modified later. Both policies will be modifiable in a future improvement to the preview feature.

## Run a vector similarity search query

Once you create a container with the desired vector policy, and insert vector data into the container, you can conduct a vector search using the [Vector Distance](query/vectordistance.md) system function in a query. Suppose you want to search for books about food recipes by looking at the description. You first need to get the embeddings for your query text. In this case, you might want to generate embeddings for the query text – "food recipe." Once you have the embedding for your search query, you can use it in the VectorDistance function in the vector search query and get all the items that are similar to your query as shown here:

```sql
SELECT c.title, VectorDistance(c.contentVector, [1,2,3,4,5,6,7,8,9,10]) AS SimilarityScore
FROM c
ORDER BY VectorDistance(c.contentVector, [1,2,3,4,5,6,7,8,9,10])
```

This query retrieves the book titles along with similarity scores with respect to your query. Here's an example in JavaScript:

```javascript
const { resources } = await container.items
  .query({
    query: "SELECT c.title, VectorDistance(c.contentVector, @embedding) AS SimilarityScore FROM c  ORDER BY VectorDistance(c.contentVector, @embedding)"
    parameters: [{ name: "@embedding", value: [1,2,3,4,5,6,7,8,9,10] }]
  })
  .fetchAll();
for (const item of resources) {
  console.log(`${itme.title}, ${item.SimilarityScore} is a capitol `);
}
```

## Related content

- [VectorDistance system function](query/vectordistance.md)
- [Vector indexing](../index-policy.md)
- [Setup Azure Cosmos DB for NoSQL for vector search](../vector-search.md).