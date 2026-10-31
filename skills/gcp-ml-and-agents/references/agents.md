# Agents

An agent is a model, tools, and a loop that feeds each tool's result back to the model so it can decide the next step. Without the loop you have a chatbot that makes one call and stops.

Instructions are input to the model, not a separate component. MCP (Model Context Protocol) is one standard way of handing tools to a model, and it sits inside the tools part. An agent does not need it.

## Does it deserve an agent?

| Situation | Build |
| --- | --- |
| You can draw the flowchart | Ordinary code. It is faster, cheaper and cannot improvise |
| The steps vary by input, and a person would branch | An agent |
| You drive it, and you watch it | Use a general assistant and build nothing |
| An event triggers it, and nobody is watching | A built service, behind an API |

Headless is normal. An agent behind an API with no chat screen is the usual production shape.

## Choose the tool

| Need | Use | Who |
| --- | --- | --- |
| Out of the box, no code | Gemini Enterprise app | Business users |
| Prototype fast, low code | Agent Studio in Agent Platform (used to be Vertex AI), with Agent Garden templates | Analysts, data scientists |
| Tailored logic, legacy systems, an exact approval sequence | ADK (Agent Development Kit) | Developers |

The usual reason to leave no-code is that it cannot reach your internal systems or cannot enforce a required behavior.

## Agent Studio

Only the prototype was run here: system instructions, a few examples, and the built-in test panel.

| Piece | Does |
| --- | --- |
| System instructions | Standing rules: role, tone, limits |
| Few-shot examples | Sample inputs with the outputs you want. The strongest lever on format and style. They do not add knowledge |
| Model settings | Change how the model answers on each request |

The screen moves faster than any written steps. In one run:

- The System instructions box was hidden until the right-hand panel was expanded with the double-chevron button.
- A field the steps told me to fill no longer existed, and the example attached silently.
- Temperature and Top-P were not offered on the model given. A "Thinking level" setting was there instead. If you need them, try another model.

## ADK

Not run here: this section comes from the ADK docs. Check them, because the framework moves quickly.

```bash
pip install google-adk
adk create my_agent
```

That creates `my_agent/` with `agent.py`, `.env` and `__init__.py`.

```python
from google.adk.agents.llm_agent import Agent

def get_ticket(ticket_id: str) -> dict:
    """Fetch a support ticket by id."""
    return ticketing_api.get(ticket_id)

root_agent = Agent(
    model="MODEL_NAME",
    name="triage_agent",
    description="Triages support tickets.",
    instruction="Triage the ticket. Check for duplicates before summarizing.",
    tools=[get_ticket],
)
```

- `root_agent` is the only required element.
- `instruction` is the prompt. Code wires up the tools, and the prompt still does the reasoning.
- The docstring under each tool is not a comment. It is how the model knows what the tool does, so write it for the model.
- You never write the loop. ADK runs it.

```bash
adk run my_agent        # In the terminal
adk web --port 8000     # A local test screen, from the parent folder
```

The docs say `adk web` is not for production.

Deployment goes to Agent Runtime (used to be Agent Engine). Read the ADK deployment docs for the current command, and the Google Cloud setup page for authenticating against a project. Cloud Run is the other common target.

A manager agent can delegate to sub-agents, with explicit rules for who hands off to whom. Start with one agent, and split only when one set of instructions stops working.

LangChain is the vendor-neutral alternative, with the same shape. If you stay on one vendor, that vendor's SDK or ADK is simpler.

## Giving the model facts

A foundation model is pretrained on general data, so it does not know your policies.

| Method | Changes | Cost | Gives |
| --- | --- | --- | --- |
| Prompt design | The instructions only | Free | Better format and behavior |
| Grounding and RAG | What goes in the prompt | Moderate | Facts, with citations |
| Parameter-efficient tuning | A small set of added weights | Moderate | Consistent format and style |
| Full fine-tuning | Every weight | High | Consistent format and style |

RAG (Retrieval-Augmented Generation) retrieves the relevant documents, puts them in the prompt, and lets the model answer from them. The model is unchanged. It only helps for what is in the retrieved set, so a question outside it can still get a confident wrong answer. Not run here: grounding, RAG and tuning.

## Before shipping one

Not run here: production agents. These follow from how they work:

- Give each tool the narrowest access that works, because the model chooses when to call it.
- Have a person approve anything that cannot be undone, at least at first.
- Treat documents and tool results as data, since text in them can try to steer the model.
- Log every tool call, so you can see why it did what it did.
- Test with a fixed set of hard cases, and rerun them when the prompt or model changes. Side-by-side prompt comparison in Agent Studio helps here.
