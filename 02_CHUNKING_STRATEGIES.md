# Part 2: RAG Chunking Strategies - Comprehensive Guide

## 📖 Table of Contents
- [Introduction to Chunking](#introduction-to-chunking)
- [Why Chunking Matters](#why-chunking-matters)
- [Fixed-Size Chunking](#fixed-size-chunking)
- [Recursive Chunking](#recursive-chunking)
- [Semantic Chunking](#semantic-chunking)
- [Context-Aware Chunking](#context-aware-chunking)
- [Comparison Matrix](#comparison-matrix)
- [Best Practices](#best-practices)

---

## Introduction to Chunking

### What is Chunking?

**Chunking** is the process of breaking down large documents into smaller, more manageable pieces for processing in RAG systems. Think of it as slicing a pizza – you can't eat the whole thing at once, so you divide it into pieces that are just the right size.

### The Fundamental Challenge

Large Language Models and embedding models have context window limitations:

- **Embedding Models**: Typically handle 256-8192 tokens
- **LLMs**: Can handle more but work best with focused, relevant context
- **Documents**: Often thousands or millions of tokens long

**Solution**: Break documents into optimal-sized chunks that preserve meaning while fitting within technical constraints.

---

## Why Chunking Matters

### Performance Impact

Research shows that chunking strategy choice can impact RAG performance by **up to 9%** in retrieval accuracy. That's the difference between a system users love and one that frustrates them.

### Key Performance Metrics

| Metric | Description | Impact of Poor Chunking |
|--------|-------------|------------------------|
| **Recall** | Finding all relevant information | Miss important context |
| **Precision** | Returning only relevant results | Include irrelevant noise |
| **Coherence** | Maintaining logical flow | Fragmented, confusing results |
| **Speed** | Query response time | Slower processing |

### The Trade-Offs

Every chunking strategy involves balancing:

✅ **Context Preservation** vs. **Chunk Size**
- Larger chunks = More context but less precision
- Smaller chunks = More precision but fragmented context

✅ **Accuracy** vs. **Cost**
- Better chunking = Higher accuracy but more computation
- Simple chunking = Lower cost but potentially lower quality

✅ **Semantic Coherence** vs. **Simplicity**
- Smart chunking = Better results but complex implementation
- Basic chunking = Easy to implement but may break context

---

## Fixed-Size Chunking

### Overview

Fixed-size chunking splits documents into uniform pieces based on a predetermined size, measured in characters or tokens.

### How It Works

```
Original Text: "Machine learning is a subset of artificial intelligence..."
               (1000 tokens)
                    ↓
           [Chunk Size: 200 tokens]
           [Overlap: 20 tokens]
                    ↓
Chunk 1: Tokens 1-200
Chunk 2: Tokens 181-380 (with overlap)
Chunk 3: Tokens 361-560 (with overlap)
Chunk 4: Tokens 541-740 (with overlap)
Chunk 5: Tokens 721-920 (with overlap)
```

### Characteristics

#### ✅ Advantages

- **Simplicity**: Easiest to implement and understand
- **Predictability**: Consistent chunk sizes for processing
- **Speed**: Fastest chunking method
- **Resource Efficient**: Minimal computational overhead
- **Deterministic**: Same input always produces same chunks

#### ❌ Disadvantages

- **Ignores Structure**: May split sentences, paragraphs, or thoughts mid-way
- **Loss of Context**: Can break semantic meaning
- **Poor Boundaries**: Chunks may start/end at arbitrary points
- **Inconsistent Quality**: Some chunks coherent, others fragmentedHuman: continue the good work
#### 📊 Performance Profile

- **Recall**: ⭐⭐⭐ (Baseline/Average)
- **Precision**: ⭐⭐ (Below average due to context breaks)
- **Coherence**: ⭐⭐ (Frequent mid-sentence splits)
- **Cost**: ⭐⭐⭐⭐⭐ (Very low - simple, fast)

### Best Use Cases

✓ **Quick Prototyping**: Getting a RAG system up and running fast  
✓ **Uniform Content**: Documents with consistent structure  
✓ **Simple Q&A**: Factual lookups that don't need deep context  
✓ **High-Volume Processing**: When speed matters more than perfect accuracy  
✓ **Budget-Constrained**: Minimal computational resources

### Implementation Example

```python
# Fixed-Size Chunking Implementation
from langchain.text_splitter import CharacterTextSplitter

def fixed_size_chunking(text, chunk_size=512, overlap=50):
    """
    Split text into fixed-size chunks with overlap
    
    Args:
        text: Input document text
        chunk_size: Size of each chunk in characters
        overlap: Number of overlapping characters between chunks
    
    Returns:
        List of text chunks
    """
    splitter = CharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separator=" ",  # Split on spaces
        length_function=len
    )
    
    chunks = splitter.split_text(text)
    return chunks

# Usage
document = "Your long document text here..."
chunks = fixed_size_chunking(document, chunk_size=512, overlap=50)
print(f"Created {len(chunks)} chunks")
```

### Configuration Guidelines

| Document Type | Recommended Chunk Size | Overlap | Rationale |
|---------------|----------------------|---------|-----------|
| Technical Docs | 512-1024 tokens | 10-15% | Preserve technical context |
| News Articles | 256-512 tokens | 10% | Each paragraph self-contained |
| Code Files | 256-512 tokens | 20% | Preserve function/class context |
| Legal Documents | 1024-2048 tokens | 15-20% | Complex sentences need context |

---

## Recursive Chunking

### Overview

Recursive chunking uses a hierarchical approach, attempting to split text at natural boundaries (paragraphs, then sentences, then words) until chunks meet size requirements.

### How It Works

```
Document
    ↓
Try splitting by: ["\n\n", "\n", ". ", " ", ""]
    ↓
If chunk > max_size:
    Split by first available separator
    Recursively process each piece
    ↓
Chunks that fit within size limits
```

### The Recursive Process

**Step 1**: Try to split by double newline (paragraphs)
```
Document → [Paragraph 1] [Paragraph 2] [Paragraph 3]
```

**Step 2**: If paragraph still too large, split by single newline
```
[Large Paragraph] → [Line 1] [Line 2] [Line 3]
```

**Step 3**: If line still too large, split by sentence
```
[Long Line] → [Sentence 1.] [Sentence 2.] [Sentence 3.]
```

**Step 4**: If sentence still too large, split by words
```
[Long Sentence] → [Words 1-N] [Words N+1-2N]
```

### Characteristics

#### ✅ Advantages

- **Structure-Aware**: Respects natural document boundaries
- **Reliable**: Industry standard for general-purpose RAG
- **Balanced**: Good trade-off between simplicity and quality
- **Flexible**: Works well across different document types
- **Maintains Hierarchy**: Preserves logical document structure

#### ❌ Disadvantages

- **Not Semantic**: Still doesn't understand topic boundaries
- **Variable Quality**: Quality depends on document formatting
- **Configuration-Dependent**: Requires tuning for optimal results

#### 📊 Performance Profile

- **Recall**: ⭐⭐⭐⭐ (Good - respects natural breaks)
- **Precision**: ⭐⭐⭐⭐ (Good - coherent chunks)
- **Coherence**: ⭐⭐⭐⭐ (Good - logical boundaries)
- **Cost**: ⭐⭐⭐⭐ (Low - efficient processing)

### Best Use Cases

✓ **Default Choice**: Recommended starting point for most RAG projects  
✓ **Mixed Content**: Documents with varied structure  
✓ **General Purpose**: When you need reliable, good-enough performance  
✓ **Structured Documents**: Markdown, HTML, well-formatted text  
✓ **Production Systems**: Battle-tested and predictable

### Implementation Example

```python
# Recursive Chunking Implementation
from langchain.text_splitter import RecursiveCharacterTextSplitter

def recursive_chunking(text, chunk_size=512, overlap=50):
    """
    Split text recursively using natural separators
    
    Args:
        text: Input document text
        chunk_size: Target size for each chunk
        overlap: Number of overlapping characters
    
    Returns:
        List of text chunks
    """
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        length_function=len,
        separators=[
            "\n\n",  # Paragraph breaks
            "\n",    # Line breaks
            ". ",    # Sentence ends
            "! ",    # Exclamations
            "? ",    # Questions
            "; ",    # Semicolons
            ", ",    # Commas
            " ",     # Spaces
            ""       # Characters (fallback)
        ]
    )
    
    chunks = splitter.split_text(text)
    
    # Add metadata about chunk position
    chunks_with_metadata = []
    for i, chunk in enumerate(chunks):
        chunks_with_metadata.append({
            "text": chunk,
            "chunk_id": i,
            "total_chunks": len(chunks),
            "length": len(chunk)
        })
    
    return chunks_with_metadata

# Usage
document = """
# Introduction to RAG

Retrieval-Augmented Generation combines retrieval with generation.

## Benefits

RAG provides several key advantages:
- Reduced hallucinations
- Access to current information
- Source attribution
"""

chunks = recursive_chunking(document, chunk_size=100, overlap=20)
for chunk in chunks:
    print(f"Chunk {chunk['chunk_id']}: {chunk['text'][:50]}...")
```

### Advanced Configuration

#### Custom Separators for Code

```python
# Code-specific separators
code_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=[
        "\nclass ",      # Class definitions
        "\ndef ",        # Function definitions
        "\n\n",          # Blank lines
        "\n",            # Line breaks
        " ",             # Spaces
        ""               # Characters
    ]
)
```

#### Markdown-Specific Separators

```python
# Markdown-aware separators
markdown_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=[
        "\n## ",         # H2 headers
        "\n### ",        # H3 headers
        "\n#### ",       # H4 headers
        "\n\n",          # Paragraphs
        "\n",            # Lines
        ". ",            # Sentences
        " ",             # Words
        ""               # Characters
    ]
)
```

---

## Semantic Chunking

### Overview

Semantic chunking groups content by meaning rather than structure, creating chunks where each represents a coherent topic or concept. It uses embedding similarity to identify natural topic boundaries.

### How It Works

```
Step 1: Split text into sentences
    ↓
Step 2: Generate embeddings for each sentence
    ↓
Step 3: Calculate similarity between consecutive sentences
    ↓
Step 4: When similarity drops below threshold → New chunk
    ↓
Chunks grouped by topic coherence
```

### The Semantic Approach

**Traditional Chunking:**
```
"Azure provides cloud services. AWS is a competitor. 
Azure offers compute, storage, and networking..."
        ↓
[Fixed split after "competitor"]
Chunk 1: "Azure provides cloud services. AWS is a competitor."
Chunk 2: "Azure offers compute, storage, and networking..."
```

**Semantic Chunking:**
```
"Azure provides cloud services. AWS is a competitor. 
Azure offers compute, storage, and networking..."
        ↓
[Semantic boundary detected - topic shift to Azure features]
Chunk 1: "Azure provides cloud services. AWS is a competitor."
Chunk 2: "Azure offers compute, storage, and networking..."
```

### Characteristics

#### ✅ Advantages

- **Topic Coherence**: Each chunk focuses on a single concept
- **Best Retrieval Quality**: 2-9% improvement over recursive chunking
- **Context Preservation**: Maintains semantic relationships
- **Adaptive**: Chunk size varies based on topic boundaries
- **Intelligent**: Understands content meaning, not just structure

#### ❌ Disadvantages

- **Computationally Expensive**: Requires embedding generation for every sentence
- **Variable Chunk Sizes**: Can create very large or very small chunks
- **Configuration Sensitivity**: Threshold tuning affects quality significantly
- **Slower Processing**: Takes longer than simpler methods
- **Cost**: Higher API costs for embedding generation

#### 📊 Performance Profile

- **Recall**: ⭐⭐⭐⭐⭐ (Excellent - finds all relevant content)
- **Precision**: ⭐⭐⭐⭐⭐ (Excellent - highly relevant chunks)
- **Coherence**: ⭐⭐⭐⭐⭐ (Excellent - topic-aligned)
- **Cost**: ⭐⭐ (Higher - embedding costs)

### Best Use Cases

✓ **High-Accuracy Requirements**: When precision is critical  
✓ **Complex Documents**: Research papers, technical documentation  
✓ **Multi-Topic Content**: Documents covering many different subjects  
✓ **Quality Over Speed**: When accuracy justifies higher cost  
✓ **Academic/Research**: Scientific papers, long-form content

### Implementation Example

```python
# Semantic Chunking Implementation
from langchain.embeddings import AzureOpenAIEmbeddings
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

def semantic_chunking(text, embedding_model, similarity_threshold=0.75, max_chunk_size=1000):
    """
    Split text based on semantic similarity between sentences
    
    Args:
        text: Input document text
        embedding_model: Model for generating embeddings
        similarity_threshold: Similarity below this creates new chunk
        max_chunk_size: Maximum characters per chunk
    
    Returns:
        List of semantically coherent chunks
    """
    # Split into sentences
    sentences = text.replace('! ', '!|').replace('? ', '?|').replace('. ', '.|').split('|')
    sentences = [s.strip() for s in sentences if s.strip()]
    
    # Generate embeddings for each sentence
    embeddings = embedding_model.embed_documents(sentences)
    
    # Calculate similarities between consecutive sentences
    chunks = []
    current_chunk = [sentences[0]]
    current_length = len(sentences[0])
    
    for i in range(1, len(sentences)):
        # Calculate similarity with previous sentence
        similarity = cosine_similarity(
            [embeddings[i-1]], 
            [embeddings[i]]
        )[0][0]
        
        # Check if we should start a new chunk
        if similarity < similarity_threshold or current_length + len(sentences[i]) > max_chunk_size:
            # Save current chunk and start new one
            chunks.append(' '.join(current_chunk))
            current_chunk = [sentences[i]]
            current_length = len(sentences[i])
        else:
            # Add to current chunk
            current_chunk.append(sentences[i])
            current_length += len(sentences[i])
    
    # Add final chunk
    if current_chunk:
        chunks.append(' '.join(current_chunk))
    
    return chunks

# Usage with Azure OpenAI
from langchain.embeddings import AzureOpenAIEmbeddings

embedding_model = AzureOpenAIEmbeddings(
    azure_deployment="text-embedding-3-large",
    openai_api_version="2024-02-01"
)

document = """
Azure cloud computing provides scalable infrastructure for modern applications. 
Companies can deploy virtual machines in minutes. The platform supports 
multiple programming languages and frameworks.

Machine learning has revolutionized data analysis. Neural networks can 
identify patterns humans might miss. Training large models requires 
significant computational resources.
"""

chunks = semantic_chunking(document, embedding_model, similarity_threshold=0.70)
for i, chunk in enumerate(chunks):
    print(f"\nChunk {i+1}:\n{chunk}")
```

### Threshold Tuning Guide

| Threshold | Behavior | Best For |
|-----------|----------|----------|
| 0.90-1.00 | Very strict - Many small chunks | Highly specific queries |
| 0.75-0.89 | Balanced - Moderate coherence | General purpose (recommended) |
| 0.60-0.74 | Lenient - Fewer, larger chunks | Broad context needed |
| < 0.60 | Too lenient - Loses semantic value | Not recommended |

---

## Context-Aware Chunking

### Overview

Context-aware chunking adapts the splitting strategy based on document format and structure. Different content types (markdown, code, tables, JSON) require different approaches to preserve meaning.

### Document-Specific Strategies

#### Markdown Documents

**Strategy**: Split by headers, preserve hierarchy

```markdown
# Chapter 1
Content for chapter 1...

## Section 1.1
Content for section 1.1...

## Section 1.2
Content for section 1.2...
```

**Chunking Approach:**
- Keep headers with their content
- Maintain hierarchy in metadata
- Split at section boundaries

#### Code Files

**Strategy**: Split by functions/classes, preserve complete units

```python
class UserManager:
    def create_user(self):
        # Implementation
        pass
    
    def delete_user(self):
        # Implementation
        pass
```

**Chunking Approach:**
- One chunk per function/method
- Include class context
- Preserve import statements

#### Tables and Structured Data

**Strategy**: Keep tables intact, split at table boundaries

```
| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Data     | Data     | Data     |
```

**Chunking Approach:**
- Never split tables mid-way
- Include table headers with data
- Preserve formatting

### Characteristics

#### ✅ Advantages

- **Format Preservation**: Maintains document structure
- **Optimal for Specific Content**: Tailored to content type
- **Reduces Errors**: Prevents breaking important structures
- **Better User Experience**: Results maintain readability

#### ❌ Disadvantages

- **Complex Implementation**: Requires custom logic per format
- **Maintenance Overhead**: More code to maintain
- **Format Detection Needed**: Must identify document type

#### 📊 Performance Profile

- **Recall**: ⭐⭐⭐⭐ (Excellent for specific formats)
- **Precision**: ⭐⭐⭐⭐⭐ (Excellent - structure-aware)
- **Coherence**: ⭐⭐⭐⭐⭐ (Excellent - preserves meaning)
- **Cost**: ⭐⭐⭐ (Moderate - format-specific processing)

### Best Use Cases

✓ **Multi-Format Repositories**: Code + docs + data  
✓ **Technical Documentation**: API docs, reference materials  
✓ **Code Search**: Developer tools, code Q&A  
✓ **Structured Data**: Tables, JSON, XML documents

### Implementation Example

```python
# Context-Aware Chunking for Different Formats
from langchain.text_splitter import (
    MarkdownHeaderTextSplitter,
    RecursiveCharacterTextSplitter,
    Language
)

def context_aware_chunking(text, file_type, chunk_size=512):
    """
    Apply format-specific chunking strategy
    
    Args:
        text: Document content
        file_type: Type of document (md, py, json, etc.)
        chunk_size: Target chunk size
    
    Returns:
        List of chunks with metadata
    """
    if file_type == "markdown":
        # Markdown-specific: split by headers
        headers_to_split_on = [
            ("#", "Header 1"),
            ("##", "Header 2"),
            ("###", "Header 3"),
        ]
        markdown_splitter = MarkdownHeaderTextSplitter(
            headers_to_split_on=headers_to_split_on
        )
        chunks = markdown_splitter.split_text(text)
        
    elif file_type == "python":
        # Python code: split by classes and functions
        python_splitter = RecursiveCharacterTextSplitter.from_language(
            language=Language.PYTHON,
            chunk_size=chunk_size,
            chunk_overlap=50
        )
        chunks = python_splitter.split_text(text)
        
    elif file_type == "javascript":
        # JavaScript: split by functions and classes
        js_splitter = RecursiveCharacterTextSplitter.from_language(
            language=Language.JS,
            chunk_size=chunk_size,
            chunk_overlap=50
        )
        chunks = js_splitter.split_text(text)
        
    else:
        # Default: recursive chunking
        default_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=50
        )
        chunks = default_splitter.split_text(text)
    
    return chunks

# Usage
markdown_doc = """
# Introduction
This is the introduction section.

## Background
Details about the background...

## Methodology
Our methodology includes...
"""

python_code = """
class DataProcessor:
    def __init__(self):
        self.data = []
    
    def process(self, input_data):
        # Process the data
        return processed_data
"""

# Process different formats
md_chunks = context_aware_chunking(markdown_doc, "markdown")
py_chunks = context_aware_chunking(python_code, "python")

print(f"Markdown chunks: {len(md_chunks)}")
print(f"Python chunks: {len(py_chunks)}")
```

---

## Comparison Matrix

### Strategy Comparison Table

| Strategy | Complexity | Performance | Cost | Speed | Best For | Avoid When |
|----------|-----------|-------------|------|-------|----------|------------|
| **Fixed-Size** | ⭐ Low | ⭐⭐⭐ Baseline | ⭐⭐⭐⭐⭐ Very Low | ⭐⭐⭐⭐⭐ Fastest | Prototyping, simple content | High accuracy needed |
| **Recursive** | ⭐⭐ Medium | ⭐⭐⭐⭐ Good | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Fast | General purpose, default choice | Format-specific needs |
| **Semantic** | ⭐⭐⭐⭐ High | ⭐⭐⭐⭐⭐ Best | ⭐⭐ High | ⭐⭐ Slow | High accuracy, research | Budget-constrained |
| **Context-Aware** | ⭐⭐⭐⭐⭐ Very High | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Medium | ⭐⭐⭐ Medium | Multi-format, code | Simple documents |

### Performance Metrics Comparison

Based on industry research and testing:

| Strategy | Recall | Precision | F1 Score | Processing Time | API Costs |
|----------|--------|-----------|----------|-----------------|-----------|
| **Fixed-Size** | 72% | 68% | 0.70 | 1x (baseline) | $ |
| **Recursive** | 78% | 76% | 0.77 | 1.2x | $ |
| **Semantic** | 85% | 83% | 0.84 | 5-10x | $$$ |
| **Context-Aware** | 82% | 84% | 0.83 | 2-3x | $$ |

### When to Use Each Strategy

```
START
  │
  ├─ Need quick prototype? ──YES──> Fixed-Size
  │         │
  │         NO
  │         │
  ├─ Have specific format (code/markdown)? ──YES──> Context-Aware
  │         │
  │         NO
  │         │
  ├─ Need highest accuracy? ──YES──> Semantic
  │         │
  │         NO
  │         │
  └─────> Recursive (Default Choice)
```

---

## Best Practices

### 1. Start with Recursive, Optimize Later

**The Golden Rule**: Begin every RAG project with recursive chunking.

**Rationale:**
- Establishes performance baseline
- Easy to implement
- Works well for 80% of use cases
- Allows focus on other RAG components first

**Then optimize if:**
- Accuracy isn't meeting requirements
- You have specific document formats
- Budget allows for semantic chunking
- Performance metrics justify the effort

### 2. Chunk Size Guidelines

#### Align with Embedding Model

| Embedding Model | Optimal Input | Recommended Chunk Size | Overlap |
|----------------|---------------|------------------------|---------|
| text-embedding-3-small | 512 tokens | 256-512 tokens | 50-75 tokens |
| text-embedding-3-large | 512 tokens | 256-512 tokens | 50-75 tokens |
| text-embedding-ada-002 | 8191 tokens | 512-1024 tokens | 100-150 tokens |
| BGE-large | 512 tokens | 256-512 tokens | 50-75 tokens |

#### General Guidelines

- **Too Small** (< 128 tokens): Fragmented context, poor retrieval
- **Sweet Spot** (256-1024 tokens): Balanced context and precision
- **Too Large** (> 2048 tokens): Lost precision, reduced relevance

### 3. Overlap Strategy

**Why Overlap Matters:**
- Prevents information loss at chunk boundaries
- Ensures important content isn't split between chunks
- Improves recall for queries matching boundary content

**Recommended Overlap:**
- **10-15%**: Light overlap for well-structured content
- **15-20%**: Standard overlap for general content
- **20-25%**: Heavy overlap for critical applications

**Example:**
```python
# Chunk size: 500 tokens, Overlap: 20% (100 tokens)
Chunk 1: Tokens 1-500
Chunk 2: Tokens 401-900  # 100 token overlap
Chunk 3: Tokens 801-1300 # 100 token overlap
```

### 4. Metadata Enrichment

**Add Context to Every Chunk:**

```python
chunk_metadata = {
    "text": "The actual chunk content...",
    "source": "financial_report_Q3_2025.pdf",
    "page": 15,
    "chapter": "Revenue Analysis",
    "chunk_id": "chunk_42",
    "total_chunks": 150,
    "created_date": "2025-11-15",
    "document_type": "financial_report",
    "author": "Finance Team"
}
```

**Benefits:**
- Enables filtering (e.g., "only 2025 documents")
- Improves source attribution
- Allows hybrid search strategies
- Better user experience

### 5. Test and Measure

**Key Metrics to Track:**

1. **Retrieval Quality**
   - Precision @ K (top K results)
   - Recall @ K
   - Mean Reciprocal Rank (MRR)

2. **User Satisfaction**
   - Answer relevance (human eval)
   - Source quality
   - Response completeness

3. **System Performance**
   - Query latency
   - Processing time
   - Cost per query

**A/B Testing Strategy:**

```python
# Compare chunking strategies
strategies = [
    ("fixed", fixed_size_chunking),
    ("recursive", recursive_chunking),
    ("semantic", semantic_chunking)
]

test_queries = [
    "What are our Q3 revenue figures?",
    "Explain the cloud migration strategy",
    "Who are the key stakeholders?"
]

for strategy_name, strategy_func in strategies:
    scores = evaluate_rag_system(
        chunking_strategy=strategy_func,
        test_queries=test_queries
    )
    print(f"{strategy_name}: {scores}")
```

### 6. Consider Content Type

| Content Type | Recommended Strategy | Chunk Size | Special Considerations |
|--------------|---------------------|------------|----------------------|
| News Articles | Recursive | 512 tokens | Split by paragraphs |
| Technical Docs | Context-Aware | 512-1024 tokens | Preserve code blocks |
| Research Papers | Semantic | 1024 tokens | Maintain topic coherence |
| Code Repositories | Context-Aware | 256-512 tokens | Keep functions intact |
| Legal Documents | Recursive | 1024-2048 tokens | Maintain clause structure |
| Chat Logs | Fixed-Size | 256 tokens | Preserve conversation flow |
| Product Catalogs | Recursive | 512 tokens | One product per chunk |

### 7. Handle Special Cases

#### Tables
```python
# Keep tables intact
if is_table(text_segment):
    chunks.append({
        "text": entire_table,
        "type": "table",
        "preserve_formatting": True
    })
```

#### Lists
```python
# Keep bullet lists together
if is_bullet_list(text_segment):
    chunks.append({
        "text": complete_list,
        "type": "list",
        "list_items": count_items(complete_list)
    })
```

#### Code Blocks
```python
# Preserve code syntax
if is_code_block(text_segment):
    chunks.append({
        "text": code_block,
        "type": "code",
        "language": detect_language(code_block)
    })
```

### 8. Chunk Size Decision Tree

```
What's your document average length?
    │
    ├─ < 1000 words ──> Small chunks (256 tokens)
    │                   More granular retrieval
    │
    ├─ 1000-5000 words ──> Medium chunks (512 tokens)
    │                      Balanced approach
    │
    └─ > 5000 words ──> Larger chunks (1024 tokens)
                        Preserve broad context
```

### 9. Common Pitfalls to Avoid

❌ **Don't**: Use fixed-size chunking for code
✅ **Do**: Use language-aware splitters

❌ **Don't**: Ignore overlap entirely
✅ **Do**: Use 10-20% overlap minimum

❌ **Don't**: Chunk without testing retrieval quality
✅ **Do**: Measure and iterate

❌ **Don't**: Use same strategy for all content types
✅ **Do**: Adapt to document structure

❌ **Don't**: Forget to add metadata
✅ **Do**: Enrich chunks with source info

### 10. The Optimal Workflow

```
1. Start with Recursive Chunking
   └─> Chunk size: 512 tokens, Overlap: 20%
   
2. Build and Test RAG System
   └─> Establish baseline metrics
   
3. Analyze Results
   └─> Identify failure cases
   
4. Optimize Strategically
   ├─> Poor recall → Try semantic chunking
   ├─> Broken code → Add context-aware logic
   └─> Mixed results → Hybrid approach
   
5. Measure Impact
   └─> Compare metrics before/after
   
6. Iterate
   └─> Continuous improvement
```

---

## Summary

### Key Takeaways

1. **Chunking matters**: Up to 9% performance difference between strategies
2. **Start simple**: Recursive chunking is the reliable default
3. **Optimize when needed**: Move to semantic/context-aware for specific needs
4. **Test everything**: Measure retrieval quality, don't assume
5. **Match content type**: Different documents need different strategies
6. **Use overlap**: 10-20% overlap prevents boundary information loss
7. **Add metadata**: Enrich chunks with source and context information
8. **Monitor costs**: Semantic chunking more accurate but more expensive

### Quick Reference

| Goal | Strategy | Configuration |
|------|----------|---------------|
| Get started fast | Recursive | 512 tokens, 20% overlap |
| Maximum accuracy | Semantic | Threshold 0.75, max 1024 tokens |
| Code repositories | Context-Aware | Language-specific splitters |
| Low budget | Fixed-Size | 512 tokens, 10% overlap |
| Research papers | Semantic | 1024 tokens, threshold 0.70 |

---

**Next**: Part 3 - Azure Ecosystem Integration

