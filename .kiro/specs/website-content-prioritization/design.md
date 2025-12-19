# Design Document

## Overview

This design improves the chatbot's source prioritization to prefer website content over PDF documents when both are available. The solution involves modifying the knowledge base retrieval process to apply ranking boosts to website sources and implementing date-aware filtering for time-sensitive queries.

## Architecture

The current system retrieves sources using semantic search from AWS Bedrock Knowledge Base, which returns results ranked by vector similarity. We will:
1. Switch from pure semantic search to hybrid search (semantic + keyword matching)
2. Increase the number of retrieved results to ensure relevant content is found
3. Add a post-retrieval reranking step that applies business logic to boost website sources and filter based on dates

**Current Flow:**
1. User query → Semantic search → Ranked results → Claude generates response

**New Flow:**
1. User query → **Hybrid search (semantic + keyword, 60 results)** → Ranked results → **Rerank (boost websites, filter dates)** → Claude generates response

## Components and Interfaces

### Backend Changes

**File**: `lambda/chatbot/lambda_function.py`

**Modified Method**: `query_knowledge_base_semantic()`
- Change `overrideSearchType` from `'SEMANTIC'` to `'HYBRID'`
- Enables keyword matching in addition to semantic search
- AWS Bedrock automatically balances semantic and keyword scores
- Increase `numberOfResults` from 40 to 60 for main domain queries
- Ensures sufficient results are retrieved to find relevant content

**New Method**: `rerank_sources_by_type()`
- Takes retrieval results and query
- Separates website sources from PDF sources
- Applies boost factor to website sources
- Reorders results with websites first

**New Method**: `filter_by_dates()`
- Extracts dates from source content
- Compares dates to current date
- Deprioritizes sources with only past dates for date-related queries

**Modified Method**: `process_knowledge_base_response()`
- Calls reranking methods before processing sources
- Maintains existing source metadata structure

**Modified Method**: `process_chat_request()`
- Calls `query_knowledge_base_semantic()` with increased result count
- Uses user query directly without preprocessing

### Source Type Detection

Sources will be classified as "website" or "pdf" based on metadata:
- **Website**: `source_url` ends with `.net` or `.com` (not `.pdf`)
- **PDF**: `source_url` ends with `.pdf` OR s3Uri contains `.pdf`

### Retrieval Strategy

For effective hybrid search:
- Use user queries directly without preprocessing
- Hybrid search combines semantic similarity with BM25 keyword matching
- AWS Bedrock automatically balances the two scoring methods
- Retrieve 60 results from main domain to ensure relevant content is found
- Retrieve 10 results from school-specific domains when applicable

**Rationale:**
- Simplicity: No complex query preprocessing needed
- Reliability: Hybrid search handles both semantic and keyword matching
- Coverage: 60 results ensures we don't miss relevant pages
- AWS handles the complexity of balancing semantic vs keyword scores

### Date Extraction

For date-aware filtering:
- Extract dates from source content using regex patterns
- Common formats: MM/DD/YYYY, Month DD, YYYY, etc.
- Compare extracted dates to `datetime.now()`
- Flag sources as "has_future_dates", "has_only_past_dates", or "no_dates"

## Data Models

### Source Ranking Metadata

```python
{
    "source_type": "website" | "pdf",
    "has_future_dates": bool,
    "has_past_dates": bool,
    "boost_score": float  # Multiplier applied to relevance
}
```

### Ranking Logic

**Approach**: Simple separation rather than score-based boosting

```python
# Separate sources by type
website_sources = [s for s in sources if is_website(s)]
pdf_sources = [s for s in sources if is_pdf(s)]

# For date-related queries, further filter by dates
if is_date_query:
    website_sources = prioritize_future_dates(website_sources)
    pdf_sources = prioritize_future_dates(pdf_sources)

# Reorder: websites first, then PDFs
reranked_sources = website_sources + pdf_sources
```

This ensures websites always appear before PDFs in the context, regardless of semantic similarity scores. Claude will naturally use the first relevant sources it encounters.

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Website Source Prioritization
*For any* query where both website and PDF sources are retrieved, website sources SHALL appear before PDF sources in the reranked results (assuming similar relevance scores).
**Validates: Requirements 1.1, 1.2, 1.3**

### Property 2: Date-Aware Ranking
*For any* date-related query, sources containing future dates SHALL be ranked higher than sources containing only past dates.
**Validates: Requirements 5.1, 5.3**

### Property 3: Consistent Ranking
*For any* query asked with different phrasing, the reranking logic SHALL apply the same boost factors and produce consistent source ordering.
**Validates: Requirements 2.1, 2.3**

### Property 4: Fallback to PDFs
*For any* query where no website sources are available, PDF sources SHALL be used without penalty.
**Validates: Requirements 4.2**

### Property 5: Keyword Matching
*For any* query containing specific terms, sources containing exact keyword matches SHALL be retrieved and ranked appropriately by the hybrid search algorithm.
**Validates: Requirements 6.1, 6.2, 6.3**

### Property 6: Sufficient Retrieval Coverage
*For any* query, retrieving 60 results SHALL provide sufficient coverage to find relevant content even when it doesn't rank in the top 20.
**Validates: Requirements 6.4**

## Error Handling

### No Website Sources Available
- **Scenario**: Query retrieves only PDF sources
- **Handling**: Use PDF sources normally, no boost applied
- **User Impact**: User gets answer from PDFs without indication of lower priority

### Date Extraction Fails
- **Scenario**: Cannot parse dates from source content
- **Handling**: Treat source as "no_dates", apply no date-based penalty
- **User Impact**: Source is ranked based on type (website vs PDF) only

### All Sources Have Past Dates
- **Scenario**: User asks about future event but all sources have past dates
- **Handling**: Use best available sources, let Claude explain dates are past
- **User Impact**: User gets informed that information may be outdated

## Testing Strategy

### Unit Tests
- Test `rerank_sources_by_type()` with mixed website/PDF sources
- Test `filter_by_dates()` with various date formats
- Verify boost calculations are correct
- Test edge cases (no sources, all PDFs, all websites)

### Integration Tests
- End-to-end test: Ask "Who are the Executive Directors?" → verify website source used
- Date test: Ask "When are parent-teacher conferences?" → verify future dates prioritized
- Fallback test: Ask question only in PDFs → verify PDFs used without issue

### Manual Testing
- Test with known queries that currently fail (Executive Directors, bus schedules)
- Verify consistency across multiple phrasings
- Check that PDF sources still work when appropriate
