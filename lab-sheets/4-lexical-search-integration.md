# Lab 4: Lexical Search Integration

In this segment we shall integrate an example Elasticsearch query into our movie application. You have a skeleton application where we shall populate a row of popular releases on the home page:

![Sample Movie Application](./screenshots/4/lab-4-starting-site.png)

## Steps

1. Ensure you are able to run the skeleton application locally:

```zsh
cd movie-rag
npm install
npm run dev
```

*Note: if you receive error **You are using Node.js 19.0.0. For Next.js, Node.js version "^18.18.0 || ^19.8.0 || >= 20.0.0" is required.**, ensure you are using a compatible version of Node.js using the below steps:*

```zsh
nvm install 20.13.1
nvm use 20.13.1
```

2. Ensure that route `/api/popular/` is created in your Next.js application. This is the route that component `Popular.tsx` will call to get results.

3. In the *Dev Tools* console in Elasticsearch, write a query to pull back the most popular results based on `vote_average` and sorted by `metadata.popularity`:

```json
GET movies/_search
{
  "query": {
    "range": {
      "metadata.vote_average": {
        "gte": 5
      }
    }
  },
  "sort": [
    { "metadata.popularity": "desc" }
    ]
}
```

4. Amend the query to only return the first chunk:

```json
GET movies/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "metadata.vote_average": {
              "gte": 5
            }
          }
        },
        {
          "term": {
            "metadata.chunk": 0
          }
        }
      ]
    }
  },
  "sort": [
    { "metadata.popularity": "desc" }
    ]
}
```

5. Configure the query to return the first 4 results, and to only return the metadata and text fields:

```json
GET movies/_search
{
  "size": 5,
  "_source": [ "metadata", "text" ],
  "sort": [{ "metadata.popularity": "desc" }],
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "metadata.vote_average": {
              "gte": 5
            }
          }
        },
        {
          "term": {
            "metadata.chunk": 0
          }
        }
      ]
    }
  }
}
```

6. Let's add this query to our application. Check that the `@elastic/elasticsearch` client is installed:

```zsh
npm install @elastic/elasticsearch
```

7. In the utility file `lib/elasticsearch.ts` instantiate a simple Elasticsearch client, using the credentials and index name from your `.env` file:

```ts
import { Client, type ClientOptions } from "@elastic/elasticsearch";

// Initialize Elasticsearch
const config: ClientOptions = {
  node: process.env.ELASTIC_DEPLOYMENT,
};

if (process.env.ELASTIC_API_KEY) {
  config.auth = {
    apiKey: process.env.ELASTIC_API_KEY,
  };
}

const client: Client = new Client(config);
const indexName: string = process.env.INDEX_NAME || "";
```

8. Complete the method `getPopularMovies` to return the results of our query to the application:

```ts
/**
 * Retrieves the most popular movies based on average vote
 * @returns most popular 5 movies
 */
export async function getPopularMovies() {
  if (!client || !indexName) {
    return;
  }

  const query = {
    index: indexName,
    query: {
      bool: {
        must: [
          {
            range: {
              "metadata.vote_average": {
                gte: 5,
              },
            },
          },
          {
            term: {
              "metadata.chunk": 0,
            },
          },
        ],
      },
    },
    sort: [{ "metadata.popularity": "desc" }],
    size: 4,
    _source: [ "metadata", "text" ]
  };

  return client.search(query);
}
```

9. Amend the `.api/popular` route to return the results of the query:

```tsx
export async function GET(req: Request) {
  try {
    const popularResponses = await getPopularMovies();
    const movies = popularResponses?.hits.hits.map((doc) => {
      return doc._source;
    })

    return Response.json({ results: movies });
  } catch (e) {
    console.error(e);
    return Response.json({
      results: [],
      errorMessage: e
    });
  }
}
```

## Expected Result

If all is well with connecting to Elasticsearch and finding the most popular movies, you will now have a row of results similar to the below:

![Final Movie Application](./screenshots/4/lab-4-popular-row.png)