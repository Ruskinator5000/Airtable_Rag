---
name: rag-workflow-generator
description: This skill should be used when the user asks to "generate a RAG workflow", "create a new RAG agent workflow", "build an n8n RAG workflow for a client", "customize the RAG template", or needs to create a client-specific RAG workflow with custom Google Drive folder, table prefixes, and system prompts.
---

# RAG Workflow Generator for ClearLight AI

## Purpose

Generate customized n8n RAG (Retrieval Augmented Generation) agent workflows for ClearLight AI clients. This skill automates the process of creating production-ready workflow JSON files from a standard template by applying client-specific configurations including Google Drive folder IDs, database table prefixes, and AI agent system prompts.

## When to Use This Skill

Use this skill when creating a new RAG workflow for a ClearLight AI client. The skill handles all necessary customizations to ensure the workflow is ready to import into the shared n8n Railway instance with properly configured:

- Google Drive data source connections
- Supabase database table names with client-specific prefixes
- AI agent behavior and personality
- ClearLight AI standard credentials

## How It Works

The skill performs template-based workflow generation by:

1. Gathering required client-specific information through interactive prompts
2. Loading the standardized RAG workflow template from assets
3. Applying systematic replacements across all workflow nodes
4. Writing a ready-to-import n8n workflow JSON file

## Required Information

### 1. Client/Project Prefix

A short identifier used to namespace database tables in the shared Supabase instance. This prefix prevents table name conflicts between different client RAG agents.

**Format requirements:**
- Lowercase letters and underscores only
- No spaces or special characters
- Short and memorable (e.g., "acme", "client_xyz", "ttolman")

**Example usage:**
- Prefix: `ttolman`
- Results in tables: `ttolman_documents`, `ttolman_document_metadata`
- Results in function: `match_ttolman_documents`

### 2. Google Drive Folder ID

The unique identifier for the Google Drive folder containing the client's source documents. The workflow monitors this folder for document updates and processes all files for RAG ingestion.

**How to find the folder ID:**
- Open the target folder in Google Drive
- Copy the ID from the URL: `https://drive.google.com/drive/folders/[FOLDER_ID]`
- Example: `1m2hXNeJQDxVGRmnHAW6oDgVRGOxbNESO`

**Used in three workflow nodes:**
- Google Drive Trigger (monitors for file updates)
- Search files and folders (initial file discovery)
- Search files and folders1 (manual execution path)

### 3. AI Agent System Prompt

The instructions that define the AI agent's behavior, personality, and response format. This prompt shapes how the agent interacts with users and utilizes the RAG-retrieved context.

**Prompt structure should include:**

```
# ROLE
[Define the agent's identity and purpose]

# TOOL USE
[Specify when and how to use RAG search and other tools]
- Use RAG search for retrieving relevant document chunks
- Access file metadata for context
- Prioritize accuracy over speculation

# OUTPUT FORMAT
[Define response structure and tone]
- Be concise and helpful
- Cite sources when referencing specific documents
- Maintain [client's preferred tone]
```

**Example for a dog training business:**
```
# ROLE
You are an expert dog training assistant for Doggy Dan's program. Help users with training questions based on the official program materials.

# TOOL USE
- Use RAG search to find relevant training techniques
- Reference specific program modules when applicable
- If information isn't in the knowledge base, acknowledge the limitation

# OUTPUT FORMAT
- Provide clear, actionable training advice
- Use a friendly, encouraging tone
- Include relevant examples from the program materials
```

## Implementation Approach

**Use Node.js for JSON manipulation** - The workflow is a large JSON file (2300+ lines) requiring careful handling:

```javascript
const fs = require('fs');
const template = fs.readFileSync('~/.claude/skills/rag-workflow-generator/assets/template.json', 'utf8');

// Escape system prompt for JSON (critical!)
const escapedPrompt = systemPrompt.replace(/\n/g, '\\n').replace(/"/g, '\\"');

let workflow = template;
workflow = workflow.replace(/pattern/g, 'replacement');

// Validate before writing
JSON.parse(workflow);
fs.writeFileSync('output.json', workflow);
```

**Critical: JSON String Escaping**
- System prompts contain newlines that MUST be escaped as `\n`
- Use `.replace(/\n/g, '\\n')` to convert literal newlines to escaped format
- Validate JSON with `JSON.parse()` before writing output
- Without proper escaping, the JSON will be invalid and fail to import

**Why Node.js over bash/sed:**
- Handles large files reliably without permission issues
- Better control over string escaping for JSON
- Can validate JSON structure before writing
- Easier to debug when issues occur

## Workflow Generation Process

### Step 1: Gather Client Information

Prompt the user for all three required pieces of information:
- Client/project prefix
- Google Drive folder ID
- System prompt text

Validate the prefix format (lowercase, underscores only) before proceeding.

### Step 2: Prepare Node.js Script

Create a Node.js script using Bash heredoc that will:
1. Load template from `~/.claude/skills/rag-workflow-generator/assets/template.json`
2. Escape the system prompt for JSON (newlines → `\n`)
3. Apply all replacements using `.replace()` methods
4. Validate JSON with `JSON.parse()`
5. Write output to `/Users/stephen/Code/clai-agent-skills/generated-workflows/{prefix}_rag_workflow.json`

### Step 3: Apply Table Prefix Replacements (in Node.js script)

The template uses `[PREFIX]_` placeholders that need to be replaced with the actual client prefix:

```javascript
// Replace all [PREFIX]_ placeholders with actual prefix
workflow = workflow.replace(/\[PREFIX\]_/g, `${prefix}_`);
```

**Simple and clean:** Single replacement handles all table names, function names, and SQL references throughout the entire workflow.

**This affects all nodes that reference:**
- `[PREFIX]_document_metadata` → `{prefix}_document_metadata`
- `[PREFIX]_documents` → `{prefix}_documents`
- `[PREFIX]_n8n_chat_histories` → `{prefix}_n8n_chat_histories`
- `match_[PREFIX]_documents` → `match_{prefix}_documents`
- `[PREFIX]_documents_embedding_idx` → `{prefix}_documents_embedding_idx`

### Step 4: Apply Google Drive Folder ID (in Node.js script)

The template uses `[GOOGLE_DRIVE_FOLDER_ID]` placeholders for the folder ID:

```javascript
// Replace folder ID placeholder
workflow = workflow.replace(/\[GOOGLE_DRIVE_FOLDER_ID\]/g, folderId);
```

**Simple replacement:** Updates all Google Drive nodes with the client's folder ID.

**Affects nodes:** Google Drive Trigger, Search files and folders, First Ingest

### Step 5: Apply System Prompt (in Node.js script)

**Critical:** Must escape newlines before insertion:

```javascript
// Escape system prompt for JSON
const escapedPrompt = systemPrompt.replace(/\n/g, '\\n').replace(/"/g, '\\"');

// Replace in AI Agent node
workflow = workflow.replace(/"systemMessage": "=[^"]*"/, `"systemMessage": "=${escapedPrompt}"`);
```

**System Prompt Template:** The new template includes comprehensive system prompt handling. Here's the structure expected:

```
# ROLE
[Agent identity and core purpose]

# TOOL USE
[When and how to use RAG search, tools, and data sources]

# OUTPUT FORMAT
[Response structure, tone, and formatting guidelines]

# [Additional sections as needed]
[Database integration notes, special instructions, boundaries]
```

**Note about system prompts:**
The default template includes a generic system prompt. For production use, clients often provide detailed 500-1000+ line system prompts covering their specific domain, methodologies, tone of voice, and knowledge bases. These comprehensive prompts define the agent's personality and expertise.

### Step 6: Update Workflow Name (in Node.js script)

```javascript
workflow = workflow.replace('"name": "Client Name | RAG Agent"', `"name": "${clientName} | RAG Agent"`);
```

### Step 7: Optional - Update LLM Model (in Node.js script)

The template defaults to `google/gemini-3-flash-preview` via OpenRouter. To use a different model:

```javascript
// Replace LLM model if needed (optional)
// Default: "google/gemini-3-flash-preview"
// Options: "anthropic/claude-haiku-4.5", "anthropic/claude-opus-4.5", etc.
workflow = workflow.replace(/"model": "google\/gemini-3-flash-preview"/g, `"model": "${llmModel}"`);
```

**Note:** LLM model change is optional. Keep the default if unsure.

### Step 8: Validate and Write Output

**Always validate JSON before writing:**

```javascript
// Validate JSON structure
try {
  JSON.parse(workflow);
  console.log('✓ JSON validation passed');
} catch(e) {
  console.error('✗ JSON validation failed:', e.message);
  process.exit(1);
}

// Write output
fs.writeFileSync(`/Users/stephen/Code/clai-agent-skills/generated-workflows/${prefix}_rag_workflow.json`, workflow);
```

Create the output directory if it doesn't exist. If validation fails, the script should exit with an error before writing.

## Workflow Enhancements (v2 - 2026-01-11)

The updated template includes important improvements to the data ingestion pipeline:

### New Nodes

**1. Get existing documents metadata** (Supabase operation)
- Runs alongside "Search files and folders" when workflow executes manually
- Retrieves all documents currently in the metadata table
- Prevents duplicate ingestion by comparing against drive files
- Supabase table: `{prefix}_document_metadata`

**2. First Ingest** (Google Drive operation)
- Provides alternative ingestion path for fresh workflows
- Can be triggered independently for initial document upload
- Uses the same Google Drive folder configuration

### Improved Data Sync Logic

The workflow now supports bidirectional file synchronization:
- **Files to ingest:** Present in Google Drive but not in database
- **Files to delete:** Present in database but not in Google Drive
- **Smart comparison:** "Compare file ids" node identifies differences
- **Parallel processing:** Filter files(Ingest) and Filter files(Delete) run simultaneously

### Postgres Chat Memory Enhancement

The Postgres Chat Memory node now includes explicit table naming:
```javascript
"tableName": "{prefix}_n8n_chat_histories"
```

This ensures conversation history is properly namespaced when multiple RAG agents share the same Supabase instance.

### Vector Store Tool Descriptions

Tool descriptions in Vector Search Supabase nodes automatically reference client-specific table names:
- Original: "retrieve full documents text"
- Generated: "retrieve full {prefix}_documents text"

This provides clearer context to the AI agent about which dataset it's querying.

## Usage Example

**User request:**
> "Create a new RAG workflow for Tyler Tolman with prefix 'ttolman', folder ID '1m2hXNeJQDxVGRmnHAW6oDgVRGOxbNESO', and make it a health coaching assistant"

**Skill execution:**
1. Confirm prefix: `ttolman`
2. Confirm folder ID: `1m2hXNeJQDxVGRmnHAW6oDgVRGOxbNESO`
3. Draft system prompt for health coaching context (or ask user to provide)
4. Load template and apply all replacements
5. Write: `generated-workflows/ttolman_rag_workflow.json`
6. Confirm completion with import instructions

## Output and Next Steps

After generating the workflow file, inform the user:

1. **File location:** Full path to the generated workflow JSON
2. **Import instructions:**
   - Open n8n on Railway
   - Click "+" → "Import from File"
   - Select the generated JSON file
   - Run the table creation nodes first
   - Configure the Google Drive trigger schedule if needed
3. **Verification checklist:**
   - Tables created in Supabase with correct prefix
   - Google Drive connection working
   - Initial document ingestion successful
   - AI agent responds appropriately

## Important Notes

### Credentials

All Postgres/Supabase nodes are pre-configured with "ClearLightAI Supabase" credentials. Do not modify these during generation.

### Shared Infrastructure

The workflow uses ClearLight AI's shared infrastructure:
- Single n8n instance on Railway (all workflows)
- Single self-hosted Supabase on Railway (shared database with prefixed tables)
- Shared Google Drive credential "ClearlightAI"

### Row Level Security (RLS)

Both tables are created with RLS enabled by default:
- `ALTER TABLE document_metadata ENABLE ROW LEVEL SECURITY;`
- `ALTER TABLE documents ENABLE ROW LEVEL SECURITY;`

This provides security at the database layer. Note that no policies are created by default, which means only the service role can access the tables. Add RLS policies after table creation if specific access patterns are needed.

### Table Prefix Best Practices

- Use client name or project abbreviation
- Keep it short (under 15 characters)
- Use underscores to separate words if needed
- Avoid changing prefix after deployment (requires data migration)

### Tables Created

Each client workflow creates three prefixed tables:
- `{prefix}_document_metadata` - Stores document file information (title, URL, type, timestamps)
- `{prefix}_documents` - Stores vector embeddings and document chunks
- `{prefix}_n8n_chat_histories` - Stores conversation history for the AI agent

All three tables have Row Level Security (RLS) enabled automatically.

### System Prompt Iteration

The system prompt can be refined after deployment by:
1. Opening the workflow in n8n
2. Editing the AI Agent node
3. Updating the system message
4. Saving and testing

## Troubleshooting

**Invalid prefix error:**
- Ensure only lowercase letters and underscores
- No spaces, hyphens, or special characters

**Google Drive folder not found:**
- Verify folder ID is correct
- Ensure folder is shared with "ClearlightAI" Google account
- Check folder permissions

**Table creation fails:**
- Verify prefix doesn't conflict with existing tables
- Check Supabase connection is active
- Ensure pgvector extension is enabled

## Template Asset

The workflow template is stored in:
```
~/.claude/skills/rag-workflow-generator/assets/template.json
```

This template is maintained separately from the skill and should be updated when the standard ClearLight AI RAG architecture changes.
