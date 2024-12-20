# Lab 2: Generate Vector Embeddings

Following our success in ingesting documents, we shall generate vectors for the movie documents in our cluster using Elasticsearch and Langchain.

## What is a vector?

Remembering back to mathematics, a [vector](https://en.wikipedia.org/wiki/Vector_(mathematics_and_physics)) is a geometric object that has length and a direction. Often they are comprised of multiple dimensions, with a numerical representation of each dimension:

```
[these, are, not, the, droid, you, look, for, no, i, am, father]
[1,   2,   1,   1,   1,   1,   1,   1,   0,   0,   0,   0]
```

All documents in our index, when converted into a vector representation, form a space where we can compare the proximity of a give query (also converted to a vector) to the documents in our vector space:

![Star Wars Sample Vector Space](./screenshots/2/lab-2-what-is-a-vector.png)

Vector search is the fundamental algorithm that underpins semantic search, which you will have seen in search engines where you ask questions and they return relevant results whose keywords may not exactly match. 

*kNN, or k-Nearest Neighbour* search, compares the embedded query with the documents in the vector space and returns the documents closest to the query through [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance):

![kNN search overview](./screenshots/2/lab-2-knn-search-overview.png)

In this lab, we shall make use of the transformer model [`mxbai-embed-large`](https://ollama.com/library/mxbai-embed-large) downloaded to our machine using [Ollama](https://ollama.com/) alongside [Langchain](https://js.langchain.com/docs/introduction/) to generate embeddings for our documents. These documents shall also be ingested into Elasticsearch, replacing the previous index of documents.

The alternative approach is to generate embeddings behind the scenes using the `semantic_text` field type based on a model imported into Elasticsearch. Instructions on this approach [is covered in this blog](https://www.elastic.co/search-labs/blog/semantic-search-simplified-semantic-text). Note that this approach uses a licensed feature, unlike the approach here.

## Steps

1. If you haven't already, install Ollama onto your machine:

```zsh
brew install ollama
brew services start ollama

ollama serve
```

For Windows follow [this guide](https://dev.to/evolvedev/how-to-install-ollama-on-windows-1ei5).

2. Download and start model `mxbai-embed-large` via the terminal, and confirm the manifest has been downloaded:

```zsh
ollama pull mxbai-embed-large
ollama list
```

3. Via the terminal, send a simple cURL request to generate embeddings for a given prompt:

```zsh
curl http://localhost:11434/api/embeddings -d '{
  "model": "mxbai-embed-large",
  "prompt":"Why is the sky blue?"
}'
```

4. Moving to the code, in the `ingestion` folder, install the required `langchain` and `ollama` dependencies:

```zsh
npm i @langchain/community @elastic/elasticsearch @langchain/ollama @langchain/core
```

5. Initialize the Ollama embeddings using Langchain:

```ts
import { OllamaEmbeddings } from "@langchain/ollama";

// Initialize Ollama embeddings
const ollamaEmbeddings = new OllamaEmbeddings({
  model: "mxbai-embed-large", // Default value
  //baseUrl: "http://localhost:11434", // Default value
});
```

6. Create a new vector store using Langchain, passing an instance of the Elasticsearch client:

```ts
import {
  ElasticVectorSearch,
  type ElasticClientArgs,
} from "@langchain/community/vectorstores/elasticsearch";

import { Client, type ClientOptions } from "@elastic/elasticsearch";

// Initialize Langchain and Elasticsearch
const config: ClientOptions = {
  node: process.env.ELASTIC_DEPLOYMENT,
};

if (process.env.ELASTIC_API_KEY) {
  config.auth = {
    apiKey: process.env.ELASTIC_API_KEY,
  };
}

const clientArgs: ElasticClientArgs = {
  client: new Client(config),
  indexName: process.env.INDEX_NAME,
  vectorSearchOptions: {
    engine: "hnsw",
    similarity: "cosine", //Default cosine
  },
};

const vectorStore = new ElasticVectorSearch(ollamaEmbeddings, clientArgs);
```

7. We're going to split the documents into smaller chunks using a `RecursiveTextSplitter` when loading the documents from `data/movies.json`:

```ts
import { Document } from "@langchain/core/documents";
import {
  RecursiveCharacterTextSplitter,
  TextSplitter,
} from "langchain/text_splitter";

import fs from "node:fs";
import { Movie, MovieCollection } from "./movies";

/**
 * Generate collection of documents from specified JSON file
 * @param pathToJSON file containing movies
 * @returns array of documents
 */
async function generateDocumentsFromJson(
  pathToJSON: string
): Promise<Document[]> {
  try {
    const jsonDocs: MovieCollection = JSON.parse(
      fs.readFileSync(pathToJSON).toString()
    );
    console.log(`Doc count: ${jsonDocs.results.length}`);

    // Split text based on overview
    const textSplitter = new RecursiveCharacterTextSplitter({
      chunkSize: 200,
      chunkOverlap: 0,
    });
    let documents: Document[] = [];

    for (const content of jsonDocs.results) {
      const splitDocs: Document[] = await splitContentByOverview(
        textSplitter,
        content
      );
      documents = documents.concat(splitDocs);
    }
    return documents;
  } catch (e) {
    console.log(e);
    return [];
  }
}
```

To do this, create a utility function to split the documents based on overview using the splitter we have created:

```ts
/**
 * Chunk content into smaller documents based on overview
 * @param textSplitter
 * @param content
 * @returns Document[] based on split document
 */
async function splitContentByOverview(
  textSplitter: TextSplitter,
  content: Movie
) {
  const splits = await textSplitter.splitText(content.overview);
  console.log(`Split doc count: ${splits.length}`);

  const splitDocs: Document[] = splits.map((split, index) => {
    return new Document({
      pageContent: split,
      metadata: {
        title: content.title,
        original_language: content.original_language,
        popularity: content.popularity,
        release_date: new Date(content.release_date),
        vote_average: content.vote_average,
        vote_count: content.vote_count,
        is_adult: content.adult,
        poster_path: `https://image.tmdb.org/t/p/original${content.poster_path}`,
        chunk: index,
      },
    });
  });
  return splitDocs;
}
```

8. Create a utility function to generate embeddings for our documents using the vector store, and add them to Elasticsearch:

```ts
// Ingest document with embedding
async function generateEmbeddings(documents: Document[]): Promise<string[]> {
  try {
    return await vectorStore.addDocuments(documents);
  } catch (e) {
    console.log(e);
    return [];
  }
}
```

9. Joining everything together in our `main` function will ingest all documents with corresponding embeddings:

```ts
async function main() {
  // Clean up index if it exists
  vectorStore.deleteIfExists();

  // Load data from JSON
  const documents = await generateDocumentsFromJson("./data/movies.json");

  // Ingest documents and generate embeddings
  const ids = await generateEmbeddings(documents);
  console.log(`Ingested ${ids.length} documents with embeddings`);
}

main();
```

10. Create a utility function `findRelevantMovies`, similar to before, that performs a vector search using a question, filtering out adult content:

```ts
/**
 * Example search function to find relevant movies
 * @param text: prompt to be used for similarity search
 * @returns
 */
async function findRelevantMovies(text: string): Promise<Document[]> {
  try {
    const filter = [
      {
        operator: "match",
        field: "title",
        value: "dance",
      },
    ];

    const similaritySearchResults = await vectorStore.similaritySearch(
      text,
      1,
      filter
    );
    return similaritySearchResults;
  } catch (e) {
    console.log(e);
    return [];
  }
}
```

11. Amend the main method to find relevant movies using the question *Find me a movie featuring a dark anti-hero*:

```ts
async function main() {
  // Prior code omitted

  // Example retrieval
  const topMovieResult = await findRelevantMovies(
    "Find me a movie featuring a dark anti-hero"
  );

  console.log(`The anti-hero movie is: "${topMovieResult[0].metadata.title}"`);
}

main();
```

## Expected Result

If all goes well you will have replaced your index with a set of movies enriched with a dense vector field similar to the below:

```json
{
  "_index": "movies-carly-richmond",
  "_id": "6835b8ef-f8fa-426a-b96f-28cc06196e54",
  "_score": 2.8742206,
  "_source": {
    "embedding": [
      0.0385989,
      -0.00013410488, ...
      ],
    "metadata": {
      "title": "Deadpool & Wolverine",
      "original_language": "en",
      "popularity": 1722.492,
      "releaseDate": "2024-07-24T00:00:00.000Z",
      "vote_average": 7.707,
      "vote_count": 4910,
      "isAdult": false,
      "posterPath": "https://image.tmdb.org/t/p/original/8cdWjvZQUExUUTzyp4t6EDMubfO.jpg",
      "chunk": 0
      },
    "text": "A listless Wade Wilson toils away in civilian life with his days as the morally flexible mercenary, Deadpool, behind him. But when his homeworld faces an existential threat, Wade must reluctantly"
  }
}
```