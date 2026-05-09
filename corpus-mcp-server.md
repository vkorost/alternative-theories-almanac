# Corpus MCP Server

## What it is

An MCP (Model Context Protocol) server that provides searchable access to the ~8.3 million-word source corpus used to write this book. The corpus includes podcast transcripts, conference recordings, and book text from five primary sources in English and Russian.

## Why it exists

Eight million words is too much to read linearly. The corpus needed to be searchable during the writing process so that claims could be verified against sources, quotes could be checked, and relevant material could be pulled into each chapter from all five voices simultaneously. The MCP server made the corpus available as a set of tools that Claude Code could call during chapter assembly.

## Corpus composition

| Source | Type | Language | Items | Words |
|--------|------|----------|-------|-------|
| Antropogenez | subtitles | Russian | 588 | ~2.9M |
| Cosmic Summit | subtitles | English | 223 | ~1.7M |
| Dedunking | subtitles | English | 199 | ~900K |
| Hancock | books | English | 10 | ~940K |
| Sklyarov | books | Russian | 32 | ~1.9M |

**Total: 1,052 sources, ~8.3 million words.**

## Architecture

### Ingestion

Raw subtitle files (.srt) and book text files were chunked into segments of approximately 500-1,000 words each, preserving paragraph and heading boundaries. Each chunk was tagged with metadata: source title, channel/author, language, publication date, source type (subtitle or book), and sequential chunk index.

### Storage

All chunks are stored in a single SQLite database (`corpus.db`, ~135 MB) with two tables:

- **`sources`**: one row per video or book, with title, channel/author, language, source type, word count, chunk count, and publication date.
- **`chunks`**: one row per text chunk, with source ID, chunk index, heading (for books), and full text.

Full-text search is provided by SQLite's FTS5 extension. A virtual table `chunks_fts` indexes the text column of the chunks table, enabling fast ranked search across the entire corpus.

### MCP Server

The server is implemented as a single Python file using the `FastMCP` framework. It exposes six tools:

#### `get_stats()`
Returns a corpus overview: total sources, chunks, and words, with breakdowns by source type, language, and channel/author.

#### `list_sources(source_type?, language?)`
Lists all channels and authors with source counts and total word counts. Filterable by source type (`subtitle` or `book`) and language (`en` or `ru`).

#### `list_titles(channel_or_author?, source_type?, language?)`
Lists all video and book titles with word counts, chunk counts, and dates. Filterable by channel/author, source type, and language.

#### `get_toc(title)`
For books: returns the heading tree with chunk indices, enabling navigation to specific sections. For subtitles: returns a scope summary showing the first and last chunk preview. Supports partial title matching.

#### `search(query, language?, channel_or_author?, limit?)`
Full-text search across all chunks using FTS5. Returns matching chunks with source metadata, chunk index, and highlighted snippets. Supports FTS5 query syntax. Default limit: 10 results.

#### `read_chunks(title, start?, end?)`
Returns full text of one or more sequential chunks from a source, identified by partial title match. Used to read extended passages after a search identifies relevant chunks.

## How it was used during writing

The typical workflow for each chapter:

1. **Search** for a topic keyword (e.g., "Younger Dryas nanodiamonds") to find relevant chunks across all five sources.
2. **Read chunks** to get the full context around a search hit.
3. **Cross-reference** by searching the same topic in different languages or sources to compare how Sklyarov, Hancock, Dedunking, Antropogenez, and Cosmic Summit presenters treated the same evidence.
4. **Verify quotes** by reading the original chunk to confirm that Cyrillic quotations were transcribed accurately and that English translations matched the source.
5. **Check facts** by searching for specific claims (dates, measurements, proper nouns) to ensure consistency across chapters.

During the encyclopedic-reference conversion (V1 to V2), the corpus was used again to:
- Verify that all Cyrillic quotations were preserved correctly
- Confirm translations against original Russian text
- Check factual claims against the primary sources

## Configuration

The server is configured via `.mcp.json` in the project root and runs as a stdio-based MCP server:

```json
{
  "mcpServers": {
    "corpus": {
      "command": "python",
      "args": ["server.py"],
      "cwd": "c:/[your-path-here]"
    }
  }
}
```

## Reuse

This approach has been reused on other projects with comparable text volumes. The key insight is that FTS5 provides fast, ranked full-text search without external dependencies, and the MCP protocol makes the search tools available to Claude Code as first-class tool calls during any session. For corpora under ~50 million words, a single SQLite file is sufficient. Beyond that, a dedicated search engine (Elasticsearch, Meilisearch) would be more appropriate.
