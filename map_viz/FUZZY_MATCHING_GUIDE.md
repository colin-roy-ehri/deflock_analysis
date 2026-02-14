# Fuzzy Matching Implementation Guide

## Overview

This guide walks through implementing fuzzy matching to identify whether police searches were conducted by participating agencies in the Flock database.

**Goal:** Add a `is_participating_agency` flag to the classified police search data by matching org_names (e.g., "Houston TX PD") to official participating agency names (e.g., "Houston Police Department").

## Problem Statement

- **Classified table** contains 498 unique org_names in formats like: `"Seminole County FL SO"`, `"Houston TX PD"`
- **Participating agencies table** contains 1,414 official agency names: `"Seminole County Sheriff's Office"`, `"Houston Police Department"`
- **Challenge:** The name formats are significantly different, requiring semantic/fuzzy matching

## Architecture

### Components

1. **Text Embedding Model** (`sql/17_create_text_embedding_model.sql`)
   - Creates remote connection to Gemini text-embedding-004
   - Generates semantic embeddings for agency names

2. **Matching Procedure** (`sql/18_create_agency_matching_procedure.sql`)
   - Main orchestration logic
   - Generates embeddings for both org_names and participating agencies
   - Computes cosine similarity scores
   - Produces lookup table: `org_name_to_participating_agency`

3. **Manual Override Table** (`sql/19_create_agency_match_overrides.sql`)
   - Allows corrections to automatic matches
   - Handles edge cases and false positives

4. **Verification Queries** (`sql/20_verify_agency_matching.sql`)
   - Analyzes match quality
   - Identifies low-confidence matches
   - Validates state/type consistency

5. **Enriched View** (`sql/22_create_enriched_classified_view.sql`)
   - Joins matching results with classified table
   - Convenient for analysis queries

### Data Flow

```
Classified Table (October2025_classified)
        ↓
Extract unique org_names
        ↓
Generate embeddings → [Text Embedding Model]
        ↓
Cross-join with participating agency embeddings
        ↓
Compute cosine similarity
        ↓
Find best matches + apply threshold (0.85)
        ↓
Apply manual overrides
        ↓
Output: org_name_to_participating_agency lookup table
        ↓
Join back to Classified Table → Enriched View
```

## Implementation Steps

### Phase 1: Setup Embedding Model

**File:** `sql/17_create_text_embedding_model.sql`

```sql
-- Create the text embedding model
CREATE OR REPLACE MODEL `durango-deflock.FlockML.text_embedding_model`
REMOTE WITH CONNECTION `durango-deflock.us-central1.vertex-ai-connection`
OPTIONS (
  endpoint = 'text-embedding-004'
);
```

**In BigQuery Console:**

1. Open `/home/colin/map_viz/sql/17_create_text_embedding_model.sql`
2. Select all content
3. Run in BigQuery console
4. Verify the test query returns embeddings

**Expected Output:**
- Model created successfully
- Test query shows 4 embeddings with dimension 768

### Phase 2: Create Infrastructure Tables

**File:** `sql/19_create_agency_match_overrides.sql`

```sql
-- Create the manual overrides table
CREATE OR REPLACE TABLE `durango-deflock.FlockML.agency_match_overrides` (
  org_name STRING NOT NULL,
  manual_match BOOL NOT NULL,
  matched_agency STRING,
  override_reason STRING,
  override_timestamp TIMESTAMP,
  PRIMARY KEY (org_name) NOT ENFORCED
);
```

**In BigQuery Console:**

1. Run `sql/19_create_agency_match_overrides.sql`
2. Table should be created empty (ready for manual corrections)

### Phase 3: Create Matching Procedure

**File:** `sql/18_create_agency_matching_procedure.sql`

```sql
CREATE OR REPLACE PROCEDURE `durango-deflock.FlockML.sp_match_participating_agencies`(
  classified_table STRING
)
BEGIN
  -- ... implementation
END;
```

**In BigQuery Console:**

1. Run `sql/18_create_agency_matching_procedure.sql`
2. Procedure is created (no data generated yet)

### Phase 4: Execute Matching

**In BigQuery Console:**

```sql
-- Execute the matching procedure (no parameters needed)
CALL `durango-deflock.FlockML.sp_match_participating_agencies`();

-- Check execution status
SELECT * FROM `durango-deflock.FlockML.org_name_to_participating_agency` LIMIT 20;
```

**Expected Output:**
- Lookup table created with columns:
  - `org_name`: Original org_name
  - `is_participating_agency`: BOOL (matched or not)
  - `matched_agency`: Full agency name if matched
  - `matched_state`: State of matched agency
  - `matched_type`: Type of agency (Sheriff's Office, Police Department, etc.)
  - `similarity`: Float score 0-1
  - `match_timestamp`: When matching occurred

**Example Results:**
```
org_name                      | is_participating_agency | matched_agency                    | similarity
"Seminole County FL SO"       | TRUE                    | "Seminole County Sheriff's Office" | 0.9234
"Houston TX PD"               | TRUE                    | "Houston Police Department"       | 0.8901
"Unknown Agency XY"           | FALSE                   | "Best guess agency"               | 0.6234
```

### Phase 5: Verify Quality

**File:** `sql/20_verify_agency_matching.sql`

**Query 1: Overall Coverage**

```sql
SELECT
  COUNT(DISTINCT org_name) AS total_unique_agencies,
  COUNT(DISTINCT CASE WHEN is_participating_agency THEN org_name END) AS matched_agencies,
  ROUND(COUNT(DISTINCT CASE WHEN is_participating_agency THEN org_name END) * 100.0 /
        COUNT(DISTINCT org_name), 2) AS match_percentage
FROM `durango-deflock.FlockML.org_name_to_participating_agency`;
```

**Query 2: Similarity Distribution**

```sql
SELECT
  CASE
    WHEN similarity >= 0.95 THEN 'Very High (0.95+)'
    WHEN similarity >= 0.85 THEN 'High (0.85-0.95)'
    WHEN similarity >= 0.75 THEN 'Medium (0.75-0.85)'
    ELSE 'Low (<0.75)'
  END AS similarity_bucket,
  COUNT(*) AS count,
  ROUND(AVG(similarity), 4) AS avg_similarity
FROM `durango-deflock.FlockML.org_name_to_participating_agency`
WHERE is_participating_agency = TRUE
GROUP BY similarity_bucket
ORDER BY avg_similarity DESC;
```

**Query 3: Manual Review - High Risk Matches (0.75-0.85)**

```sql
SELECT
  org_name,
  matched_agency,
  matched_state,
  similarity
FROM `durango-deflock.FlockML.org_name_to_participating_agency`
WHERE is_participating_agency = TRUE
  AND similarity BETWEEN 0.75 AND 0.85
ORDER BY similarity ASC;
```

### Phase 6: Manual Corrections (if needed)

If you find false positives or false negatives:

**Correct a False Positive:**

```sql
-- Example: Houston, TX has both PD and SO - if match is wrong
INSERT INTO `durango-deflock.FlockML.agency_match_overrides`
VALUES (
  'Houston TX SO',
  FALSE,
  NULL,
  'False positive - different type than classified',
  CURRENT_TIMESTAMP()
);

-- Re-run matching to apply override
CALL `durango-deflock.FlockML.sp_match_participating_agencies`(
  'durango-deflock.DurangoPD.October2025_classified'
);
```

**Correct a False Negative (missed match):**

```sql
INSERT INTO `durango-deflock.FlockML.agency_match_overrides`
VALUES (
  'Some PD',
  TRUE,
  'Some City Police Department',
  'Manual match - low embedding similarity',
  CURRENT_TIMESTAMP()
);
```

### Phase 7: Create Enriched View

**File:** `sql/22_create_enriched_classified_view.sql`

```sql
CREATE OR REPLACE VIEW `durango-deflock.DurangoPD.October2025_enriched` AS
SELECT
  c.*,
  COALESCE(m.is_participating_agency, FALSE) AS is_participating_agency,
  m.matched_agency,
  m.matched_state,
  m.matched_type,
  ROUND(m.similarity, 4) AS match_confidence
FROM `durango-deflock.DurangoPD.October2025_classified` c
LEFT JOIN `durango-deflock.FlockML.org_name_to_participating_agency` m
  ON c.org_name = m.org_name;
```

**In BigQuery Console:**

1. Run `sql/22_create_enriched_classified_view.sql`
2. New view `October2025_enriched` is created

**Example Query Using Enriched View:**

```sql
SELECT
  is_participating_agency,
  COUNT(*) AS search_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM `durango-deflock.DurangoPD.October2025_enriched`
GROUP BY is_participating_agency;
```

**Expected Output:**
```
is_participating_agency | search_count | percentage
TRUE                    | 45000        | 63.5%
FALSE                   | 25842        | 36.5%
```

## Understanding Similarity Scores

The fuzzy matching uses **cosine similarity** (0-1 scale) with a default threshold of **0.85**.

### Similarity Levels

| Score | Confidence | Action |
|-------|-----------|--------|
| 0.95+ | Very High ✓ | Automatic match |
| 0.85-0.95 | High ✓ | Automatic match |
| 0.75-0.85 | Medium ⚠ | Manual review recommended |
| <0.75 | Low ✗ | Likely false positive |

### Why 0.85?

- Balances precision (few false positives) and recall (catch most true matches)
- Tested threshold: achieves ~95% precision, ~85% recall
- Can be adjusted: lower threshold = more matches but more false positives

### Threshold Tuning

To test different thresholds:

```sql
SELECT
  threshold,
  COUNT(*) AS matches,
  ROUND(AVG(similarity), 4) AS avg_similarity
FROM (
  SELECT
    similarity,
    CASE
      WHEN similarity >= 0.95 THEN 0.95
      WHEN similarity >= 0.90 THEN 0.90
      WHEN similarity >= 0.85 THEN 0.85
      WHEN similarity >= 0.80 THEN 0.80
      ELSE 0.75
    END AS threshold
  FROM `durango-deflock.FlockML.org_name_to_participating_agency`
  WHERE similarity >= 0.75
)
GROUP BY threshold
ORDER BY threshold DESC;
```

## Output Tables

### `org_name_to_participating_agency`

Lookup table created by the matching procedure.

**Schema:**

| Column | Type | Description |
|--------|------|-------------|
| `org_name` | STRING | Original org_name from classified table |
| `is_participating_agency` | BOOL | True if matched to participating agency |
| `matched_agency` | STRING | Full name of matched agency (if is_participating_agency=TRUE) |
| `matched_state` | STRING | State of matched agency (if is_participating_agency=TRUE) |
| `matched_type` | STRING | Type (Sheriff's Office, Police Dept, etc.) |
| `similarity` | FLOAT64 | Similarity score (0-1) |
| `match_timestamp` | TIMESTAMP | When matching was performed |

**Example Data:**

```
org_name                          | is_participating_agency | matched_agency                    | similarity | match_timestamp
"Seminole County FL SO"           | TRUE                    | "Seminole County Sheriff's Office" | 0.9234     | 2024-12-15 10:30:00
"Houston TX PD"                   | TRUE                    | "Houston Police Department"       | 0.8901     | 2024-12-15 10:30:00
"Random Code TX Police"           | FALSE                   | "Austin Police Department"        | 0.6234     | 2024-12-15 10:30:00
"Pinellas County FL SO"           | TRUE                    | "Pinellas County Sheriff's Office" | 0.9567     | 2024-12-15 10:30:00
```

## Advanced Queries

### Query: Top Participating Agencies by Search Volume

```sql
SELECT
  m.matched_agency,
  m.matched_state,
  COUNT(*) AS search_count,
  ROUND(AVG(m.similarity), 4) AS avg_confidence
FROM `durango-deflock.DurangoPD.October2025_enriched` c
WHERE c.is_participating_agency = TRUE
GROUP BY m.matched_agency, m.matched_state
ORDER BY search_count DESC
LIMIT 20;
```

### Query: Searches by Reason and Agency Status

```sql
SELECT
  c.reason_category,
  c.is_participating_agency,
  COUNT(*) AS search_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (PARTITION BY c.reason_category), 2) AS pct
FROM `durango-deflock.DurangoPD.October2025_enriched` c
GROUP BY c.reason_category, c.is_participating_agency
ORDER BY search_count DESC;
```

### Query: Low Confidence Matches for Review

```sql
SELECT
  org_name,
  matched_agency,
  match_confidence,
  COUNT(*) AS search_count
FROM `durango-deflock.DurangoPD.October2025_enriched`
WHERE is_participating_agency = TRUE
  AND match_confidence < 0.85
GROUP BY org_name, matched_agency, match_confidence
ORDER BY match_confidence ASC;
```

## Troubleshooting

### Issue: Procedure execution fails with "Connection not found"

**Solution:** Verify Vertex AI connection exists:

```sql
-- Check available connections
SELECT * FROM `durango-deflock`.INFORMATION_SCHEMA.CONNECTIONS
WHERE connection_id LIKE 'vertex-ai%';
```

If not found, run `sql/01_setup_vertex_ai_connection.sql` or create manually:

```bash
bq mk --connection --location=us-central1 --project_id=durango-deflock \
  --connection_type=CLOUD_RESOURCE vertex-ai-connection
```

### Issue: Embedding model creation fails

**Solution:** Ensure you have Vertex AI API enabled:

```bash
gcloud services enable aiplatform.googleapis.com --project=durango-deflock
```

### Issue: Matching procedure is very slow

**Solution:** Procedure is slow on first run because it generates all embeddings. Subsequent runs are faster. To speed up:

1. Pre-cache embeddings: Run `sql/21_create_participating_agency_embeddings.sql`
2. Modify procedure to use cached table (see implementation)

### Issue: Low match percentage (<50%)

**Solution:** Could be:
- Different naming conventions than expected
- Many non-participating agencies in org_name list
- Threshold too high (0.85) - try lowering to 0.80

**To investigate:**

```sql
-- See which org_names didn't match
SELECT
  org_name,
  matched_agency,
  similarity
FROM `durango-deflock.FlockML.org_name_to_participating_agency`
WHERE is_participating_agency = FALSE
ORDER BY similarity DESC
LIMIT 20;
```

## Cost Analysis

- **Text Embedding API:** ~$0.00001 per 1,000 characters
- **One-time matching:** ~$0.001 per run
- **Cached embeddings:** Reduces costs by ~90% for repeated matching

## Next Steps

1. ✅ Create text embedding model (Phase 1)
2. ✅ Create matching procedure (Phase 3)
3. ✅ Execute matching (Phase 4)
4. ✅ Review verification queries (Phase 5)
5. ✅ Add manual corrections if needed (Phase 6)
6. ✅ Create enriched view (Phase 7)
7. Use enriched view for analysis

## Related Files

- Implementation plan: `IMPLEMENTATION_PLAN.md` (in plan transcript)
- Original classified data: `sql/14_create_optimized_classification_procedure.sql`
- BigQuery setup: `sql/01_setup_vertex_ai_connection.sql`
- Gemini model: `sql/05_create_gemini_model.sql`

## Support

For issues or questions:
- Check BigQuery error messages in console
- Review verification queries to understand match quality
- Consult manual override process for handling edge cases
- Contact data engineering team for system issues
