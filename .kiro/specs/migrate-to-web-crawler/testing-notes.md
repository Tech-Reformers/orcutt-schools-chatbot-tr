# Testing Notes for KB v3 (PHUCWA33C9)

## Knowledge Base Configuration

**KB ID:** PHUCWA33C9  
**Name:** OrcuttSchoolsKB-WebCrawler-v3  
**Chunking Strategy:** Semantic, 500 tokens, buffer 1 (20% overlap), breakpoint 95  
**Search Type:** Hybrid (semantic + BM25 keyword matching)  
**Retrieval Count:** 60 results  

**Changes from v2 (V3OVONSOBC):**
- Same chunking parameters (500 tokens, buffer 1)
- Comment in code says "correct URL" - suggests metadata or URL extraction fix
- Testing to see if retrieval quality improves

## Test Queries

### Critical Queries (Previously Failing)

#### 1. "Who are the Executive Directors?"
**Expected:** Should find http://orcuttschools.net/33975_3  
**Status:** ⏳ PENDING TEST  
**Notes:**

#### 2. "How do I sign up for a classroom pizza party?"
**Expected:** Should find https://www.orcuttschools.net/34729_3  
**Status:** ⏳ PENDING TEST  
**Notes:**

#### 3. "pizza" (short query)
**Expected:** Should find pizza party catering page  
**Status:** ⏳ PENDING TEST  
**Notes:**

### Baseline Queries (Should Continue Working)

#### 4. "Who is the Superintendent?"
**Expected:** Dr. Holly Edds  
**Status:** ⏳ PENDING TEST  
**Notes:**

#### 5. "When are parent-teacher conferences?"
**Expected:** Upcoming conference dates  
**Status:** ⏳ PENDING TEST  
**Notes:**

### School-Specific Filtering

#### 6. Pine Grove: "When does school start?"
**Expected:** Pine Grove-specific hours  
**Status:** ⏳ PENDING TEST  
**Notes:**

#### 7. Lakeview: "When does school start?"
**Expected:** Lakeview-specific hours  
**Status:** ⏳ PENDING TEST  
**Notes:**

## Comparison with v2

| Metric | v2 (V3OVONSOBC) | v3 (PHUCWA33C9) | Notes |
|--------|-----------------|-----------------|-------|
| Executive Directors query | ❌ Failed | ⏳ Testing | |
| Pizza party query | ❌ Failed | ⏳ Testing | |
| Superintendent query | ✅ Passed | ⏳ Testing | |
| Date-related queries | ✅ Passed | ⏳ Testing | |
| School filtering | ✅ Passed | ⏳ Testing | |

## Issues Found

_Document any issues discovered during testing_

## Recommendations

_Document any recommendations based on test results_

---

**Testing Date:** 2026-02-07  
**Tester:** _TBD_  
**Status:** In Progress
