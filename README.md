# agentic_mkt_assistant
Project of an AI agentic flow for brainstorm-based marketing assistance. Through LangGraph framework and AutoGen, the main script presents a flow integrating Tavily tools, an AI agentic marketing team and the concept of human-in-the-loop (you can participate in the discussion to generate ideas and have insights).  

# AI Marketing Brainstorm Assistant (LangGraph + AutoGen + Human-in-the-Loop)

An agentic workflow for brainstorm-based marketing assistance. Using the LangGraph framework and AutoGen, the notebook combines Tavily web research tools, an AI marketing team and the human-in-the-loop concept: after every round you read the team's proposals and either give a new direction, which starts another round of research and ideas with the whole history carried over, or end the session.

The notebook ships configured for a small Brazilian manufacturer of porcelain bathroom niches, countertops and sinks, but the business context lives in a few text prompts and can be replaced for any company (see [Customizing for your business](#customizing-for-your-business)).

## Reference

The inspiration for this project came from the Udemy course *Master Large Language Models, Retrieval Augmented Generation, LangGraph, MCP, CrewAI, AutoGen, N8N, & OpenAI Agents SDK*, by [Prof. Ryan Ahmed, PhD, MBA](https://www.udemy.com/user/ryan-ahmed/), which was used as a learning reference. The logical structure of some blocks of the tool-calling research agent (tools list, `make_call_model_with_tools`, `ToolNode`, `should_continue` and `build_graph_one_tool`) was adapted from patterns presented in the course, while the integration of LangGraph and AutoGen in a single flow, the use of Tavily Extract, the marketing team, the human-in-the-loop loop and the business context were developed for this project.

Claude (Anthropic) was used as a support tool during development, for debugging, explanations and documentation. The project's logic, structure and design decisions were defined, understood and supervised by the author, who chose and reviewed every change.

## How it works

```
OUTER GRAPH (LangGraph, state = BrainstormState): the brainstorm loop

START ──> run_tools ──> brand_marketer ──> cmo ──> designer ──> human_review ──┬──> END   (you type "exit")
              ^                                                                │
              └────────────────────────────────────────────────────────────────┘  (you type a new direction)


INNER GRAPH (LangGraph, state = AgentState): the research agent, used inside run_tools

START ──> agent ──should_continue──> action (ToolNode: Tavily search / extract)
            ^                            │
            └────────────────────────────┘      no tool call ──> END
```

Each iteration runs these steps:

1. **`run_tools`**: a tool-calling LLM decides which web searches to run (in Brazilian Portuguese and English) and reads up to 3 of the most relevant pages in full (Tavily Search and Tavily Extract), then writes a research report with facts, numbers, competitors and trends, each with its source URL.
2. **`brand_marketer`**: an AutoGen agent turns the research, the history and your direction into brand insights, citing the research sources, separating facts from hypotheses and ending with 2 or 3 questions to steer the next iteration.
3. **`cmo`**: an AutoGen agent with a Chief Marketing Officer role proposes positioning, target audience, messaging, digital and physical channels and campaign ideas for a small company with limited resources, reading what the Brand Marketer wrote.
4. **`designer`**: an AutoGen agent writes ideas and image-generation prompts for social media posts, since the company has no image editor, building on both previous answers.
5. **`human_review`**: the graph pauses (`interrupt`) and shows you the three answers. Your reply becomes the direction of the next iteration, or `exit` (also `quit`, `stop` or `end`) ends the session.

**State.** Each agent writes to its own field (`marketer_insight`, `cmo_insight`, `designer_insight`), so no answer overwrites another. The `history` field accumulates your directions, the tools called and every agent answer across iterations; the raw research is kept only for the current iteration, so the prompts don't grow with page texts.

**Autonomy limits.** The agents follow a fixed order, the team agents have no tools, the research agent is capped by `RESEARCH_RECURSION_LIMIT` and the loop by `MAX_ITERATIONS`.

## Requirements

* Python 3.10+ (tested on Google Colab)
* API keys:

| Key | Used for | Required |
|---|---|---|
| `OPENAI_API_KEY` | research agent and team agents (default provider) | yes |
| `TAVILY_API_KEY` | web search and page extraction | yes |
| `GOOGLE_API_KEY` | team agents, only if `MARKETER_PROVIDER = "gemini"` | optional |

On Colab, add the keys in the **Secrets** panel (key icon on the left). Elsewhere, put them in a `.env` file in the same folder as the notebook.

## How to run

1. Open `MKT_PROJECT_v3.ipynb` and run the install cell (section 1).
2. Run the cells in order up to section 11.
3. In section 12, write your `first_prompt` and run the cell.
4. At each pause, read the proposals and type a new direction, or `exit` to finish.
5. The last cell prints the full accumulated history.

## Customizing for your business

Everything that describes the company lives in plain-text prompts. To adapt the assistant to another business, edit the items below; the graph structure does not need to change.

### 1. Business context (main change)

The company description appears in four prompts. Replace it in all of them with your own context: what the company sells, what it does **not** sell (useful to stop the models from mixing up similar products), its size, age, region, business model (B2B/B2C), sales channels and main constraints.

| Where | Variable | Section |
|---|---|---|
| Research agent instructions | `RESEARCH_SYSTEM` | 9 |
| Brand Marketer role | `brand_marketer_agent` → `system_message` | 5 |
| CMO role | `cmo_agent` → `system_message` | 5 |
| Marketing Designer role | `designer_agent` → `system_message` | 5 |

The starting question (`first_prompt`, section 12) also mentions the current products, so rewrite it for the new business too.

### 2. What each agent does

Each agent's behaviour is defined only by its `system_message` (section 5). Some examples of what can be changed:

* **Brand Marketer**: focus (brand awareness, repositioning, launch of a new product line), required output format, number of questions at the end.
* **CMO**: budget level, priority channels (digital, physical, trade shows, sales reps), metrics to track.
* **Marketing Designer**: target platforms (Instagram, LinkedIn, catalogues, packaging), tools available to the company, visual style, whether to output image-generation prompts or design guidelines.

### 3. The team itself (adding, removing or reordering agents)

To add an agent, for example a Sales Specialist:

1. **Agent** (section 5): create a new `AssistantAgent` with a name and `system_message`.
2. **State** (section 6): add a field for its answer, such as `sales_insight: str`.
3. **Node** (section 9): copy one of the agent nodes, change the agent it runs and the field it returns.
4. **Edges** (section 10): `add_node` the new node and place it in the chain, for example `designer → sales → human_review`.
5. **Human review** (section 9): add the new field to the `insight_to_evaluate` text in `node_human_review`.

To remove an agent, do the reverse and reconnect the edges. To reorder, change only the edges in section 10. Keep a single chain: a node with two plain outgoing edges runs both destinations in parallel, which breaks the loop.

### 4. Research behaviour

| What | Where | Section |
|---|---|---|
| Search languages (currently Brazilian Portuguese and English), number of pages read in full (currently 3), report format | `RESEARCH_SYSTEM` | 9 |
| Results per search (currently 3) | `TavilySearch(max_results=3)` | 7 |
| Depth of page extraction (currently `"advanced"`; the other option is `"basic"`) | `TavilyExtract(extract_depth="advanced")` | 7 |
| How much recent history the research agent sees (currently the last 6 entries) | `state.get("history", [])[-6:]` in `node_run_tools` | 9 |

**New tools.** Section 7 has a commented example. Write a function with the `@tool` decorator (the docstring is what the LLM reads to decide when to use it) and add it to `tools_list_single`. `bind_tools` and `ToolNode` pick it up automatically.

### 5. Models and providers

| What | Where | Section |
|---|---|---|
| Team provider (`"openai"` by default, or `"gemini"`) | `MARKETER_PROVIDER` | 3 |
| Team model names (`"gpt-5-mini"` and `"gemini-2.5-flash"` by default) | `OPENAI_MARKETER_MODEL`, `GEMINI_MARKETER_MODEL` | 3 |
| Research agent model (`"gpt-5-mini"` by default) | `llm = ChatOpenAI(model=...)` | 4 |

The three team agents share the same model client (`marketer_client`), so changing the provider changes all of them.

### 6. Limits and session control

| What | Where | Section |
|---|---|---|
| Maximum brainstorm iterations (default 5) | `MAX_ITERATIONS` | 3 |
| Maximum research steps per iteration (default 12) | `RESEARCH_RECURSION_LIMIT` | 3 |
| Words that end the session (`exit`, `quit`, `stop`, `end`) | `EXIT_WORDS` | 3 |

### 7. The starting question

Edit `first_prompt` in section 12 with the question that opens the brainstorm (market analysis, campaign for a product launch, repositioning, and so on). The prompts and agent answers are in English.

## Notebook structure

| Section | Content |
|---|---|
| 1 | Package installation |
| 2 | Imports and API keys |
| 3 | Settings (limits, provider, models) |
| 4 | LangChain LLM for the research agent |
| 5 | AutoGen marketing team (agents and roles) |
| 6 | States (`AgentState`, `BrainstormState`) |
| 7 | Tools list |
| 8 | Research agent graph (`make_call_model_with_tools`, `should_continue`, `build_graph_one_tool`) |
| 9 | Nodes of the outer graph |
| 10 | Outer graph wiring and checkpointer |
| 11 | Driver loop (`run_brainstorm`) |
| 12 | Run and inspect the history |

## Notes

* **Session memory.** `MemorySaver` keeps sessions in RAM, so they are lost when the runtime restarts. For persistent sessions, replace it with `SqliteSaver` or `PostgresSaver`.
* **Costs.** Every iteration makes several LLM calls and several Tavily calls. Keep `MAX_ITERATIONS` and `RESEARCH_RECURSION_LIMIT` low while testing.
* **Research quality.** The research agent and the Brand Marketer are instructed to cite source URLs, and the Brand Marketer to separate facts from hypotheses, but outputs should still be checked before being used in real decisions.
