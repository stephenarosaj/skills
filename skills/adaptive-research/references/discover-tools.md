---
name: discover-tools
description: "Analyzes a research goal to autonomously discover and select the most authoritative web sources, frameworks, internal tools, and APIs required to execute the research."
argument-hint: "[Research Goal] [Context/Domain] [Known Resources (Optional)]"
---
# Tool Discovery & Selection Strategist

This skill is designed to analyze a specific research goal and actively discover the most authoritative sources, tools, frameworks, and APIs needed to accomplish it. It moves beyond merely selecting from a predefined list, actively formulating strategies to find the best available resources.

## Objective
Given a research goal and its context, identify and compile a comprehensive list of tools, authoritative web domains, internal APIs, and/or search queries that will be most effective in gathering required evidence and insights.

## 📥 Inputs
When invoking this skill, you must provide:
1. **Research Goal**: The primary objective of the research.
2. **Context/Domain**: Background information, specific constraints, or the general domain (e.g., "React frontend performance", "Python data engineering", "Internal Google infrastructure").
3. **Known Resources (Optional)**: Any starting tools or sources already identified.

## ⚙️ Methodology

1. **Analyze the Goal & Domain**:
   - Break down the research goal into core components.
   - Identify the technical domain and the types of information needed (e.g., official documentation, academic papers, community discussions, source code, internal wikis).

2. **Source Category Mapping**:
   Map the needed information to specific categories of sources:
   - **Official/Authoritative**: Documentation sites (e.g., `docs.python.org`, `react.dev`), specifications, RFCs.
   - **Academic/Research**: Google Scholar, arXiv, ResearchGate.
   - **Community/Practical**: StackOverflow, GitHub issues, specialized forums, Reddit (for sentiment/issues).
   - **Internal/Proprietary**: (If applicable) internal code search (`code_search`), internal wikis (`moma_search`), specialized internal tools.
   - **Current News**: Google News, trusted journals or news outlets covering the topic, etc.
   - **Your Own Suggestions**: If you believe other sources would be valuable, include them in the list.

3. **Active Discovery Strategy (The "How-To-Find")**:
   If the specific authoritative sources are not immediately known, formulate a plan to find them:
   - Construct precise search queries (e.g., `"web framework" performance benchmark 2024 filetype:pdf`).
   - Identify hub sites or directories that list relevant tools.

4. **Tool Selection**:
   Select the specific agent tools (e.g., `search_web`, `read_url_content`, `grep_search`, `code_search`) that will be required to interface with the identified sources.

## 📤 Output Format

Return a structured JSON object containing the discovery plan. This object will be passed to the Persona Architect to equip the research subagents.

### Output Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Discovery Plan",
  "description": "Structured output schema defining the source and tool strategy for research subagents.",
  "type": "object",
  "required": [
    "research_goal",
    "recommended_agent_tools",
    "authoritative_sources",
    "discovery_queries"
  ],
  "properties": {
    "research_goal": {
      "type": "string",
      "description": "The synthesized core objective of the research task."
    },
    "recommended_agent_tools": {
      "type": "array",
      "description": "List of actual system tools (e.g., 'search_web', 'read_url_content', 'grep_search') the subagent must be equipped with to access the identified sources.",
      "items": {
        "type": "object",
        "required": ["tool_name", "purpose"],
        "properties": {
          "tool_name": {
            "type": "string",
            "description": "Exact name of the tool."
          },
          "purpose": {
            "type": "string",
            "description": "Why this tool is necessary for this specific research goal."
          }
        }
      }
    },
    "authoritative_sources": {
      "type": "array",
      "description": "Specific domains, files, or platforms that the subagents should prioritize or restrict their searches to. This prevents lazy searching and bounds the subagents to high-signal targets.",
      "items": {
        "type": "object",
        "required": ["sources", "source_type", "justification"],
        "properties": {
          "sources": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Specific URLs, domains (e.g., 'react.dev'), absolute file paths, etc."
          },
          "source_type": {
            "type": "string",
            "description": "Category of the source (e.g., 'official documentation', 'research paper', 'internal design document', etc.)."
          },
          "justification": {
            "type": "string",
            "description": "Why these sources are considered useful for this goal."
          }
        }
      }
    },
    "discovery_queries": {
      "type": "array",
      "description": "Pre-computed, highly specific search queries (e.g., 'site:github.com nextjs memory leak') designed to jump-start the subagents' investigation and save token cycles.",
      "items": { "type": "string" }
    }
  }
}
```

## 🛠️ Execution Steps for the Discoverer
1. Review the inputs carefully.
2. If the domain is highly specialized and you are unsure of the best sources, you MAY perform a preliminary `search_web` to discover the top frameworks or authoritative sites in that space before finalizing the output.
3. Construct the JSON output ensuring it is comprehensive, targeted, and directly addresses the research goal.
