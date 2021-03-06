# Information Retrieval
Given a *query*, we will search a large collection for matching documents. 


This articles focuses on the case where *query* is a search query or a question and the documents are paragraphs answering the search query / question. Also have look at [semantic search](../semantic-search/README.md) for the case where query and documents are of the same size.

## Retrieval Pipeline

A pipeline for information retrieval / question answering retrieval that works well is the following. All components are provided and explained in this article:

![InformationRetrieval](https://raw.githubusercontent.com/UKPLab/sentence-transformers/master/docs/img/InformationRetrieval.png)

Given a search query, we first use a **retrieval system** that retrieves a large list of e.g. 100 possible hits which are potentially relevant for the query. For the retrieval, we can use either lexical search, e.g. with ElasticSearch, or we can use dense retrieval with a bi-encoder. 

However, the retrieval system might retrieve documents that are not that relevant for the search query. Hence, in a second stage, we use a **re-ranker** based on a **cross-encoder** that scores the relevancy of all candidates for the given search query. 

The output will be a ranked list of hits we can present to the user.

## Retrieval: Bi-Encoder
For the retrieval of the candidate set, we can either use lexical search, e.g. with [ElasticSearch](https://www.elastic.co/elasticsearch/), or we can use a bi-encoder which is implemented in this repository.

Lexical search looks for literal matches of the query words in your document collection. It will not recognize synonyms, acronyms or spelling variations. In contrast, semantic search (or dense retrieval) encodes the search query into vector space and retrieves the document embeddings that are close in vector space. 

![SemanticSearch](https://raw.githubusercontent.com/UKPLab/sentence-transformers/master/docs/img/SemanticSearch.png)

Semantic search overcomes the short comings of lexical search and can recognize synonym and acronyms. Have a look at the [semantic search article](../semantic-search/README.md)  for different options to implement semantic search.


## Re-Ranker: Cross-Encoder

The retriever has to be efficient for large document collections with millions of entries. However, it might return irrelevant candidates.

A re-ranker based on a Cross-Encoder can substantially improve the final results for the user. The query and a possible document is passed simultaneously to transformer network, which then outputs a single score between 0 and 1 indicating how relevant the document is for the given query. 

![CrossEncoder](https://raw.githubusercontent.com/UKPLab/sentence-transformers/master/docs/img/CrossEncoder.png)

The advantage of Cross-Encoders is the higher performance, as they perform attention across the query and the document. 

Scoring thousands or millions of (query, document)-pairs would be rather slow. Hence, we use the retriever to create a set of e.g. 100 possible candidates which are then re-ranked by the Cross-Encoder.

## Example Scripts

* **[qa_retrieval_simple_wikipedia.py](qa_retrieval_simple_wikipedia.py)**: This script uses the smaller [Simple English Wikipedia](https://simple.wikipedia.org/wiki/Main_Page) as document collection to provide answers to user questions / search queries. First, we split all Wikipedia articles into paragraphs and encode them with a bi-encoder. If a new query / question is entered, it is encoded by the same bi-encoder and the paragraphs with the highest cosine-similarity are retrieved (see [semantic search](../semantic-search/README.md)). Next, the retrieved candidates are scored by a Cross-Encoder re-ranker and the 5 passages with the highest score from the Cross-Encoder are presented to the user.
- **[in_document_search_crossencoder.py](in_document_search_crossencoder.py):** If have only have a small set of paragraphs, we don't the retrieval stage. This is for example the case if you want to perform search within a single document. In this example, take the Wikipedia article about Europe and split it into paragraphs. Then, the search query / question and all paragraphs are scored using the Cross-Encoder re-ranker. The most relevant passages for the query are returned.


## Pre-trained Bi-Encoders (Retrieval)

The bi-encoder produces embeddings independently for your paragraphs and for your search queries. You can use it like this:

```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('model_name')

docs = ["My first paragraph. That contains information", "Python is a programming language."]
document_embeddings = model.encode(docs)

query = "What is Python?"
query_embedding = model.encode(query)
```

For more details how to compare the embeddings, see [semantic search](../semantic-search/README.md).

We provide pre-trained models based on:
- **MS MARCO:** 500k real user queries from Bing search engine. See [MS MARCO models](https://www.sbert.net/docs/pretrained-models/msmarco-v2.html) 

## Pre-trained Cross-Encoders (Re-ranker)

Pre-trained models can be used like this:
```python
from sentence_transformers import CrossEncoder
model = CrossEncoder('model_name', max_length=512)
scores = model.predict([('Query', 'Paragraph1'), ('Query', 'Paragraph2') , ('Query', 'Paragraph3')])
```

In the following table, we provide various pre-trained Cross-Encoders together with their performance on the [TREC Deep Learning 2019](https://microsoft.github.io/TREC-2019-Deep-Learning/) and the [MS Marco Passage Reranking](https://github.com/microsoft/MSMARCO-Passage-Ranking/) dataset. 


| Model-Name        | NDCG@10 (TREC DL 19) | MRR@10 (MS Marco Dev)  | Docs / Sec (BertTokenizerFast) | Docs / Sec |
| ------------- |:-------------| -----| --- | --- |
| cross-encoder/ms-marco-TinyBERT-L-2  | 67.43 | 30.15  | 9000 | 780
| cross-encoder/ms-marco-TinyBERT-L-4  | 68.09 | 34.50  | 2900 | 760
| cross-encoder/ms-marco-TinyBERT-L-6 |  69.57 | 36.13  | 680 | 660
| cross-encoder/ms-marco-electra-base | 71.99 | 36.41 | 340 | 340
| *Other models* | | | |
| nboost/pt-tinybert-msmarco | 63.63 | 28.80 | 2900 | 760
| nboost/pt-bert-base-uncased-msmarco | 70.94 | 34.75 | 340 | 340|
| nboost/pt-bert-large-msmarco | 73.36 | 36.48 | 100 | 100 |
| Capreolus/electra-base-msmarco | 71.23 | 36.89 | 340 | 340 |
| amberoad/bert-multilingual-passage-reranking-msmarco | 68.40 | 35.54 | 330 | 330 
 
 Note: Runtime was computed on a V100 GPU. A bottleneck for smaller models is the Python-based tokenizer from Huggingface that was used in Huggingface Version 3. Starting with Huggingface Version 4, fast tokenizer based on Rust is used.
