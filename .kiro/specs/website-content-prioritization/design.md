# Design Document

## Overview

This design improves the chatbot's ability to retrieve relevant website content for specific queries. The solution uses **selective reranking** and **improved prompt engineering** rather than query expansion.

## Problem Statement

When users query "pizza party", the system was not consistently retrieving the Child Nutrition Services catering page. The issue was related to retrieval configuration and prompt handling of conflicting information (wellness policies vs. actual services offered).

## Solution: Selective Reranking + Smart Prompt Engineering

### Implementation
**Location**: `lambda/chatbot/lambda_function.py`

### Approach 1: Selective Reranking (lines 248-250)
Reranking is **disabled** for main domain queries but **enabled** for school-specific queries.

```python
# Rerank sources to prioritize website content
# DISABLED: Reranking was interfering with retrieval order
# kb_response_main_domain = self.rerank_kb_response(kb_response_main_domain, message)
if kb_response_school_specific:
    kb_response_school_specific = self.rerank_kb_response(kb_response_school_specific, message)
```

**Why**: Bedrock's native ranking works well for main domain queries. Reranking only helps for school-specific queries where domain filtering is applied.

### Approach 2: Prompt Engineering (lines 800-830)
Added "CRITICAL - HANDLING CONFLICTING INFORMATION" section to system prompt that instructs Claude to prioritize official services over restrictive policies.

**Key instruction**:
```
When sources contain both district services AND policies that seem to conflict:
1. District-provided services are OFFICIAL OFFERINGS and take precedence
2. Policies describe general guidelines, but services are specific exceptions/implementations
3. If a service exists for what the user is asking about, provide that service information
4. Do NOT cite restrictive policies when an official service is available
```

### Why This Works
- Bedrock's retrieval finds both the catering page AND wellness policy
- Without guidance, Claude would cite the restrictive policy
- Prompt engineering tells Claude to prioritize the actual service offered
- Result: "pizza party" queries now correctly return catering information

### Status
✅ **IMPLEMENTED AND WORKING**
- Deployed: February 9, 2026
- Production URL: https://d3dolln1x7yei7.cloudfront.net
- Test confirmed: "pizza party" now retrieves catering page

## Reranking Implementation

## Reranking Implementation

### Reranking Code (lines 666-691)
The `rerank_sources()` method separates website sources from PDF sources and prioritizes:
1. Website sources (current, authoritative)
2. PDF sources (archived documents)

For date-related queries, sources with future dates are prioritized within each category.

### Current Configuration
- **Main domain queries**: Reranking **DISABLED** (line 248-249 commented out)
- **School-specific queries**: Reranking **ENABLED** (line 250)
- **Reason**: Bedrock's native ranking is optimal for main domain; reranking helps school-specific filtered results

## Configuration

### Knowledge Base Settings
- **KB ID**: GCERPWLGOK (2806 files from web crawler)
- **Number of results**: 40 for main domain, 10 for school-specific
- **Search type**: Default (HYBRID - semantic + keyword)
- **Domain filter**: None for main queries (searches all content)

### Query Processing Flow
1. User submits query (e.g., "pizza party")
2. **KB Retrieval**: Search with hybrid search (semantic + keyword)
3. **Selective Reranking**: Apply only to school-specific queries
4. **Response Generation**: Claude uses retrieved sources with smart prompt guidance

## Testing Results

### Test Cases - All Passed ✅
1. ✅ "pizza party" → Retrieves catering page and prioritizes service over policy
2. ✅ "executive directors" → Retrieves correct info
3. ✅ "superintendent" → Works correctly
4. ✅ School-specific queries → Reranking works correctly
5. ✅ Other queries → No regression

### Success Metrics
- ✅ Food queries retrieve catering page
- ✅ No degradation in other query types
- ✅ Response times under 10 seconds
- ✅ System stable and working for demo

## Key Learnings

1. **Bedrock's native ranking is good**: Don't interfere unless necessary
2. **Prompt engineering matters**: Teaching Claude to prioritize services over policies solved the core issue
3. **Selective reranking**: Only apply where it helps (school-specific queries)
4. **Test with actual queries**: Real-world testing revealed the policy vs. service conflict

## Rollback Plan
If issues arise:
1. Re-enable reranking for main domain (uncomment lines 248-249)
2. Redeploy Lambda
3. Monitor query performance

## Future Enhancements
1. Machine learning to learn optimal ranking strategies
2. User feedback to refine prompt instructions
3. Expand conflict resolution logic to other domains
