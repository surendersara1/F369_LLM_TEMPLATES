<!-- Template Version: 1.0 | strands-agents: 1.x+ | Format: Structured Markdown with RFC 2119 -->

# Template 23 — Agent SOP Authoring

## Purpose
Generate a complete Agent SOP (Standard Operating Procedure) authoring framework: structured markdown SOP templates with RFC 2119 keywords (MUST, SHOULD, MAY, MUST NOT, SHOULD NOT) for defining agent obligation levels, parameterized inputs using `{{parameter}}` syntax with type annotations and defaults, SOP-to-SOP chaining for composable sub-procedures, progress tracking markers for step completion reporting, resumability patterns for interrupted workflow recovery, example SOPs for data retrieval, report generation, and multi-step approval workflows, a Python loader for parameterizing and loading SOPs as Strands Agent system prompts, a validator for SOP structural correctness, and a progress tracker for monitoring step completion.

---

## Role Definition

You are an expert AI workflow engineer specializing in Agent SOP authoring with expertise in:
- Agent SOP design: structured markdown workflows that balance flexibility and control for AI agents using natural language procedural steps
- RFC 2119 keywords: precise use of MUST, SHOULD, MAY, MUST NOT, and SHOULD NOT to define obligation levels — MUST for mandatory behavior, SHOULD for recommended behavior, MAY for optional behavior, MUST NOT and SHOULD NOT for prohibitions
- Parameterized inputs: `{{parameter_name}}` syntax with type annotations (string, int, float, bool, list, dict), default values, and REQUIRED/OPTIONAL markers for runtime substitution
- SOP chaining: SOP-to-SOP references that allow one SOP to invoke another as a sub-procedure with parameter passing, enabling composable multi-step workflows
- Progress tracking: `[PROGRESS: Step N complete]` markers that enable agents to report completion status for each procedural step, with optional metadata (record counts, durations, intermediate results)
- Resumability: patterns that allow agents to resume an SOP from a specific step after interruption, including state checkpointing and step verification on resume
- Strands Agents SDK: loading SOPs as `system_prompt` in `Agent()`, parameterizing SOPs at runtime, and combining SOPs with tool-equipped agents for executable workflows
- Error handling in SOPs: conditional error branches using IF/THEN patterns, partial result reporting, retry guidance, and escalation procedures
- SOP validation: structural correctness checks for required sections, parameter references, RFC 2119 keyword usage, and chaining references

Generate complete, production-ready SOP templates and supporting Python tooling.


---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - region where the Strands agent executing SOPs will run]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

SOP_NAME:               [REQUIRED - identifier for the SOP, used in file naming and chaining references]
                        Example: "data_retrieval", "report_generation", "approval_workflow"

SOP_PURPOSE:            [REQUIRED - one-paragraph description of what the SOP accomplishes]
                        Example: "Retrieve data from specified sources, validate completeness,
                        and generate a formatted report with summary metrics."

SOP_INPUTS:             [REQUIRED - JSON list of SOP input parameters with types and defaults]
                        Example: [
                            {
                                "name": "data_source",
                                "type": "string",
                                "required": true,
                                "description": "The data source identifier"
                            },
                            {
                                "name": "report_format",
                                "type": "string",
                                "required": false,
                                "default": "markdown",
                                "description": "Output format: markdown | html | pdf"
                            },
                            {
                                "name": "date_range",
                                "type": "string",
                                "required": false,
                                "description": "Date range for data retrieval (ISO 8601)"
                            }
                        ]

SOP_CHAINING_ENABLED:   [OPTIONAL: false]
                        When true, generates SOP-to-SOP reference patterns that allow
                        this SOP to invoke other SOPs as sub-procedures with parameter
                        passing. Also generates sop_chaining.md reference documentation.

PROGRESS_TRACKING_ENABLED: [OPTIONAL: true]
                        When true, includes [PROGRESS: Step N complete] markers after
                        each procedural step for agent completion reporting.

RESUMABILITY_ENABLED:   [OPTIONAL: false]
                        When true, includes resume point patterns that allow agents to
                        restart from a specific step after interruption, with state
                        checkpointing and step verification on resume.

EXAMPLE_SOPS:           [OPTIONAL: true]
                        When true, generates three example SOP templates:
                        data_retrieval.md, report_generation.md, approval_workflow.md

SOP_OUTPUTS:            [OPTIONAL - JSON list of SOP output descriptions]
                        Example: [
                            {
                                "name": "report",
                                "type": "string",
                                "description": "Generated report in the specified format"
                            },
                            {
                                "name": "summary",
                                "type": "dict",
                                "description": "Data retrieval summary with record counts"
                            }
                        ]
```


---

## Task

Generate complete Agent SOP authoring framework:

```
{PROJECT_NAME}-agent-sops/
├── sops/
│   ├── templates/
│   │   ├── data_retrieval.md          # Data retrieval SOP template
│   │   ├── report_generation.md       # Report generation SOP template
│   │   └── approval_workflow.md       # Multi-step approval SOP template
│   ├── sop_schema.md                 # SOP format specification
│   └── sop_chaining.md              # SOP-to-SOP reference patterns
├── loader/
│   ├── sop_loader.py                 # Load and parameterize SOPs
│   ├── sop_validator.py              # Validate SOP structure
│   └── progress_tracker.py           # Track step completion
├── examples/
│   ├── run_sop_agent.py              # Example: run agent with SOP
│   └── chain_sops.py                # Example: chain multiple SOPs
└── requirements.txt
```


**sop_schema.md**: SOP format specification document defining the canonical structure every SOP MUST follow:
- **Title**: `# SOP: {SOP_NAME}` — human-readable SOP name
- **Purpose**: One paragraph describing what the SOP accomplishes, referencing SOP_PURPOSE
- **Scope**: Defines when this SOP applies and any preconditions
- **Inputs**: Parameter list using `{{parameter_name}}` syntax with type annotation, REQUIRED/OPTIONAL marker, default value, and description. Types: string, int, float, bool, list, dict
- **Outputs**: Expected outputs with type and description
- **Procedure**: Numbered steps (`### Step N: Title`) with RFC 2119 keywords:
  - MUST — absolute requirement, agent cannot skip or deviate
  - SHOULD — recommended, agent may skip with documented justification
  - MAY — optional, agent decides based on context
  - MUST NOT — absolute prohibition, agent must never perform this action
  - SHOULD NOT — discouraged, agent may perform only with strong justification
- **Progress Markers**: `[PROGRESS: Step N complete]` after each step (when PROGRESS_TRACKING_ENABLED)
- **Error Handling**: IF/THEN conditional branches for error scenarios
- **Resume Point**: Instructions for resuming from any step after interruption (when RESUMABILITY_ENABLED)
- **Chaining References**: `[CHAIN: sop_name(param1=value1, param2=value2)]` syntax for invoking sub-SOPs (when SOP_CHAINING_ENABLED)
- Include a complete example SOP demonstrating every section

**sop_chaining.md**: SOP-to-SOP reference patterns (generated when SOP_CHAINING_ENABLED is true):
- Define the `[CHAIN: sop_name(param=value)]` syntax for invoking another SOP as a sub-procedure
- Explain parameter passing: parent SOP parameters and intermediate results can be passed to child SOPs
- Define return value handling: child SOP outputs are available to subsequent steps in the parent SOP
- Define error propagation: if a child SOP fails, the parent SOP's error handling section applies
- Include examples of linear chaining (SOP A → SOP B → SOP C) and conditional chaining (IF condition THEN chain SOP B ELSE chain SOP C)
- Document the `[CHAIN_RESULT: variable_name]` syntax for capturing child SOP output into a named variable

**data_retrieval.md**: Example SOP template for data retrieval tasks (generated when EXAMPLE_SOPS is true):
- Title: `# SOP: Data Retrieval and Validation`
- Inputs: `{{data_source}}` (string, REQUIRED), `{{date_range}}` (string, OPTIONAL), `{{max_records}}` (int, OPTIONAL, default: 1000)
- Outputs: Retrieved dataset, validation summary with record counts
- Procedure steps:
  - Step 1: Validate inputs — agent MUST validate `{{data_source}}` is a recognized source, SHOULD check connectivity before proceeding, MUST NOT proceed if unreachable after 3 retries
  - Step 2: Retrieve data — agent MUST query using `{{date_range}}`, SHOULD apply pagination for results exceeding `{{max_records}}`, MAY cache intermediate results
  - Step 3: Validate data — agent MUST check for null values and schema conformance, SHOULD report data quality metrics
  - Step 4: Return results — agent MUST return the dataset and validation summary
- Error handling: data source errors, partial retrieval, schema violations
- Progress markers after each step
- Resume point support

**report_generation.md**: Example SOP template for report generation tasks (generated when EXAMPLE_SOPS is true):
- Title: `# SOP: Report Generation`
- Inputs: `{{data}}` (dict, REQUIRED — output from data retrieval), `{{report_format}}` (string, OPTIONAL, default: markdown), `{{include_charts}}` (bool, OPTIONAL, default: false)
- Outputs: Formatted report, generation metadata
- Procedure steps:
  - Step 1: Analyze data — agent MUST compute summary statistics, SHOULD identify trends and outliers
  - Step 2: Structure report — agent MUST create sections: Executive Summary, Data Overview, Detailed Findings, Recommendations
  - Step 3: Format output — agent MUST format according to `{{report_format}}`, SHOULD include table of contents for reports exceeding 5 sections
  - Step 4: Quality check — agent SHOULD verify all data references are accurate, MUST NOT include raw data dumps exceeding 100 rows
- Demonstrates chaining: references data_retrieval SOP as upstream input source
- Progress markers and resume support

**approval_workflow.md**: Example SOP template for multi-step approval workflows (generated when EXAMPLE_SOPS is true):
- Title: `# SOP: Multi-Step Approval Workflow`
- Inputs: `{{request_type}}` (string, REQUIRED), `{{requester}}` (string, REQUIRED), `{{approval_chain}}` (list, REQUIRED — ordered list of approver roles), `{{auto_approve_threshold}}` (float, OPTIONAL, default: 1000.0)
- Outputs: Approval decision, audit trail
- Procedure steps:
  - Step 1: Validate request — agent MUST verify request completeness, MUST NOT proceed with incomplete requests
  - Step 2: Check auto-approval — agent MAY auto-approve if request value is below `{{auto_approve_threshold}}`
  - Step 3: Route to approvers — agent MUST route to each approver in `{{approval_chain}}` sequentially, SHOULD include context summary for each approver
  - Step 4: Collect decisions — agent MUST wait for each approval, SHOULD set timeout per approver, MUST NOT skip any approver in the chain
  - Step 5: Record decision — agent MUST log the final decision with full audit trail including timestamps, approver identities, and comments
- Demonstrates conditional branching, timeout handling, and audit logging
- Progress markers and resume support (critical for multi-day approval workflows)

**sop_loader.py**: Python module for loading and parameterizing SOPs:
- `load_sop(sop_path: str) -> str`: Read SOP markdown file from disk and return raw content
- `parameterize_sop(sop_content: str, params: dict) -> str`: Replace all `{{parameter_name}}` placeholders with values from the params dict. Raise `ValueError` for any REQUIRED parameter that is missing from params. Use default values for OPTIONAL parameters not provided.
- `load_and_parameterize(sop_path: str, params: dict) -> str`: Convenience function combining load and parameterize
- `extract_parameters(sop_content: str) -> list[dict]`: Parse the Inputs section and return a list of parameter definitions with name, type, required flag, and default value
- `validate_params(sop_content: str, params: dict) -> list[str]`: Return list of validation errors (missing required params, type mismatches)
- All functions include docstrings and type hints

**sop_validator.py**: Python module for validating SOP structural correctness:
- `validate_structure(sop_content: str) -> list[str]`: Check that the SOP contains all required sections (Title, Purpose, Scope, Inputs, Outputs, Procedure, Error Handling). Return list of missing section errors.
- `validate_rfc2119(sop_content: str) -> list[str]`: Check that RFC 2119 keywords (MUST, SHOULD, MAY, MUST NOT, SHOULD NOT) are used in the Procedure section. Warn if no RFC 2119 keywords are found in any step. Return list of warnings.
- `validate_parameters(sop_content: str) -> list[str]`: Check that all `{{parameter}}` references in the Procedure section are defined in the Inputs section. Return list of undefined parameter errors.
- `validate_chaining(sop_content: str, available_sops: list[str]) -> list[str]`: Check that all `[CHAIN: sop_name(...)]` references point to SOPs in the available_sops list. Return list of broken reference errors.
- `validate_progress_markers(sop_content: str) -> list[str]`: Check that `[PROGRESS: Step N complete]` markers exist after each numbered step. Return list of missing marker warnings.
- `validate_sop(sop_path: str, available_sops: list[str] = None) -> dict`: Run all validations and return a summary dict with `valid: bool`, `errors: list`, `warnings: list`
- All functions include docstrings, type hints, and line number references in error messages

**progress_tracker.py**: Python module for tracking SOP step completion:
- `ProgressTracker` class:
  - `__init__(self, sop_name: str, total_steps: int)`: Initialize tracker with SOP name and total step count
  - `start_step(self, step_number: int, step_title: str)`: Mark a step as started with timestamp
  - `complete_step(self, step_number: int, metadata: dict = None)`: Mark a step as complete with optional metadata (record counts, durations, intermediate results). Emit `[PROGRESS: Step N complete]` marker.
  - `fail_step(self, step_number: int, error: str)`: Mark a step as failed with error details
  - `get_status(self) -> dict`: Return current progress status: completed steps, current step, total steps, percentage, elapsed time
  - `get_resume_point(self) -> int`: Return the step number to resume from (first incomplete step)
  - `to_json(self) -> str`: Serialize tracker state to JSON for persistence
  - `from_json(cls, json_str: str) -> ProgressTracker`: Deserialize tracker state from JSON for resumption
- Include structured logging for each state transition

**run_sop_agent.py**: Example script demonstrating how to run a Strands Agent with an SOP as its system prompt:
- Load an SOP from `sops/templates/data_retrieval.md` using `sop_loader.load_and_parameterize()`
- Validate the SOP using `sop_validator.validate_sop()`
- Initialize a `strands.Agent()` with `BedrockModel()` and the parameterized SOP as `system_prompt`
- Add relevant tools (e.g., database query tool, file writer tool) to the agent
- Initialize a `ProgressTracker` for the SOP
- Invoke the agent with a task prompt that triggers SOP execution
- Print the agent response and progress status
- Include error handling and structured logging

**chain_sops.py**: Example script demonstrating SOP chaining (generated when SOP_CHAINING_ENABLED is true):
- Load a parent SOP (report_generation.md) and a child SOP (data_retrieval.md)
- Parameterize both SOPs with appropriate values
- Execute the child SOP first using a Strands Agent, capture the output
- Pass the child SOP output as input to the parent SOP
- Execute the parent SOP using a Strands Agent with the child output as context
- Demonstrate the full chain: data_retrieval → report_generation
- Include progress tracking across the chained execution
- Include error handling: if child SOP fails, parent SOP receives error context

**requirements.txt**: Python dependencies:
- `strands-agents>=1.0.0`
- `boto3>=1.35.0`


---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**RFC 2119 Keywords:** All SOP procedural steps MUST use RFC 2119 keywords to define obligation levels. Use MUST for mandatory agent behavior that cannot be skipped. Use SHOULD for recommended behavior the agent may skip with justification. Use MAY for optional behavior the agent decides based on context. Use MUST NOT for absolute prohibitions. Use SHOULD NOT for discouraged actions. Keywords MUST be written in ALL CAPS within procedural steps to distinguish them from regular text.

**Parameter Syntax:** All parameterized inputs use `{{parameter_name}}` double-brace syntax. Parameters MUST be defined in the Inputs section with type, required/optional marker, and description. REQUIRED parameters without values cause a validation error at load time. OPTIONAL parameters without values use their declared default.

**SOP Structure:** Every SOP MUST contain these sections in order: Title, Purpose, Scope, Inputs, Outputs, Procedure (numbered steps), Error Handling. Progress markers and Resume Point sections are included when their respective flags are enabled. The validator enforces this structure.

**Progress Tracking:** When enabled, every procedural step MUST end with a `[PROGRESS: Step N complete]` marker. Markers MAY include metadata in the format `[PROGRESS: Step N complete — {{record_count}} records retrieved]`. Progress markers enable external monitoring of agent workflow execution.

**Resumability:** When enabled, the SOP MUST include a Resume Point section that instructs the agent to verify the state of previously completed steps before continuing from the resume point. The `ProgressTracker.get_resume_point()` method returns the first incomplete step number. On resume, the agent SHOULD NOT re-execute completed steps unless their outputs are no longer valid.

**SOP Chaining:** When enabled, SOPs MAY reference other SOPs using `[CHAIN: sop_name(param1=value1)]` syntax. The loader resolves chaining references by loading and parameterizing the referenced SOP. Circular chaining references MUST NOT be created — the validator detects and rejects circular chains.

**Naming:** SOP files follow `{sop_name}.md` naming convention within the `sops/templates/` directory. Project resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- SOP project directory: `{PROJECT_NAME}-agent-sops`
- SOP template files: `sops/templates/{sop_name}.md`

**Security:** SOPs MUST NOT contain hardcoded credentials, API keys, or secrets. Sensitive values SHOULD be passed as parameters at runtime. SOPs SHOULD NOT instruct agents to bypass security controls or access unauthorized resources.

**Cost:** SOPs are pure markdown authoring artifacts with no direct AWS service costs. Cost is incurred when SOPs are loaded as Strands Agent system prompts and executed — longer SOPs consume more input tokens per agent invocation. Keep SOPs concise to minimize token usage.

---

## Code Scaffolding Hints

**SOP template structure with RFC 2119 keywords:**
```markdown
# SOP: Data Retrieval and Report Generation

## Purpose
Retrieve data from specified sources and generate a formatted report.

## Scope
This SOP applies to all data retrieval tasks involving internal databases
and external APIs.

## Inputs
- {{data_source}}: string — The data source identifier (REQUIRED)
- {{report_format}}: string — Output format: markdown | html | pdf (DEFAULT: markdown)
- {{date_range}}: string — Date range for data retrieval (OPTIONAL)

## Outputs
- Generated report in the specified format
- Data retrieval summary with record counts

## Procedure

### Step 1: Validate Inputs
The agent MUST validate that {{data_source}} is a recognized source.
The agent SHOULD check connectivity to the data source before proceeding.
The agent MUST NOT proceed if the data source is unreachable after 3 retries.

[PROGRESS: Step 1 complete]

### Step 2: Retrieve Data
The agent MUST query the data source using the provided {{date_range}}.
The agent SHOULD apply pagination for result sets exceeding 1000 records.
The agent MAY cache intermediate results for large datasets.

[PROGRESS: Step 2 complete — {{record_count}} records retrieved]

### Step 3: Generate Report
The agent MUST format the output according to {{report_format}}.
The agent SHOULD include a summary section with key metrics.
The agent SHOULD NOT include raw data dumps exceeding 100 rows.

[PROGRESS: Step 3 complete — Report generated]

## Error Handling
IF data source returns an error THEN the agent MUST log the error and
notify the user with the error details.
IF partial data is retrieved THEN the agent SHOULD generate a partial
report and indicate incomplete data.

## Resume Point
This SOP supports resumption from any step. On resume, the agent MUST
verify the state of previously completed steps before continuing.
```

**Loading SOP as Strands Agent system prompt:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel


def load_sop(sop_path: str) -> str:
    """Read SOP markdown file from disk."""
    with open(sop_path) as f:
        return f.read()


def parameterize_sop(sop_content: str, params: dict) -> str:
    """Replace {{parameter}} placeholders with provided values."""
    result = sop_content
    for key, value in params.items():
        result = result.replace("{{" + key + "}}", str(value))
    return result


# Load and parameterize the SOP
sop_content = load_sop("sops/templates/data_retrieval.md")
sop_prompt = parameterize_sop(sop_content, {
    "data_source": "sales_db",
    "report_format": "markdown",
    "date_range": "2024-10-01/2024-12-31",
})

# Create agent with SOP as system prompt
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
    temperature=0.7,
)

agent = Agent(
    model=model,
    system_prompt=sop_prompt,
    tools=[db_query_tool, report_generator_tool],
)

response = agent("Execute the data retrieval SOP for Q4 2024")
print(response)
```

**Parameter extraction and validation:**
```python
import re


def extract_parameters(sop_content: str) -> list[dict]:
    """Parse the Inputs section and return parameter definitions."""
    params = []
    # Match lines like: - {{name}}: type — description (REQUIRED) or (DEFAULT: value)
    pattern = r"-\s*\{\{(\w+)\}\}:\s*(\w+)\s*—\s*(.+?)(?:\(REQUIRED\)|\(DEFAULT:\s*(.+?)\)|\(OPTIONAL\))"
    for match in re.finditer(pattern, sop_content):
        param = {
            "name": match.group(1),
            "type": match.group(2),
            "description": match.group(3).strip(),
            "required": "REQUIRED" in match.group(0),
            "default": match.group(4).strip() if match.group(4) else None,
        }
        params.append(param)
    return params


def validate_params(sop_content: str, params: dict) -> list[str]:
    """Validate provided params against SOP input definitions."""
    errors = []
    definitions = extract_parameters(sop_content)
    for defn in definitions:
        if defn["required"] and defn["name"] not in params:
            errors.append(f"Missing required parameter: {{{{{defn['name']}}}}}")
    return errors
```

**SOP validator:**
```python
import re


REQUIRED_SECTIONS = ["Purpose", "Scope", "Inputs", "Outputs", "Procedure", "Error Handling"]
RFC_2119_KEYWORDS = ["MUST", "SHOULD", "MAY", "MUST NOT", "SHOULD NOT"]


def validate_structure(sop_content: str) -> list[str]:
    """Check that the SOP contains all required sections."""
    errors = []
    for section in REQUIRED_SECTIONS:
        if f"## {section}" not in sop_content:
            errors.append(f"Missing required section: ## {section}")
    return errors


def validate_rfc2119(sop_content: str) -> list[str]:
    """Check RFC 2119 keyword usage in the Procedure section."""
    warnings = []
    # Extract Procedure section
    proc_match = re.search(r"## Procedure\n(.*?)(?=\n## |\Z)", sop_content, re.DOTALL)
    if not proc_match:
        return ["No Procedure section found"]

    procedure_text = proc_match.group(1)
    found_keywords = [kw for kw in RFC_2119_KEYWORDS if kw in procedure_text]
    if not found_keywords:
        warnings.append("No RFC 2119 keywords found in Procedure section")
    return warnings


def validate_parameters(sop_content: str) -> list[str]:
    """Check that all {{parameter}} references are defined in Inputs."""
    errors = []
    # Find all parameter references
    all_refs = set(re.findall(r"\{\{(\w+)\}\}", sop_content))

    # Find parameters defined in Inputs section
    inputs_match = re.search(r"## Inputs\n(.*?)(?=\n## |\Z)", sop_content, re.DOTALL)
    if inputs_match:
        defined = set(re.findall(r"\{\{(\w+)\}\}", inputs_match.group(1)))
    else:
        defined = set()

    for ref in all_refs - defined:
        errors.append(f"Undefined parameter reference: {{{{{ref}}}}}")
    return errors


def validate_sop(sop_path: str, available_sops: list[str] = None) -> dict:
    """Run all validations and return summary."""
    with open(sop_path) as f:
        content = f.read()

    errors = []
    warnings = []

    errors.extend(validate_structure(content))
    warnings.extend(validate_rfc2119(content))
    errors.extend(validate_parameters(content))

    if available_sops:
        chain_refs = re.findall(r"\[CHAIN:\s*(\w+)\(", content)
        for ref in chain_refs:
            if ref not in available_sops:
                errors.append(f"Broken chaining reference: [CHAIN: {ref}(...)]")

    return {
        "valid": len(errors) == 0,
        "errors": errors,
        "warnings": warnings,
    }
```

**Progress tracker:**
```python
import json
import time
import logging

logger = logging.getLogger(__name__)


class ProgressTracker:
    """Track SOP step completion with timestamps and metadata."""

    def __init__(self, sop_name: str, total_steps: int):
        self.sop_name = sop_name
        self.total_steps = total_steps
        self.steps: dict[int, dict] = {}
        self.start_time = time.time()

    def start_step(self, step_number: int, step_title: str):
        """Mark a step as started."""
        self.steps[step_number] = {
            "title": step_title,
            "status": "in_progress",
            "started_at": time.time(),
        }
        logger.info("[SOP: %s] Step %d started: %s", self.sop_name, step_number, step_title)

    def complete_step(self, step_number: int, metadata: dict = None):
        """Mark a step as complete with optional metadata."""
        if step_number in self.steps:
            self.steps[step_number]["status"] = "complete"
            self.steps[step_number]["completed_at"] = time.time()
            self.steps[step_number]["metadata"] = metadata or {}
        logger.info("[PROGRESS: Step %d complete]", step_number)

    def fail_step(self, step_number: int, error: str):
        """Mark a step as failed."""
        if step_number in self.steps:
            self.steps[step_number]["status"] = "failed"
            self.steps[step_number]["error"] = error
        logger.error("[SOP: %s] Step %d failed: %s", self.sop_name, step_number, error)

    def get_status(self) -> dict:
        """Return current progress status."""
        completed = sum(1 for s in self.steps.values() if s["status"] == "complete")
        return {
            "sop_name": self.sop_name,
            "completed_steps": completed,
            "total_steps": self.total_steps,
            "percentage": round(completed / self.total_steps * 100, 1),
            "elapsed_seconds": round(time.time() - self.start_time, 2),
        }

    def get_resume_point(self) -> int:
        """Return the first incomplete step number."""
        for step_num in range(1, self.total_steps + 1):
            if step_num not in self.steps or self.steps[step_num]["status"] != "complete":
                return step_num
        return self.total_steps + 1  # All steps complete

    def to_json(self) -> str:
        """Serialize tracker state to JSON for persistence."""
        return json.dumps({
            "sop_name": self.sop_name,
            "total_steps": self.total_steps,
            "steps": self.steps,
            "start_time": self.start_time,
        })

    @classmethod
    def from_json(cls, json_str: str) -> "ProgressTracker":
        """Deserialize tracker state from JSON for resumption."""
        data = json.loads(json_str)
        tracker = cls(data["sop_name"], data["total_steps"])
        tracker.steps = {int(k): v for k, v in data["steps"].items()}
        tracker.start_time = data["start_time"]
        return tracker
```

**SOP chaining pattern:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel


def chain_sops(sop_chain: list[dict], model: BedrockModel, tools: list) -> dict:
    """Execute a chain of SOPs, passing outputs as inputs to the next SOP.

    Args:
        sop_chain: List of dicts with "sop_path", "params", and "prompt" keys.
        model: BedrockModel instance for all agents in the chain.
        tools: Tools available to all agents in the chain.

    Returns:
        Dict with results from each SOP in the chain.
    """
    results = {}
    previous_output = None

    for i, step in enumerate(sop_chain):
        # Load and parameterize the SOP
        sop_content = load_sop(step["sop_path"])

        # Merge previous output into params if available
        params = step.get("params", {})
        if previous_output:
            params["previous_result"] = str(previous_output)

        sop_prompt = parameterize_sop(sop_content, params)

        # Create agent with SOP as system prompt
        agent = Agent(
            model=model,
            system_prompt=sop_prompt,
            tools=tools,
        )

        # Execute the SOP
        response = agent(step.get("prompt", "Execute this SOP."))
        previous_output = str(response)
        results[f"step_{i + 1}"] = previous_output

    return results


# Example: chain data_retrieval → report_generation
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

chain_results = chain_sops(
    sop_chain=[
        {
            "sop_path": "sops/templates/data_retrieval.md",
            "params": {"data_source": "sales_db", "date_range": "2024-Q4"},
            "prompt": "Retrieve Q4 2024 sales data.",
        },
        {
            "sop_path": "sops/templates/report_generation.md",
            "params": {"report_format": "markdown"},
            "prompt": "Generate a quarterly sales report from the retrieved data.",
        },
    ],
    model=model,
    tools=[db_query_tool, report_generator_tool],
)
```

**Running an agent with SOP and progress tracking:**
```python
from strands import Agent
from strands.models.bedrock import BedrockModel

# Load and parameterize SOP
sop_content = load_sop("sops/templates/data_retrieval.md")
sop_prompt = parameterize_sop(sop_content, {
    "data_source": "sales_db",
    "report_format": "markdown",
    "date_range": "2024-10-01/2024-12-31",
})

# Validate SOP before use
validation = validate_sop("sops/templates/data_retrieval.md")
if not validation["valid"]:
    print(f"SOP validation errors: {validation['errors']}")
    exit(1)

# Initialize progress tracker
tracker = ProgressTracker(sop_name="data_retrieval", total_steps=3)

# Create agent with SOP
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=4096,
)

agent = Agent(
    model=model,
    system_prompt=sop_prompt,
    tools=[db_query_tool, report_generator_tool],
)

# Execute SOP
tracker.start_step(1, "Validate Inputs")
response = agent("Execute the data retrieval SOP for Q4 2024 sales data.")
tracker.complete_step(1)

# Check progress
status = tracker.get_status()
print(f"Progress: {status['percentage']}% ({status['completed_steps']}/{status['total_steps']})")

# Persist tracker state for resumability
tracker_json = tracker.to_json()
# Save tracker_json to DynamoDB, file, or other storage

# Resume from checkpoint
restored_tracker = ProgressTracker.from_json(tracker_json)
resume_step = restored_tracker.get_resume_point()
print(f"Resume from step: {resume_step}")
```


---

## Integration Points

- **Upstream**: None — SOPs are standalone authoring artifacts that do not depend on other templates for creation. They are pure markdown documents authored by humans or generated by this template.
- **Downstream**: `mlops/20` → Strands Agent Lambda deployment loads SOPs as system prompts for serverless agent execution. SOPs define the agent's workflow behavior within the Lambda handler.
- **Downstream**: `mlops/21` → Multi-agent patterns use SOPs as workflow definitions for individual agents within Graph, Swarm, or Workflow orchestrations. Each agent in a multi-agent pattern can operate under a different SOP.
- **Downstream**: `mlops/24` → Bedrock Prompt Management versions SOPs as managed prompts, enabling lifecycle management (draft → review → approved → deployed), A/B testing of SOP variants, and centralized SOP retrieval via the Prompt Management API.
- **Downstream**: `mlops/22` → AgentCore deployment loads SOPs as system prompts for long-running, stateful agent sessions where SOP resumability is critical.
- **Related**: `devops/04` → IAM roles are required when agents executing SOPs need permissions to access AWS resources referenced in SOP procedural steps.
