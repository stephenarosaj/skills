# Evidence & JSON Schema Specification

This document defines the standardized reporting structure and citation guidelines for subagents operating within the Adaptive Research Orchestrator. By adhering to this schema, subagents ensure that the orchestrator receives highly structured, verifiable, and hallucination-resistant information for final synthesis.

## 1. JSON Schema Specification

When a subagent completes its research task, it MUST report its findings back to the orchestrator using a JSON object that validates against the following schema. This structured data transfer prevents information loss and allows the orchestrator to programmatically assess confidence and source coverage.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Subagent Research Report",
  "type": "object",
  "properties": {
    "research_goal": {
      "type": "string",
      "description": "The specific goal or question this subagent was tasked with researching."
    },
    "findings": {
      "type": "array",
      "description": "A list of detailed analytical findings with inlined citations.",
      "items": {
        "type": "object",
        "properties": {
          "claim": {
            "type": "string",
            "description": "A concise statement of the finding."
          },
          "details": {
            "type": "string",
            "description": "Detailed explanation of the finding. MUST include inline citations in Markdown format."
          },
          "certainty": {
            "type": "string",
            "enum": ["High", "Medium", "Low"],
            "description": "Confidence level in this specific finding based on the quality and consensus of sources."
          }
        },
        "required": ["claim", "details", "certainty"]
      }
    },
    "overall_certainty": {
      "type": "string",
      "enum": ["High", "Medium", "Low"],
      "description": "The overall confidence in the completeness and accuracy of the research conducted by this subagent."
    },
    "open_questions": {
      "type": "array",
      "description": "Questions that remain unanswered or require further investigation.",
      "items": {
        "type": "string"
      }
    },
    "cited_sources": {
      "type": "array",
      "description": "A comprehensive list of all sources cited in the findings.",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "type": "string",
            "description": "A unique identifier for the source (e.g., '[1]', 'doc-a', or the URL itself), used in inline citations."
          },
          "title": {
            "type": "string",
            "description": "The title or name of the source."
          },
          "url_or_path": {
            "type": "string",
            "description": "The exact URL, absolute file path, or verifiable reference link to the source."
          },
          "reliability": {
            "type": "string",
            "enum": ["Authoritative", "Primary", "Secondary", "Questionable"],
            "description": "Assessment of the source's reliability."
          }
        },
        "required": ["id", "title", "url_or_path", "reliability"]
      }
    }
  },
  "required": ["research_goal", "findings", "overall_certainty", "open_questions", "cited_sources"]
}
```

## 2. Inlined Citation Instructions for Subagents

To guarantee that the orchestrator can trace every claim back to its origin, subagents MUST follow these strict rules when writing the `details` field of their `findings`:

1.  **Mandatory Citations**: Every factual statement, architectural detail, metric, or significant claim MUST be supported by an inline citation.
2.  **Markdown Format**: Use standard Markdown link syntax for all citations: `[Link Text](Destination)`.
    *   **Link Text**: Should be a brief, recognizable name for the source or a reference ID (e.g., `[1]`, `React Docs`, `app.py`).
    *   **Destination**: Must be the exact URL, absolute file path (using `file:///...`), or an exact match to the `id` of a source listed in the `cited_sources` array.
3.  **Proximity**: Place the inline citation immediately following the claim it supports, *before* the punctuation mark.
    *   *Correct*: `The API rate limit is 100 requests per minute [1].`
    *   *Incorrect*: `The API rate limit is 100 requests per minute. [1]`
4.  **No Hallucination**: Do not invent URLs, file paths, or sources. You may only cite sources that you have directly accessed and verified during your execution.
5.  **Examples**:
    *   *Web Source*: `The framework introduces concurrent rendering in version 18 ([React v18 Release Notes](https://react.dev/blog/2022/03/29/react-v18)).`
    *   *Local File*: `Database connections are pooled as configured in [database.ts](file:///workspace/src/config/database.ts).`
    *   *Reference ID*: `Security patches must be applied within 48 hours [[1]](#1).` (Assuming `[1]` is defined in `cited_sources`).

## 3. Integration with the Orchestrator

When the orchestrator synthesizes the final research report, it will:
1.  Parse the JSON from each subagent.
2.  Compile a unified bibliography from all `cited_sources`.
3.  Weave the `findings.details` together, automatically carrying over the inlined Markdown citations.
4.  Highlight `open_questions` for the user, prioritizing those from subagents that reported `Low` `overall_certainty`.
