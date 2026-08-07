# Vector Search Quick Start Guide

This guide walks you through testing the vector search feature that's been added to your app.

## What's New

Your app now has semantic search capabilities! Users can:
- Search through news articles using natural language questions
- Find relevant documents even if they don't contain exact keyword matches
- Search at two levels: full articles (faster) or article chunks (more precise)

## Files Added/Modified

### New Files
1. **`templates/search.html`** - Beautiful search UI with:
   - Natural language query input
   - Toggle between document and chunk search
   - Results display with similarity scores
   - Clean, modern design based on day-1's improved UI

2. **`VECTOR_SEARCH_GUIDE.md`** (this file) - Testing guide

### Modified Files
1. **`app.py`** - Added:
   - `sentence-transformers` import and model initialization
   - `/search` route (renders the search UI)
   - `/api/search` endpoint (performs vector search)
   - `get_embedding_model()` helper (lazy-loads the embedding model)

2. **`requirements.txt`** - Added:
   - `sentence-transformers>=2.2.0`
   - `torch>=2.0.0`

3. **`templates/index.html`** - Added navigation link to search page

4. **`README.md`** - Added comprehensive vector search documentation

## Quick Test Steps

### 1. Prerequisites

Before testing vector search, ensure you have:

✅ Run the embedding setup SQL scripts:
```sql
-- In your Lakebase Postgres database:
-- sql/02_setup_embeddings_table.sql (replace {{EMBEDDING_DIM}} with 384)
-- sql/03_setup_chunk_embeddings_table.sql (replace {{EMBEDDING_DIM}} with 384)
```

✅ Run the embeddings notebook at least once:
```python
# notebooks/ingest_ticker_news_embeddings.py
# This fetches news and generates embeddings
```

### 2. Local Testing

If running locally:

```bash
# Install new dependencies
pip install -r requirements.txt

# Start the app
python app.py
```

Navigate to:
- Main app: `http://localhost:8000/`
- Search UI: `http://localhost:8000/search`

### 3. Test Queries

Try these example queries to see semantic search in action:

#### Technology & Production
- "What's happening with Tesla's production?"
- "Show me news about chip shortages"
- "Which companies are building AI chips?"

#### Financial & Market
- "What caused the recent market volatility?"
- "Tell me about tech stock performance"
- "Are there any merger announcements?"

#### Industry Trends
- "What's the latest on electric vehicles?"
- "Show me semiconductor industry news"
- "Which companies are investing in renewable energy?"

### 4. Understanding Results

Each result shows:
- **Similarity score** (0-100%): How relevant the document is to your query
  - 70%+: Highly relevant
  - 50-70%: Moderately relevant  
  - <50%: Loosely related
- **Ticker symbol**: Which stock the article is about
- **Published date**: When the article was published
- **Title** (documents mode) or **Chunk text** (chunks mode)

### 5. Search Modes

**Full Articles (documents)**
- Searches against title + description embeddings
- Faster (fewer vectors to compare)
- Good for: "What's the overall topic coverage?"
- Returns: Complete article metadata

**Article Chunks**
- Searches against 800-character text chunks
- More precise (finds specific passages)
- Good for: "Find the exact paragraph that mentions X"
- Returns: Specific text snippets with context

## API Testing

You can also test the search API directly:

```bash
curl -X POST http://localhost:8000/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Tesla doing with production?",
    "limit": 5,
    "search_type": "documents"
  }'
```

Response format:
```json
{
  "query": "What is Tesla doing with production?",
  "search_type": "documents",
  "count": 5,
  "results": [
    {
      "id": "article-123",
      "ticker": "TSLA",
      "title": "Tesla Ramps Up Model Y Production",
      "published_utc": "2026-08-01T10:00:00Z",
      "model_name": "sentence-transformers/all-MiniLM-L6-v2",
      "similarity": 0.847
    },
    ...
  ]
}
```

## Troubleshooting

### "No results found"
**Cause**: Embeddings table is empty or the news table has no data  
**Fix**: Run the `notebooks/ingest_ticker_news_embeddings.py` notebook to fetch news and generate embeddings

### "Error loading embedding model"
**Cause**: `sentence-transformers` or `torch` not installed  
**Fix**: `pip install -r requirements.txt`

### "Relation 'ticker_news_embeddings' does not exist"
**Cause**: Embeddings tables not created yet  
**Fix**: Run the SQL setup scripts in `sql/02_setup_embeddings_table.sql` and `sql/03_setup_chunk_embeddings_table.sql`

### Slow first query
**Cause**: The embedding model loads lazily on first use  
**Expected**: First query takes 5-10 seconds, subsequent queries are fast

## Performance Tips

1. **HNSW Indexes**: The SQL setup scripts create HNSW indexes for fast similarity search. Make sure they're created:
   ```sql
   -- Check if indexes exist:
   SELECT * FROM pg_indexes WHERE tablename IN ('ticker_news_embeddings', 'ticker_news_chunk_embeddings');
   ```

2. **Embedding Cache**: The model loads once and stays in memory. Restart the app to free memory if needed.

3. **Result Limits**: Keep `limit` reasonable (5-20). Higher limits are slower but not much more useful.

## Next Steps

1. **Schedule the embeddings job**: Set up the notebook as a recurring Databricks Workflow to keep embeddings fresh (see main README)

2. **Add more tickers**: Add stocks to your watchlist - the embeddings job will automatically fetch their news

3. **Tune chunking**: Adjust `chunk_size` and `chunk_overlap` in the notebook if you want different granularity

4. **Try different models**: Change `EMBEDDING_MODEL` to a different sentence-transformers model for different trade-offs:
   - `all-MiniLM-L6-v2`: 384 dim, fastest (current)
   - `all-mpnet-base-v2`: 768 dim, more accurate
   - `BAAI/bge-base-en-v1.5`: 768 dim, state-of-the-art

## Architecture

```
User Query
    ↓
[sentence-transformers] → Query Embedding (384-dim vector)
    ↓
[pgvector in Lakebase] → Cosine Similarity Search (HNSW index)
    ↓
[Flask /api/search] → Ranked Results
    ↓
[search.html] → Beautiful UI
```

The beauty of this approach:
- **No external vector DB needed**: pgvector runs inside your Lakebase Postgres
- **Fast**: HNSW indexes make cosine similarity search O(log n)
- **Semantic**: Finds articles by meaning, not just keywords
- **Self-contained**: Everything runs in your Databricks environment

Enjoy your new semantic search feature! 🚀