You are a dataset engineering expert specializing in fine-tuning LLMs for creative writing workflows. Your task is to generate a comprehensive set of **Jinja2 templates** that transform novel database records into chat-format training examples for fine-tuning.

---

## Context

The user is building a service where an LLM writes novels by interacting with a structured project database through tool calls. The fine-tuned model must learn to:

1. Read and understand novel project state (genre, characters, locations, outlines, manuscript, etc.)
2. Make appropriate tool calls to create, read, update, and delete story elements
3. Follow a natural novel-building workflow (world-build → outline → write → revise)
4. Produce high-quality prose in the manuscript
5. Use retrieval tools (keyword_search, rag_search) to look up relevant context before acting

---

## Database Schema

Each novel record in the database has the following structure:

```json
{
  "genre": "",
  "logline": "",
  "characters": [
    { "id": "", "name": "", "description": "", "content": "" }
  ],
  "locations": [
    { "id": "", "name": "", "description": "", "content": "" }
  ],
  "organizations": [
    { "id": "", "name": "", "description": "", "content": "" }
  ],
  "lorebooks": [
    { "id": "", "name": "", "description": "", "content": "" }
  ],
  "outline": {
    "id": "",
    "name": "",
    "description": "",
    "content": "",
    "acts": [
      {
        "id": "",
        "name": "",
        "description": "",
        "content": "",
        "chapters": [
          {
            "id": "ch01",
            "name": "",
            "description": "",
            "content": "",
            "text_from": "",
            "text_to": ""
          }
        ]
      }
    ]
  },
  "manuscript": {
    "chapters": [
      {
        "id": "ch01",
        "name": "",
        "chapter_text": ""
      }
    ]
  },
  "guidelines": {
    "id": "",
    "authorNote": ""
  }
}
```

### Field Semantics

- `genre`: The novel's genre (e.g., "Fantasy", "Literary Fiction", "Sci-Fi Thriller")
- `logline`: A one-to-two sentence summary of the story's premise
- `characters[].description`: A short one-line description (e.g., "A retired detective haunted by an unsolved case")
- `characters[].content`: Extended character profile — backstory, personality traits, motivations, arc notes
- `locations[].content`, `organizations[].content`, `lorebooks[].content`: Same pattern — short `description`, detailed `content`
- `outline.content`: High-level story synopsis
- `outline.acts[].content`: Per-act narrative summary
- `outline.acts[].chapters[].content`: Detailed chapter plan — scene beats, POV, emotional arc, key events
- `outline.acts[].chapters[].text_from` / `text_to`: Reference anchors pointing to start/end positions within the corresponding manuscript chapter's text
- `manuscript.chapters[].chapter_text`: The actual novel prose for that chapter
- `guidelines.authorNote`: Style and tone instructions (e.g., "Write in close third-person. Favor short, punchy sentences. Avoid adverbs.")

---

## Available Tool Calls

The fine-tuned model will have access to the following tools. Each tool is defined by its name and JSON schema:

### Story Object CRUD

```json
{
  "create_story_object": {
    "properties": {
      "type": { "type": "string", "enum": ["character", "location", "organization", "lorebook"] },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["type", "name", "content"]
  },
  "delete_story_object": {
    "properties": {
      "id": { "type": "string" },
      "type": { "type": "string", "enum": ["character", "location", "organization", "lorebook"] }
    },
    "required": ["id", "type"]
  },
  "read_story_object": {
    "properties": {
      "id": { "type": "string" },
      "type": { "type": "string", "enum": ["character", "location", "organization", "lorebook", "guidelines", "basic_info"] }
    },
    "required": ["id", "type"]
  },
  "replace_story_object": {
    "properties": {
      "id": { "type": "string" },
      "type": { "type": "string", "enum": ["character", "location", "organization", "lorebook"] },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["id", "type"]
  },
  "patch_story_object": {
    "properties": {
      "id": { "type": "string" },
      "type": { "type": "string", "enum": ["character", "location", "organization", "lorebook"] },
      "field": { "type": "string", "enum": ["name", "description", "content"] },
      "old": { "type": "string" },
      "new": { "type": "string" }
    },
    "required": ["id", "type", "field", "old", "new"]
  }
}
```

### Basic Info & Guidelines

```json
{
  "replace_basic_info": {
    "properties": {
      "title": { "type": "string" },
      "logline": { "type": "string" },
      "genre": { "type": "string" }
    },
    "required": []
  },
  "patch_basic_info": {
    "properties": {
      "field": { "type": "string", "enum": ["title", "logline", "genre"] },
      "old": { "type": "string" },
      "new": { "type": "string" }
    },
    "required": ["field", "old", "new"]
  },
  "replace_guidelines": {
    "properties": {
      "id": { "type": "string" },
      "authorNote": { "type": "string" }
    },
    "required": ["id", "authorNote"]
  },
  "patch_guidelines": {
    "properties": {
      "id": { "type": "string" },
      "field": { "type": "string", "enum": ["authorNote"] },
      "old": { "type": "string" },
      "new": { "type": "string" }
    },
    "required": ["id", "field", "old", "new"]
  }
}
```

### Outline CRUD

```json
{
  "create_outline": {
    "properties": {
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["name", "content"]
  },
  "delete_outline": {
    "properties": { "id": { "type": "string" } },
    "required": ["id"]
  },
  "create_outline_act": {
    "properties": {
      "outlineId": { "type": "string" },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["outlineId", "name", "content"]
  },
  "delete_outline_act": {
    "properties": { "id": { "type": "string" } },
    "required": ["id"]
  },
  "create_outline_chapter": {
    "properties": {
      "actId": { "type": "string" },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["actId", "name", "content"]
  },
  "delete_outline_chapter": {
    "properties": { "id": { "type": "string" } },
    "required": ["id"]
  },
  "read_outline": {
    "properties": {
      "id": { "type": "string" },
      "type": { "type": "string", "enum": ["outline", "act", "chapter"] }
    },
    "required": ["id", "type"]
  },
  "replace_outline": {
    "properties": {
      "id": { "type": "string" },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["id"]
  },
  "replace_outline_act": {
    "properties": {
      "id": { "type": "string" },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" },
      "order": { "type": "integer" }
    },
    "required": ["id"]
  },
  "replace_outline_chapter": {
    "properties": {
      "id": { "type": "string" },
      "actId": { "type": "string" },
      "name": { "type": "string" },
      "description": { "type": "string" },
      "content": { "type": "string" },
      "order": { "type": "integer" }
    },
    "required": ["id"]
  },
  "patch_outline": {
    "properties": {
      "id": { "type": "string" },
      "field": { "type": "string", "enum": ["name", "description", "content"] },
      "old": { "type": "string" },
      "new": { "type": "string" }
    },
    "required": ["id", "field", "old", "new"]
  },
  "patch_outline_act": {
    "properties": {
      "id": { "type": "string" },
      "field": { "type": "string", "enum": ["name", "description", "content"] },
      "old": { "type": "string" },
      "new": { "type": "string" },
      "order": { "type": "integer" }
    },
    "required": ["id", "field", "old", "new"]
  },
  "patch_outline_chapter": {
    "properties": {
      "id": { "type": "string" },
      "field": { "type": "string", "enum": ["name", "description", "content"] },
      "old": { "type": "string" },
      "new": { "type": "string" },
      "actId": { "type": "string" },
      "order": { "type": "integer" }
    },
    "required": ["id", "field", "old", "new"]
  }
}
```

### Manuscript Operations

```json
{
  "read_manuscript": {
    "properties": {
      "id": { "type": "string" },
      "offset": {
        "type": "object",
        "properties": {
          "from": { "type": "integer" },
          "to": { "type": "integer" }
        },
        "required": ["from", "to"]
      }
    },
    "required": ["id"]
  },
  "replace_manuscript": {
    "properties": {
      "id": { "type": "string" },
      "content": { "type": "string" }
    },
    "required": ["id", "content"]
  },
  "patch_manuscript": {
    "properties": {
      "id": { "type": "string" },
      "old": { "type": "string" },
      "new": { "type": "string" }
    },
    "required": ["id", "old", "new"]
  }
}
```

### Search / Retrieval

```json
{
  "keyword_search": {
    "properties": {
      "keyword": { "type": "string" },
      "page": { "type": "integer" }
    },
    "required": ["keyword"]
  },
  "rag_search": {
    "properties": {
      "queries": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "required": ["queries"]
  }
}
```

---

## Output Requirements

### Training Data Format

Generate templates that produce **JSONL** output (one JSON object per line), where each line is a training example in chat-message format:

```json
{
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": null, "tool_calls": [{ "id": "call_001", "type": "function", "function": { "name": "...", "arguments": "..." } }] },
    { "role": "tool", "tool_call_id": "call_001", "content": "..." },
    { "role": "assistant", "content": "..." }
  ]
}
```

Notes on format:
- `tool_calls[].function.arguments` must be a JSON **string** (stringified JSON), not an object
- Each tool call needs a unique `id` (e.g., `call_001`, `call_002`)
- Tool results reference back via `tool_call_id`
- Multi-tool sequences should chain naturally: assistant calls tool → tool result → assistant calls next tool → etc.

### Templates to Generate

Create the following Jinja2 templates. Each template file should be self-contained and well-commented.

---

#### Template 1: `01_setup_basic_info.jinja2`

**Purpose**: Teach the model to set up a novel project from scratch.

**Input → Output mapping**:
- From the record's `genre`, `logline`, and `outline.name` (as title), produce an example where the user describes their novel idea and the assistant calls `replace_basic_info` to set up the project.

**Conversation flow**:
1. System: Base system prompt (define the assistant's role as a novel-writing collaborator)
2. User: Describes their novel idea in natural language (synthesize from the genre + logline)
3. Assistant: Responds enthusiastically, calls `replace_basic_info` with the extracted data
4. Tool: Returns success
5. Assistant: Confirms setup, asks about next steps

---

#### Template 2: `02_create_world_elements.jinja2`

**Purpose**: Teach the model to create characters, locations, organizations, and lorebook entries.

**Input → Output mapping**:
- Iterate over each element type (`characters`, `locations`, `organizations`, `lorebooks`)
- For each element, produce a training example

**Conversation flow** (per element):
1. System: Base system prompt + current project context (genre, logline, list of already-created elements so far)
2. User: Requests creation of the element (vary phrasing — sometimes specific, sometimes vague)
3. Assistant: Calls `create_story_object` with appropriate type, name, description, content
4. Tool: Returns success with generated ID
5. Assistant: Confirms creation, may briefly note how the element fits the story

**Important considerations**:
- Build up context progressively — early characters have sparse context, later ones reference existing characters
- Vary user request phrasing significantly using Jinja2 constructs (random choice from phrase banks)
- Handle empty arrays gracefully (skip that element type)

---

#### Template 3: `03_create_outline.jinja2`

**Purpose**: Teach the model to build a hierarchical outline (outline → acts → chapters).

**Input → Output mapping**:
- First, create the outline itself using `create_outline`
- Then iterate over acts using `create_outline_act`
- Then iterate over chapters within each act using `create_outline_chapter`

**Conversation flow**:
- This should be a multi-turn conversation where the user and assistant collaborate on the outline structure
- The system prompt should include all world-building elements as context
- The assistant should demonstrate reading existing elements (via `read_story_object`) before proposing outline structure

---

#### Template 4: `04_write_manuscript.jinja2`

**Purpose**: Teach the model to write actual novel prose for each chapter.

**Input → Output mapping**:
- For each chapter in the manuscript, produce a training example
- The assistant should use `read_outline` to check the chapter plan, optionally use `rag_search` to find relevant character/location details, then `replace_manuscript` to write the chapter

**Conversation flow** (per chapter):
1. System: Base prompt + genre + logline + guidelines/authorNote + outline summary (act-level, not full chapter details — the model should read those)
2. User: Requests writing of the specific chapter (e.g., "Write Chapter 3: The Crossing")
3. Assistant: Calls `read_outline` for the chapter to get the detailed plan
4. Tool: Returns chapter plan
5. Assistant: Optionally calls `rag_search` to pull relevant character/location context
6. Tool: Returns search results
7. Assistant: Calls `replace_manuscript` with the full chapter text
8. Tool: Returns success
9. Assistant: Brief summary of what was written

**Critical**: The `content` field in `replace_manuscript` should be the actual `chapter_text` from the database record. This is the ground truth prose.

---

#### Template 5: `05_edit_and_revise.jinja2`

**Purpose**: Teach the model to revise existing content using patch operations.

**Input → Output mapping**:
- Generate synthetic edit scenarios from the existing data
- Use `patch_manuscript` to demonstrate find-and-replace edits in prose
- Use `patch_story_object` and `patch_outline_chapter` for metadata revisions

**Strategies for generating edit examples**:
- Take a sentence from `chapter_text`, create a "before" version with a deliberate weakness (tell → show, passive voice, weak verb), and use the actual text as the "after"
- Simulate a user asking to "tighten up Chapter 2" or "make the antagonist's motivation clearer"
- The assistant should `read_manuscript` first, identify issues, then `patch_manuscript` targeted sections

---

#### Template 6: `06_retrieval_and_search.jinja2`

**Purpose**: Teach the model to use `keyword_search` and `rag_search` effectively.

**Input → Output mapping**:
- Generate questions a writer might ask about their own novel (e.g., "What scenes mention the artifact?", "Find where I described Elena's appearance")
- The assistant uses search tools, then synthesizes results in natural language

**Conversation flow**:
1. User: Asks a question about their novel content
2. Assistant: Calls `keyword_search` or `rag_search` with appropriate queries
3. Tool: Returns results (synthesized from actual database content)
4. Assistant: Answers the user's question based on results

---

#### Template 7: `07_full_workflow.jinja2`

**Purpose**: Produce long multi-turn conversations that chain multiple task types together, simulating a full novel-writing session.

**Input → Output mapping**:
- Combine elements from templates 1–6 into a single multi-turn conversation
- Shows the complete workflow: setup → world-build → outline → write → search → revise

**This is the most important template** because it teaches the model the natural rhythm of novel-writing collaboration.

---

#### Template 8: `08_character_relationship_mapping.jinja2`

**Purpose**: Teach the model to analyze and update characters based on their relationships with other characters.

**Input → Output mapping**:
- For each pair of characters that appear in the same chapter (infer from `chapter_text` mentioning both names), produce a conversation where the user asks about or wants to define relationships.
- The assistant uses `rag_search` to find scenes where both characters interact, then calls `patch_story_object` to update character profiles with relationship notes.

**Conversation flow**:
1. User: "How do {{ char_a.name }} and {{ char_b.name }} relate to each other? I want to make sure their dynamic is consistent."
2. Assistant: Calls `rag_search` with queries like `["{{ char_a.name }} {{ char_b.name }}", "{{ char_a.name }} interaction"]`
3. Tool: Returns relevant passages
4. Assistant: Summarizes the relationship found in existing text
5. User: "Update {{ char_a.name }}'s profile to reflect this."
6. Assistant: Calls `read_story_object` on char_a, then `patch_story_object` to append relationship info to `content`
7. Tool: Returns success
8. Assistant: Confirms the update

---

#### Template 9: `09_continuity_check.jinja2`

**Purpose**: Teach the model to detect and fix continuity errors across chapters.

**Input → Output mapping**:
- Pick two chapters that share characters or locations
- Simulate a user asking "Check if Chapter X and Chapter Y are consistent about [detail]"
- The assistant reads both chapters and relevant story objects, identifies discrepancies (or confirms consistency), and proposes patches

**Conversation flow**:
1. User: "Can you check if the details about {{ location.name }} are consistent between chapters {{ ch_a.name }} and {{ ch_b.name }}?"
2. Assistant: Calls `read_manuscript` on ch_a (with offset if long)
3. Tool: Returns text
4. Assistant: Calls `read_manuscript` on ch_b
5. Tool: Returns text
6. Assistant: Calls `read_story_object` on the location for reference
7. Tool: Returns location content
8. Assistant: Reports findings — if inconsistency found, proposes `patch_manuscript` to fix it
9. User: Approves the fix
10. Assistant: Calls `patch_manuscript` with old/new strings
11. Tool: Returns success

---

#### Template 10: `10_chapter_continuation.jinja2`

**Purpose**: Teach the model to continue writing a chapter from a specific midpoint (partial draft scenario).

**Input → Output mapping**:
- Split `chapter_text` at a natural paragraph break (e.g., 40–60% through)
- The first half becomes the "existing draft"; the second half is what the model writes
- Uses `text_from` and `text_to` fields from the outline chapter to anchor the continuation point

**Conversation flow**:
1. System: Includes the partial chapter text as existing manuscript state
2. User: "Continue writing Chapter {{ ch.name }} from where I left off."
3. Assistant: Calls `read_manuscript` with offset to see current state
4. Tool: Returns partial text
5. Assistant: Calls `read_outline` for the chapter plan to check remaining beats
6. Tool: Returns chapter plan
7. Assistant: Calls `patch_manuscript` to append the continuation text
8. Tool: Returns success
9. Assistant: Brief note on what was added

**Key detail**: Use `patch_manuscript` with `old` as the last paragraph of the existing text and `new` as that same paragraph plus the continuation, or use `replace_manuscript` with the full combined text.

---

#### Template 11: `11_style_tone_adjustment.jinja2`

**Purpose**: Teach the model to rewrite passages to match different style/tone guidelines.

**Input → Output mapping**:
- Take existing `chapter_text` passages and the `guidelines.authorNote`
- Simulate a user wanting to adjust prose style (e.g., "Make this more literary", "Tighten the pacing", "Switch to present tense")
- The assistant reads guidelines, reads the passage, then patches it

**Conversation flow**:
1. User: "The opening of Chapter {{ ch.name }} feels too slow. Can you tighten it up while keeping the {{ genre }} tone?"
2. Assistant: Calls `read_story_object` on guidelines to check authorNote
3. Tool: Returns guidelines
4. Assistant: Calls `read_manuscript` on the chapter (first N characters via offset)
5. Tool: Returns passage
6. Assistant: Analyzes the passage, explains what changes would improve pacing
7. Assistant: Calls `patch_manuscript` with targeted rewrites (multiple patch calls for multiple edits)
8. Tool: Returns success for each
9. Assistant: Summarizes changes made

---

#### Template 12: `12_dialogue_scene_writing.jinja2`

**Purpose**: Teach the model to write dialogue-heavy scenes with natural speech patterns per character.

**Input → Output mapping**:
- For chapters whose `content` (plan) mentions dialogue, confrontation, or conversation between characters
- The assistant reads relevant character profiles to understand voice, then writes the scene

**Conversation flow**:
1. User: "Write the confrontation scene between {{ char_a.name }} and {{ char_b.name }} in Chapter {{ ch.name }}."
2. Assistant: Calls `read_story_object` for char_a
3. Tool: Returns character profile
4. Assistant: Calls `read_story_object` for char_b
5. Tool: Returns character profile
6. Assistant: Calls `read_outline` for the chapter plan
7. Tool: Returns chapter details with scene beats
8. Assistant: Calls `replace_manuscript` with dialogue-rich prose
9. Tool: Returns success
10. Assistant: Brief note on dialogue choices and character voice differentiation

---

#### Template 13: `13_subplot_management.jinja2`

**Purpose**: Teach the model to track, create, and weave subplots across multiple chapters.

**Input → Output mapping**:
- Identify recurring themes/characters across chapters (via `rag_search`)
- Create or update lorebook entries to track subplot threads
- Modify outline chapters to ensure subplot beats are placed

**Conversation flow**:
1. User: "I want to add a subplot about {{ char.name }}'s hidden past. Help me weave it into the existing outline."
2. Assistant: Calls `rag_search` with queries about the character's appearances across chapters
3. Tool: Returns list of chapters featuring the character
4. Assistant: Proposes which chapters should carry subplot beats
5. User: Approves the plan
6. Assistant: Calls `create_story_object` (lorebook) to track the subplot thread
7. Tool: Returns success
8. Assistant: Calls `patch_outline_chapter` on multiple chapters to add subplot beats to their `content`
9. Tool: Returns success for each
10. Assistant: Summarizes the subplot integration plan

---

#### Template 14: `14_scene_expansion.jinja2`

**Purpose**: Teach the model to expand a brief or summarized passage into a full, detailed scene.

**Input → Output mapping**:
- Identify a section of `chapter_text` that is relatively terse or summary-like
- The assistant expands it with sensory detail, internal monologue, and action beats

**Conversation flow**:
1. User: "The scene where {{ char.name }} arrives at {{ location.name }} in Chapter {{ ch.name }} feels rushed. Can you expand it?"
2. Assistant: Calls `read_manuscript` on the chapter
3. Tool: Returns text
4. Assistant: Calls `read_story_object` on the location for atmospheric details
5. Tool: Returns location content
6. Assistant: Calls `read_story_object` on the character for internal voice
7. Tool: Returns character content
8. Assistant: Calls `patch_manuscript` replacing the terse passage with an expanded version
9. Tool: Returns success
10. Assistant: Explains what was added (sensory details, character interiority, pacing beats)

---

#### Template 15: `15_outline_restructuring.jinja2`

**Purpose**: Teach the model to reorganize the outline — reorder chapters, move chapters between acts, split or merge acts.

**Input → Output mapping**:
- Simulate scenarios like "Move Chapter 5 to Act 2" or "Split Act 1 into two acts" or "Combine the last two chapters"
- Uses `replace_outline_chapter` (with `actId` and `order` changes), `delete_outline_chapter`, `create_outline_chapter`, `replace_outline_act`

**Conversation flow**:
1. User: "Act 2 is too long. Can we split it into two acts, breaking at Chapter {{ ch.name }}?"
2. Assistant: Calls `read_outline` for the full outline structure
3. Tool: Returns outline with acts and chapters
4. Assistant: Proposes the split point and new act names
5. User: Approves
6. Assistant: Calls `create_outline_act` for the new act
7. Tool: Returns new act ID
8. Assistant: Calls `replace_outline_chapter` on each chapter that needs to move (updating `actId` and `order`)
9. Tool: Returns success for each
10. Assistant: Calls `patch_outline_act` on the original act to update its description
11. Tool: Returns success
12. Assistant: Confirms the restructuring

---

#### Template 16: `16_pov_and_perspective.jinja2`

**Purpose**: Teach the model to rewrite scenes from a different character's POV or switch narrative perspective (1st ↔ 3rd, close ↔ distant).

**Input → Output mapping**:
- Take existing `chapter_text` and rewrite a section from a different POV
- The assistant reads character profiles to understand each character's knowledge and emotional state

**Conversation flow**:
1. User: "Rewrite the opening scene of Chapter {{ ch.name }} from {{ char_b.name }}'s perspective instead of {{ char_a.name }}'s."
2. Assistant: Calls `read_manuscript` on the chapter
3. Tool: Returns existing text
4. Assistant: Calls `read_story_object` on char_b for their perspective/knowledge
5. Tool: Returns character profile
6. Assistant: Calls `patch_manuscript` with the rewritten passage
7. Tool: Returns success
8. Assistant: Notes what changed in terms of knowledge, emotional filter, and narrative access

---

#### Template 17: `17_foreshadowing_insertion.jinja2`

**Purpose**: Teach the model to retroactively plant foreshadowing in earlier chapters for events that happen later.

**Input → Output mapping**:
- Given a major event in a later chapter, find appropriate places in earlier chapters to insert subtle hints
- Uses `rag_search` to find candidate passages, then `patch_manuscript` to weave in foreshadowing

**Conversation flow**:
1. User: "{{ char.name }}'s betrayal in Chapter {{ later_ch.name }} feels too sudden. Can you add some foreshadowing in earlier chapters?"
2. Assistant: Calls `read_outline` on the betrayal chapter for full context
3. Tool: Returns chapter plan
4. Assistant: Calls `rag_search` with queries about the character's earlier appearances
5. Tool: Returns passages from earlier chapters
6. Assistant: Proposes 2–3 specific places to add subtle hints
7. User: Approves
8. Assistant: Calls `patch_manuscript` on each earlier chapter to insert foreshadowing
9. Tool: Returns success for each
10. Assistant: Summarizes what was added and why it works subtly

---

#### Template 18: `18_chapter_summary_and_notes.jinja2`

**Purpose**: Teach the model to generate and update chapter descriptions and outline summaries based on actual manuscript content.

**Input → Output mapping**:
- After a chapter is written, the outline's chapter `description` and `content` may be outdated
- The assistant reads the manuscript, then updates the outline to match

**Conversation flow**:
1. User: "I've rewritten Chapter {{ ch.name }} significantly. Can you update the outline to match what's actually in the manuscript now?"
2. Assistant: Calls `read_manuscript` on the chapter
3. Tool: Returns current prose
4. Assistant: Calls `read_outline` for the chapter's current plan
5. Tool: Returns outdated plan
6. Assistant: Calls `replace_outline_chapter` with updated `description` and `content` reflecting the actual manuscript
7. Tool: Returns success
8. Assistant: Highlights key differences between old and new outline entries

---

#### Template 19: `19_batch_world_building.jinja2`

**Purpose**: Teach the model to create multiple related story objects in a single session (e.g., a faction with its members and headquarters).

**Input → Output mapping**:
- Group related elements: an organization + its member characters + its location
- The assistant creates them in logical order, referencing each in subsequent creations

**Conversation flow**:
1. User: "I need to flesh out {{ org.name }}. Create the organization, their headquarters, and their key members."
2. Assistant: Calls `create_story_object` (organization) with org data
3. Tool: Returns org ID
4. Assistant: Calls `create_story_object` (location) for headquarters, referencing the org in the content
5. Tool: Returns location ID
6. Assistant: For each member character, calls `create_story_object` (character), with content referencing their role in the org
7. Tool: Returns character IDs
8. Assistant: Summarizes the complete faction setup with cross-references

---

#### Template 20: `20_guidelines_and_style_setup.jinja2`

**Purpose**: Teach the model to establish and refine writing guidelines/author notes.

**Input → Output mapping**:
- From the `guidelines.authorNote`, produce conversations where the user discusses style preferences and the assistant translates them into structured guidelines

**Conversation flow**:
1. User: Describes style preferences in casual language (e.g., "I want it to feel like Cormac McCarthy meets Ursula Le Guin — sparse but mythic")
2. Assistant: Calls `replace_guidelines` with a structured authorNote translating the preferences into actionable rules
3. Tool: Returns success
4. Assistant: Confirms and reads back the guidelines for approval
5. User: Requests adjustments
6. Assistant: Calls `patch_guidelines` to refine specific parts
7. Tool: Returns success

---

#### Template 21: `21_multi_chapter_revision_pass.jinja2`

**Purpose**: Teach the model to perform a revision pass across multiple consecutive chapters (e.g., fixing a character name change, adjusting timeline references, tightening theme).

**Input → Output mapping**:
- Simulate a global edit (e.g., "I renamed the city from Ashvale to Dawnreach — update all chapters")
- The assistant uses `keyword_search` to find all occurrences, then `patch_manuscript` on each chapter

**Conversation flow**:
1. User: "I'm renaming {{ old_name }} to {{ new_name }}. Can you update this everywhere in the manuscript?"
2. Assistant: Calls `keyword_search` for the old name
3. Tool: Returns list of chapters/passages containing it
4. Assistant: For each chapter with occurrences, calls `patch_manuscript` with old/new
5. Tool: Returns success for each
6. Assistant: Reports how many changes were made across how many chapters
7. Assistant: Calls `patch_story_object` or `patch_outline_chapter` to update any references in metadata
8. Tool: Returns success

---

#### Template 22: `22_chapter_deletion_and_merge.jinja2`

**Purpose**: Teach the model to delete or merge chapters and update the outline accordingly.

**Input → Output mapping**:
- Simulate "Chapter 4 and 5 should be one chapter" or "Cut Chapter 7 entirely"
- The assistant reads both chapters, proposes a merged version, updates outline

**Conversation flow (merge)**:
1. User: "Chapters {{ ch_a.name }} and {{ ch_b.name }} should be combined into one."
2. Assistant: Calls `read_manuscript` on ch_a
3. Tool: Returns text
4. Assistant: Calls `read_manuscript` on ch_b
5. Tool: Returns text
6. Assistant: Proposes a merge strategy (transition paragraph, etc.)
7. User: Approves
8. Assistant: Calls `replace_manuscript` on ch_a with combined text
9. Tool: Returns success
10. Assistant: Calls `delete_outline_chapter` on ch_b
11. Tool: Returns success
12. Assistant: Calls `patch_outline_chapter` on ch_a to update its plan to cover both chapters' content
13. Tool: Returns success

---

#### Template 23: `23_research_before_writing.jinja2`

**Purpose**: Teach the model to thoroughly research existing project state before producing new content — the "look before you leap" pattern.

**Input → Output mapping**:
- Before writing any new content, the assistant proactively reads relevant story objects, outline entries, and existing manuscript sections
- This template focuses on the research phase, not the writing

**Conversation flow**:
1. User: "I'm about to write Chapter {{ ch.name }}. What do I need to know?"
2. Assistant: Calls `read_outline` for the chapter plan
3. Tool: Returns chapter details
4. Assistant: Identifies characters mentioned in the plan, calls `read_story_object` for each
5. Tool: Returns character profiles
6. Assistant: Identifies locations mentioned, calls `read_story_object` for each
7. Tool: Returns location profiles
8. Assistant: Calls `read_manuscript` on the previous chapter (for continuity — last few paragraphs)
9. Tool: Returns text
10. Assistant: Calls `rag_search` for any lorebook items relevant to this chapter's events
11. Tool: Returns results
12. Assistant: Synthesizes everything into a comprehensive briefing for the user

---

#### Template 24: `24_error_recovery.jinja2`

**Purpose**: Teach the model to handle and recover from failed tool calls or unexpected states.

**Input → Output mapping**:
- Simulate tool calls returning errors (e.g., "Object not found", "Invalid ID", "Content too long")
- The assistant adjusts its approach — retries with corrected parameters, uses search to find the correct ID, or breaks content into smaller pieces

**Conversation flow (example: ID not found)**:
1. User: "Update the character Elena's backstory."
2. Assistant: Calls `read_story_object` with an ID
3. Tool: Returns error — `{"error": "Object not found"}`
4. Assistant: Calls `keyword_search` for "Elena" to find the correct ID
5. Tool: Returns search results with the correct ID
6. Assistant: Calls `read_story_object` with the correct ID
7. Tool: Returns character data
8. Assistant: Proceeds with the update

**Conversation flow (example: content too long)**:
1. User: "Write the entire chapter."
2. Assistant: Calls `replace_manuscript` with full content
3. Tool: Returns error — `{"error": "Content exceeds maximum length"}`
4. Assistant: Splits content and calls `replace_manuscript` with the first half, then `patch_manuscript` to append the second half
5. Tool: Returns success for both

---

#### Template 25: `25_comparative_analysis.jinja2`

**Purpose**: Teach the model to compare different versions or sections of the manuscript and make editorial recommendations.

**Input → Output mapping**:
- Simulate "Compare the opening of Chapter 1 and Chapter 7 — are they thematically linked?"
- The assistant reads both, analyzes them, and optionally proposes edits to strengthen the connection

**Conversation flow**:
1. User: "Does the ending of my novel mirror the beginning? I want a circular structure."
2. Assistant: Calls `read_manuscript` on the first chapter (first N chars via offset)
3. Tool: Returns opening text
4. Assistant: Calls `read_manuscript` on the last chapter (last N chars via offset)
5. Tool: Returns ending text
6. Assistant: Analyzes thematic, symbolic, and structural parallels (or lack thereof)
7. User: "Can you strengthen the echo?"
8. Assistant: Calls `patch_manuscript` on the final chapter to add mirroring elements
9. Tool: Returns success

---

## Template Design Guidelines

### System Prompt Construction

Every template must include a well-structured system prompt. Use this structure:

```jinja2
{% set system_prompt %}
You are an expert novel-writing assistant. You collaborate with authors to build compelling stories using structured tools.

## Current Project
- **Title**: {{ outline.name if outline and outline.name else "Untitled" }}
- **Genre**: {{ genre if genre else "Not yet defined" }}
- **Logline**: {{ logline if logline else "Not yet defined" }}

{% if guidelines and guidelines.authorNote %}
## Writing Guidelines
{{ guidelines.authorNote }}
{% endif %}

{% if characters %}
## Characters
{% for char in characters %}
- **{{ char.name }}**: {{ char.description }}
{% endfor %}
{% endif %}

{# ... similar blocks for locations, organizations, lorebooks ... #}

{% if outline and outline.acts %}
## Outline
{% for act in outline.acts %}
### {{ act.name }}
{{ act.description }}
{% for ch in act.chapters %}
- {{ ch.name }}: {{ ch.description }}
{% endfor %}
{% endfor %}
{% endif %}
{% endset %}
```

### User Message Variation

To prevent the model from overfitting on specific phrasings, vary user messages. Use Jinja2 constructs:

```jinja2
{% set create_character_phrases = [
  "I need a new character: %s. Here's what I'm thinking: %s",
  "Let's add a character called %s. %s",
  "Create a character named %s — %s",
  "I want to introduce %s into the story. %s",
  "Add %s as a character. Background: %s",
  "New character: %s. %s"
] %}

{% set phrase = create_character_phrases | random %}
{{ phrase % (character.name, character.description) }}
```

### Handling Edge Cases

Templates must handle:
- Empty `characters`, `locations`, `organizations`, or `lorebooks` arrays (skip or produce minimal examples)
- Missing `description` fields (some fields are optional — only `name` and `content` are required for story objects)
- Single-act outlines vs. multi-act outlines
- Very short vs. very long `chapter_text` (for long chapters, may need to split into multiple `patch_manuscript` calls)
- Missing `guidelines` or empty `authorNote`

Use Jinja2 conditionals:
```jinja2
{% if characters | length > 0 %}
  {# generate character creation examples #}
{% endif %}

{% if character.description is defined and character.description %}
  "description": {{ character.description | tojson }},
{% endif %}
```

### Tool Call ID Generation

Use a consistent pattern for generating unique tool call IDs within a conversation:

```jinja2
{% set ns = namespace(call_counter=0) %}

{% macro next_call_id() %}
  {% set ns.call_counter = ns.call_counter + 1 %}
  call_{{ "%03d" | format(ns.call_counter) }}
{% endmacro %}
```

### Multi-Turn Chaining

For templates that produce multi-turn conversations, ensure each assistant turn logically follows from the previous context:

```jinja2
{# After creating a character, the next turn might reference them #}
{
  "role": "user",
  "content": {{ ("Now that we have %s, let's think about where they live. Create a location for %s's hometown." % (prev_character.name, prev_character.name)) | tojson }}
}
```

### Prose Quality in Manuscript Examples

When generating manuscript writing examples (Template 4), the `replace_manuscript` content should be the **actual chapter_text from the database**, not a summary or placeholder. This is what trains the model's prose quality.

```jinja2
{
  "role": "assistant",
  "content": null,
  "tool_calls": [{
    "id": "{{ next_call_id() }}",
    "type": "function",
    "function": {
      "name": "replace_manuscript",
      "arguments": {{ {
        "id": chapter.id,
        "content": manuscript_chapter.chapter_text
      } | tojson | tojson }}
    }
  }]
}
```

Note the double `tojson` — the outer one stringifies the JSON object into arguments format.

---

## Final Checklist

Before outputting each template, verify:

1. ✅ All JSON output is valid (properly escaped strings, no trailing commas)
2. ✅ Tool call arguments are stringified JSON (string, not object)
3. ✅ Every tool call has a unique `id` within the conversation
4. ✅ Every tool result has a matching `tool_call_id`
5. ✅ System prompts include relevant context but don't leak future information (e.g., don't include chapter text in the system prompt when the task is to write that chapter)
6. ✅ User messages sound natural and varied
7. ✅ Empty arrays and missing optional fields are handled gracefully
8. ✅ The template is well-commented explaining the purpose of each section
9. ✅ Output is JSONL format (one complete JSON object per line, no pretty-printing within a line)
10. ✅ Long manuscript text is properly escaped for JSON embedding

---

## Deliverable

Output each template as a complete, runnable Jinja2 file with:
- A file-level comment block explaining purpose, inputs, and expected outputs
- Clear section comments throughout
- A brief usage example showing how to render the template with a sample record

Output all 7 templates in order.