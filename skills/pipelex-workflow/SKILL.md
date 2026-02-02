---
name: pipelex-workflow
description: |
  Write and edit Pipelex workflow bundles (.plx files) - a domain-specific language for building AI pipelines. Use when:
  (1) Creating new .plx workflow files with domains, concepts, and pipes
  (2) Editing existing Pipelex bundles
  (3) Defining concepts with inline structures or Python classes
  (4) Creating PipeLLM, PipeSequence, PipeCondition, PipeExtract, PipeCompose, PipeImgGen, or PipeFunc pipes
  (5) Validating pipelines with `pipelex validate`
  Triggers: "write a pipeline", "create a .plx file", "pipelex workflow", "AI workflow", "define concepts", "pipe definition"
---

# Pipelex Workflow Bundle Guide

Write AI pipelines using the Pipelex language in `.plx` files.

## Workflow

1. Plan in natural language first, then transcribe to Pipelex
2. Write the `.plx` file (POSIX standard: empty line at end, no trailing whitespace)
3. **Generate inputs**: `.venv/bin/pipelex --no-logo build inputs my_bundle.plx`
4. **Validate**: `.venv/bin/pipelex --no-logo validate my_bundle.plx`
5. **Dry run first**: `.venv/bin/pipelex --no-logo run my_bundle.plx --dry-run --mock-inputs`
6. **Run**: `.venv/bin/pipelex --no-logo run my_bundle.plx --inputs inputs.json`
7. Iterate until validation passes

## Bundle Structure

```toml
domain = "domain_code"
description = "Domain description"  # Optional
main_pipe = "pipe_code"             # Optional - default pipe to run

[concept]
ConceptName = "Description"

[concept.StructuredConcept]
description = "Concept with fields"

[concept.StructuredConcept.structure]
field_name = "Field description"  # Simple text field (optional)
typed_field = { type = "text", description = "...", required = true }

[pipe.pipe_code]
type = "PipeLLM"
description = "What this pipe does"
inputs = { input_name = "ConceptName" }
output = "OutputConcept"
prompt = """
Your prompt here with @block_var and $inline_var
"""
```

## Naming Conventions

- **Domain**: `snake_case` (e.g., `invoice_processing`)
- **Concepts**: `PascalCase`, singular, no circumstantial adjectives (e.g., `Invoice` not `Invoices` or `LargeInvoice`)
- **Pipes**: `snake_case` (e.g., `extract_invoice`)

## Native Concepts

Use directly without defining: `Text`, `Image`, `PDF`, `Document`, `TextAndImages`, `Number`, `Page`, `JSON`, `ImgGenPrompt`

## Concept Definitions

### Simple Concept (refines native)

```toml
[concept.Landscape]
description = "A scenic outdoor photograph"
refines = "Image"
```

### Inline Structure (recommended)

```toml
[concept.Invoice]
description = "A commercial invoice"

[concept.Invoice.structure]
invoice_number = "The unique identifier"  # Optional text field
issue_date = { type = "date", description = "Issue date", required = true }
total_amount = { type = "number", description = "Total", required = true }
line_items = { type = "list", item_type = "text", description = "Items" }
```

**Field types**: `text`, `integer`, `boolean`, `number`, `date`, `list`, `dict`, `concept`

**Choices (enum-like values)**:
```toml
[concept.Order.structure]
status = { choices = ["pending", "processing", "shipped", "delivered"], description = "Order status" }
priority = { choices = ["low", "medium", "high"], required = true, description = "Priority level" }
```
This generates Python `Literal` types (e.g., `Literal["pending", "processing", "shipped", "delivered"]`), providing type-safe constrained string values.

**Concept references**:
```toml
customer = { type = "concept", concept_ref = "myapp.Customer", description = "..." }
items = { type = "list", item_type = "concept", item_concept_ref = "myapp.LineItem", description = "..." }
```

## Pipe Types

### PipeLLM - Generate text/objects with LLMs

```toml
[pipe.summarize]
type = "PipeLLM"
description = "Summarize text"
inputs = { text = "Text" }
output = "Summary"
prompt = """
Summarize this text:

@text
"""
```

**With system prompt and model settings**:
```toml
[pipe.expert_analysis]
type = "PipeLLM"
description = "Expert analysis"
inputs = { document = "Document" }
output = "Analysis"
system_prompt = "You are a financial analyst expert."
model = { model = "gpt-4o", temperature = 0.2 }
prompt = """
Analyze this document:

@document
"""
```

**Multiple outputs**:
- `output = "Idea[3]"` - exactly 3 items
- `output = "Idea[]"` - variable number

**Vision (images)**:
```toml
[pipe.analyze_image]
type = "PipeLLM"
inputs = { image = "Image" }
output = "ImageAnalysis"
prompt = "Describe this image: $image"
```

### PipeSequence - Chain pipes sequentially

```toml
[pipe.process_document]
type = "PipeSequence"
description = "Extract, summarize, translate"
inputs = { document = "PDF" }
output = "FrenchSummary"
steps = [
    { pipe = "extract_text", result = "extracted" },
    { pipe = "summarize", result = "summary" },
    { pipe = "translate_french", result = "french" }
]
```

**Batch processing** (parallel over list):
```toml
steps = [
    { pipe = "process_item", batch_over = "items", batch_as = "item", result = "processed" }
]
```

### PipeCondition - Conditional branching

```toml
[pipe.route_by_category]
type = "PipeCondition"
description = "Route based on category"
inputs = { input_data = "CategorizedInput" }
output = "Text"
expression = "input_data.category"
default_outcome = "process_medium"

[pipe.route_by_category.outcomes]
small = "process_small"
medium = "process_medium"
large = "process_large"
```

Use `default_outcome = "fail"` for strict matching.

### PipeExtract - Extract text/images from PDF/Image

```toml
[pipe.extract_document]
type = "PipeExtract"
description = "Extract content from PDF"
inputs = { document = "PDF" }
output = "Page"
```

Output is always `Page` (a `ListContent` of pages with `text_and_images` and `page_view`).

### PipeCompose - Template composition

```toml
[pipe.compose_email]
type = "PipeCompose"
description = "Compose email from template"
inputs = { customer = "Customer", deal = "Deal" }
output = "Text"
template = """
Hi $customer.name,

Following up on $deal.product_name...

@deal.details
"""
```

**Construct mode** (build structured objects):
```toml
[pipe.build_invoice]
type = "PipeCompose"
inputs = { order = "Order", customer = "Customer" }
output = "Invoice"

[pipe.build_invoice.construct]
invoice_number = { template = "INV-$order.id" }
customer_name = { from = "customer.name" }
total = { from = "order.total" }
```

**Construct field methods:**
| Method | Syntax | Use case |
|--------|--------|----------|
| `template` | `{ template = "text $var" }` | String interpolation |
| `from` | `{ from = "input.field" }` | Reference input or nested field |
| Direct value | `"string"` or `123` or `[...]` | Static/fixed values |

**Static values in construct** - assign directly without wrapping:
```toml
[pipe.build_config.construct]
name = { template = "$input.name Config" }    # Dynamic from input
source = { from = "input.source" }            # Reference input field
version = "1.0"                               # Static string
count = 5                                     # Static number
tags = ["tag1", "tag2", "tag3"]               # Static list
enabled = true                                # Static boolean
```

**Common mistake:** Do NOT use `{ value = [...] }` - just assign the value directly.

### PipeImgGen - Generate images

```toml
[pipe.generate_image]
type = "PipeImgGen"
description = "Generate image"
inputs = { img_prompt = "ImgGenPrompt" }
output = "Image"
prompt = "$img_prompt"
model = "$gen-image"
aspect_ratio = "landscape_16_9"
```

**Required fields:**
- `prompt` - Must reference the input variable (e.g., `"$img_prompt"`)
- `model` - Model preset (e.g., `"$gen-image"`, `"@default-premium"`)

**Aspect ratio values** (use enum names, not ratios):
`square`, `landscape_4_3`, `landscape_3_2`, `landscape_16_9`, `landscape_21_9`, `portrait_3_4`, `portrait_2_3`, `portrait_9_16`, `portrait_9_21`

**Common mistake:** The `prompt` field is required separately from inputs - you must explicitly reference the input variable.

### PipeFunc - Custom Python functions

```toml
[pipe.process_data]
type = "PipeFunc"
description = "Custom processing"
inputs = { data = "InputData" }
output = "ProcessedData"
function_name = "my_registered_function"
```

Function must be registered in `func_registry` and accept `working_memory: WorkingMemory`.

## Prompt Variable Syntax

- `@variable` - Block insertion (multi-line, with delimiters). Put alone on its own line.
- `$variable` - Inline insertion (short text). Use within sentences.

**Structured content is auto-expanded**: When you use `@structured_var`, Pipelex automatically formats ALL fields of the structured concept. No need to manually enumerate fields.

```toml
# GOOD - concise, auto-expands all fields
prompt = """
Based on this theme configuration, create a prompt template:

@theme
"""

# BAD - verbose, manually listing fields (unnecessary)
prompt = """
Based on this theme:
- Primary: $theme.palette.primary
- Secondary: $theme.palette.secondary
...
"""
```

Use `$var.field` only when you need a specific field inline within a sentence.

## Input Multiplicity

```toml
inputs = { doc = "Text" }        # Single item
inputs = { docs = "Text[]" }     # Variable list
inputs = { pair = "Image[2]" }   # Exactly 2 items
```

## Cross-Domain References

Same domain (no prefix): `inputs = { invoice = "Invoice" }`

Different domain (prefix required): `inputs = { invoice = "finance.Invoice" }`

## Model Configuration

**Direct model**:
```toml
model = { model = "gpt-4o", temperature = 0.7 }
```

**Preset** (defined in deck):
```toml
model = "$writing-creative"
```

## CLI Commands

**Always use `--no-logo`** to avoid wasting tokens on ASCII art output.

Run commands directly from the venv (no activation needed):

```bash
# Generate example input JSON (saved next to the .plx file as inputs.json)
.venv/bin/pipelex --no-logo build inputs my_bundle.plx
.venv/bin/pipelex --no-logo build inputs my_bundle.plx --pipe specific_pipe

# Validate a bundle
.venv/bin/pipelex --no-logo validate my_bundle.plx

# Dry run (no API calls, validates logic)
.venv/bin/pipelex --no-logo run my_bundle.plx --dry-run
.venv/bin/pipelex --no-logo run my_bundle.plx --dry-run --mock-inputs  # Auto-generate mock data

# Run with inputs
.venv/bin/pipelex --no-logo run my_bundle.plx --inputs inputs.json
.venv/bin/pipelex --no-logo run my_bundle.plx --pipe specific_pipe --inputs inputs.json
```

On Windows, use `.venv\Scripts\pipelex` instead of `.venv/bin/pipelex`.

## Common Errors

**`missing_input_variable`**: Add the missing input to the parent pipe's `inputs`. PipeSequence/PipeCondition must declare ALL inputs used by sub-pipes (except intermediate results).

**Inputs must be on one line**:
```toml
# WRONG
inputs = {
    a = "A",
    b = "B"
}

# CORRECT
inputs = { a = "A", b = "B" }
```

## Pipe Ordering

**Put controller pipes before the pipes they reference.** Place the main pipe first, then sub-pipes in execution order. This makes the workflow easier to read top-down.

## Complete Example

```toml
domain = "document_processing"
description = "Document processing workflows"
main_pipe = "process_invoice"

[concept.InvoiceData]
description = "Extracted invoice information"

[concept.InvoiceData.structure]
vendor = { type = "text", description = "Vendor name", required = true }
total = { type = "number", description = "Total amount", required = true }
items = { type = "list", item_type = "text", description = "Line items" }

# Main pipe first (controller)
[pipe.process_invoice]
type = "PipeSequence"
description = "Full invoice processing pipeline"
inputs = { document = "PDF" }
output = "InvoiceData"
steps = [
    { pipe = "extract_from_pdf", result = "pages" },
    { pipe = "analyze_invoice", result = "invoice_data" }
]

# Then sub-pipes in execution order
[pipe.extract_from_pdf]
type = "PipeExtract"
description = "Extract content from PDF"
inputs = { document = "PDF" }
output = "Page"

[pipe.analyze_invoice]
type = "PipeLLM"
description = "Extract invoice data from text"
inputs = { page = "Page" }
output = "InvoiceData"
prompt = """
Extract invoice information from this document:

@page
"""
```
