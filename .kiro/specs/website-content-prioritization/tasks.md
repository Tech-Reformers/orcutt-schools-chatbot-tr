# Implementation Plan

- [x] 1. Add source type detection method
  - Create `is_website_source()` method to identify website vs PDF sources
  - Check if source URL ends with `.pdf` or if s3Uri contains `.pdf`
  - Return True for website sources, False for PDF sources
  - _Requirements: 1.1, 1.2_

- [x] 2. Add date extraction and filtering method
  - Create `extract_dates_from_content()` method to find dates in source text
  - Support common date formats (MM/DD/YYYY, Month DD YYYY, etc.)
  - Create `has_future_dates()` method to check if dates are after current date
  - Create `prioritize_future_dates()` method to reorder sources by date relevance
  - _Requirements: 5.1, 5.2, 5.3_

- [x] 3. Add query classification for date-related questions
  - Create `is_date_query()` method to detect date-related questions
  - Check for keywords: "when", "schedule", "calendar", "date", "conference", etc.
  - Return True if query is asking about dates/events
  - _Requirements: 5.1_

- [x] 4. Implement source reranking logic
  - Create `rerank_sources()` method in OrcuttChatbot class
  - Separate sources into website_sources and pdf_sources lists
  - If date query, apply date filtering to both lists
  - Return reordered list: website_sources + pdf_sources
  - _Requirements: 1.1, 1.2, 1.3, 2.3, 4.1_

- [x] 5. Integrate reranking into knowledge base processing
  - Modify `process_knowledge_base_response()` to call `rerank_sources()`
  - Pass query to reranking method for date detection
  - Apply reranking before creating context string
  - Ensure source metadata is preserved
  - _Requirements: All_

- [x] 6. Update prompt to reference current date
  - Ensure prompt includes current date context
  - Add guidance to use current date when answering date-related questions
  - Add example: "The next parent-teacher conferences are..."
  - _Requirements: 5.4_

- [x] 7. Enable hybrid search for keyword matching
  - Modify `query_knowledge_base_semantic()` method
  - Change `overrideSearchType` from `'SEMANTIC'` to `'HYBRID'`
  - This enables both semantic similarity and keyword matching (BM25)
  - _Requirements: 6.1, 6.2, 6.3_

- [ ] 8. Test and deploy
  - Deploy backend changes
  - Test with "Who are the Executive Directors?" query (should now find exact match)
  - Test with "When are parent-teacher conferences?" query
  - Test with PDF-only query to ensure fallback works
  - Verify consistency across different phrasings
  - Test that existing queries still work correctly
  - _Requirements: All_
