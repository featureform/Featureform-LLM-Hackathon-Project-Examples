# Quickstart: Building a Newscatcher Article Search Engine with Featureform, HuggingFace, and Pinecone

Welcome, fellow hackers!

In this tutorial, we're going to build a search engine that can find articles related to natural phenomena based on a user's text query. We'll be using Featureform for managing our data transformations and features, Huggingface's sentence-transformers for generating embeddings, and Pinecone Vector Database to store and retrieve these embeddings efficiently.

By the end of this guide, you'll be able to input a query about a natural phenomenon and get a list of relevant article links as search results.

Here's what you need to get started:

## Prerequisites 📚

- Python 3.7
- A Pinecone account (for creating and managing the vector database)
- A Huggingface account (for accessing the sentence-transformers model)
- .env file with your Pinecone and/or Huggingface credentials ([Pinecone Instructions](https://www.pinecone.io/start/) / [Huggingface Instructions](https://huggingface.co/transformers/installation.html))

## Setting Up Your Workspace 💻

1. **Install the Necessary Libraries**: Make sure to have the following Python libraries installed: `Pandas`, `Transformers`, `Featureform` 

```
pip install pandas
```

```
pip install sentence-transformers
```

```
pip install featureform
```

2. **Setup Your Credentials**: Store your Pinecone and Huggingface credentials in your .env file. Make sure to add the .env file to your .gitignore to keep your credentials private.

```
PINECONE_PROJECT_ID=
PINECONE_ENVIRONMENT=
PINECONE_API_KEY=
HUGGING_FACE_TOKEN=
```

## Getting Started with the Project 🚀

1. **Register the Newscatcher Data as Source**: With Featureform, register the Newscatcher dataset as your data source.

```python
import pandas as pd

df = pd.read_csv("labelled_newscatcher_dataset.csv", sep=";")
df.to_csv("newscatcher_dataset.csv", index=False)
)
```

```python
import featureform as ff
from featureform import local
import dotenv
import os

dotenv.load_dotenv(".env")

pinecone = ff.register_pinecone(
    name="pinecone4",
    project_id=os.getenv("PINECONE_PROJECT_ID", ""),
    environment=os.getenv("PINECONE_ENVIRONMENT", ""),
    api_key=os.getenv("PINECONE_API_KEY", ""),
)

news = local.register_file(
    name="news",
    description="108,774 news articles labelled with 8 topics (balanced)",
    path="ucb-hackathon/newscatcher_dataset.csv",
)
```



### Create an instance of the Featureform client and apply the changes
We'll be using client after each step to apply our changes and checkpoint our work.

```python
from featureform import Client

client = Client(local=True)

client.apply()
```
Use the CLI command to check

```
!featureform list sources --local
```

2. **Register Transformations**
Given the size of the Newscatcher dataset, we'll limit the context we'll create embeddings for to only science-related articles. Once we've filtered by the topic, we'll use [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) to create embeddings of the article titles so we can perform searches on them later.

```python
@local.df_transformation(inputs=[news])
def vectorize_science_news(news_df):
    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer("all-MiniLM-L6-v2")

    science_news = news_df[news_df["topic"] == "SCIENCE"]

    embeddings = model.encode(science_news["title"].tolist())

    science_news["title_embedding"] = embeddings.tolist()

    print(science_news)

    return science_news
```

```python
client.apply()
```

3. **Register Entity and Feature**
We'll now register an entity and a feature, which will kick off the materialization process.

NOTE: This may take some time to complete. See the progress bar for status.

```python
@ff.entity
class NewsDomain:
    science_news = ff.Embedding(
        vectorize_science_news[["link", "title_embedding"]],
        dims=384,
        vector_db=pinecone,
    )

```

```python
client.apply()
```

4. **Register On-Demand Feature**

We'll want to query the embeddings we created, and we can do so using Featureform's on-demand feature decorator. This creates a feature that's calculated on the client at serving time.

```python
@ff.ondemand_feature(name="science_news_search")
def search_science_news(serving_client, params, entity):
    from sentence_transformers import SentenceTransformer

    model = SentenceTransformer("all-MiniLM-L6-v2")
    search_vector = model.encode(params[0])
    feature, variant = params[1]

    return serving_client.nearest(feature, variant, search_vector.tolist(), k=2)

```

```python
client.apply()
```

 5. **Serve Features for Semantic Search**
Now we'll query the vector database via our on-demand feature.

 ```python
 query = "asteroids over England"
embedding_feature_variant = ("science_news", "quizzical_goldstine")
ondemand_feature_variant = ("science_news_search", "quizzical_goldstine")

features = client.features(
    [(ondemand_feature_variant)],
    {"link": ""},
    params=[query, embedding_feature_variant],
)

df = features[0]
results = "\n".join(
    [
        url
        for url in df[
            f"{ondemand_feature_variant[0]}.{ondemand_feature_variant[1]}"
        ].values[0]
    ]
)

print(f"SEARCH RESULTS:\n{results}")

```

```python
client.apply()
```
