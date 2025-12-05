# Part 3: Azure Ecosystem Integration for RAG

## 📖 Table of Contents
- [Azure OpenAI for RAG](#azure-openai-for-rag)
- [Claude Models in Azure AI Foundry](#claude-models-in-azure-ai-foundry)
- [Azure AI Foundry IQ](#azure-ai-foundry-iq)
- [Multi-Model Strategy](#multi-model-strategy)
- [Implementation Architecture](#implementation-architecture)
- [Best Practices](#best-practices)

---

## Azure OpenAI for RAG

### Overview

Azure OpenAI Service provides enterprise-grade access to OpenAI's most advanced models, including GPT-4o and text-embedding models, with the security, compliance, and regional availability that enterprises require.

### Key Models for RAG

#### GPT-4o (Generation Model)

**Capabilities:**
- **Context Window**: 128,000 tokens
- **Max Output**: 16,384 tokens
- **Multimodal**: Text, images, audio
- **Best For**: Complex reasoning, creative writing, code generation

**Pricing (Approximate):**
- Input: $2.50 per 1M tokens
- Output: $10.00 per 1M tokens

**When to Use:**
- Complex queries requiring deep reasoning
- Multi-step analysis
- Code generation from documentation
- High-stakes applications

#### GPT-4o-mini (Efficient Generation)

**Capabilities:**
- **Context Window**: 128,000 tokens
- **Faster**: Lower latency than GPT-4o
- **Cost-Effective**: Significantly cheaper
- **Best For**: Simpler queries, high-volume applications

**Pricing (Approximate):**
- Input: $0.15 per 1M tokens
- Output: $0.60 per 1M tokens

**When to Use:**
- Straightforward Q&A
- High-volume requests
- Budget-conscious deployments
- Simple summarization tasks

#### text-embedding-3-large (Embedding Model)

**Capabilities:**
- **Dimensions**: 3072 (configurable down to 256)
- **Performance**: Best-in-class retrieval quality
- **Multilingual**: Excellent cross-language support
- **Best For**: High-accuracy RAG applications

**Pricing:**
- $0.13 per 1M tokens

**When to Use:**
- Maximum retrieval accuracy needed
- Multilingual content
- Complex domain knowledge
- Production RAG systems

#### text-embedding-3-small (Efficient Embedding)

**Capabilities:**
- **Dimensions**: 1536
- **Cost-Effective**: Lower price point
- **Good Performance**: Solid retrieval quality
- **Best For**: Budget-conscious deployments

**Pricing:**
- $0.02 per 1M tokens

**When to Use:**
- Cost optimization priority
- Large-scale indexing
- Prototyping and development
- Acceptable accuracy trade-off

### Azure OpenAI RAG Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     AZURE OPENAI RAG STACK                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [User Query] ──────────────────────────────────────────┐  │
│       │                                                   │  │
│       ▼                                                   │  │
│  ┌──────────────────┐                                    │  │
│  │ Query Embedding  │ text-embedding-3-large             │  │
│  └────────┬─────────┘                                    │  │
│           │                                               │  │
│           ▼                                               │  │
│  ┌──────────────────┐                                    │  │
│  │  Azure AI Search │ ← Vector + Hybrid Search           │  │
│  │  Vector Index    │                                    │  │
│  └────────┬─────────┘                                    │  │
│           │                                               │  │
│           ▼                                               │  │
│  ┌──────────────────┐                                    │  │
│  │ Top K Chunks     │ ← Ranked by relevance              │  │
│  │ + Metadata       │                                    │  │
│  └────────┬─────────┘                                    │  │
│           │                                               │  │
│           ▼                                               │  │
│  ┌──────────────────┐                                    │  │
│  │ Prompt Assembly  │ ← Context + Query                  │  │
│  └────────┬─────────┘                                    │  │
│           │                                               │  │
│           ▼                                               │  │
│  ┌──────────────────┐                                    │  │
│  │   GPT-4o Model   │ ← Generate Response                │  │
│  └────────┬─────────┘                                    │  │
│           │                                               │  │
│           ▼                                               │  │
│  [Answer with Sources] ─────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Setup and Configuration

#### 1. Azure OpenAI Resource Setup

```python
# Azure OpenAI Configuration
import os
from openai import AzureOpenAI

# Initialize Azure OpenAI client
client = AzureOpenAI(
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    api_version="2024-08-01-preview",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT")
)

# Alternative: Using Azure Active Directory authentication (recommended)
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
token_provider = credential.get_token("https://cognitiveservices.azure.com/.default")

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    api_version="2024-08-01-preview",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT")
)
```

#### 2. Embedding Generation

```python
# Generate embeddings for documents
def generate_embeddings(texts, model="text-embedding-3-large"):
    """
    Generate embeddings for a list of texts

    Args:
        texts: List of text strings to embed
        model: Azure OpenAI embedding model deployment name

    Returns:
        List of embedding vectors
    """
    response = client.embeddings.create(
        model=model,
        input=texts
    )

    embeddings = [item.embedding for item in response.data]
    return embeddings

# Usage
documents = [
    "Azure provides cloud computing services",
    "Machine learning enables predictive analytics",
    "RAG combines retrieval with generation"
]

embeddings = generate_embeddings(documents)
print(f"Generated {len(embeddings)} embeddings")
print(f"Embedding dimension: {len(embeddings[0])}")
```

#### 3. Azure AI Search Integration

```python
# Azure AI Search for vector storage and retrieval
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex,
    SearchField,
    SearchFieldDataType,
    VectorSearch,
    VectorSearchProfile,
    HnswAlgorithmConfiguration,
)
from azure.core.credentials import AzureKeyCredential

# Initialize search client
search_endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
search_key = os.getenv("AZURE_SEARCH_KEY")
index_name = "rag-knowledge-base"

credential = AzureKeyCredential(search_key)
index_client = SearchIndexClient(search_endpoint, credential)

# Define search index with vector field
def create_search_index(index_name, embedding_dimensions=3072):
    """
    Create Azure AI Search index for RAG

    Args:
        index_name: Name of the search index
        embedding_dimensions: Size of embedding vectors

    Returns:
        Created index
    """
    fields = [
        SearchField(
            name="id",
            type=SearchFieldDataType.String,
            key=True,
            filterable=True
        ),
        SearchField(
            name="content",
            type=SearchFieldDataType.String,
            searchable=True
        ),
        SearchField(
            name="content_vector",
            type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            searchable=True,
            vector_search_dimensions=embedding_dimensions,
            vector_search_profile_name="my-vector-profile"
        ),
        SearchField(
            name="source",
            type=SearchFieldDataType.String,
            filterable=True,
            facetable=True
        ),
        SearchField(
            name="category",
            type=SearchFieldDataType.String,
            filterable=True,
            facetable=True
        ),
        SearchField(
            name="created_date",
            type=SearchFieldDataType.DateTimeOffset,
            filterable=True,
            sortable=True
        )
    ]

    # Configure vector search
    vector_search = VectorSearch(
        profiles=[
            VectorSearchProfile(
                name="my-vector-profile",
                algorithm_configuration_name="my-hnsw-config"
            )
        ],
        algorithms=[
            HnswAlgorithmConfiguration(
                name="my-hnsw-config",
                parameters={
                    "m": 4,
                    "efConstruction": 400,
                    "efSearch": 500,
                    "metric": "cosine"
                }
            )
        ]
    )

    index = SearchIndex(
        name=index_name,
        fields=fields,
        vector_search=vector_search
    )

    result = index_client.create_or_update_index(index)
    return result

# Create the index
index = create_search_index(index_name)
print(f"Created index: {index.name}")
```

#### 4. Document Indexing

```python
# Index documents with embeddings
from azure.search.documents import SearchClient
import uuid
from datetime import datetime

def index_documents(documents, index_name):
    """
    Index documents into Azure AI Search

    Args:
        documents: List of document dictionaries
        index_name: Name of the search index

    Returns:
        Indexing result
    """
    search_client = SearchClient(
        search_endpoint,
        index_name,
        AzureKeyCredential(search_key)
    )

    # Prepare documents for indexing
    indexed_docs = []
    for doc in documents:
        # Generate embedding for content
        embedding = generate_embeddings([doc["content"]])[0]

        indexed_doc = {
            "id": str(uuid.uuid4()),
            "content": doc["content"],
            "content_vector": embedding,
            "source": doc.get("source", "unknown"),
            "category": doc.get("category", "general"),
            "created_date": datetime.now().isoformat()
        }
        indexed_docs.append(indexed_doc)

    # Upload documents
    result = search_client.upload_documents(documents=indexed_docs)
    return result

# Usage
sample_docs = [
    {
        "content": "Azure OpenAI provides enterprise access to GPT-4o with enhanced security features.",
        "source": "azure_docs",
        "category": "cloud_services"
    },
    {
        "content": "RAG systems combine retrieval mechanisms with generative models for improved accuracy.",
        "source": "technical_guide",
        "category": "ai_architecture"
    }
]

result = index_documents(sample_docs, index_name)
print(f"Indexed {len(result)} documents")
```

#### 5. Hybrid Search (Vector + Keyword)

```python
# Perform hybrid search combining vector and keyword search
from azure.search.documents.models import VectorizedQuery

def hybrid_search(query, index_name, top_k=5):
    """
    Perform hybrid search (vector + keyword)

    Args:
        query: Search query string
        index_name: Name of the search index
        top_k: Number of results to return

    Returns:
        Search results
    """
    search_client = SearchClient(
        search_endpoint,
        index_name,
        AzureKeyCredential(search_key)
    )

    # Generate query embedding
    query_embedding = generate_embeddings([query])[0]

    # Create vector query
    vector_query = VectorizedQuery(
        vector=query_embedding,
        k_nearest_neighbors=top_k,
        fields="content_vector"
    )

    # Perform hybrid search
    results = search_client.search(
        search_text=query,  # Keyword search
        vector_queries=[vector_query],  # Vector search
        select=["content", "source", "category"],
        top=top_k
    )

    # Process results
    retrieved_docs = []
    for result in results:
        retrieved_docs.append({
            "content": result["content"],
            "source": result["source"],
            "category": result["category"],
            "score": result["@search.score"]
        })

    return retrieved_docs

# Usage
query = "How does Azure OpenAI enhance security?"
results = hybrid_search(query, index_name, top_k=3)

for i, doc in enumerate(results):
    print(f"\nResult {i+1} (Score: {doc['score']:.4f}):")
    print(f"Content: {doc['content']}")
    print(f"Source: {doc['source']}")
```

#### 6. RAG Response Generation

```python
# Generate RAG response using retrieved context
def generate_rag_response(query, index_name, model="gpt-4o"):
    """
    Generate RAG response with source attribution

    Args:
        query: User query
        index_name: Search index name
        model: Azure OpenAI model deployment name

    Returns:
        Dictionary with answer and sources
    """
    # Retrieve relevant documents
    retrieved_docs = hybrid_search(query, index_name, top_k=3)

    # Build context from retrieved documents
    context = "\n\n".join([
        f"[Source: {doc['source']}]\n{doc['content']}"
        for doc in retrieved_docs
    ])

    # Create prompt with context
    messages = [
        {
            "role": "system",
            "content": """You are a helpful assistant that answers questions based on the provided context.
Always cite your sources using [Source: source_name] notation.
If the context doesn't contain enough information to answer the question, say so clearly."""
        },
        {
            "role": "user",
            "content": f"""Context:
{context}

Question: {query}

Please provide a comprehensive answer based on the context above."""
        }
    ]

    # Generate response
    response = client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=0.3,  # Lower temperature for factual responses
        max_tokens=1000
    )

    answer = response.choices[0].message.content

    return {
        "answer": answer,
        "sources": [doc["source"] for doc in retrieved_docs],
        "retrieved_docs": retrieved_docs
    }

# Usage
query = "What are the benefits of using Azure OpenAI for RAG?"
result = generate_rag_response(query, index_name)

print(f"Question: {query}\n")
print(f"Answer: {result['answer']}\n")
print(f"Sources: {', '.join(result['sources'])}")
```

### Advanced Features

#### Semantic Ranking

Azure AI Search provides L2 (learning to rank) semantic ranking for improved results:

```python
# Enable semantic ranking
from azure.search.documents.models import QueryType, QueryCaptionType, QueryAnswerType

def semantic_hybrid_search(query, index_name, top_k=5):
    """
    Hybrid search with semantic ranking
    """
    search_client = SearchClient(
        search_endpoint,
        index_name,
        AzureKeyCredential(search_key)
    )

    query_embedding = generate_embeddings([query])[0]
    vector_query = VectorizedQuery(
        vector=query_embedding,
        k_nearest_neighbors=50,  # Get more candidates for reranking
        fields="content_vector"
    )

    results = search_client.search(
        search_text=query,
        vector_queries=[vector_query],
        query_type=QueryType.SEMANTIC,
        semantic_configuration_name="my-semantic-config",
        query_caption=QueryCaptionType.EXTRACTIVE,
        query_answer=QueryAnswerType.EXTRACTIVE,
        top=top_k
    )

    return results
```

#### Filtering by Metadata

```python
# Search with filters
def filtered_search(query, index_name, category=None, date_from=None):
    """
    Hybrid search with metadata filtering
    """
    search_client = SearchClient(
        search_endpoint,
        index_name,
        AzureKeyCredential(search_key)
    )

    # Build filter expression
    filters = []
    if category:
        filters.append(f"category eq '{category}'")
    if date_from:
        filters.append(f"created_date ge {date_from}")

    filter_expression = " and ".join(filters) if filters else None

    query_embedding = generate_embeddings([query])[0]
    vector_query = VectorizedQuery(
        vector=query_embedding,
        k_nearest_neighbors=5,
        fields="content_vector"
    )

    results = search_client.search(
        search_text=query,
        vector_queries=[vector_query],
        filter=filter_expression,
        top=5
    )

    return results

# Usage
results = filtered_search(
    query="Azure OpenAI features",
    index_name=index_name,
    category="cloud_services"
)
```

### Best Practices for Azure OpenAI RAG

1. **Use Managed Identity**: Prefer Azure AD authentication over API keys
2. **Enable Semantic Ranking**: Improves retrieval quality significantly
3. **Implement Caching**: Cache embeddings and frequent queries
4. **Monitor Costs**: Track token usage and optimize accordingly
5. **Use Hybrid Search**: Combine vector and keyword search for best results
6. **Chunk Appropriately**: Align chunk size with embedding model capabilities
7. **Add Metadata**: Enable filtering and better source attribution
8. **Handle Errors**: Implement retry logic and fallback strategies

---

## Claude Models in Azure AI Foundry

### Overview

Azure AI Foundry (formerly Azure AI Studio) now offers Anthropic's Claude models, providing customers access to both Claude and GPT models on a single platform. This is a unique advantage of the Azure ecosystem.

### Available Claude Models (2025)

#### Claude Sonnet 4.5

**Capabilities:**
- **Context Window**: 200,000 tokens
- **Best For**: Balanced performance across complex tasks
- **Strengths**: Code generation, analysis, creative writing
- **Speed**: Fast inference

**Pricing (Approximate):**
- Input: $3.00 per 1M tokens
- Output: $15.00 per 1M tokens

**When to Use for RAG:**
- Large context requirements (long documents)
- Complex reasoning over retrieved content
- Code-heavy documentation
- Multi-turn conversations

#### Claude Haiku 4.5

**Capabilities:**
- **Context Window**: 200,000 tokens
- **Best For**: High-speed, cost-effective processing
- **Strengths**: Fast responses, efficient processing
- **Speed**: Fastest in Claude family

**Pricing (Approximate):**
- Input: $0.25 per 1M tokens
- Output: $1.25 per 1M tokens

**When to Use for RAG:**
- High-volume query processing
- Simple Q&A scenarios
- Cost optimization priority
- Real-time applications

#### Claude Opus 4.1

**Capabilities:**
- **Context Window**: 200,000 tokens
- **Best For**: Maximum capability and accuracy
- **Strengths**: Complex analysis, research, expert-level tasks
- **Speed**: Slower but most capable

**Pricing (Approximate):**
- Input: $15.00 per 1M tokens
- Output: $75.00 per 1M tokens

**When to Use for RAG:**
- Critical accuracy requirements
- Complex research queries
- Expert-level analysis
- When cost is secondary to quality

### Claude RAG Implementation

```python
# Claude on Azure AI Foundry
import os
from anthropic import AnthropicAzure

# Initialize Claude client
claude_client = AnthropicAzure(
    api_key=os.getenv("AZURE_ANTHROPIC_API_KEY"),
    azure_endpoint=os.getenv("AZURE_ANTHROPIC_ENDPOINT")
)

def generate_claude_rag_response(query, retrieved_docs, model="claude-sonnet-4-5"):
    """
    Generate RAG response using Claude on Azure

    Args:
        query: User query
        retrieved_docs: List of retrieved document dictionaries
        model: Claude model to use

    Returns:
        Claude's response with citations
    """
    # Build context from retrieved documents
    context_parts = []
    for i, doc in enumerate(retrieved_docs):
        context_parts.append(f"""
Document {i+1} [Source: {doc['source']}]:
{doc['content']}
""")

    context = "\n".join(context_parts)

    # Create prompt for Claude
    prompt = f"""You are a helpful assistant answering questions based on provided documents.

Retrieved Documents:
{context}

User Question: {query}

Please provide a comprehensive answer based on the documents above. Always cite which document(s) you're referencing using the format [Document N]."""

    # Generate response with Claude
    response = claude_client.messages.create(
        model=model,
        max_tokens=2000,
        temperature=0.3,
        messages=[
            {
                "role": "user",
                "content": prompt
            }
        ]
    )

    return response.content[0].text

# Usage with Azure AI Search results
query = "How does RAG improve AI accuracy?"
search_results = hybrid_search(query, index_name, top_k=3)
answer = generate_claude_rag_response(query, search_results, model="claude-sonnet-4-5")

print(f"Claude's Answer:\n{answer}")
```

### Claude Advantages for RAG

**200K Context Window:**
- Can process entire documents without chunking
- Better for "Long RAG" scenarios
- Fewer retrieval calls needed

**Superior Code Understanding:**
- Excellent for technical documentation RAG
- Better code generation from docs
- Strong at analyzing code examples

**Multilingual Capabilities:**
- Strong performance across languages
- Good for international knowledge bases

**Citation Quality:**
- Naturally provides better source attribution
- More careful about factual claims

---

## Azure AI Foundry IQ

### Overview

**Azure AI Foundry IQ** represents the next generation of RAG, reimagining retrieval as a dynamic reasoning process rather than a one-time lookup. It's powered by Azure AI Search and provides agentic RAG capabilities.

### Key Features

#### 1. Agentic Retrieval

Traditional RAG: Query → Retrieve → Generate

Agentic RAG with Foundry IQ:
```
Query → Agent analyzes → Breaks into sub-queries →
Multiple retrievals → Reflection → Iterative refinement →
Final answer
```

**Benefits:**
- Handles complex, multi-hop questions
- Iteratively improves retrieval
- Self-corrects based on intermediate results

#### 2. Centralized Grounding API

**Single Entry Point** for multiple data sources:
- SharePoint documents
- OneDrive files
- Internal databases
- External APIs
- Web search results

**Simplified Orchestration:**
```python
# Foundry IQ unifies access to multiple sources
from azure.ai.foundry import FoundryIQClient

foundry_client = FoundryIQClient(
    endpoint=os.getenv("FOUNDRY_ENDPOINT"),
    credential=DefaultAzureCredential()
)

# Query across multiple sources with one API call
response = foundry_client.grounded_query(
    query="What were our Q3 cloud costs and how do they compare to industry benchmarks?",
    sources=[
        "sharepoint://finance/reports",
        "database://financial_data",
        "web://industry_reports"
    ],
    enable_agentic_retrieval=True,
    respect_permissions=True
)

print(response.answer)
print(f"Sources consulted: {response.sources}")
print(f"Reasoning trace: {response.reasoning_steps}")
```

#### 3. Permission-Aware Retrieval

Foundry IQ respects user access permissions:
- Queries only return content user can access
- Maintains data classification boundaries
- Integrates with Azure AD permissions

```python
# Permission-aware query
response = foundry_client.grounded_query(
    query="Show me confidential project updates",
    user_principal=user_identity,  # Current user's identity
    respect_permissions=True  # Only show what user can access
)
```

#### 4. Multi-Source Selection

Intelligently decides which sources to query:

```python
# Automatic source selection
response = foundry_client.grounded_query(
    query="Compare our pricing with competitors",
    available_sources={
        "internal_pricing": {
            "type": "database",
            "contains": "Our product pricing"
        },
        "market_research": {
            "type": "sharepoint",
            "contains": "Competitor analysis"
        },
        "web_search": {
            "type": "bing",
            "contains": "Public competitor pricing"
        }
    },
    auto_select_sources=True
)

# Foundry IQ automatically queries relevant sources
print(f"Sources used: {response.sources_consulted}")
```

#### 5. Iterative Retrieval with Reflection

Agentic RAG can retrieve multiple times:

```
Initial Query: "What caused our Q3 sales decline?"
    ↓
Retrieval 1: Gets Q3 sales figures
    ↓
Agent Reflection: "Need to compare with Q2 and external factors"
    ↓
Retrieval 2: Gets Q2 data and market conditions
    ↓
Agent Reflection: "Should check product-specific trends"
    ↓
Retrieval 3: Gets product breakdown
    ↓
Final Answer: Comprehensive analysis with all context
```

### Implementation Example

```python
# Complete Foundry IQ RAG Example
from azure.ai.foundry import FoundryIQClient, SourceConfig
from azure.identity import DefaultAzureCredential

class FoundryRAGSystem:
    def __init__(self, endpoint):
        self.client = FoundryIQClient(
            endpoint=endpoint,
            credential=DefaultAzureCredential()
        )

    def configure_sources(self):
        """Configure data sources for RAG"""
        sources = [
            SourceConfig(
                name="company_docs",
                type="sharepoint",
                connection_string=os.getenv("SHAREPOINT_CONNECTION"),
                description="Internal company documentation"
            ),
            SourceConfig(
                name="knowledge_base",
                type="azure_search",
                connection_string=os.getenv("SEARCH_CONNECTION"),
                index_name="kb-index",
                description="Curated knowledge base"
            ),
            SourceConfig(
                name="web_search",
                type="bing",
                api_key=os.getenv("BING_API_KEY"),
                description="Web search for current information"
            )
        ]

        self.client.register_sources(sources)

    def query_with_agentic_rag(self, query, user_id=None, max_iterations=3):
        """
        Perform agentic RAG query with Foundry IQ

        Args:
            query: User question
            user_id: User principal for permission checks
            max_iterations: Maximum retrieval iterations

        Returns:
            Response with answer, sources, and reasoning
        """
        response = self.client.grounded_query(
            query=query,
            user_principal=user_id,
            agentic_mode=True,
            max_retrieval_iterations=max_iterations,
            enable_reflection=True,
            enable_source_selection=True,
            respect_permissions=True,
            response_format="detailed"  # Include reasoning trace
        )

        return {
            "answer": response.answer,
            "sources": response.sources,
            "reasoning_steps": response.reasoning_trace,
            "confidence": response.confidence_score,
            "iterations_used": response.retrieval_count
        }

# Usage
rag_system = FoundryRAGSystem(endpoint=os.getenv("FOUNDRY_ENDPOINT"))
rag_system.configure_sources()

result = rag_system.query_with_agentic_rag(
    query="How should we optimize our cloud infrastructure costs based on current usage and industry best practices?",
    user_id="user@company.com"
)

print(f"Answer: {result['answer']}\n")
print(f"Confidence: {result['confidence']}\n")
print(f"Retrieval iterations: {result['iterations_used']}\n")
print(f"Reasoning steps:")
for i, step in enumerate(result['reasoning_steps']):
    print(f"  {i+1}. {step}")
print(f"\nSources: {', '.join(result['sources'])}")
```

### Foundry IQ vs. Traditional RAG

| Aspect | Traditional RAG | Foundry IQ |
|--------|-----------------|------------|
| **Retrieval** | One-time lookup | Iterative, multi-round |
| **Query Handling** | Single query | Breaks into sub-queries |
| **Sources** | One index | Multiple, auto-selected |
| **Reasoning** | None | Built-in reflection |
| **Permissions** | Manual handling | Automatic enforcement |
| **Complexity** | Simple queries | Complex, multi-hop questions |

---

## Multi-Model Strategy

### Choosing the Right Model

Different RAG stages can use different models for optimal performance and cost:

#### Strategy 1: Model per Stage

```
Embedding: text-embedding-3-small (cost-effective)
    ↓
Retrieval: Azure AI Search
    ↓
Simple Queries → GPT-4o-mini (fast, cheap)
Complex Queries → Claude Sonnet 4.5 (capable, good value)
Critical Queries → GPT-4o or Claude Opus (best quality)
```

#### Strategy 2: Fallback Approach

```
Try GPT-4o-mini first
    ↓
If confidence < threshold → Retry with Claude Sonnet
    ↓
If still uncertain → Escalate to GPT-4o or Opus
```

#### Strategy 3: Task-Specific Routing

```python
def route_to_best_model(query, retrieved_docs):
    """
    Route query to optimal model based on characteristics
    """
    # Analyze query complexity
    is_code_heavy = "code" in query.lower() or any("```" in doc['content'] for doc in retrieved_docs)
    is_long_context = sum(len(doc['content']) for doc in retrieved_docs) > 10000
    is_complex = len(query.split()) > 20 or "?" in query multiple times

    # Route to appropriate model
    if is_code_heavy:
        return "claude-sonnet-4-5"  # Claude excels at code
    elif is_long_context:
        return "claude-haiku-4-5"  # Large context window
    elif is_complex:
        return "gpt-4o"  # Complex reasoning
    else:
        return "gpt-4o-mini"  # Simple, fast
```

### Cost Optimization Matrix

| Use Case | Volume | Model Choice | Est. Cost per 1K queries |
|----------|--------|--------------|-------------------------|
| Simple FAQ | High | GPT-4o-mini | $2-5 |
| Technical Docs | Medium | Claude Sonnet 4.5 | $15-25 |
| Research Analysis | Low | GPT-4o or Claude Opus | $50-100 |
| Code Q&A | Medium | Claude Sonnet 4.5 | $15-30 |

### Implementation: Multi-Model RAG

```python
class MultiModelRAG:
    def __init__(self):
        self.gpt_client = AzureOpenAI(...)
        self.claude_client = AnthropicAzure(...)
        self.search_client = SearchClient(...)

    def query(self, question, strategy="auto"):
        """
        Multi-model RAG with intelligent routing
        """
        # Retrieve relevant documents
        docs = self.hybrid_search(question)

        # Select model based on strategy
        if strategy == "auto":
            model, provider = self.select_model(question, docs)
        else:
            model, provider = strategy.split(":")

        # Generate response with selected model
        if provider == "azure_openai":
            return self.generate_with_gpt(question, docs, model)
        elif provider == "claude":
            return self.generate_with_claude(question, docs, model)

    def select_model(self, question, docs):
        """Auto-select best model"""
        context_length = sum(len(d['content']) for d in docs)

        if context_length > 50000:
            return "claude-haiku-4-5", "claude"
        elif "code" in question.lower():
            return "claude-sonnet-4-5", "claude"
        elif len(question.split()) < 10:
            return "gpt-4o-mini", "azure_openai"
        else:
            return "gpt-4o", "azure_openai"
```

---

## Implementation Architecture

### Complete Azure RAG Stack

```
┌──────────────────────────────────────────────────────────────┐
│                    COMPLETE AZURE RAG SYSTEM                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              DATA SOURCES                                ││
│  │  • SharePoint   • Databases   • File Storage            ││
│  │  • APIs         • Web         • OneDrive                ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │         DOCUMENT PROCESSING & CHUNKING                   ││
│  │  • LangChain Text Splitters                             ││
│  │  • Custom Chunking Logic                                ││
│  │  • Metadata Extraction                                  ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │            EMBEDDING GENERATION                          ││
│  │  • Azure OpenAI text-embedding-3-large                  ││
│  │  • Batch processing for efficiency                      ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │            VECTOR STORAGE                                ││
│  │  • Azure AI Search (with hybrid search)                 ││
│  │  • Semantic ranking enabled                             ││
│  │  • Metadata filtering support                           ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │         QUERY PROCESSING                                 ││
│  │  • Query embedding generation                           ││
│  │  • Hybrid search (vector + keyword)                     ││
│  │  • Semantic reranking                                   ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │      INTELLIGENT MODEL ROUTING                           ││
│  │  ┌──────────┬──────────────┬──────────────┐           ││
│  │  │ Simple   │   Code/Tech  │   Complex    │           ││
│  │  │   ↓      │      ↓       │      ↓       │           ││
│  │  │GPT-4o-   │   Claude     │   GPT-4o     │           ││
│  │  │  mini    │  Sonnet 4.5  │  /Opus 4.1   │           ││
│  │  └──────────┴──────────────┴──────────────┘           ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │         RESPONSE GENERATION                              ││
│  │  • Context assembly                                     ││
│  │  • Prompt engineering                                   ││
│  │  • Source attribution                                   ││
│  └────────────────────┬────────────────────────────────────┘│
│                       │                                      │
│  ┌────────────────────▼────────────────────────────────────┐│
│  │      MONITORING & OPTIMIZATION                           ││
│  │  • Azure Monitor     • Cost tracking                    ││
│  │  • Performance metrics • Quality evaluation             ││
│  └─────────────────────────────────────────────────────────┘│
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### 1. Security & Compliance

✅ **Use Managed Identity**: Azure AD authentication instead of API keys
✅ **Enable Private Endpoints**: Keep data within Azure network
✅ **Implement RBAC**: Role-based access control for all resources
✅ **Enable Logging**: Azure Monitor for audit trails
✅ **Data Residency**: Choose appropriate Azure regions
✅ **Encryption**: At-rest and in-transit encryption

### 2. Performance Optimization

✅ **Enable Caching**: Cache embeddings and frequent queries
✅ **Batch Processing**: Generate embeddings in batches
✅ **Async Operations**: Use async calls for better throughput
✅ **Connection Pooling**: Reuse connections to Azure services
✅ **CDN for Static Content**: Azure CDN for frequently accessed docs

### 3. Cost Management

✅ **Right-Size Models**: Use smallest model that meets requirements
✅ **Monitor Token Usage**: Track and optimize token consumption
✅ **Implement Quotas**: Set spending limits
✅ **Cache Aggressively**: Reduce redundant API calls
✅ **Use Cheaper Embeddings**: text-embedding-3-small where appropriate
✅ **Batch Operations**: Group requests to reduce overhead

### 4. Quality Assurance

✅ **Evaluation Framework**: Regular testing of RAG quality
✅ **Human Feedback Loop**: Collect and incorporate user feedback
✅ **A/B Testing**: Test different configurations
✅ **Monitor Metrics**: Track precision, recall, user satisfaction
✅ **Version Control**: Track changes to prompts and configurations

### 5. Scalability

✅ **Auto-Scaling**: Enable auto-scale for Azure resources
✅ **Load Balancing**: Distribute load across deployments
✅ **Queue-Based Processing**: Azure Service Bus for async processing
✅ **Partitioning**: Partition large indexes for better performance
✅ **Global Distribution**: Use multiple regions for worldwide access

---

## Summary

### Azure RAG Ecosystem Advantages

1. **Unified Platform**: GPT and Claude models in one ecosystem
2. **Enterprise-Grade**: Security, compliance, SLAs
3. **Flexible Integration**: Multiple services work together seamlessly
4. **Agentic Capabilities**: Foundry IQ for next-gen RAG
5. **Cost Control**: Fine-grained monitoring and optimization
6. **Global Reach**: Worldwide availability with data residency options

### Quick Decision Guide

| Requirement | Recommended Solution |
|-------------|---------------------|
| Best embedding quality | text-embedding-3-large |
| Cost-effective embeddings | text-embedding-3-small |
| Simple Q&A | GPT-4o-mini |
| Code-heavy docs | Claude Sonnet 4.5 |
| Large context | Claude models (200K tokens) |
| Maximum quality | GPT-4o or Claude Opus 4.1 |
| Complex multi-hop queries | Azure AI Foundry IQ |
| Permission-aware retrieval | Foundry IQ with AD integration |

---

**Next**: Part 4 - Code Examples and Implementation Patterns

