# SQL Fuzzy Matching Implementation Files

This directory contains SQL scripts for implementing fuzzy matching between classified police search org_names and official participating agencies.

## Overview

The fuzzy matching system uses BigQuery ML text embeddings and semantic similarity to match police agency abbreviations (e.g., "Houston TX PD") to official participating agency names (e.g., "Houston Police Department").

## Files (in execution order)

### Phase 1: Model Setup

#### `sql/17_create_text_embedding_model.sql`
- **Purpose:** Create Gemini text embedding model
- **Dependencies:** Vertex AI connection (from `sql/01_setup_vertex_ai_connection.sql`)
- **Output:** Remote model `FlockML.text_embedding_model`
- **Execution Time:** ~2 seconds
- **Cost:** ~$0 (model definition only)
- **When to run:** First, before any matching

```sql
-- Command to run:
bq query < sql/17_create_text_embedding_model.sql
```

### Phase 2: Infrastructure Tables

#### `sql/19_create_agency_match_overrides.sql`
- **Purpose:** Create manual override table for edge cases
- **Dependencies:** None
- **Output:** Empty table `FlockML.agency_match_overrides`
- **Execution Time:** <1 second
- **When to run:** Before matching procedure (can be run anytime)

```sql
-- Command to run:
bq query < sql/19_create_agency_match_overrides.sql
```

### Phase 3: Matching Procedure

#### `sql/18_create_agency_matching_procedure.sql`
- **Purpose:** Main fuzzy matching stored procedure
- **Dependencies:**
  - `text_embedding_model` (from Phase 1)
  - `participatingAgencies` table (source data)
  - `agency_match_overrides` table (optional)
- **Input:** None (optimized for October2025_classified table)
- **Output:** Lookup table `FlockML.org_name_to_participating_agency`
- **Execution Time:** 3-5 minutes
- **Cost:** ~$0.001 per execution
- **When to run:** After Phase 1 setup
- **Note:** To match other tables, modify the source table reference in Step 1 of the procedure

```sql
-- Command to run:
bq query < sql/18_create_agency_matching_procedure.sql

-- Then execute:
bq query --use_legacy_sql=false <<'EOF'
CALL `durango-deflock.FlockML.sp_match_participating_agencies`(
  'durango-deflock.DurangoPD.October2025_classified'
);
EOF
```

### Phase 4: Verification & Analysis

#### `sql/20_verify_agency_matching.sql`
- **Purpose:** 10 verification queries to analyze match quality
- **Dependencies:** `org_name_to_participating_agency` table (from Phase 3)
- **Output:** Analysis queries (run individually)
- **Execution Time:** <10 seconds per query
- **Cost:** ~$0 (read-only)
- **When to run:** After matching procedure completes

```sql
-- Run individual verification queries:
bq query < sql/20_verify_agency_matching.sql
```

**Key Queries:**
1. Overall coverage statistics
2. Similarity distribution
3. Sample matches for manual review
4. High-risk matches (0.75-0.85 similarity)
5. Unmatched agencies
6. Manual overrides applied
7. Threshold tuning analysis
8. Agency type validation
9. State-based validation

### Phase 5: Performance Optimization (Optional)

#### `sql/21_create_participating_agency_embeddings.sql`
- **Purpose:** Pre-cache participating agency embeddings
- **Dependencies:** `text_embedding_model` (from Phase 1)
- **Output:** Cached table `FlockML.participating_agency_embeddings`
- **Execution Time:** ~30 seconds
- **Cost:** ~$0.0005 per execution
- **When to run:** Optional, after initial matching for faster reruns
- **Benefit:** Reduces execution time of matching procedure by ~30%

```sql
-- Command to run:
bq query < sql/21_create_participating_agency_embeddings.sql
```

**Note:** If using cached embeddings, the matching procedure should be modified to reference the cached table instead of computing embeddings on-the-fly.

### Phase 6: Integration with Classified Data

#### `sql/22_create_enriched_classified_view.sql`
- **Purpose:** Create view joining matching results with classified table
- **Dependencies:** `org_name_to_participating_agency` table (from Phase 3)
- **Output:** View `DurangoPD.October2025_enriched`
- **Execution Time:** <1 second
- **Cost:** ~$0 (view definition only)
- **When to run:** After matching procedure completes
- **Usage:** Use this view instead of classified table for analysis

```sql
-- Command to run:
bq query < sql/22_create_enriched_classified_view.sql
```

**Columns Added:**
- `is_participating_agency` (BOOL)
- `matched_agency` (STRING)
- `matched_state` (STRING)
- `matched_type` (STRING)
- `match_confidence` (FLOAT64)

## Complete Execution Workflow

### Initial Setup (One-time)

```bash
# Phase 1: Create embedding model
bq query < sql/17_create_text_embedding_model.sql

# Phase 2: Create override table
bq query < sql/19_create_agency_match_overrides.sql

# Phase 3: Create matching procedure
bq query < sql/18_create_agency_matching_procedure.sql

# Phase 5 (Optional): Cache embeddings for faster reruns
bq query < sql/21_create_participating_agency_embeddings.sql
```

### Run Matching

```bash
# Execute the stored procedure (no parameters needed)
bq query --use_legacy_sql=false <<'EOF'
CALL `durango-deflock.FlockML.sp_match_participating_agencies`();
EOF

# Check results
bq query --use_legacy_sql=false <<'EOF'
SELECT
  COUNT(DISTINCT org_name) as total_agencies,
  COUNT(DISTINCT CASE WHEN is_participating_agency THEN org_name END) as matched_agencies,
  ROUND(COUNT(DISTINCT CASE WHEN is_participating_agency THEN org_name END) * 100.0 / COUNT(DISTINCT org_name), 2) as match_pct
FROM `durango-deflock.FlockML.org_name_to_participating_agency`;
EOF
```

### Verification & Analysis

```bash
# Run verification queries
# See sql/20_verify_agency_matching.sql for individual queries

# Query 1: Coverage statistics
bq query --use_legacy_sql=false < sql/20_verify_agency_matching.sql

# Query 2: Similarity distribution (uncomment and run)
# See file for additional queries
```

### Integration with Classified Table

```bash
# Create enriched view
bq query < sql/22_create_enriched_classified_view.sql

# Now use for analysis
bq query --use_legacy_sql=false <<'EOF'
SELECT
  is_participating_agency,
  COUNT(*) as search_count
FROM `durango-deflock.DurangoPD.October2025_enriched`
GROUP BY is_participating_agency;
EOF
```

## Data Dependencies

### Input Tables

- `durango-deflock.FlockML.participatingAgencies`
  - Source of official agency names
  - Columns: `LAW ENFORCEMENT AGENCY`, `STATE`, `TYPE`, others
  - ~1,414 records

- `durango-deflock.DurangoPD.October2025_classified`
  - Classified police search data
  - Column: `org_name` (abbreviations like "Houston TX PD")
  - ~70,842 records, ~498 unique org_names

### Output Tables

- `durango-deflock.FlockML.text_embedding_model` (Remote model)
- `durango-deflock.FlockML.agency_match_overrides` (Manual corrections table)
- `durango-deflock.FlockML.org_name_to_participating_agency` (Matching results)
- `durango-deflock.FlockML.participating_agency_embeddings` (Optional cache)

### Output Views

- `durango-deflock.DurangoPD.October2025_enriched` (Classified + matching results)

## Performance Characteristics

### Execution Times

| Operation | Time | Cost |
|-----------|------|------|
| Create embedding model | 2s | $0 |
| Create procedure | 1s | $0 |
| Run matching (first time) | 3-5m | $0.001 |
| Run matching (with cache) | 1-2m | $0.0005 |
| Verification queries | <10s | $0 |
| Create enriched view | <1s | $0 |

### Scaling

- **500 unique org_names:** ~3 minutes
- **1,400 participating agencies:** ~1 minute per embedding batch
- **Embeddings:** 768-dimensional vectors

## Key Parameters

### Similarity Threshold

**Default: 0.85**

- Controls precision vs recall trade-off
- Similarities ≥0.85: Automatic match
- Similarities 0.75-0.85: Manual review recommended
- Can be adjusted in the matching procedure

### Manual Overrides

The `agency_match_overrides` table allows:
- Correcting false positives (mark as not matching)
- Correcting false negatives (mark as matching)
- Custom matches with full control

## Troubleshooting

### "Connection not found" error

Ensure Vertex AI connection exists:

```bash
bq ls --connection --project_id=durango-deflock
```

Create if missing:

```bash
bq mk --connection --location=us-central1 --project_id=durango-deflock \
  --connection_type=CLOUD_RESOURCE vertex-ai-connection
```

### "Permission denied" when calling procedure

Ensure service account has `Vertex AI User` role:

```bash
gcloud projects add-iam-policy-binding durango-deflock \
  --member=serviceAccount:SERVICE_ACCOUNT_EMAIL \
  --role=roles/aiplatform.user
```

### Slow execution

- First run generates all embeddings (normal)
- Run `sql/21_create_participating_agency_embeddings.sql` to cache
- Subsequent runs will be ~2x faster

### Low match percentage

Check similarity distribution:

```sql
SELECT
  CASE
    WHEN similarity >= 0.95 THEN '0.95+'
    WHEN similarity >= 0.90 THEN '0.90-0.95'
    WHEN similarity >= 0.85 THEN '0.85-0.90'
    ELSE '<0.85'
  END as bucket,
  COUNT(*) as count
FROM `durango-deflock.FlockML.org_name_to_participating_agency`
GROUP BY bucket;
```

If most matches are <0.85, try lowering threshold in matching procedure.

## Related Files

- **Plan:** `/IMPLEMENTATION_PLAN.md` (in plan transcript)
- **Guide:** `/FUZZY_MATCHING_GUIDE.md` (comprehensive implementation guide)
- **Vertex AI Setup:** `sql/01_setup_vertex_ai_connection.sql`
- **Gemini Model:** `sql/05_create_gemini_model.sql`
- **Classification:** `sql/14_create_optimized_classification_procedure.sql`

## Next Steps

1. ✅ Run Phase 1-2 setup scripts
2. ✅ Run Phase 3 matching procedure
3. ✅ Review Phase 4 verification queries
4. ✅ Apply manual corrections if needed
5. ✅ Run Phase 6 enriched view
6. Use for analysis and reporting

## Support

For questions about:
- **BigQuery ML:** See BigQuery documentation
- **Text embeddings:** See Vertex AI Embeddings docs
- **Cosine similarity:** Standard vector similarity metric (0-1 scale)
- **This implementation:** See FUZZY_MATCHING_GUIDE.md
