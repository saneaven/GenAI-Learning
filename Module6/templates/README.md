# Novel Fine-Tuning Jinja2 Templates

## Overview

This collection of 25 Jinja2 templates + 1 shared macros file transforms novel database records into chat-format JSONL training examples for fine-tuning an LLM to collaborate on novel writing through tool calls.

## Files

| File | Purpose |
|------|---------|
| `_macros.jinja2` | Shared macros: call ID generation, system prompts, message builders |
| `01_setup_basic_info.jinja2` | Set up a novel project from scratch |
| `02_create_world_elements.jinja2` | Create characters, locations, organizations, lorebooks |
| `03_create_outline.jinja2` | Build hierarchical outline (outline → acts → chapters) |
| `04_write_manuscript.jinja2` | Write novel prose for each chapter |
| `05_edit_and_revise.jinja2` | Revise content using patch operations |
| `06_retrieval_and_search.jinja2` | Use keyword_search and rag_search effectively |
| `07_full_workflow.jinja2` | Complete session: setup → world-build → outline → write → revise |
| `08_character_relationship_mapping.jinja2` | Analyze and document character relationships |
| `09_continuity_check.jinja2` | Detect and fix continuity errors across chapters |
| `10_chapter_continuation.jinja2` | Continue writing from a midpoint (partial draft) |
| `11_style_tone_adjustment.jinja2` | Rewrite passages to match style/tone guidelines |
| `12_dialogue_scene_writing.jinja2` | Write dialogue-heavy scenes with distinct character voices |
| `13_subplot_management.jinja2` | Track and weave subplots across chapters |
| `14_scene_expansion.jinja2` | Expand terse passages into detailed scenes |
| `15_outline_restructuring.jinja2` | Reorder, split, or merge acts and chapters |
| `16_pov_and_perspective.jinja2` | Rewrite scenes from different POVs |
| `17_foreshadowing_insertion.jinja2` | Plant foreshadowing in earlier chapters |
| `18_chapter_summary_and_notes.jinja2` | Update outline to match written manuscript |
| `19_batch_world_building.jinja2` | Create related elements together (org + members + HQ) |
| `20_guidelines_and_style_setup.jinja2` | Establish and refine writing guidelines |
| `21_multi_chapter_revision_pass.jinja2` | Global find-and-replace across all chapters |
| `22_chapter_deletion_and_merge.jinja2` | Delete or merge chapters with outline updates |
| `23_research_before_writing.jinja2` | "Look before you leap" research pattern |
| `24_error_recovery.jinja2` | Handle and recover from failed tool calls |
| `25_comparative_analysis.jinja2` | Compare manuscript sections for thematic/tonal analysis |

## Input Format

Each template expects a novel record as a flat namespace of variables:

```json
{
  "genre": "Fantasy",
  "logline": "A reluctant mage must...",
  "characters": [...],
  "locations": [...],
  "organizations": [...],
  "lorebooks": [...],
  "outline": { "id": "", "name": "", "acts": [{
        "id": "",
        "name": "",
        "description": "",
        "content": "",
        "chapters": [
          {
            "id": "",
            "name": "",
            "description": "",
            "content": ""
          }
        ]
      }
    ]
  },
  "manuscript": { "chapters": [
      {
        "id": "",
        "name": "",
        "chapter_text": ""
      }
    ] },
  "guidelines": { "id": "", "authorNote": "" }
}
```

## Output Format

Each template produces one or more JSONL lines in OpenAI chat-completion format:

```json
{"messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}, ...]}
```

Tool calls use the standard format with stringified JSON arguments.

## Usage

```python
import json
from jinja2 import Environment, FileSystemLoader

env = Environment(loader=FileSystemLoader('./templates'))

# Load a novel record
with open('novel_record.json') as f:
    record = json.load(f)

# Render each template
for template_name in sorted(env.list_templates()):
    if template_name.startswith('_'):
        continue
    template = env.get_template(template_name)
    output = template.render(**record)
    for line in output.strip().split('\n'):
        line = line.strip()
        if line:
            parsed = json.loads(line)  # validate JSON
            print(json.dumps(parsed))  # write to JSONL
```

## Key Design Decisions

1. **Progressive context**: Template 02 builds up the system prompt as elements are created
2. **Phrase variation**: User messages are drawn from phrase banks to prevent overfitting
3. **Actual prose as ground truth**: Template 04 uses real `chapter_text` in `replace_manuscript` calls
4. **Double tojson**: Tool call arguments are stringified JSON (string within JSON)
5. **Graceful empty handling**: All templates check for empty arrays before iterating
6. **Deterministic randomness**: Phrase selection uses string length modulo, not true random, for reproducibility
