# OpenAI Agents SDK
The [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) enables you to build agentic AI apps in a lightweight, easy-to-use package with very few abstractions. It's a production-ready upgrade of our previous experimentation for agents, [Swarm](https://github.com/openai/swarm/tree/main). The Agents SDK has a very small set of primitives:

*   **Agents**, which are LLMs equipped with instructions and tools
*   **Handoffs**, which allow agents to delegate to other agents for specific tasks
*   **Guardrails**, which enable the inputs to agents to be validated

In combination with Python, these primitives are powerful enough to express complex relationships between tools and agents, and allow you to build real-world applications without a steep learning curve. In addition, the SDK comes with built-in **tracing** that lets you visualize and debug your agentic flows, as well as evaluate them and even fine-tune models for your application.

Why use the Agents SDK
----------------------

The SDK has two driving design principles:

1.  Enough features to be worth using, but few enough primitives to make it quick to learn.
2.  Works great out of the box, but you can customize exactly what happens.

Here are the main features of the SDK:

*   Agent loop: Built-in agent loop that handles calling tools, sending results to the LLM, and looping until the LLM is done.
*   Python-first: Use built-in language features to orchestrate and chain agents, rather than needing to learn new abstractions.
*   Handoffs: A powerful feature to coordinate and delegate between multiple agents.
*   Guardrails: Run input validations and checks in parallel to your agents, breaking early if the checks fail.
*   Function tools: Turn any Python function into a tool, with automatic schema generation and Pydantic-powered validation.
*   Tracing: Built-in tracing that lets you visualize, debug and monitor your workflows, as well as use the OpenAI suite of evaluation, fine-tuning and distillation tools.

Installation
------------

```
pip install openai-agents

```


Hello world example
-------------------

```
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="You are a helpful assistant")

result = Runner.run_sync(agent, "Write a haiku about recursion in programming.")
print(result.final_output)

# Code within the code,
# Functions calling themselves,
# Infinite loop's dance.

```


(_If running this, ensure you set the `OPENAI_API_KEY` environment variable_)

```
export OPENAI_API_KEY=sk-...

```

# Quickstart - OpenAI Agents SDK
Create a project and virtual environment
----------------------------------------

You'll only need to do this once.

```
mkdir my_project
cd my_project
python -m venv .venv

```


### Activate the virtual environment

Do this every time you start a new terminal session.

```
source .venv/bin/activate

```


### Install the Agents SDK

```
pip install openai-agents # or `uv add openai-agents`, etc

```


### Set an OpenAI API key

If you don't have one, follow [these instructions](https://platform.openai.com/docs/quickstart#create-and-export-an-api-key) to create an OpenAI API key.

```
export OPENAI_API_KEY=sk-...

```


Create your first agent
-----------------------

Agents are defined with instructions, a name, and optional config (such as `model_config`)

```
from agents import Agent

agent = Agent(
    name="Math Tutor",
    instructions="You provide help with math problems. Explain your reasoning at each step and include examples",
)

```


Add a few more agents
---------------------

Additional agents can be defined in the same way. `handoff_descriptions` provide additional context for determining handoff routing

```
from agents import Agent

history_tutor_agent = Agent(
    name="History Tutor",
    handoff_description="Specialist agent for historical questions",
    instructions="You provide assistance with historical queries. Explain important events and context clearly.",
)

math_tutor_agent = Agent(
    name="Math Tutor",
    handoff_description="Specialist agent for math questions",
    instructions="You provide help with math problems. Explain your reasoning at each step and include examples",
)

```


Define your handoffs
--------------------

On each agent, you can define an inventory of outgoing handoff options that the agent can choose from to decide how to make progress on their task.

```
triage_agent = Agent(
    name="Triage Agent",
    instructions="You determine which agent to use based on the user's homework question",
    handoffs=[history_tutor_agent, math_tutor_agent]
)

```


Run the agent orchestration
---------------------------

Let's check that the workflow runs and the triage agent correctly routes between the two specialist agents.

```
from agents import Runner

async def main():
    result = await Runner.run(triage_agent, "What is the capital of France?")
    print(result.final_output)

```


Add a guardrail
---------------

You can define custom guardrails to run on the input or output.

```
from agents import GuardrailFunctionOutput, Agent, Runner
from pydantic import BaseModel

class HomeworkOutput(BaseModel):
    is_homework: bool
    reasoning: str

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the user is asking about homework.",
    output_type=HomeworkOutput,
)

async def homework_guardrail(ctx, agent, input_data):
    result = await Runner.run(guardrail_agent, input_data, context=ctx.context)
    final_output = result.final_output_as(HomeworkOutput)
    return GuardrailFunctionOutput(
        output_info=final_output,
        tripwire_triggered=not final_output.is_homework,
    )

```


Put it all together
-------------------

Let's put it all together and run the entire workflow, using handoffs and the input guardrail.

```
from agents import Agent, InputGuardrail,GuardrailFunctionOutput, Runner
from pydantic import BaseModel
import asyncio

class HomeworkOutput(BaseModel):
    is_homework: bool
    reasoning: str

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the user is asking about homework.",
    output_type=HomeworkOutput,
)

math_tutor_agent = Agent(
    name="Math Tutor",
    handoff_description="Specialist agent for math questions",
    instructions="You provide help with math problems. Explain your reasoning at each step and include examples",
)

history_tutor_agent = Agent(
    name="History Tutor",
    handoff_description="Specialist agent for historical questions",
    instructions="You provide assistance with historical queries. Explain important events and context clearly.",
)


async def homework_guardrail(ctx, agent, input_data):
    result = await Runner.run(guardrail_agent, input_data, context=ctx.context)
    final_output = result.final_output_as(HomeworkOutput)
    return GuardrailFunctionOutput(
        output_info=final_output,
        tripwire_triggered=not final_output.is_homework,
    )

triage_agent = Agent(
    name="Triage Agent",
    instructions="You determine which agent to use based on the user's homework question",
    handoffs=[history_tutor_agent, math_tutor_agent],
    input_guardrails=[
        InputGuardrail(guardrail_function=homework_guardrail),
    ],
)

async def main():
    result = await Runner.run(triage_agent, "who was the first president of the united states?")
    print(result.final_output)

    result = await Runner.run(triage_agent, "what is life")
    print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())

```


View your traces
----------------

To review what happened during your agent run, navigate to the [Trace viewer in the OpenAI Dashboard](https://platform.openai.com/traces) to view traces of your agent runs.

Next steps
----------

Learn how to build more complex agentic flows:

*   Learn about how to configure [Agents](../agents/).
*   Learn about [running agents](../running_agents/).
*   Learn about [tools](../tools/), [guardrails](../guardrails/) and [models](../models/).

# Examples - OpenAI Agents SDK
Check out a variety of sample implementations of the SDK in the examples section of the [repo](https://github.com/openai/openai-agents-python/tree/main/examples). The examples are organized into several categories that demonstrate different patterns and capabilities.

Categories
----------

*   **agent\_patterns:** Examples in this category illustrate common agent design patterns, such as
    
    *   Deterministic workflows
    *   Agents as tools
    *   Parallel agent execution
*   **basic:** These examples showcase foundational capabilities of the SDK, such as
    
    *   Dynamic system prompts
    *   Streaming outputs
    *   Lifecycle events
*   **tool examples:** Learn how to implement OAI hosted tools such as web search and file search, and integrate them into your agents.
    
*   **model providers:** Explore how to use non-OpenAI models with the SDK.
    
*   **handoffs:** See practical examples of agent handoffs.
    
*   **customer\_service** and **research\_bot:** Two more built-out examples that illustrate real-world applications
    
    *   **customer\_service**: Example customer service system for an airline.
    *   **research\_bot**: Simple deep research clone.

# Agents - OpenAI Agents SDK
Agents are the core building block in your apps. An agent is a large language model (LLM), configured with instructions and tools.

Basic configuration
-------------------

The most common properties of an agent you'll configure are:

*   `instructions`: also known as a developer message or system prompt.
*   `model`: which LLM to use, and optional `model_settings` to configure model tuning parameters like temperature, top\_p, etc.
*   `tools`: Tools that the agent can use to achieve its tasks.

```
from agents import Agent, ModelSettings, function_tool

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Haiku agent",
    instructions="Always respond in haiku form",
    model="o3-mini",
    tools=[get_weather],
)

```


Context
-------

Agents are generic on their `context` type. Context is a dependency-injection tool: it's an object you create and pass to `Runner.run()`, that is passed to every agent, tool, handoff etc, and it serves as a grab bag of dependencies and state for the agent run. You can provide any Python object as the context.

```
@dataclass
class UserContext:
  uid: str
  is_pro_user: bool

  async def fetch_purchases() -> list[Purchase]:
     return ...

agent = Agent[UserContext](
    ...,
)

```


Output types
------------

By default, agents produce plain text (i.e. `str`) outputs. If you want the agent to produce a particular type of output, you can use the `output_type` parameter. A common choice is to use [Pydantic](https://docs.pydantic.dev/) objects, but we support any type that can be wrapped in a Pydantic [TypeAdapter](https://docs.pydantic.dev/latest/api/type_adapter/) - dataclasses, lists, TypedDict, etc.

```
from pydantic import BaseModel
from agents import Agent


class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar extractor",
    instructions="Extract calendar events from text",
    output_type=CalendarEvent,
)

```


Note

When you pass an `output_type`, that tells the model to use [structured outputs](https://platform.openai.com/docs/guides/structured-outputs) instead of regular plain text responses.

Handoffs
--------

Handoffs are sub-agents that the agent can delegate to. You provide a list of handoffs, and the agent can choose to delegate to them if relevant. This is a powerful pattern that allows orchestrating modular, specialized agents that excel at a single task. Read more in the [handoffs](../handoffs/) documentation.

```
from agents import Agent

booking_agent = Agent(...)
refund_agent = Agent(...)

triage_agent = Agent(
    name="Triage agent",
    instructions=(
        "Help the user with their questions."
        "If they ask about booking, handoff to the booking agent."
        "If they ask about refunds, handoff to the refund agent."
    ),
    handoffs=[booking_agent, refund_agent],
)

```


Dynamic instructions
--------------------

In most cases, you can provide instructions when you create the agent. However, you can also provide dynamic instructions via a function. The function will receive the agent and context, and must return the prompt. Both regular and `async` functions are accepted.

```
def dynamic_instructions(
    context: RunContextWrapper[UserContext], agent: Agent[UserContext]
) -> str:
    return f"The user's name is {context.context.name}. Help them with their questions."


agent = Agent[UserContext](
    name="Triage agent",
    instructions=dynamic_instructions,
)

```


Lifecycle events (hooks)
------------------------

Sometimes, you want to observe the lifecycle of an agent. For example, you may want to log events, or pre-fetch data when certain events occur. You can hook into the agent lifecycle with the `hooks` property. Subclass the [`AgentHooks`](about:blank/ref/lifecycle/#agents.lifecycle.AgentHooks "AgentHooks") class, and override the methods you're interested in.

Guardrails
----------

Guardrails allow you to run checks/validations on user input, in parallel to the agent running. For example, you could screen the user's input for relevance. Read more in the [guardrails](../guardrails/) documentation.

Cloning/copying agents
----------------------

By using the `clone()` method on an agent, you can duplicate an Agent, and optionally change any properties you like.

```
pirate_agent = Agent(
    name="Pirate",
    instructions="Write like a pirate",
    model="o3-mini",
)

robot_agent = pirate_agent.clone(
    name="Robot",
    instructions="Write like a robot",
)

```


Supplying a list of tools doesn't always mean the LLM will use a tool. You can force tool use by setting [`ModelSettings.tool_choice`](about:blank/ref/model_settings/#agents.model_settings.ModelSettings.tool_choice "tool_choice
class-attribute
instance-attribute
"). Valid values are:

1.  `auto`, which allows the LLM to decide whether or not to use a tool.
2.  `required`, which requires the LLM to use a tool (but it can intelligently decide which tool).
3.  `none`, which requires the LLM to _not_ use a tool.
4.  Setting a specific string e.g. `my_tool`, which requires the LLM to use that specific tool.

Note

To prevent infinite loops, the framework automatically resets `tool_choice` to "auto" after a tool call. This behavior is configurable via [`agent.reset_tool_choice`](about:blank/ref/agent/#agents.agent.Agent.reset_tool_choice "reset_tool_choice
class-attribute
instance-attribute
"). The infinite loop is because tool results are sent to the LLM, which then generates another tool call because of `tool_choice`, ad infinitum.

If you want the Agent to completely stop after a tool call (rather than continuing with auto mode), you can set \[`Agent.tool_use_behavior="stop_on_first_tool"`\] which will directly use the tool output as the final response without further LLM processing.

# Running agents - OpenAI Agents SDK
You can run agents via the [`Runner`](about:blank/ref/run/#agents.run.Runner "Runner") class. You have 3 options:

1.  [`Runner.run()`](about:blank/ref/run/#agents.run.Runner.run "run
    async
    classmethod
    "), which runs async and returns a [`RunResult`](about:blank/ref/result/#agents.result.RunResult "RunResult
    dataclass
    ").
2.  [`Runner.run_sync()`](about:blank/ref/run/#agents.run.Runner.run_sync "run_sync
    classmethod
    "), which is a sync method and just runs `.run()` under the hood.
3.  [`Runner.run_streamed()`](about:blank/ref/run/#agents.run.Runner.run_streamed "run_streamed
    classmethod
    "), which runs async and returns a [`RunResultStreaming`](about:blank/ref/result/#agents.result.RunResultStreaming "RunResultStreaming
    dataclass
    "). It calls the LLM in streaming mode, and streams those events to you as they are received.

```
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="You are a helpful assistant")

    result = await Runner.run(agent, "Write a haiku about recursion in programming.")
    print(result.final_output)
    # Code within the code,
    # Functions calling themselves,
    # Infinite loop's dance.

```


Read more in the [results guide](../results/).

The agent loop
--------------

When you use the run method in `Runner`, you pass in a starting agent and input. The input can either be a string (which is considered a user message), or a list of input items, which are the items in the OpenAI Responses API.

The runner then runs a loop:

1.  We call the LLM for the current agent, with the current input.
2.  The LLM produces its output.
    1.  If the LLM returns a `final_output`, the loop ends and we return the result.
    2.  If the LLM does a handoff, we update the current agent and input, and re-run the loop.
    3.  If the LLM produces tool calls, we run those tool calls, append the results, and re-run the loop.
3.  If we exceed the `max_turns` passed, we raise a [`MaxTurnsExceeded`](about:blank/ref/exceptions/#agents.exceptions.MaxTurnsExceeded "MaxTurnsExceeded") exception.

Note

The rule for whether the LLM output is considered as a "final output" is that it produces text output with the desired type, and there are no tool calls.

Streaming
---------

Streaming allows you to additionally receive streaming events as the LLM runs. Once the stream is done, the [`RunResultStreaming`](about:blank/ref/result/#agents.result.RunResultStreaming "RunResultStreaming
dataclass
") will contain the complete information about the run, including all the new outputs produces. You can call `.stream_events()` for the streaming events. Read more in the [streaming guide](../streaming/).

Run config
----------

The `run_config` parameter lets you configure some global settings for the agent run:

*   [`model`](about:blank/ref/run/#agents.run.RunConfig.model "model
    class-attribute
    instance-attribute
    "): Allows setting a global LLM model to use, irrespective of what `model` each Agent has.
*   [`model_provider`](about:blank/ref/run/#agents.run.RunConfig.model_provider "model_provider
    class-attribute
    instance-attribute
    "): A model provider for looking up model names, which defaults to OpenAI.
*   [`model_settings`](about:blank/ref/run/#agents.run.RunConfig.model_settings "model_settings
    class-attribute
    instance-attribute
    "): Overrides agent-specific settings. For example, you can set a global `temperature` or `top_p`.
*   [`input_guardrails`](about:blank/ref/run/#agents.run.RunConfig.input_guardrails "input_guardrails
    class-attribute
    instance-attribute
    "), [`output_guardrails`](about:blank/ref/run/#agents.run.RunConfig.output_guardrails "output_guardrails
    class-attribute
    instance-attribute
    "): A list of input or output guardrails to include on all runs.
*   [`handoff_input_filter`](about:blank/ref/run/#agents.run.RunConfig.handoff_input_filter "handoff_input_filter
    class-attribute
    instance-attribute
    "): A global input filter to apply to all handoffs, if the handoff doesn't already have one. The input filter allows you to edit the inputs that are sent to the new agent. See the documentation in [`Handoff.input_filter`](about:blank/ref/handoffs/#agents.handoffs.Handoff.input_filter "input_filter
    class-attribute
    instance-attribute
    ") for more details.
*   [`tracing_disabled`](about:blank/ref/run/#agents.run.RunConfig.tracing_disabled "tracing_disabled
    class-attribute
    instance-attribute
    "): Allows you to disable [tracing](../tracing/) for the entire run.
*   [`trace_include_sensitive_data`](about:blank/ref/run/#agents.run.RunConfig.trace_include_sensitive_data "trace_include_sensitive_data
    class-attribute
    instance-attribute
    "): Configures whether traces will include potentially sensitive data, such as LLM and tool call inputs/outputs.
*   [`workflow_name`](about:blank/ref/run/#agents.run.RunConfig.workflow_name "workflow_name
    class-attribute
    instance-attribute
    "), [`trace_id`](about:blank/ref/run/#agents.run.RunConfig.trace_id "trace_id
    class-attribute
    instance-attribute
    "), [`group_id`](about:blank/ref/run/#agents.run.RunConfig.group_id "group_id
    class-attribute
    instance-attribute
    "): Sets the tracing workflow name, trace ID and trace group ID for the run. We recommend at least setting `workflow_name`. The session ID is an optional field that lets you link traces across multiple runs.
*   [`trace_metadata`](about:blank/ref/run/#agents.run.RunConfig.trace_metadata "trace_metadata
    class-attribute
    instance-attribute
    "): Metadata to include on all traces.

Conversations/chat threads
--------------------------

Calling any of the run methods can result in one or more agents running (and hence one or more LLM calls), but it represents a single logical turn in a chat conversation. For example:

1.  User turn: user enter text
2.  Runner run: first agent calls LLM, runs tools, does a handoff to a second agent, second agent runs more tools, and then produces an output.

At the end of the agent run, you can choose what to show to the user. For example, you might show the user every new item generated by the agents, or just the final output. Either way, the user might then ask a followup question, in which case you can call the run method again.

You can use the base [`RunResultBase.to_input_list()`](about:blank/ref/result/#agents.result.RunResultBase.to_input_list "to_input_list") method to get the inputs for the next turn.

```
async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?")
        print(result.final_output)
        # San Francisco

        # Second turn
        new_input = result.to_input_list() + [{"role": "user", "content": "What state is it in?"}]
        result = await Runner.run(agent, new_input)
        print(result.final_output)
        # California

```


Exceptions
----------

The SDK raises exceptions in certain cases. The full list is in [`agents.exceptions`](about:blank/ref/exceptions/#agents.exceptions). As an overview:

*   [`AgentsException`](about:blank/ref/exceptions/#agents.exceptions.AgentsException "AgentsException") is the base class for all exceptions raised in the SDK.
*   [`MaxTurnsExceeded`](about:blank/ref/exceptions/#agents.exceptions.MaxTurnsExceeded "MaxTurnsExceeded") is raised when the run exceeds the `max_turns` passed to the run methods.
*   [`ModelBehaviorError`](about:blank/ref/exceptions/#agents.exceptions.ModelBehaviorError "ModelBehaviorError") is raised when the model produces invalid outputs, e.g. malformed JSON or using non-existent tools.
*   [`UserError`](about:blank/ref/exceptions/#agents.exceptions.UserError "UserError") is raised when you (the person writing code using the SDK) make an error using the SDK.
*   [`InputGuardrailTripwireTriggered`](about:blank/ref/exceptions/#agents.exceptions.InputGuardrailTripwireTriggered "InputGuardrailTripwireTriggered"), [`OutputGuardrailTripwireTriggered`](about:blank/ref/exceptions/#agents.exceptions.OutputGuardrailTripwireTriggered "OutputGuardrailTripwireTriggered") is raised when a [guardrail](../guardrails/) is tripped.



# Results - OpenAI Agents SDK
When you call the `Runner.run` methods, you either get a:

*   [`RunResult`](about:blank/ref/result/#agents.result.RunResult "RunResult
    dataclass
    ") if you call `run` or `run_sync`
*   [`RunResultStreaming`](about:blank/ref/result/#agents.result.RunResultStreaming "RunResultStreaming
    dataclass
    ") if you call `run_streamed`

Both of these inherit from [`RunResultBase`](about:blank/ref/result/#agents.result.RunResultBase "RunResultBase
dataclass
"), which is where most useful information is present.

Final output
------------

The [`final_output`](about:blank/ref/result/#agents.result.RunResultBase.final_output "final_output
instance-attribute
") property contains the final output of the last agent that ran. This is either:

*   a `str`, if the last agent didn't have an `output_type` defined
*   an object of type `last_agent.output_type`, if the agent had an output type defined.

Note

`final_output` is of type `Any`. We can't statically type this, because of handoffs. If handoffs occur, that means any Agent might be the last agent, so we don't statically know the set of possible output types.

Inputs for the next turn
------------------------

You can use [`result.to_input_list()`](about:blank/ref/result/#agents.result.RunResultBase.to_input_list "to_input_list") to turn the result into an input list that concatenates the original input you provided, to the items generated during the agent run. This makes it convenient to take the outputs of one agent run and pass them into another run, or to run it in a loop and append new user inputs each time.

Last agent
----------

The [`last_agent`](about:blank/ref/result/#agents.result.RunResultBase.last_agent "last_agent
abstractmethod
property
") property contains the last agent that ran. Depending on your application, this is often useful for the next time the user inputs something. For example, if you have a frontline triage agent that hands off to a language-specific agent, you can store the last agent, and re-use it the next time the user messages the agent.

New items
---------

The [`new_items`](about:blank/ref/result/#agents.result.RunResultBase.new_items "new_items
instance-attribute
") property contains the new items generated during the run. The items are [`RunItem`](about:blank/ref/items/#agents.items.RunItem "RunItem
module-attribute
")s. A run item wraps the raw item generated by the LLM.

*   [`MessageOutputItem`](about:blank/ref/items/#agents.items.MessageOutputItem "MessageOutputItem
    dataclass
    ") indicates a message from the LLM. The raw item is the message generated.
*   [`HandoffCallItem`](about:blank/ref/items/#agents.items.HandoffCallItem "HandoffCallItem
    dataclass
    ") indicates that the LLM called the handoff tool. The raw item is the tool call item from the LLM.
*   [`HandoffOutputItem`](about:blank/ref/items/#agents.items.HandoffOutputItem "HandoffOutputItem
    dataclass
    ") indicates that a handoff occurred. The raw item is the tool response to the handoff tool call. You can also access the source/target agents from the item.
*   [`ToolCallItem`](about:blank/ref/items/#agents.items.ToolCallItem "ToolCallItem
    dataclass
    ") indicates that the LLM invoked a tool.
*   [`ToolCallOutputItem`](about:blank/ref/items/#agents.items.ToolCallOutputItem "ToolCallOutputItem
    dataclass
    ") indicates that a tool was called. The raw item is the tool response. You can also access the tool output from the item.
*   [`ReasoningItem`](about:blank/ref/items/#agents.items.ReasoningItem "ReasoningItem
    dataclass
    ") indicates a reasoning item from the LLM. The raw item is the reasoning generated.

Other information
-----------------

### Guardrail results

The [`input_guardrail_results`](about:blank/ref/result/#agents.result.RunResultBase.input_guardrail_results "input_guardrail_results
instance-attribute
") and [`output_guardrail_results`](about:blank/ref/result/#agents.result.RunResultBase.output_guardrail_results "output_guardrail_results
instance-attribute
") properties contain the results of the guardrails, if any. Guardrail results can sometimes contain useful information you want to log or store, so we make these available to you.

### Raw responses

The [`raw_responses`](about:blank/ref/result/#agents.result.RunResultBase.raw_responses "raw_responses
instance-attribute
") property contains the [`ModelResponse`](about:blank/ref/items/#agents.items.ModelResponse "ModelResponse
dataclass
")s generated by the LLM.

### Original input

The [`input`](about:blank/ref/result/#agents.result.RunResultBase.input "input
instance-attribute
") property contains the original input you provided to the `run` method. In most cases you won't need this, but it's available in case you do.

# Streaming - OpenAI Agents SDK
Streaming lets you subscribe to updates of the agent run as it proceeds. This can be useful for showing the end-user progress updates and partial responses.

To stream, you can call [`Runner.run_streamed()`](about:blank/ref/run/#agents.run.Runner.run_streamed "run_streamed
classmethod
"), which will give you a [`RunResultStreaming`](about:blank/ref/result/#agents.result.RunResultStreaming "RunResultStreaming
dataclass
"). Calling `result.stream_events()` gives you an async stream of [`StreamEvent`](about:blank/ref/stream_events/#agents.stream_events.StreamEvent "StreamEvent
module-attribute
") objects, which are described below.

Raw response events
-------------------

[`RawResponsesStreamEvent`](about:blank/ref/stream_events/#agents.stream_events.RawResponsesStreamEvent "RawResponsesStreamEvent
dataclass
") are raw events passed directly from the LLM. They are in OpenAI Responses API format, which means each event has a type (like `response.created`, `response.output_text.delta`, etc) and data. These events are useful if you want to stream response messages to the user as soon as they are generated.

For example, this will output the text generated by the LLM token-by-token.

```
import asyncio
from openai.types.responses import ResponseTextDeltaEvent
from agents import Agent, Runner

async def main():
    agent = Agent(
        name="Joker",
        instructions="You are a helpful assistant.",
    )

    result = Runner.run_streamed(agent, input="Please tell me 5 jokes.")
    async for event in result.stream_events():
        if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
            print(event.data.delta, end="", flush=True)


if __name__ == "__main__":
    asyncio.run(main())

```


Run item events and agent events
--------------------------------

[`RunItemStreamEvent`](about:blank/ref/stream_events/#agents.stream_events.RunItemStreamEvent "RunItemStreamEvent
dataclass
")s are higher level events. They inform you when an item has been fully generated. This allows you to push progress updates at the level of "message generated", "tool ran", etc, instead of each token. Similarly, [`AgentUpdatedStreamEvent`](about:blank/ref/stream_events/#agents.stream_events.AgentUpdatedStreamEvent "AgentUpdatedStreamEvent
dataclass
") gives you updates when the current agent changes (e.g. as the result of a handoff).

For example, this will ignore raw events and stream updates to the user.

```
import asyncio
import random
from agents import Agent, ItemHelpers, Runner, function_tool

@function_tool
def how_many_jokes() -> int:
    return random.randint(1, 10)


async def main():
    agent = Agent(
        name="Joker",
        instructions="First call the `how_many_jokes` tool, then tell that many jokes.",
        tools=[how_many_jokes],
    )

    result = Runner.run_streamed(
        agent,
        input="Hello",
    )
    print("=== Run starting ===")

    async for event in result.stream_events():
        # We'll ignore the raw responses event deltas
        if event.type == "raw_response_event":
            continue
        # When the agent updates, print that
        elif event.type == "agent_updated_stream_event":
            print(f"Agent updated: {event.new_agent.name}")
            continue
        # When items are generated, print them
        elif event.type == "run_item_stream_event":
            if event.item.type == "tool_call_item":
                print("-- Tool was called")
            elif event.item.type == "tool_call_output_item":
                print(f"-- Tool output: {event.item.output}")
            elif event.item.type == "message_output_item":
                print(f"-- Message output:\n {ItemHelpers.text_message_output(event.item)}")
            else:
                pass  # Ignore other event types

    print("=== Run complete ===")


if __name__ == "__main__":
    asyncio.run(main())

```

# Tools - OpenAI Agents SDK
Tools let agents take actions: things like fetching data, running code, calling external APIs, and even using a computer. There are three classes of tools in the Agent SDK:

*   Hosted tools: these run on LLM servers alongside the AI models. OpenAI offers retrieval, web search and computer use as hosted tools.
*   Function calling: these allow you to use any Python function as a tool.
*   Agents as tools: this allows you to use an agent as a tool, allowing Agents to call other agents without handing off to them.

OpenAI offers a few built-in tools when using the [`OpenAIResponsesModel`](about:blank/ref/models/openai_responses/#agents.models.openai_responses.OpenAIResponsesModel "OpenAIResponsesModel"):

*   The [`WebSearchTool`](about:blank/ref/tool/#agents.tool.WebSearchTool "WebSearchTool
    dataclass
    ") lets an agent search the web.
*   The [`FileSearchTool`](about:blank/ref/tool/#agents.tool.FileSearchTool "FileSearchTool
    dataclass
    ") allows retrieving information from your OpenAI Vector Stores.
*   The [`ComputerTool`](about:blank/ref/tool/#agents.tool.ComputerTool "ComputerTool
    dataclass
    ") allows automating computer use tasks.

```
from agents import Agent, FileSearchTool, Runner, WebSearchTool

agent = Agent(
    name="Assistant",
    tools=[
        WebSearchTool(),
        FileSearchTool(
            max_num_results=3,
            vector_store_ids=["VECTOR_STORE_ID"],
        ),
    ],
)

async def main():
    result = await Runner.run(agent, "Which coffee shop should I go to, taking into account my preferences and the weather today in SF?")
    print(result.final_output)

```


You can use any Python function as a tool. The Agents SDK will setup the tool automatically:

*   The name of the tool will be the name of the Python function (or you can provide a name)
*   Tool description will be taken from the docstring of the function (or you can provide a description)
*   The schema for the function inputs is automatically created from the function's arguments
*   Descriptions for each input are taken from the docstring of the function, unless disabled

We use Python's `inspect` module to extract the function signature, along with [`griffe`](https://mkdocstrings.github.io/griffe/) to parse docstrings and `pydantic` for schema creation.

```
import json

from typing_extensions import TypedDict, Any

from agents import Agent, FunctionTool, RunContextWrapper, function_tool


class Location(TypedDict):
    lat: float
    long: float

@function_tool  # (1)!
async def fetch_weather(location: Location) -> str:
    # (2)!
    """Fetch the weather for a given location.

    Args:
        location: The location to fetch the weather for.
    """
    # In real life, we'd fetch the weather from a weather API
    return "sunny"


@function_tool(name_override="fetch_data")  # (3)!
def read_file(ctx: RunContextWrapper[Any], path: str, directory: str | None = None) -> str:
    """Read the contents of a file.

    Args:
        path: The path to the file to read.
        directory: The directory to read the file from.
    """
    # In real life, we'd read the file from the file system
    return "<file contents>"


agent = Agent(
    name="Assistant",
    tools=[fetch_weather, read_file],  # (4)!
)

for tool in agent.tools:
    if isinstance(tool, FunctionTool):
        print(tool.name)
        print(tool.description)
        print(json.dumps(tool.params_json_schema, indent=2))
        print()

```


1.  You can use any Python types as arguments to your functions, and the function can be sync or async.
2.  Docstrings, if present, are used to capture descriptions and argument descriptions
3.  Functions can optionally take the `context` (must be the first argument). You can also set overrides, like the name of the tool, description, which docstring style to use, etc.
4.  You can pass the decorated functions to the list of tools.

Expand to see output

```
fetch_weather
Fetch the weather for a given location.
{
"$defs": {
  "Location": {
    "properties": {
      "lat": {
        "title": "Lat",
        "type": "number"
      },
      "long": {
        "title": "Long",
        "type": "number"
      }
    },
    "required": [
      "lat",
      "long"
    ],
    "title": "Location",
    "type": "object"
  }
},
"properties": {
  "location": {
    "$ref": "#/$defs/Location",
    "description": "The location to fetch the weather for."
  }
},
"required": [
  "location"
],
"title": "fetch_weather_args",
"type": "object"
}

fetch_data
Read the contents of a file.
{
"properties": {
  "path": {
    "description": "The path to the file to read.",
    "title": "Path",
    "type": "string"
  },
  "directory": {
    "anyOf": [
      {
        "type": "string"
      },
      {
        "type": "null"
      }
    ],
    "default": null,
    "description": "The directory to read the file from.",
    "title": "Directory"
  }
},
"required": [
  "path"
],
"title": "fetch_data_args",
"type": "object"
}

```


### Custom function tools

Sometimes, you don't want to use a Python function as a tool. You can directly create a [`FunctionTool`](about:blank/ref/tool/#agents.tool.FunctionTool "FunctionTool
dataclass
") if you prefer. You'll need to provide:

*   `name`
*   `description`
*   `params_json_schema`, which is the JSON schema for the arguments
*   `on_invoke_tool`, which is an async function that receives the context and the arguments as a JSON string, and must return the tool output as a string.

```
from typing import Any

from pydantic import BaseModel

from agents import RunContextWrapper, FunctionTool



def do_some_work(data: str) -> str:
    return "done"


class FunctionArgs(BaseModel):
    username: str
    age: int


async def run_function(ctx: RunContextWrapper[Any], args: str) -> str:
    parsed = FunctionArgs.model_validate_json(args)
    return do_some_work(data=f"{parsed.username} is {parsed.age} years old")


tool = FunctionTool(
    name="process_user",
    description="Processes extracted user data",
    params_json_schema=FunctionArgs.model_json_schema(),
    on_invoke_tool=run_function,
)

```


### Automatic argument and docstring parsing

As mentioned before, we automatically parse the function signature to extract the schema for the tool, and we parse the docstring to extract descriptions for the tool and for individual arguments. Some notes on that:

1.  The signature parsing is done via the `inspect` module. We use type annotations to understand the types for the arguments, and dynamically build a Pydantic model to represent the overall schema. It supports most types, including Python primitives, Pydantic models, TypedDicts, and more.
2.  We use `griffe` to parse docstrings. Supported docstring formats are `google`, `sphinx` and `numpy`. We attempt to automatically detect the docstring format, but this is best-effort and you can explicitly set it when calling `function_tool`. You can also disable docstring parsing by setting `use_docstring_info` to `False`.

The code for the schema extraction lives in [`agents.function_schema`](about:blank/ref/function_schema/#agents.function_schema).

In some workflows, you may want a central agent to orchestrate a network of specialized agents, instead of handing off control. You can do this by modeling agents as tools.

```
from agents import Agent, Runner
import asyncio

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You translate the user's message to Spanish",
)

french_agent = Agent(
    name="French agent",
    instructions="You translate the user's message to French",
)

orchestrator_agent = Agent(
    name="orchestrator_agent",
    instructions=(
        "You are a translation agent. You use the tools given to you to translate."
        "If asked for multiple translations, you call the relevant tools."
    ),
    tools=[
        spanish_agent.as_tool(
            tool_name="translate_to_spanish",
            tool_description="Translate the user's message to Spanish",
        ),
        french_agent.as_tool(
            tool_name="translate_to_french",
            tool_description="Translate the user's message to French",
        ),
    ],
)

async def main():
    result = await Runner.run(orchestrator_agent, input="Say 'Hello, how are you?' in Spanish.")
    print(result.final_output)

```


When you create a function tool via `@function_tool`, you can pass a `failure_error_function`. This is a function that provides an error response to the LLM in case the tool call crashes.

*   By default (i.e. if you don't pass anything), it runs a `default_tool_error_function` which tells the LLM an error occurred.
*   If you pass your own error function, it runs that instead, and sends the response to the LLM.
*   If you explicitly pass `None`, then any tool call errors will be re-raised for you to handle. This could be a `ModelBehaviorError` if the model produced invalid JSON, or a `UserError` if your code crashed, etc.

If you are manually creating a `FunctionTool` object, then you must handle errors inside the `on_invoke_tool` function.

# Model context protocol (MCP) - OpenAI Agents SDK
The [Model context protocol](https://modelcontextprotocol.io/introduction) (aka MCP) is a way to provide tools and context to the LLM. From the MCP docs:

> MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools.

The Agents SDK has support for MCP. This enables you to use a wide range of MCP servers to provide tools to your Agents.

MCP servers
-----------

Currently, the MCP spec defines two kinds of servers, based on the transport mechanism they use:

1.  **stdio** servers run as a subprocess of your application. You can think of them as running "locally".
2.  **HTTP over SSE** servers run remotely. You connect to them via a URL.

You can use the [`MCPServerStdio`](about:blank/ref/mcp/server/#agents.mcp.server.MCPServerStdio "MCPServerStdio") and [`MCPServerSse`](about:blank/ref/mcp/server/#agents.mcp.server.MCPServerSse "MCPServerSse") classes to connect to these servers.

For example, this is how you'd use the [official MCP filesystem server](https://www.npmjs.com/package/@modelcontextprotocol/server-filesystem).

```
async with MCPServerStdio(
    params={
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", samples_dir],
    }
) as server:
    tools = await server.list_tools()

```


Using MCP servers
-----------------

MCP servers can be added to Agents. The Agents SDK will call `list_tools()` on the MCP servers each time the Agent is run. This makes the LLM aware of the MCP server's tools. When the LLM calls a tool from an MCP server, the SDK calls `call_tool()` on that server.

```
agent=Agent(
    name="Assistant",
    instructions="Use the tools to achieve the task",
    mcp_servers=[mcp_server_1, mcp_server_2]
)

```


Caching
-------

Every time an Agent runs, it calls `list_tools()` on the MCP server. This can be a latency hit, especially if the server is a remote server. To automatically cache the list of tools, you can pass `cache_tools_list=True` to both [`MCPServerStdio`](about:blank/ref/mcp/server/#agents.mcp.server.MCPServerStdio "MCPServerStdio") and [`MCPServerSse`](about:blank/ref/mcp/server/#agents.mcp.server.MCPServerSse "MCPServerSse"). You should only do this if you're certain the tool list will not change.

If you want to invalidate the cache, you can call `invalidate_tools_cache()` on the servers.

End-to-end examples
-------------------

View complete working examples at [examples/mcp](https://github.com/openai/openai-agents-python/tree/main/examples/mcp).

Tracing
-------

[Tracing](../tracing/) automatically captures MCP operations, including:

1.  Calls to the MCP server to list tools
2.  MCP-related info on function calls

![MCP Tracing Screenshot](../assets/images/mcp-tracing.jpg)

# Handoffs - OpenAI Agents SDK
Handoffs allow an agent to delegate tasks to another agent. This is particularly useful in scenarios where different agents specialize in distinct areas. For example, a customer support app might have agents that each specifically handle tasks like order status, refunds, FAQs, etc.

Handoffs are represented as tools to the LLM. So if there's a handoff to an agent named `Refund Agent`, the tool would be called `transfer_to_refund_agent`.

Creating a handoff
------------------

All agents have a [`handoffs`](about:blank/ref/agent/#agents.agent.Agent.handoffs "handoffs
class-attribute
instance-attribute
") param, which can either take an `Agent` directly, or a `Handoff` object that customizes the Handoff.

You can create a handoff using the [`handoff()`](about:blank/ref/handoffs/#agents.handoffs.handoff "handoff") function provided by the Agents SDK. This function allows you to specify the agent to hand off to, along with optional overrides and input filters.

### Basic Usage

Here's how you can create a simple handoff:

```
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

# (1)!
triage_agent = Agent(name="Triage agent", handoffs=[billing_agent, handoff(refund_agent)])

```


1.  You can use the agent directly (as in `billing_agent`), or you can use the `handoff()` function.

### Customizing handoffs via the `handoff()` function

The [`handoff()`](about:blank/ref/handoffs/#agents.handoffs.handoff "handoff") function lets you customize things.

*   `agent`: This is the agent to which things will be handed off.
*   `tool_name_override`: By default, the `Handoff.default_tool_name()` function is used, which resolves to `transfer_to_<agent_name>`. You can override this.
*   `tool_description_override`: Override the default tool description from `Handoff.default_tool_description()`
*   `on_handoff`: A callback function executed when the handoff is invoked. This is useful for things like kicking off some data fetching as soon as you know a handoff is being invoked. This function receives the agent context, and can optionally also receive LLM generated input. The input data is controlled by the `input_type` param.
*   `input_type`: The type of input expected by the handoff (optional).
*   `input_filter`: This lets you filter the input received by the next agent. See below for more.

```
from agents import Agent, handoff, RunContextWrapper

def on_handoff(ctx: RunContextWrapper[None]):
    print("Handoff called")

agent = Agent(name="My agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    tool_name_override="custom_handoff_tool",
    tool_description_override="Custom description",
)

```


Handoff inputs
--------------

In certain situations, you want the LLM to provide some data when it calls a handoff. For example, imagine a handoff to an "Escalation agent". You might want a reason to be provided, so you can log it.

```
from pydantic import BaseModel

from agents import Agent, handoff, RunContextWrapper

class EscalationData(BaseModel):
    reason: str

async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    print(f"Escalation agent called with reason: {input_data.reason}")

agent = Agent(name="Escalation agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    input_type=EscalationData,
)

```


Input filters
-------------

When a handoff occurs, it's as though the new agent takes over the conversation, and gets to see the entire previous conversation history. If you want to change this, you can set an [`input_filter`](about:blank/ref/handoffs/#agents.handoffs.Handoff.input_filter "input_filter
class-attribute
instance-attribute
"). An input filter is a function that receives the existing input via a [`HandoffInputData`](about:blank/ref/handoffs/#agents.handoffs.HandoffInputData "HandoffInputData
dataclass
"), and must return a new `HandoffInputData`.

There are some common patterns (for example removing all tool calls from the history), which are implemented for you in [`agents.extensions.handoff_filters`](about:blank/ref/extensions/handoff_filters/#agents.extensions.handoff_filters)

```
from agents import Agent, handoff
from agents.extensions import handoff_filters

agent = Agent(name="FAQ agent")

handoff_obj = handoff(
    agent=agent,
    input_filter=handoff_filters.remove_all_tools, # (1)!
)

```


1.  This will automatically remove all tools from the history when `FAQ agent` is called.

Recommended prompts
-------------------

To make sure that LLMs understand handoffs properly, we recommend including information about handoffs in your agents. We have a suggested prefix in [`agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX`](about:blank/ref/extensions/handoff_prompt/#agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX "RECOMMENDED_PROMPT_PREFIX
module-attribute
"), or you can call [`agents.extensions.handoff_prompt.prompt_with_handoff_instructions`](about:blank/ref/extensions/handoff_prompt/#agents.extensions.handoff_prompt.prompt_with_handoff_instructions "prompt_with_handoff_instructions") to automatically add recommended data to your prompts.

```
from agents import Agent
from agents.extensions.handoff_prompt import RECOMMENDED_PROMPT_PREFIX

billing_agent = Agent(
    name="Billing agent",
    instructions=f"""{RECOMMENDED_PROMPT_PREFIX}
    <Fill in the rest of your prompt here>.""",
)

```

# Tracing - OpenAI Agents SDK
The Agents SDK includes built-in tracing, collecting a comprehensive record of events during an agent run: LLM generations, tool calls, handoffs, guardrails, and even custom events that occur. Using the [Traces dashboard](https://platform.openai.com/traces), you can debug, visualize, and monitor your workflows during development and in production.

Note

Tracing is enabled by default. There are two ways to disable tracing:

1.  You can globally disable tracing by setting the env var `OPENAI_AGENTS_DISABLE_TRACING=1`
2.  You can disable tracing for a single run by setting [`agents.run.RunConfig.tracing_disabled`](about:blank/ref/run/#agents.run.RunConfig.tracing_disabled "tracing_disabled
    class-attribute
    instance-attribute
    ") to `True`

**_For organizations operating under a Zero Data Retention (ZDR) policy using OpenAI's APIs, tracing is unavailable._**

Traces and spans
----------------

*   **Traces** represent a single end-to-end operation of a "workflow". They're composed of Spans. Traces have the following properties:
    *   `workflow_name`: This is the logical workflow or app. For example "Code generation" or "Customer service".
    *   `trace_id`: A unique ID for the trace. Automatically generated if you don't pass one. Must have the format `trace_<32_alphanumeric>`.
    *   `group_id`: Optional group ID, to link multiple traces from the same conversation. For example, you might use a chat thread ID.
    *   `disabled`: If True, the trace will not be recorded.
    *   `metadata`: Optional metadata for the trace.
*   **Spans** represent operations that have a start and end time. Spans have:
    *   `started_at` and `ended_at` timestamps.
    *   `trace_id`, to represent the trace they belong to
    *   `parent_id`, which points to the parent Span of this Span (if any)
    *   `span_data`, which is information about the Span. For example, `AgentSpanData` contains information about the Agent, `GenerationSpanData` contains information about the LLM generation, etc.

Default tracing
---------------

By default, the SDK traces the following:

*   The entire `Runner.{run, run_sync, run_streamed}()` is wrapped in a `trace()`.
*   Each time an agent runs, it is wrapped in `agent_span()`
*   LLM generations are wrapped in `generation_span()`
*   Function tool calls are each wrapped in `function_span()`
*   Guardrails are wrapped in `guardrail_span()`
*   Handoffs are wrapped in `handoff_span()`
*   Audio inputs (speech-to-text) are wrapped in a `transcription_span()`
*   Audio outputs (text-to-speech) are wrapped in a `speech_span()`
*   Related audio spans may be parented under a `speech_group_span()`

By default, the trace is named "Agent trace". You can set this name if you use `trace`, or you can can configure the name and other properties with the [`RunConfig`](about:blank/ref/run/#agents.run.RunConfig "RunConfig
dataclass
").

In addition, you can set up [custom trace processors](#custom-tracing-processors) to push traces to other destinations (as a replacement, or secondary destination).

Higher level traces
-------------------

Sometimes, you might want multiple calls to `run()` to be part of a single trace. You can do this by wrapping the entire code in a `trace()`.

```
from agents import Agent, Runner, trace

async def main():
    agent = Agent(name="Joke generator", instructions="Tell funny jokes.")

    with trace("Joke workflow"): # (1)!
        first_result = await Runner.run(agent, "Tell me a joke")
        second_result = await Runner.run(agent, f"Rate this joke: {first_result.final_output}")
        print(f"Joke: {first_result.final_output}")
        print(f"Rating: {second_result.final_output}")

```


1.  Because the two calls to `Runner.run` are wrapped in a `with trace()`, the individual runs will be part of the overall trace rather than creating two traces.

Creating traces
---------------

You can use the [`trace()`](about:blank/ref/tracing/#agents.tracing.trace "trace") function to create a trace. Traces need to be started and finished. You have two options to do so:

1.  **Recommended**: use the trace as a context manager, i.e. `with trace(...) as my_trace`. This will automatically start and end the trace at the right time.
2.  You can also manually call [`trace.start()`](about:blank/ref/tracing/#agents.tracing.Trace.start "start
    abstractmethod
    ") and [`trace.finish()`](about:blank/ref/tracing/#agents.tracing.Trace.finish "finish
    abstractmethod
    ").

The current trace is tracked via a Python [`contextvar`](https://docs.python.org/3/library/contextvars.html). This means that it works with concurrency automatically. If you manually start/end a trace, you'll need to pass `mark_as_current` and `reset_current` to `start()`/`finish()` to update the current trace.

Creating spans
--------------

You can use the various [`*_span()`](about:blank/ref/tracing/create/#agents.tracing.create) methods to create a span. In general, you don't need to manually create spans. A [`custom_span()`](about:blank/ref/tracing/#agents.tracing.custom_span "custom_span") function is available for tracking custom span information.

Spans are automatically part of the current trace, and are nested under the nearest current span, which is tracked via a Python [`contextvar`](https://docs.python.org/3/library/contextvars.html).

Sensitive data
--------------

Certain spans may capture potentially sensitive data.

The `generation_span()` stores the inputs/outputs of the LLM generation, and `function_span()` stores the inputs/outputs of function calls. These may contain sensitive data, so you can disable capturing that data via [`RunConfig.trace_include_sensitive_data`](about:blank/ref/run/#agents.run.RunConfig.trace_include_sensitive_data "trace_include_sensitive_data
class-attribute
instance-attribute
").

Similarly, Audio spans include base64-encoded PCM data for input and output audio by default. You can disable capturing this audio data by configuring [`VoicePipelineConfig.trace_include_sensitive_audio_data`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_audio_data "trace_include_sensitive_audio_data
class-attribute
instance-attribute
").

Custom tracing processors
-------------------------

The high level architecture for tracing is:

*   At initialization, we create a global [`TraceProvider`](about:blank/ref/tracing/setup/#agents.tracing.setup.TraceProvider "TraceProvider"), which is responsible for creating traces.
*   We configure the `TraceProvider` with a [`BatchTraceProcessor`](about:blank/ref/tracing/processors/#agents.tracing.processors.BatchTraceProcessor "BatchTraceProcessor") that sends traces/spans in batches to a [`BackendSpanExporter`](about:blank/ref/tracing/processors/#agents.tracing.processors.BackendSpanExporter "BackendSpanExporter"), which exports the spans and traces to the OpenAI backend in batches.

To customize this default setup, to send traces to alternative or additional backends or modifying exporter behavior, you have two options:

1.  [`add_trace_processor()`](about:blank/ref/tracing/#agents.tracing.add_trace_processor "add_trace_processor") lets you add an **additional** trace processor that will receive traces and spans as they are ready. This lets you do your own processing in addition to sending traces to OpenAI's backend.
2.  [`set_trace_processors()`](about:blank/ref/tracing/#agents.tracing.set_trace_processors "set_trace_processors") lets you **replace** the default processors with your own trace processors. This means traces will not be sent to the OpenAI backend unless you include a `TracingProcessor` that does so.

External tracing processors list
--------------------------------

*   [Weights & Biases](https://weave-docs.wandb.ai/guides/integrations/openai_agents)
*   [Arize-Phoenix](https://docs.arize.com/phoenix/tracing/integrations-tracing/openai-agents-sdk)
*   [MLflow](https://mlflow.org/docs/latest/tracing/integrations/openai-agent)
*   [Braintrust](https://braintrust.dev/docs/guides/traces/integrations#openai-agents-sdk)
*   [Pydantic Logfire](https://logfire.pydantic.dev/docs/integrations/llms/openai/#openai-agents)
*   [AgentOps](https://docs.agentops.ai/v1/integrations/agentssdk)
*   [Scorecard](https://docs.scorecard.io/docs/documentation/features/tracing#openai-agents-sdk-integration)
*   [Keywords AI](https://docs.keywordsai.co/integration/development-frameworks/openai-agent)
*   [LangSmith](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_openai_agents_sdk)
*   [Maxim AI](https://www.getmaxim.ai/docs/observe/integrations/openai-agents-sdk)
*   [Comet Opik](https://www.comet.com/docs/opik/tracing/integrations/openai_agents)
*   [Langfuse](https://langfuse.com/docs/integrations/openaiagentssdk/openai-agents)
*   [Langtrace](https://docs.langtrace.ai/supported-integrations/llm-frameworks/openai-agents-sdk)


# Context management - OpenAI Agents SDK
Context is an overloaded term. There are two main classes of context you might care about:

1.  Context available locally to your code: this is data and dependencies you might need when tool functions run, during callbacks like `on_handoff`, in lifecycle hooks, etc.
2.  Context available to LLMs: this is data the LLM sees when generating a response.

Local context
-------------

This is represented via the [`RunContextWrapper`](about:blank/ref/run_context/#agents.run_context.RunContextWrapper "RunContextWrapper
dataclass
") class and the [`context`](about:blank/ref/run_context/#agents.run_context.RunContextWrapper.context "context
instance-attribute
") property within it. The way this works is:

1.  You create any Python object you want. A common pattern is to use a dataclass or a Pydantic object.
2.  You pass that object to the various run methods (e.g. `Runner.run(..., **context=whatever**))`.
3.  All your tool calls, lifecycle hooks etc will be passed a wrapper object, `RunContextWrapper[T]`, where `T` represents your context object type which you can access via `wrapper.context`.

The **most important** thing to be aware of: every agent, tool function, lifecycle etc for a given agent run must use the same _type_ of context.

You can use the context for things like:

*   Contextual data for your run (e.g. things like a username/uid or other information about the user)
*   Dependencies (e.g. logger objects, data fetchers, etc)
*   Helper functions

Note

The context object is **not** sent to the LLM. It is purely a local object that you can read from, write to and call methods on it.

```
import asyncio
from dataclasses import dataclass

from agents import Agent, RunContextWrapper, Runner, function_tool

@dataclass
class UserInfo:  # (1)!
    name: str
    uid: int

@function_tool
async def fetch_user_age(wrapper: RunContextWrapper[UserInfo]) -> str:  # (2)!
    return f"User {wrapper.context.name} is 47 years old"

async def main():
    user_info = UserInfo(name="John", uid=123)

    agent = Agent[UserInfo](  # (3)!
        name="Assistant",
        tools=[fetch_user_age],
    )

    result = await Runner.run(  # (4)!
        starting_agent=agent,
        input="What is the age of the user?",
        context=user_info,
    )

    print(result.final_output)  # (5)!
    # The user John is 47 years old.

if __name__ == "__main__":
    asyncio.run(main())

```


1.  This is the context object. We've used a dataclass here, but you can use any type.
2.  This is a tool. You can see it takes a `RunContextWrapper[UserInfo]`. The tool implementation reads from the context.
3.  We mark the agent with the generic `UserInfo`, so that the typechecker can catch errors (for example, if we tried to pass a tool that took a different context type).
4.  The context is passed to the `run` function.
5.  The agent correctly calls the tool and gets the age.

Agent/LLM context
-----------------

When an LLM is called, the **only** data it can see is from the conversation history. This means that if you want to make some new data available to the LLM, you must do it in a way that makes it available in that history. There are a few ways to do this:

1.  You can add it to the Agent `instructions`. This is also known as a "system prompt" or "developer message". System prompts can be static strings, or they can be dynamic functions that receive the context and output a string. This is a common tactic for information that is always useful (for example, the user's name or the current date).
2.  Add it to the `input` when calling the `Runner.run` functions. This is similar to the `instructions` tactic, but allows you to have messages that are lower in the [chain of command](https://cdn.openai.com/spec/model-spec-2024-05-08.html#follow-the-chain-of-command).
3.  Expose it via function tools. This is useful for _on-demand_ context - the LLM decides when it needs some data, and can call the tool to fetch that data.
4.  Use retrieval or web search. These are special tools that are able to fetch relevant data from files or databases (retrieval), or from the web (web search). This is useful for "grounding" the response in relevant contextual data.

# Guardrails - OpenAI Agents SDK
Guardrails run _in parallel_ to your agents, enabling you to do checks and validations of user input. For example, imagine you have an agent that uses a very smart (and hence slow/expensive) model to help with customer requests. You wouldn't want malicious users to ask the model to help them with their math homework. So, you can run a guardrail with a fast/cheap model. If the guardrail detects malicious usage, it can immediately raise an error, which stops the expensive model from running and saves you time/money.

There are two kinds of guardrails:

1.  Input guardrails run on the initial user input
2.  Output guardrails run on the final agent output

Input guardrails
----------------

Input guardrails run in 3 steps:

1.  First, the guardrail receives the same input passed to the agent.
2.  Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`](about:blank/ref/guardrail/#agents.guardrail.GuardrailFunctionOutput "GuardrailFunctionOutput
    dataclass
    "), which is then wrapped in an [`InputGuardrailResult`](about:blank/ref/guardrail/#agents.guardrail.InputGuardrailResult "InputGuardrailResult
    dataclass
    ")
3.  Finally, we check if [`.tripwire_triggered`](about:blank/ref/guardrail/#agents.guardrail.GuardrailFunctionOutput.tripwire_triggered "tripwire_triggered
    instance-attribute
    ") is true. If true, an [`InputGuardrailTripwireTriggered`](about:blank/ref/exceptions/#agents.exceptions.InputGuardrailTripwireTriggered "InputGuardrailTripwireTriggered") exception is raised, so you can appropriately respond to the user or handle the exception.

Note

Input guardrails are intended to run on user input, so an agent's guardrails only run if the agent is the _first_ agent. You might wonder, why is the `guardrails` property on the agent instead of passed to `Runner.run`? It's because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

Output guardrails
-----------------

Output guardrails run in 3 steps:

1.  First, the guardrail receives the same input passed to the agent.
2.  Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`](about:blank/ref/guardrail/#agents.guardrail.GuardrailFunctionOutput "GuardrailFunctionOutput
    dataclass
    "), which is then wrapped in an [`OutputGuardrailResult`](about:blank/ref/guardrail/#agents.guardrail.OutputGuardrailResult "OutputGuardrailResult
    dataclass
    ")
3.  Finally, we check if [`.tripwire_triggered`](about:blank/ref/guardrail/#agents.guardrail.GuardrailFunctionOutput.tripwire_triggered "tripwire_triggered
    instance-attribute
    ") is true. If true, an [`OutputGuardrailTripwireTriggered`](about:blank/ref/exceptions/#agents.exceptions.OutputGuardrailTripwireTriggered "OutputGuardrailTripwireTriggered") exception is raised, so you can appropriately respond to the user or handle the exception.

Note

Output guardrails are intended to run on the final agent output, so an agent's guardrails only run if the agent is the _last_ agent. Similar to the input guardrails, we do this because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

Tripwires
---------

If the input or output fails the guardrail, the Guardrail can signal this with a tripwire. As soon as we see a guardrail that has triggered the tripwires, we immediately raise a `{Input,Output}GuardrailTripwireTriggered` exception and halt the Agent execution.

Implementing a guardrail
------------------------

You need to provide a function that receives input, and returns a [`GuardrailFunctionOutput`](about:blank/ref/guardrail/#agents.guardrail.GuardrailFunctionOutput "GuardrailFunctionOutput
dataclass
"). In this example, we'll do this by running an Agent under the hood.

```
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    TResponseInputItem,
    input_guardrail,
)

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent( # (1)!
    name="Guardrail check",
    instructions="Check if the user is asking you to do their math homework.",
    output_type=MathHomeworkOutput,
)


@input_guardrail
async def math_guardrail( # (2)!
    ctx: RunContextWrapper[None], agent: Agent, input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output, # (3)!
        tripwire_triggered=result.final_output.is_math_homework,
    )


agent = Agent(  # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    input_guardrails=[math_guardrail],
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except InputGuardrailTripwireTriggered:
        print("Math homework guardrail tripped")

```


1.  We'll use this agent in our guardrail function.
2.  This is the guardrail function that receives the agent's input/context, and returns the result.
3.  We can include extra information in the guardrail result.
4.  This is the actual agent that defines the workflow.

Output guardrails are similar.

```
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    OutputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    output_guardrail,
)
class MessageOutput(BaseModel): # (1)!
    response: str

class MathOutput(BaseModel): # (2)!
    reasoning: str
    is_math: bool

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the output includes any math.",
    output_type=MathOutput,
)

@output_guardrail
async def math_guardrail(  # (3)!
    ctx: RunContextWrapper, agent: Agent, output: MessageOutput
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, output.response, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math,
    )

agent = Agent( # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    output_guardrails=[math_guardrail],
    output_type=MessageOutput,
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except OutputGuardrailTripwireTriggered:
        print("Math output guardrail tripped")

```


1.  This is the actual agent's output type.
2.  This is the guardrail's output type.
3.  This is the guardrail function that receives the agent's output, and returns the result.
4.  This is the actual agent that defines the workflow.

# Orchestrating multiple agents - OpenAI Agents SDK
Orchestration refers to the flow of agents in your app. Which agents run, in what order, and how do they decide what happens next? There are two main ways to orchestrate agents:

1.  Allowing the LLM to make decisions: this uses the intelligence of an LLM to plan, reason, and decide on what steps to take based on that.
2.  Orchestrating via code: determining the flow of agents via your code.

You can mix and match these patterns. Each has their own tradeoffs, described below.

Orchestrating via LLM
---------------------

An agent is an LLM equipped with instructions, tools and handoffs. This means that given an open-ended task, the LLM can autonomously plan how it will tackle the task, using tools to take actions and acquire data, and using handoffs to delegate tasks to sub-agents. For example, a research agent could be equipped with tools like:

*   Web search to find information online
*   File search and retrieval to search through proprietary data and connections
*   Computer use to take actions on a computer
*   Code execution to do data analysis
*   Handoffs to specialized agents that are great at planning, report writing and more.

This pattern is great when the task is open-ended and you want to rely on the intelligence of an LLM. The most important tactics here are:

1.  Invest in good prompts. Make it clear what tools are available, how to use them, and what parameters it must operate within.
2.  Monitor your app and iterate on it. See where things go wrong, and iterate on your prompts.
3.  Allow the agent to introspect and improve. For example, run it in a loop, and let it critique itself; or, provide error messages and let it improve.
4.  Have specialized agents that excel in one task, rather than having a general purpose agent that is expected to be good at anything.
5.  Invest in [evals](https://platform.openai.com/docs/guides/evals). This lets you train your agents to improve and get better at tasks.

Orchestrating via code
----------------------

While orchestrating via LLM is powerful, orchestrating via code makes tasks more deterministic and predictable, in terms of speed, cost and performance. Common patterns here are:

*   Using [structured outputs](https://platform.openai.com/docs/guides/structured-outputs) to generate well formed data that you can inspect with your code. For example, you might ask an agent to classify the task into a few categories, and then pick the next agent based on the category.
*   Chaining multiple agents by transforming the output of one into the input of the next. You can decompose a task like writing a blog post into a series of steps - do research, write an outline, write the blog post, critique it, and then improve it.
*   Running the agent that performs the task in a `while` loop with an agent that evaluates and provides feedback, until the evaluator says the output passes certain criteria.
*   Running multiple agents in parallel, e.g. via Python primitives like `asyncio.gather`. This is useful for speed when you have multiple tasks that don't depend on each other.

We have a number of examples in [`examples/agent_patterns`](https://github.com/openai/openai-agents-python/tree/main/examples/agent_patterns).


# Models - OpenAI Agents SDK
The Agents SDK comes with out-of-the-box support for OpenAI models in two flavors:

*   **Recommended**: the [`OpenAIResponsesModel`](about:blank/ref/models/openai_responses/#agents.models.openai_responses.OpenAIResponsesModel "OpenAIResponsesModel"), which calls OpenAI APIs using the new [Responses API](https://platform.openai.com/docs/api-reference/responses).
*   The [`OpenAIChatCompletionsModel`](about:blank/ref/models/openai_chatcompletions/#agents.models.openai_chatcompletions.OpenAIChatCompletionsModel "OpenAIChatCompletionsModel"), which calls OpenAI APIs using the [Chat Completions API](https://platform.openai.com/docs/api-reference/chat).

Mixing and matching models
--------------------------

Within a single workflow, you may want to use different models for each agent. For example, you could use a smaller, faster model for triage, while using a larger, more capable model for complex tasks. When configuring an [`Agent`](about:blank/ref/agent/#agents.agent.Agent "Agent
dataclass
"), you can select a specific model by either:

1.  Passing the name of an OpenAI model.
2.  Passing any model name + a [`ModelProvider`](about:blank/ref/models/interface/#agents.models.interface.ModelProvider "ModelProvider") that can map that name to a Model instance.
3.  Directly providing a [`Model`](about:blank/ref/models/interface/#agents.models.interface.Model "Model") implementation.

Note

While our SDK supports both the [`OpenAIResponsesModel`](about:blank/ref/models/openai_responses/#agents.models.openai_responses.OpenAIResponsesModel "OpenAIResponsesModel") and the [`OpenAIChatCompletionsModel`](about:blank/ref/models/openai_chatcompletions/#agents.models.openai_chatcompletions.OpenAIChatCompletionsModel "OpenAIChatCompletionsModel") shapes, we recommend using a single model shape for each workflow because the two shapes support a different set of features and tools. If your workflow requires mixing and matching model shapes, make sure that all the features you're using are available on both.

```
from agents import Agent, Runner, AsyncOpenAI, OpenAIChatCompletionsModel
import asyncio

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You only speak Spanish.",
    model="o3-mini", # (1)!
)

english_agent = Agent(
    name="English agent",
    instructions="You only speak English",
    model=OpenAIChatCompletionsModel( # (2)!
        model="gpt-4o",
        openai_client=AsyncOpenAI()
    ),
)

triage_agent = Agent(
    name="Triage agent",
    instructions="Handoff to the appropriate agent based on the language of the request.",
    handoffs=[spanish_agent, english_agent],
    model="gpt-3.5-turbo",
)

async def main():
    result = await Runner.run(triage_agent, input="Hola, ¿cómo estás?")
    print(result.final_output)

```


1.  Sets the name of an OpenAI model directly.
2.  Provides a [`Model`](about:blank/ref/models/interface/#agents.models.interface.Model "Model") implementation.

Using other LLM providers
-------------------------

You can use other LLM providers in 3 ways (examples [here](https://github.com/openai/openai-agents-python/tree/main/examples/model_providers/)):

1.  [`set_default_openai_client`](about:blank/ref/#agents.set_default_openai_client "set_default_openai_client") is useful in cases where you want to globally use an instance of `AsyncOpenAI` as the LLM client. This is for cases where the LLM provider has an OpenAI compatible API endpoint, and you can set the `base_url` and `api_key`. See a configurable example in [examples/model\_providers/custom\_example\_global.py](https://github.com/openai/openai-agents-python/tree/main/examples/model_providers/custom_example_global.py).
2.  [`ModelProvider`](about:blank/ref/models/interface/#agents.models.interface.ModelProvider "ModelProvider") is at the `Runner.run` level. This lets you say "use a custom model provider for all agents in this run". See a configurable example in [examples/model\_providers/custom\_example\_provider.py](https://github.com/openai/openai-agents-python/tree/main/examples/model_providers/custom_example_provider.py).
3.  [`Agent.model`](about:blank/ref/agent/#agents.agent.Agent.model "model
    class-attribute
    instance-attribute
    ") lets you specify the model on a specific Agent instance. This enables you to mix and match different providers for different agents. See a configurable example in [examples/model\_providers/custom\_example\_agent.py](https://github.com/openai/openai-agents-python/tree/main/examples/model_providers/custom_example_agent.py).

In cases where you do not have an API key from `platform.openai.com`, we recommend disabling tracing via `set_tracing_disabled()`, or setting up a [different tracing processor](../tracing/).

Note

In these examples, we use the Chat Completions API/model, because most LLM providers don't yet support the Responses API. If your LLM provider does support it, we recommend using Responses.

Common issues with using other LLM providers
--------------------------------------------

### Tracing client error 401

If you get errors related to tracing, this is because traces are uploaded to OpenAI servers, and you don't have an OpenAI API key. You have three options to resolve this:

1.  Disable tracing entirely: [`set_tracing_disabled(True)`](about:blank/ref/#agents.set_tracing_disabled "set_tracing_disabled").
2.  Set an OpenAI key for tracing: [`set_tracing_export_api_key(...)`](about:blank/ref/#agents.set_tracing_export_api_key "set_tracing_export_api_key"). This API key will only be used for uploading traces, and must be from [platform.openai.com](https://platform.openai.com/).
3.  Use a non-OpenAI trace processor. See the [tracing docs](about:blank/tracing/#custom-tracing-processors).

### Responses API support

The SDK uses the Responses API by default, but most other LLM providers don't yet support it. You may see 404s or similar issues as a result. To resolve, you have two options:

1.  Call [`set_default_openai_api("chat_completions")`](about:blank/ref/#agents.set_default_openai_api "set_default_openai_api"). This works if you are setting `OPENAI_API_KEY` and `OPENAI_BASE_URL` via environment vars.
2.  Use [`OpenAIChatCompletionsModel`](about:blank/ref/models/openai_chatcompletions/#agents.models.openai_chatcompletions.OpenAIChatCompletionsModel "OpenAIChatCompletionsModel"). There are examples [here](https://github.com/openai/openai-agents-python/tree/main/examples/model_providers/).

### Structured outputs support

Some model providers don't have support for [structured outputs](https://platform.openai.com/docs/guides/structured-outputs). This sometimes results in an error that looks something like this:

```
BadRequestError: Error code: 400 - {'error': {'message': "'response_format.type' : value is not one of the allowed values ['text','json_object']", 'type': 'invalid_request_error'}}

```


This is a shortcoming of some model providers - they support JSON outputs, but don't allow you to specify the `json_schema` to use for the output. We are working on a fix for this, but we suggest relying on providers that do have support for JSON schema output, because otherwise your app will often break because of malformed JSON.

# Configuring the SDK - OpenAI Agents SDK
API keys and clients
--------------------

By default, the SDK looks for the `OPENAI_API_KEY` environment variable for LLM requests and tracing, as soon as it is imported. If you are unable to set that environment variable before your app starts, you can use the [set\_default\_openai\_key()](about:blank/ref/#agents.set_default_openai_key "set_default_openai_key") function to set the key.

```
from agents import set_default_openai_key

set_default_openai_key("sk-...")

```


Alternatively, you can also configure an OpenAI client to be used. By default, the SDK creates an `AsyncOpenAI` instance, using the API key from the environment variable or the default key set above. You can change this by using the [set\_default\_openai\_client()](about:blank/ref/#agents.set_default_openai_client "set_default_openai_client") function.

```
from openai import AsyncOpenAI
from agents import set_default_openai_client

custom_client = AsyncOpenAI(base_url="...", api_key="...")
set_default_openai_client(custom_client)

```


Finally, you can also customize the OpenAI API that is used. By default, we use the OpenAI Responses API. You can override this to use the Chat Completions API by using the [set\_default\_openai\_api()](about:blank/ref/#agents.set_default_openai_api "set_default_openai_api") function.

```
from agents import set_default_openai_api

set_default_openai_api("chat_completions")

```


Tracing
-------

Tracing is enabled by default. It uses the OpenAI API keys from the section above by default (i.e. the environment variable or the default key you set). You can specifically set the API key used for tracing by using the [`set_tracing_export_api_key`](about:blank/ref/#agents.set_tracing_export_api_key "set_tracing_export_api_key") function.

```
from agents import set_tracing_export_api_key

set_tracing_export_api_key("sk-...")

```


You can also disable tracing entirely by using the [`set_tracing_disabled()`](about:blank/ref/#agents.set_tracing_disabled "set_tracing_disabled") function.

```
from agents import set_tracing_disabled

set_tracing_disabled(True)

```


Debug logging
-------------

The SDK has two Python loggers without any handlers set. By default, this means that warnings and errors are sent to `stdout`, but other logs are suppressed.

To enable verbose logging, use the [`enable_verbose_stdout_logging()`](about:blank/ref/#agents.enable_verbose_stdout_logging "enable_verbose_stdout_logging") function.

```
from agents import enable_verbose_stdout_logging

enable_verbose_stdout_logging()

```


Alternatively, you can customize the logs by adding handlers, filters, formatters, etc. You can read more in the [Python logging guide](https://docs.python.org/3/howto/logging.html).

```
import logging

logger =  logging.getLogger("openai.agents") # or openai.agents.tracing for the Tracing logger

# To make all logs show up
logger.setLevel(logging.DEBUG)
# To make info and above show up
logger.setLevel(logging.INFO)
# To make warning and above show up
logger.setLevel(logging.WARNING)
# etc

# You can customize this as needed, but this will output to `stderr` by default
logger.addHandler(logging.StreamHandler())

```


### Sensitive data in logs

Certain logs may contain sensitive data (for example, user data). If you want to disable this data from being logged, set the following environment variables.

To disable logging LLM inputs and outputs:

```
export OPENAI_AGENTS_DONT_LOG_MODEL_DATA=1

```


To disable logging tool inputs and outputs:

```
export OPENAI_AGENTS_DONT_LOG_TOOL_DATA=1

```

# Agent Visualization - OpenAI Agents SDK
Agent visualization allows you to generate a structured graphical representation of agents and their relationships using **Graphviz**. This is useful for understanding how agents, tools, and handoffs interact within an application.

Installation
------------

Install the optional `viz` dependency group:

```
pip install "openai-agents[viz]"

```


Generating a Graph
------------------

You can generate an agent visualization using the `draw_graph` function. This function creates a directed graph where:

*   **Agents** are represented as yellow boxes.
*   **Tools** are represented as green ellipses.
*   **Handoffs** are directed edges from one agent to another.

### Example Usage

```
from agents import Agent, function_tool
from agents.extensions.visualization import draw_graph

@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You only speak Spanish.",
)

english_agent = Agent(
    name="English agent",
    instructions="You only speak English",
)

triage_agent = Agent(
    name="Triage agent",
    instructions="Handoff to the appropriate agent based on the language of the request.",
    handoffs=[spanish_agent, english_agent],
    tools=[get_weather],
)

draw_graph(triage_agent)

```


![Agent Graph](../assets/images/graph.png)

This generates a graph that visually represents the structure of the **triage agent** and its connections to sub-agents and tools.

Understanding the Visualization
-------------------------------

The generated graph includes:

*   A **start node** (`__start__`) indicating the entry point.
*   Agents represented as **rectangles** with yellow fill.
*   Tools represented as **ellipses** with green fill.
*   Directed edges indicating interactions:
*   **Solid arrows** for agent-to-agent handoffs.
*   **Dotted arrows** for tool invocations.
*   An **end node** (`__end__`) indicating where execution terminates.

Customizing the Graph
---------------------

### Showing the Graph

By default, `draw_graph` displays the graph inline. To show the graph in a separate window, write the following:

```
draw_graph(triage_agent).view()

```


### Saving the Graph

By default, `draw_graph` displays the graph inline. To save it as a file, specify a filename:

```
draw_graph(triage_agent, filename="agent_graph.png")

```


This will generate `agent_graph.png` in the working directory.

# Quickstart - OpenAI Agents SDK
Prerequisites
-------------

Make sure you've followed the base [quickstart instructions](../../quickstart/) for the Agents SDK, and set up a virtual environment. Then, install the optional voice dependencies from the SDK:

```
pip install 'openai-agents[voice]'

```


Concepts
--------

The main concept to know about is a [`VoicePipeline`](about:blank/ref/voice/pipeline/#agents.voice.pipeline.VoicePipeline "VoicePipeline"), which is a 3 step process:

1.  Run a speech-to-text model to turn audio into text.
2.  Run your code, which is usually an agentic workflow, to produce a result.
3.  Run a text-to-speech model to turn the result text back into audio.

```
graph LR
    %% Input
    A["🎤 Audio Input"]

    %% Voice Pipeline
    subgraph Voice_Pipeline [Voice Pipeline]
        direction TB
        B["Transcribe (speech-to-text)"]
        C["Your Code"]:::highlight
        D["Text-to-speech"]
        B --> C --> D
    end

    %% Output
    E["🎧 Audio Output"]

    %% Flow
    A --> Voice_Pipeline
    Voice_Pipeline --> E

    %% Custom styling
    classDef highlight fill:#ffcc66,stroke:#333,stroke-width:1px,font-weight:700;

```


Agents
------

First, let's set up some Agents. This should feel familiar to you if you've built any agents with this SDK. We'll have a couple of Agents, a handoff, and a tool.

```
import asyncio
import random

from agents import (
    Agent,
    function_tool,
)
from agents.extensions.handoff_prompt import prompt_with_handoff_instructions



@function_tool
def get_weather(city: str) -> str:
    """Get the weather for a given city."""
    print(f"[debug] get_weather called with city: {city}")
    choices = ["sunny", "cloudy", "rainy", "snowy"]
    return f"The weather in {city} is {random.choice(choices)}."


spanish_agent = Agent(
    name="Spanish",
    handoff_description="A spanish speaking agent.",
    instructions=prompt_with_handoff_instructions(
        "You're speaking to a human, so be polite and concise. Speak in Spanish.",
    ),
    model="gpt-4o-mini",
)

agent = Agent(
    name="Assistant",
    instructions=prompt_with_handoff_instructions(
        "You're speaking to a human, so be polite and concise. If the user speaks in Spanish, handoff to the spanish agent.",
    ),
    model="gpt-4o-mini",
    handoffs=[spanish_agent],
    tools=[get_weather],
)

```


Voice pipeline
--------------

We'll set up a simple voice pipeline, using [`SingleAgentVoiceWorkflow`](about:blank/ref/voice/workflow/#agents.voice.workflow.SingleAgentVoiceWorkflow "SingleAgentVoiceWorkflow") as the workflow.

```
from agents.voice import SingleAgentVoiceWorkflow, VoicePipeline
pipeline = VoicePipeline(workflow=SingleAgentVoiceWorkflow(agent))

```


Run the pipeline
----------------

```
import numpy as np
import sounddevice as sd
from agents.voice import AudioInput

# For simplicity, we'll just create 3 seconds of silence
# In reality, you'd get microphone data
buffer = np.zeros(24000 * 3, dtype=np.int16)
audio_input = AudioInput(buffer=buffer)

result = await pipeline.run(audio_input)

# Create an audio player using `sounddevice`
player = sd.OutputStream(samplerate=24000, channels=1, dtype=np.int16)
player.start()

# Play the audio stream as it comes in
async for event in result.stream():
    if event.type == "voice_stream_event_audio":
        player.write(event.data)

```


Put it all together
-------------------

```
import asyncio
import random

import numpy as np
import sounddevice as sd

from agents import (
    Agent,
    function_tool,
    set_tracing_disabled,
)
from agents.voice import (
    AudioInput,
    SingleAgentVoiceWorkflow,
    VoicePipeline,
)
from agents.extensions.handoff_prompt import prompt_with_handoff_instructions


@function_tool
def get_weather(city: str) -> str:
    """Get the weather for a given city."""
    print(f"[debug] get_weather called with city: {city}")
    choices = ["sunny", "cloudy", "rainy", "snowy"]
    return f"The weather in {city} is {random.choice(choices)}."


spanish_agent = Agent(
    name="Spanish",
    handoff_description="A spanish speaking agent.",
    instructions=prompt_with_handoff_instructions(
        "You're speaking to a human, so be polite and concise. Speak in Spanish.",
    ),
    model="gpt-4o-mini",
)

agent = Agent(
    name="Assistant",
    instructions=prompt_with_handoff_instructions(
        "You're speaking to a human, so be polite and concise. If the user speaks in Spanish, handoff to the spanish agent.",
    ),
    model="gpt-4o-mini",
    handoffs=[spanish_agent],
    tools=[get_weather],
)


async def main():
    pipeline = VoicePipeline(workflow=SingleAgentVoiceWorkflow(agent))
    buffer = np.zeros(24000 * 3, dtype=np.int16)
    audio_input = AudioInput(buffer=buffer)

    result = await pipeline.run(audio_input)

    # Create an audio player using `sounddevice`
    player = sd.OutputStream(samplerate=24000, channels=1, dtype=np.int16)
    player.start()

    # Play the audio stream as it comes in
    async for event in result.stream():
        if event.type == "voice_stream_event_audio":
            player.write(event.data)


if __name__ == "__main__":
    asyncio.run(main())

```


If you run this example, the agent will speak to you! Check out the example in [examples/voice/static](https://github.com/openai/openai-agents-python/tree/main/examples/voice/static) to see a demo where you can speak to the agent yourself.

# Pipelines and workflows - OpenAI Agents SDK
[`VoicePipeline`](about:blank/ref/voice/pipeline/#agents.voice.pipeline.VoicePipeline "VoicePipeline") is a class that makes it easy to turn your agentic workflows into a voice app. You pass in a workflow to run, and the pipeline takes care of transcribing input audio, detecting when the audio ends, calling your workflow at the right time, and turning the workflow output back into audio.

```
graph LR
    %% Input
    A["🎤 Audio Input"]

    %% Voice Pipeline
    subgraph Voice_Pipeline [Voice Pipeline]
        direction TB
        B["Transcribe (speech-to-text)"]
        C["Your Code"]:::highlight
        D["Text-to-speech"]
        B --> C --> D
    end

    %% Output
    E["🎧 Audio Output"]

    %% Flow
    A --> Voice_Pipeline
    Voice_Pipeline --> E

    %% Custom styling
    classDef highlight fill:#ffcc66,stroke:#333,stroke-width:1px,font-weight:700;

```


Configuring a pipeline
----------------------

When you create a pipeline, you can set a few things:

1.  The [`workflow`](about:blank/ref/voice/workflow/#agents.voice.workflow.VoiceWorkflowBase "VoiceWorkflowBase"), which is the code that runs each time new audio is transcribed.
2.  The [`speech-to-text`](about:blank/ref/voice/model/#agents.voice.model.STTModel "STTModel") and [`text-to-speech`](about:blank/ref/voice/model/#agents.voice.model.TTSModel "TTSModel") models used
3.  The [`config`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig "VoicePipelineConfig
    dataclass
    "), which lets you configure things like:
    *   A model provider, which can map model names to models
    *   Tracing, including whether to disable tracing, whether audio files are uploaded, the workflow name, trace IDs etc.
    *   Settings on the TTS and STT models, like the prompt, language and data types used.

Running a pipeline
------------------

You can run a pipeline via the [`run()`](about:blank/ref/voice/pipeline/#agents.voice.pipeline.VoicePipeline.run "run
async
") method, which lets you pass in audio input in two forms:

1.  [`AudioInput`](about:blank/ref/voice/input/#agents.voice.input.AudioInput "AudioInput
    dataclass
    ") is used when you have a full audio transcript, and just want to produce a result for it. This is useful in cases where you don't need to detect when a speaker is done speaking; for example, when you have pre-recorded audio or in push-to-talk apps where it's clear when the user is done speaking.
2.  [`StreamedAudioInput`](about:blank/ref/voice/input/#agents.voice.input.StreamedAudioInput "StreamedAudioInput") is used when you might need to detect when a user is done speaking. It allows you to push audio chunks as they are detected, and the voice pipeline will automatically run the agent workflow at the right time, via a process called "activity detection".

Results
-------

The result of a voice pipeline run is a [`StreamedAudioResult`](about:blank/ref/voice/result/#agents.voice.result.StreamedAudioResult "StreamedAudioResult"). This is an object that lets you stream events as they occur. There are a few kinds of [`VoiceStreamEvent`](about:blank/ref/voice/events/#agents.voice.events.VoiceStreamEvent "VoiceStreamEvent
module-attribute
"), including:

1.  [`VoiceStreamEventAudio`](about:blank/ref/voice/events/#agents.voice.events.VoiceStreamEventAudio "VoiceStreamEventAudio
    dataclass
    "), which contains a chunk of audio.
2.  [`VoiceStreamEventLifecycle`](about:blank/ref/voice/events/#agents.voice.events.VoiceStreamEventLifecycle "VoiceStreamEventLifecycle
    dataclass
    "), which informs you of lifecycle events like a turn starting or ending.
3.  [`VoiceStreamEventError`](about:blank/ref/voice/events/#agents.voice.events.VoiceStreamEventError "VoiceStreamEventError
    dataclass
    "), is an error event.

```
result = await pipeline.run(input)

async for event in result.stream():
    if event.type == "voice_stream_event_audio":
        # play audio
    elif event.type == "voice_stream_event_lifecycle":
        # lifecycle
    elif event.type == "voice_stream_event_error"
        # error
    ...

```


Best practices
--------------

### Interruptions

The Agents SDK currently does not support any built-in interruptions support for [`StreamedAudioInput`](about:blank/ref/voice/input/#agents.voice.input.StreamedAudioInput "StreamedAudioInput"). Instead for every detected turn it will trigger a separate run of your workflow. If you want to handle interruptions inside your application you can listen to the [`VoiceStreamEventLifecycle`](about:blank/ref/voice/events/#agents.voice.events.VoiceStreamEventLifecycle "VoiceStreamEventLifecycle
dataclass
") events. `turn_started` will indicate that a new turn was transcribed and processing is beginning. `turn_ended` will trigger after all the audio was dispatched for a respective turn. You could use these events to mute the microphone of the speaker when the model starts a turn and unmute it after you flushed all the related audio for a turn.

# Tracing - OpenAI Agents SDK
Just like the way [agents are traced](../../tracing/), voice pipelines are also automatically traced.

You can read the tracing doc above for basic tracing information, but you can additionally configure tracing of a pipeline via [`VoicePipelineConfig`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig "VoicePipelineConfig
dataclass
").

Key tracing related fields are:

*   [`tracing_disabled`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.tracing_disabled "tracing_disabled
    class-attribute
    instance-attribute
    "): controls whether tracing is disabled. By default, tracing is enabled.
*   [`trace_include_sensitive_data`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_data "trace_include_sensitive_data
    class-attribute
    instance-attribute
    "): controls whether traces include potentially sensitive data, like audio transcripts. This is specifically for the voice pipeline, and not for anything that goes on inside your Workflow.
*   [`trace_include_sensitive_audio_data`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_audio_data "trace_include_sensitive_audio_data
    class-attribute
    instance-attribute
    "): controls whether traces include audio data.
*   [`workflow_name`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.workflow_name "workflow_name
    class-attribute
    instance-attribute
    "): The name of the trace workflow.
*   [`group_id`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.group_id "group_id
    class-attribute
    instance-attribute
    "): The `group_id` of the trace, which lets you link multiple traces.
*   [`trace_metadata`](about:blank/ref/voice/pipeline_config/#agents.voice.pipeline_config.VoicePipelineConfig.tracing_disabled "tracing_disabled
    class-attribute
    instance-attribute
    "): Additional metadata to include with the trace.

# Agents module - OpenAI Agents SDK
### set\_default\_openai\_key

```
set_default_openai_key(
    key: str, use_for_tracing: bool = True
) -> None

```


Set the default OpenAI API key to use for LLM requests (and optionally tracing(). This is only necessary if the OPENAI\_API\_KEY environment variable is not already set.

If provided, this key will be used instead of the OPENAI\_API\_KEY environment variable.

Parameters:



* Name:                 key            
  * Type:                   str            
  * Description:               The OpenAI key to use.            
  * Default:                 required            
* Name:                 use_for_tracing            
  * Type:                   bool            
  * Description:               Whether to also use this key to send traces to OpenAI. Defaults to TrueIf False, you'll either need to set the OPENAI_API_KEY environment variable or callset_tracing_export_api_key() with the API key you want to use for tracing.            
  * Default:                   True            


Source code in `src/agents/__init__.py`



### set\_default\_openai\_client

```
set_default_openai_client(
    client: AsyncOpenAI, use_for_tracing: bool = True
) -> None

```


Set the default OpenAI client to use for LLM requests and/or tracing. If provided, this client will be used instead of the default OpenAI client.

Parameters:



* Name:                 client            
  * Type:                   AsyncOpenAI            
  * Description:               The OpenAI client to use.            
  * Default:                 required            
* Name:                 use_for_tracing            
  * Type:                   bool            
  * Description:               Whether to use the API key from this client for uploading traces. If False,you'll either need to set the OPENAI_API_KEY environment variable or callset_tracing_export_api_key() with the API key you want to use for tracing.            
  * Default:                   True            


Source code in `src/agents/__init__.py`



### set\_default\_openai\_api

```
set_default_openai_api(
    api: Literal["chat_completions", "responses"],
) -> None

```


Set the default API to use for OpenAI LLM requests. By default, we will use the responses API but you can set this to use the chat completions API instead.

Source code in `src/agents/__init__.py`



### set\_tracing\_export\_api\_key

```
set_tracing_export_api_key(api_key: str) -> None

```


Set the OpenAI API key for the backend exporter.

Source code in `src/agents/tracing/__init__.py`



### set\_tracing\_disabled

```
set_tracing_disabled(disabled: bool) -> None

```


Set whether tracing is globally disabled.

Source code in `src/agents/tracing/__init__.py`



### set\_trace\_processors

Set the list of trace processors. This will replace the current list of processors.

Source code in `src/agents/tracing/__init__.py`



### enable\_verbose\_stdout\_logging

```
enable_verbose_stdout_logging()

```


Enables verbose logging to stdout. This is useful for debugging.

Source code in `src/agents/__init__.py`



Table of contents

*   [agent](#agents.agent)
*   [ToolsToFinalOutputFunction](#agents.agent.ToolsToFinalOutputFunction)
*   [ToolsToFinalOutputResult](#agents.agent.ToolsToFinalOutputResult)
    
    *   [is\_final\_output](#agents.agent.ToolsToFinalOutputResult.is_final_output)
    *   [final\_output](#agents.agent.ToolsToFinalOutputResult.final_output)
    
*   [StopAtTools](#agents.agent.StopAtTools)
    
    *   [stop\_at\_tool\_names](#agents.agent.StopAtTools.stop_at_tool_names)
    
*   [Agent](#agents.agent.Agent)
    
    *   [name](#agents.agent.Agent.name)
    *   [instructions](#agents.agent.Agent.instructions)
    *   [handoff\_description](#agents.agent.Agent.handoff_description)
    *   [handoffs](#agents.agent.Agent.handoffs)
    *   [model](#agents.agent.Agent.model)
    *   [model\_settings](#agents.agent.Agent.model_settings)
    *   [tools](#agents.agent.Agent.tools)
    *   [mcp\_servers](#agents.agent.Agent.mcp_servers)
    *   [input\_guardrails](#agents.agent.Agent.input_guardrails)
    *   [output\_guardrails](#agents.agent.Agent.output_guardrails)
    *   [output\_type](#agents.agent.Agent.output_type)
    *   [hooks](#agents.agent.Agent.hooks)
    *   [tool\_use\_behavior](#agents.agent.Agent.tool_use_behavior)
    *   [reset\_tool\_choice](#agents.agent.Agent.reset_tool_choice)
    *   [clone](#agents.agent.Agent.clone)
    *   [as\_tool](#agents.agent.Agent.as_tool)
    *   [get\_system\_prompt](#agents.agent.Agent.get_system_prompt)
    *   [get\_mcp\_tools](#agents.agent.Agent.get_mcp_tools)
    *   [get\_all\_tools](#agents.agent.Agent.get_all_tools)
    

`Agents`
========

### ToolsToFinalOutputFunction `module-attribute`

```
ToolsToFinalOutputFunction: TypeAlias = Callable[
    [RunContextWrapper[TContext], list[FunctionToolResult]],
    MaybeAwaitable[ToolsToFinalOutputResult],
]

```


A function that takes a run context and a list of tool results, and returns a `ToolToFinalOutputResult`.

### ToolsToFinalOutputResult `dataclass`

Source code in `src/agents/agent.py`



#### is\_final\_output `instance-attribute`

```
is_final_output: bool

```


Whether this is the final output. If False, the LLM will run again and receive the tool call output.

#### final\_output `class-attribute` `instance-attribute`

```
final_output: Any | None = None

```


The final output. Can be None if `is_final_output` is False, otherwise must match the `output_type` of the agent.

### StopAtTools

Bases: `TypedDict`

Source code in `src/agents/agent.py`



#### stop\_at\_tool\_names `instance-attribute`

```
stop_at_tool_names: list[str]

```


A list of tool names, any of which will stop the agent from running further.

### Agent `dataclass`

Bases: `Generic[TContext]`

An agent is an AI model configured with instructions, tools, guardrails, handoffs and more.

We strongly recommend passing `instructions`, which is the "system prompt" for the agent. In addition, you can pass `handoff_description`, which is a human-readable description of the agent, used when the agent is used inside tools/handoffs.

Agents are generic on the context type. The context is a (mutable) object you create. It is passed to tool functions, handoffs, guardrails, etc.

Source code in `src/agents/agent.py`



#### name `instance-attribute`

```
name: str

```


The name of the agent.

#### instructions `class-attribute` `instance-attribute`

```
instructions: (
    str
    | Callable[
        [RunContextWrapper[TContext], Agent[TContext]],
        MaybeAwaitable[str],
    ]
    | None
) = None

```


The instructions for the agent. Will be used as the "system prompt" when this agent is invoked. Describes what the agent should do, and how it responds.

Can either be a string, or a function that dynamically generates instructions for the agent. If you provide a function, it will be called with the context and the agent instance. It must return a string.

#### handoff\_description `class-attribute` `instance-attribute`

```
handoff_description: str | None = None

```


A description of the agent. This is used when the agent is used as a handoff, so that an LLM knows what it does and when to invoke it.

#### handoffs `class-attribute` `instance-attribute`

```
handoffs: list[Agent[Any] | Handoff[TContext]] = field(
    default_factory=list
)

```


Handoffs are sub-agents that the agent can delegate to. You can provide a list of handoffs, and the agent can choose to delegate to them if relevant. Allows for separation of concerns and modularity.

#### model `class-attribute` `instance-attribute`

```
model: str | Model | None = None

```


The model implementation to use when invoking the LLM.

By default, if not set, the agent will use the default model configured in `model_settings.DEFAULT_MODEL`.

#### model\_settings `class-attribute` `instance-attribute`

```
model_settings: ModelSettings = field(
    default_factory=ModelSettings
)

```


Configures model-specific tuning parameters (e.g. temperature, top\_p).

#### tools `class-attribute` `instance-attribute`

```
tools: list[Tool] = field(default_factory=list)

```


A list of tools that the agent can use.

#### mcp\_servers `class-attribute` `instance-attribute`

```
mcp_servers: list[MCPServer] = field(default_factory=list)

```


A list of [Model Context Protocol](https://modelcontextprotocol.io/) servers that the agent can use. Every time the agent runs, it will include tools from these servers in the list of available tools.

NOTE: You are expected to manage the lifecycle of these servers. Specifically, you must call `server.connect()` before passing it to the agent, and `server.cleanup()` when the server is no longer needed.

#### input\_guardrails `class-attribute` `instance-attribute`

```
input_guardrails: list[InputGuardrail[TContext]] = field(
    default_factory=list
)

```


A list of checks that run in parallel to the agent's execution, before generating a response. Runs only if the agent is the first agent in the chain.

#### output\_guardrails `class-attribute` `instance-attribute`

```
output_guardrails: list[OutputGuardrail[TContext]] = field(
    default_factory=list
)

```


A list of checks that run on the final output of the agent, after generating a response. Runs only if the agent produces a final output.

#### output\_type `class-attribute` `instance-attribute`

```
output_type: type[Any] | None = None

```


The type of the output object. If not provided, the output will be `str`.

#### hooks `class-attribute` `instance-attribute`

```
hooks: AgentHooks[TContext] | None = None

```


A class that receives callbacks on various lifecycle events for this agent.

#### tool\_use\_behavior `class-attribute` `instance-attribute`

```
tool_use_behavior: (
    Literal["run_llm_again", "stop_on_first_tool"]
    | StopAtTools
    | ToolsToFinalOutputFunction
) = "run_llm_again"

```


This lets you configure how tool use is handled. - "run\_llm\_again": The default behavior. Tools are run, and then the LLM receives the results and gets to respond. - "stop\_on\_first\_tool": The output of the first tool call is used as the final output. This means that the LLM does not process the result of the tool call. - A list of tool names: The agent will stop running if any of the tools in the list are called. The final output will be the output of the first matching tool call. The LLM does not process the result of the tool call. - A function: If you pass a function, it will be called with the run context and the list of tool results. It must return a `ToolToFinalOutputResult`, which determines whether the tool calls result in a final output.

NOTE: This configuration is specific to FunctionTools. Hosted tools, such as file search, web search, etc are always processed by the LLM.

#### reset\_tool\_choice `class-attribute` `instance-attribute`

```
reset_tool_choice: bool = True

```


Whether to reset the tool choice to the default value after a tool has been called. Defaults to True. This ensures that the agent doesn't enter an infinite loop of tool usage.

#### clone

```
clone(**kwargs: Any) -> Agent[TContext]

```


Make a copy of the agent, with the given arguments changed. For example, you could do:

```
new_agent = agent.clone(instructions="New instructions")

```


Source code in `src/agents/agent.py`



#### as\_tool

```
as_tool(
    tool_name: str | None,
    tool_description: str | None,
    custom_output_extractor: Callable[
        [RunResult], Awaitable[str]
    ]
    | None = None,
) -> Tool

```


Transform this agent into a tool, callable by other agents.

This is different from handoffs in two ways: 1. In handoffs, the new agent receives the conversation history. In this tool, the new agent receives generated input. 2. In handoffs, the new agent takes over the conversation. In this tool, the new agent is called as a tool, and the conversation is continued by the original agent.

Parameters:



* Name:                 tool_name            
  * Type:                   str | None            
  * Description:                               The name of the tool. If not provided, the agent's name will be used.                          
  * Default:                 required            
* Name:                 tool_description            
  * Type:                   str | None            
  * Description:                               The description of the tool, which should indicate what it does andwhen to use it.                          
  * Default:                 required            
* Name:                 custom_output_extractor            
  * Type:                   Callable[[RunResult], Awaitable[str]] | None            
  * Description:                               A function that extracts the output from the agent. If notprovided, the last message from the agent will be used.                          
  * Default:                   None            


Source code in `src/agents/agent.py`



#### get\_system\_prompt `async`

```
get_system_prompt(
    run_context: RunContextWrapper[TContext],
) -> str | None

```


Get the system prompt for the agent.

Source code in `src/agents/agent.py`



#### get\_mcp\_tools `async`

```
get_mcp_tools() -> list[Tool]

```


Fetches the available tools from the MCP servers.

Source code in `src/agents/agent.py`



#### get\_all\_tools `async`

```
get_all_tools() -> list[Tool]

```


All agent tools, including MCP tools and function tools.

Source code in `src/agents/agent.py`



Table of contents

*   [run](#agents.run)
*   [Runner](#agents.run.Runner)
    
    *   [run](#agents.run.Runner.run)
    *   [run\_sync](#agents.run.Runner.run_sync)
    *   [run\_streamed](#agents.run.Runner.run_streamed)
    
*   [RunConfig](#agents.run.RunConfig)
    
    *   [model](#agents.run.RunConfig.model)
    *   [model\_provider](#agents.run.RunConfig.model_provider)
    *   [model\_settings](#agents.run.RunConfig.model_settings)
    *   [handoff\_input\_filter](#agents.run.RunConfig.handoff_input_filter)
    *   [input\_guardrails](#agents.run.RunConfig.input_guardrails)
    *   [output\_guardrails](#agents.run.RunConfig.output_guardrails)
    *   [tracing\_disabled](#agents.run.RunConfig.tracing_disabled)
    *   [trace\_include\_sensitive\_data](#agents.run.RunConfig.trace_include_sensitive_data)
    *   [workflow\_name](#agents.run.RunConfig.workflow_name)
    *   [trace\_id](#agents.run.RunConfig.trace_id)
    *   [group\_id](#agents.run.RunConfig.group_id)
    *   [trace\_metadata](#agents.run.RunConfig.trace_metadata)
    

`Runner`
========

### Runner

Source code in `src/agents/run.py`



#### run `async` `classmethod`

```
run(
    starting_agent: Agent[TContext],
    input: str | list[TResponseInputItem],
    *,
    context: TContext | None = None,
    max_turns: int = DEFAULT_MAX_TURNS,
    hooks: RunHooks[TContext] | None = None,
    run_config: RunConfig | None = None,
) -> RunResult

```


Run a workflow starting at the given agent. The agent will run in a loop until a final output is generated. The loop runs like so: 1. The agent is invoked with the given input. 2. If there is a final output (i.e. the agent produces something of type `agent.output_type`, the loop terminates. 3. If there's a handoff, we run the loop again, with the new agent. 4. Else, we run tool calls (if any), and re-run the loop.

In two cases, the agent may raise an exception: 1. If the max\_turns is exceeded, a MaxTurnsExceeded exception is raised. 2. If a guardrail tripwire is triggered, a GuardrailTripwireTriggered exception is raised.

Note that only the first agent's input guardrails are run.

Parameters:



* Name:                 starting_agent            
  * Type:                   Agent[TContext]            
  * Description:                               The starting agent to run.                          
  * Default:                 required            
* Name:                 input            
  * Type:                   str | list[TResponseInputItem]            
  * Description:                               The initial input to the agent. You can pass a single string for a user message,or a list of input items.                          
  * Default:                 required            
* Name:                 context            
  * Type:                   TContext | None            
  * Description:                               The context to run the agent with.                          
  * Default:                   None            
* Name:                 max_turns            
  * Type:                   int            
  * Description:                               The maximum number of turns to run the agent for. A turn is defined as oneAI invocation (including any tool calls that might occur).                          
  * Default:                   DEFAULT_MAX_TURNS            
* Name:                 hooks            
  * Type:                   RunHooks[TContext] | None            
  * Description:                               An object that receives callbacks on various lifecycle events.                          
  * Default:                   None            
* Name:                 run_config            
  * Type:                   RunConfig | None            
  * Description:                               Global settings for the entire agent run.                          
  * Default:                   None            


Returns:



* Type:                   RunResult            
  * Description:                               A run result containing all the inputs, guardrail results and the output of the last                          
* Type:                   RunResult            
  * Description:                               agent. Agents may perform handoffs, so we don't know the specific type of the output.                          


Source code in `src/agents/run.py`



#### run\_sync `classmethod`

```
run_sync(
    starting_agent: Agent[TContext],
    input: str | list[TResponseInputItem],
    *,
    context: TContext | None = None,
    max_turns: int = DEFAULT_MAX_TURNS,
    hooks: RunHooks[TContext] | None = None,
    run_config: RunConfig | None = None,
) -> RunResult

```


Run a workflow synchronously, starting at the given agent. Note that this just wraps the `run` method, so it will not work if there's already an event loop (e.g. inside an async function, or in a Jupyter notebook or async context like FastAPI). For those cases, use the `run` method instead.

The agent will run in a loop until a final output is generated. The loop runs like so: 1. The agent is invoked with the given input. 2. If there is a final output (i.e. the agent produces something of type `agent.output_type`, the loop terminates. 3. If there's a handoff, we run the loop again, with the new agent. 4. Else, we run tool calls (if any), and re-run the loop.

In two cases, the agent may raise an exception: 1. If the max\_turns is exceeded, a MaxTurnsExceeded exception is raised. 2. If a guardrail tripwire is triggered, a GuardrailTripwireTriggered exception is raised.

Note that only the first agent's input guardrails are run.

Parameters:



* Name:                 starting_agent            
  * Type:                   Agent[TContext]            
  * Description:                               The starting agent to run.                          
  * Default:                 required            
* Name:                 input            
  * Type:                   str | list[TResponseInputItem]            
  * Description:                               The initial input to the agent. You can pass a single string for a user message,or a list of input items.                          
  * Default:                 required            
* Name:                 context            
  * Type:                   TContext | None            
  * Description:                               The context to run the agent with.                          
  * Default:                   None            
* Name:                 max_turns            
  * Type:                   int            
  * Description:                               The maximum number of turns to run the agent for. A turn is defined as oneAI invocation (including any tool calls that might occur).                          
  * Default:                   DEFAULT_MAX_TURNS            
* Name:                 hooks            
  * Type:                   RunHooks[TContext] | None            
  * Description:                               An object that receives callbacks on various lifecycle events.                          
  * Default:                   None            
* Name:                 run_config            
  * Type:                   RunConfig | None            
  * Description:                               Global settings for the entire agent run.                          
  * Default:                   None            


Returns:



* Type:                   RunResult            
  * Description:                               A run result containing all the inputs, guardrail results and the output of the last                          
* Type:                   RunResult            
  * Description:                               agent. Agents may perform handoffs, so we don't know the specific type of the output.                          


Source code in `src/agents/run.py`



#### run\_streamed `classmethod`

```
run_streamed(
    starting_agent: Agent[TContext],
    input: str | list[TResponseInputItem],
    context: TContext | None = None,
    max_turns: int = DEFAULT_MAX_TURNS,
    hooks: RunHooks[TContext] | None = None,
    run_config: RunConfig | None = None,
) -> RunResultStreaming

```


Run a workflow starting at the given agent in streaming mode. The returned result object contains a method you can use to stream semantic events as they are generated.

The agent will run in a loop until a final output is generated. The loop runs like so: 1. The agent is invoked with the given input. 2. If there is a final output (i.e. the agent produces something of type `agent.output_type`, the loop terminates. 3. If there's a handoff, we run the loop again, with the new agent. 4. Else, we run tool calls (if any), and re-run the loop.

In two cases, the agent may raise an exception: 1. If the max\_turns is exceeded, a MaxTurnsExceeded exception is raised. 2. If a guardrail tripwire is triggered, a GuardrailTripwireTriggered exception is raised.

Note that only the first agent's input guardrails are run.

Parameters:



* Name:                 starting_agent            
  * Type:                   Agent[TContext]            
  * Description:                               The starting agent to run.                          
  * Default:                 required            
* Name:                 input            
  * Type:                   str | list[TResponseInputItem]            
  * Description:                               The initial input to the agent. You can pass a single string for a user message,or a list of input items.                          
  * Default:                 required            
* Name:                 context            
  * Type:                   TContext | None            
  * Description:                               The context to run the agent with.                          
  * Default:                   None            
* Name:                 max_turns            
  * Type:                   int            
  * Description:                               The maximum number of turns to run the agent for. A turn is defined as oneAI invocation (including any tool calls that might occur).                          
  * Default:                   DEFAULT_MAX_TURNS            
* Name:                 hooks            
  * Type:                   RunHooks[TContext] | None            
  * Description:                               An object that receives callbacks on various lifecycle events.                          
  * Default:                   None            
* Name:                 run_config            
  * Type:                   RunConfig | None            
  * Description:                               Global settings for the entire agent run.                          
  * Default:                   None            


Returns:



* Type:                   RunResultStreaming            
  * Description:                               A result object that contains data about the run, as well as a method to stream events.                          


Source code in `src/agents/run.py`



### RunConfig `dataclass`

Configures settings for the entire agent run.

Source code in `src/agents/run.py`



#### model `class-attribute` `instance-attribute`

```
model: str | Model | None = None

```


The model to use for the entire agent run. If set, will override the model set on every agent. The model\_provider passed in below must be able to resolve this model name.

#### model\_provider `class-attribute` `instance-attribute`

```
model_provider: ModelProvider = field(
    default_factory=OpenAIProvider
)

```


The model provider to use when looking up string model names. Defaults to OpenAI.

#### model\_settings `class-attribute` `instance-attribute`

```
model_settings: ModelSettings | None = None

```


Configure global model settings. Any non-null values will override the agent-specific model settings.

#### handoff\_input\_filter `class-attribute` `instance-attribute`

```
handoff_input_filter: HandoffInputFilter | None = None

```


A global input filter to apply to all handoffs. If `Handoff.input_filter` is set, then that will take precedence. The input filter allows you to edit the inputs that are sent to the new agent. See the documentation in `Handoff.input_filter` for more details.

#### input\_guardrails `class-attribute` `instance-attribute`

```
input_guardrails: list[InputGuardrail[Any]] | None = None

```


A list of input guardrails to run on the initial run input.

#### output\_guardrails `class-attribute` `instance-attribute`

```
output_guardrails: list[OutputGuardrail[Any]] | None = None

```


A list of output guardrails to run on the final output of the run.

#### tracing\_disabled `class-attribute` `instance-attribute`

```
tracing_disabled: bool = False

```


Whether tracing is disabled for the agent run. If disabled, we will not trace the agent run.

#### trace\_include\_sensitive\_data `class-attribute` `instance-attribute`

```
trace_include_sensitive_data: bool = True

```


Whether we include potentially sensitive data (for example: inputs/outputs of tool calls or LLM generations) in traces. If False, we'll still create spans for these events, but the sensitive data will not be included.

#### workflow\_name `class-attribute` `instance-attribute`

```
workflow_name: str = 'Agent workflow'

```


The name of the run, used for tracing. Should be a logical name for the run, like "Code generation workflow" or "Customer support agent".

#### trace\_id `class-attribute` `instance-attribute`

```
trace_id: str | None = None

```


A custom trace ID to use for tracing. If not provided, we will generate a new trace ID.

#### group\_id `class-attribute` `instance-attribute`

```
group_id: str | None = None

```


A grouping identifier to use for tracing, to link multiple traces from the same conversation or process. For example, you might use a chat thread ID.

#### trace\_metadata `class-attribute` `instance-attribute`

```
trace_metadata: dict[str, Any] | None = None

```


An optional dictionary of additional metadata to include with the trace.

Table of contents

*   [tool](#agents.tool)
*   [Tool](#agents.tool.Tool)
*   [FunctionToolResult](#agents.tool.FunctionToolResult)
    
    *   [tool](#agents.tool.FunctionToolResult.tool)
    *   [output](#agents.tool.FunctionToolResult.output)
    *   [run\_item](#agents.tool.FunctionToolResult.run_item)
    
*   [FunctionTool](#agents.tool.FunctionTool)
    
    *   [name](#agents.tool.FunctionTool.name)
    *   [description](#agents.tool.FunctionTool.description)
    *   [params\_json\_schema](#agents.tool.FunctionTool.params_json_schema)
    *   [on\_invoke\_tool](#agents.tool.FunctionTool.on_invoke_tool)
    *   [strict\_json\_schema](#agents.tool.FunctionTool.strict_json_schema)
    
*   [FileSearchTool](#agents.tool.FileSearchTool)
    
    *   [vector\_store\_ids](#agents.tool.FileSearchTool.vector_store_ids)
    *   [max\_num\_results](#agents.tool.FileSearchTool.max_num_results)
    *   [include\_search\_results](#agents.tool.FileSearchTool.include_search_results)
    *   [ranking\_options](#agents.tool.FileSearchTool.ranking_options)
    *   [filters](#agents.tool.FileSearchTool.filters)
    
*   [WebSearchTool](#agents.tool.WebSearchTool)
    
    *   [user\_location](#agents.tool.WebSearchTool.user_location)
    *   [search\_context\_size](#agents.tool.WebSearchTool.search_context_size)
    
*   [ComputerTool](#agents.tool.ComputerTool)
    
    *   [computer](#agents.tool.ComputerTool.computer)
    
*   [default\_tool\_error\_function](#agents.tool.default_tool_error_function)
*   [function\_tool](#agents.tool.function_tool)

`Tools`
=======

### Tool `module-attribute`

```
Tool = Union[
    FunctionTool,
    FileSearchTool,
    WebSearchTool,
    ComputerTool,
]

```


A tool that can be used in an agent.

### FunctionToolResult `dataclass`

Source code in `src/agents/tool.py`



#### tool `instance-attribute`

```
tool: FunctionTool

```


The tool that was run.

#### output `instance-attribute`

```
output: Any

```


The output of the tool.

#### run\_item `instance-attribute`

```
run_item: RunItem

```


The run item that was produced as a result of the tool call.

### FunctionTool `dataclass`

A tool that wraps a function. In most cases, you should use the `function_tool` helpers to create a FunctionTool, as they let you easily wrap a Python function.

Source code in `src/agents/tool.py`



#### name `instance-attribute`

```
name: str

```


The name of the tool, as shown to the LLM. Generally the name of the function.

#### description `instance-attribute`

```
description: str

```


A description of the tool, as shown to the LLM.

#### params\_json\_schema `instance-attribute`

```
params_json_schema: dict[str, Any]

```


The JSON schema for the tool's parameters.

#### on\_invoke\_tool `instance-attribute`

```
on_invoke_tool: Callable[
    [RunContextWrapper[Any], str], Awaitable[Any]
]

```


A function that invokes the tool with the given context and parameters. The params passed are: 1. The tool run context. 2. The arguments from the LLM, as a JSON string.

You must return a string representation of the tool output, or something we can call `str()` on. In case of errors, you can either raise an Exception (which will cause the run to fail) or return a string error message (which will be sent back to the LLM).

#### strict\_json\_schema `class-attribute` `instance-attribute`

```
strict_json_schema: bool = True

```


Whether the JSON schema is in strict mode. We **strongly** recommend setting this to True, as it increases the likelihood of correct JSON input.

### FileSearchTool `dataclass`

A hosted tool that lets the LLM search through a vector store. Currently only supported with OpenAI models, using the Responses API.

Source code in `src/agents/tool.py`



#### vector\_store\_ids `instance-attribute`

```
vector_store_ids: list[str]

```


The IDs of the vector stores to search.

#### max\_num\_results `class-attribute` `instance-attribute`

```
max_num_results: int | None = None

```


The maximum number of results to return.

#### include\_search\_results `class-attribute` `instance-attribute`

```
include_search_results: bool = False

```


Whether to include the search results in the output produced by the LLM.

#### ranking\_options `class-attribute` `instance-attribute`

```
ranking_options: RankingOptions | None = None

```


Ranking options for search.

#### filters `class-attribute` `instance-attribute`

```
filters: Filters | None = None

```


A filter to apply based on file attributes.

### WebSearchTool `dataclass`

A hosted tool that lets the LLM search the web. Currently only supported with OpenAI models, using the Responses API.

Source code in `src/agents/tool.py`



#### user\_location `class-attribute` `instance-attribute`

```
user_location: UserLocation | None = None

```


Optional location for the search. Lets you customize results to be relevant to a location.

#### search\_context\_size `class-attribute` `instance-attribute`

```
search_context_size: Literal["low", "medium", "high"] = (
    "medium"
)

```


The amount of context to use for the search.

### ComputerTool `dataclass`

A hosted tool that lets the LLM control a computer.

Source code in `src/agents/tool.py`



#### computer `instance-attribute`

```
computer: Computer | AsyncComputer

```


The computer implementation, which describes the environment and dimensions of the computer, as well as implements the computer actions like click, screenshot, etc.

### default\_tool\_error\_function

```
default_tool_error_function(
    ctx: RunContextWrapper[Any], error: Exception
) -> str

```


The default tool error function, which just returns a generic error message.

Source code in `src/agents/tool.py`



### function\_tool

```
function_tool(
    func: ToolFunction[...],
    *,
    name_override: str | None = None,
    description_override: str | None = None,
    docstring_style: DocstringStyle | None = None,
    use_docstring_info: bool = True,
    failure_error_function: ToolErrorFunction | None = None,
    strict_mode: bool = True,
) -> FunctionTool

```


```
function_tool(
    *,
    name_override: str | None = None,
    description_override: str | None = None,
    docstring_style: DocstringStyle | None = None,
    use_docstring_info: bool = True,
    failure_error_function: ToolErrorFunction | None = None,
    strict_mode: bool = True,
) -> Callable[[ToolFunction[...]], FunctionTool]

```


```
function_tool(
    func: ToolFunction[...] | None = None,
    *,
    name_override: str | None = None,
    description_override: str | None = None,
    docstring_style: DocstringStyle | None = None,
    use_docstring_info: bool = True,
    failure_error_function: ToolErrorFunction
    | None = default_tool_error_function,
    strict_mode: bool = True,
) -> (
    FunctionTool
    | Callable[[ToolFunction[...]], FunctionTool]
)

```


Decorator to create a FunctionTool from a function. By default, we will: 1. Parse the function signature to create a JSON schema for the tool's parameters. 2. Use the function's docstring to populate the tool's description. 3. Use the function's docstring to populate argument descriptions. The docstring style is detected automatically, but you can override it.

If the function takes a `RunContextWrapper` as the first argument, it _must_ match the context type of the agent that uses the tool.

Parameters:



* Name:                 func            
  * Type:                   ToolFunction[...] | None            
  * Description:                               The function to wrap.                          
  * Default:                   None            
* Name:                 name_override            
  * Type:                   str | None            
  * Description:                               If provided, use this name for the tool instead of the function's name.                          
  * Default:                   None            
* Name:                 description_override            
  * Type:                   str | None            
  * Description:                               If provided, use this description for the tool instead of thefunction's docstring.                          
  * Default:                   None            
* Name:                 docstring_style            
  * Type:                   DocstringStyle | None            
  * Description:                               If provided, use this style for the tool's docstring. If not provided,we will attempt to auto-detect the style.                          
  * Default:                   None            
* Name:                 use_docstring_info            
  * Type:                   bool            
  * Description:                               If True, use the function's docstring to populate the tool'sdescription and argument descriptions.                          
  * Default:                   True            
* Name:                 failure_error_function            
  * Type:                   ToolErrorFunction | None            
  * Description:                               If provided, use this function to generate an error message whenthe tool call fails. The error message is sent to the LLM. If you pass None, then noerror message will be sent and instead an Exception will be raised.                          
  * Default:                   default_tool_error_function            
* Name:                 strict_mode            
  * Type:                   bool            
  * Description:                               Whether to enable strict mode for the tool's JSON schema. We stronglyrecommend setting this to True, as it increases the likelihood of correct JSON input.If False, it allows non-strict JSON schemas. For example, if a parameter has a defaultvalue, it will be optional, additional properties are allowed, etc. See here for more:https://platform.openai.com/docs/guides/structured-outputs?api-mode=responses#supported-schemas                          
  * Default:                   True            


Source code in `src/agents/tool.py`



Table of contents

*   [result](#agents.result)
*   [RunResultBase](#agents.result.RunResultBase)
    
    *   [input](#agents.result.RunResultBase.input)
    *   [new\_items](#agents.result.RunResultBase.new_items)
    *   [raw\_responses](#agents.result.RunResultBase.raw_responses)
    *   [final\_output](#agents.result.RunResultBase.final_output)
    *   [input\_guardrail\_results](#agents.result.RunResultBase.input_guardrail_results)
    *   [output\_guardrail\_results](#agents.result.RunResultBase.output_guardrail_results)
    *   [last\_agent](#agents.result.RunResultBase.last_agent)
    *   [final\_output\_as](#agents.result.RunResultBase.final_output_as)
    *   [to\_input\_list](#agents.result.RunResultBase.to_input_list)
    
*   [RunResult](#agents.result.RunResult)
    
    *   [input](#agents.result.RunResult.input)
    *   [new\_items](#agents.result.RunResult.new_items)
    *   [raw\_responses](#agents.result.RunResult.raw_responses)
    *   [final\_output](#agents.result.RunResult.final_output)
    *   [input\_guardrail\_results](#agents.result.RunResult.input_guardrail_results)
    *   [output\_guardrail\_results](#agents.result.RunResult.output_guardrail_results)
    *   [last\_agent](#agents.result.RunResult.last_agent)
    *   [final\_output\_as](#agents.result.RunResult.final_output_as)
    *   [to\_input\_list](#agents.result.RunResult.to_input_list)
    
*   [RunResultStreaming](#agents.result.RunResultStreaming)
    
    *   [input](#agents.result.RunResultStreaming.input)
    *   [new\_items](#agents.result.RunResultStreaming.new_items)
    *   [raw\_responses](#agents.result.RunResultStreaming.raw_responses)
    *   [input\_guardrail\_results](#agents.result.RunResultStreaming.input_guardrail_results)
    *   [output\_guardrail\_results](#agents.result.RunResultStreaming.output_guardrail_results)
    *   [current\_agent](#agents.result.RunResultStreaming.current_agent)
    *   [current\_turn](#agents.result.RunResultStreaming.current_turn)
    *   [max\_turns](#agents.result.RunResultStreaming.max_turns)
    *   [final\_output](#agents.result.RunResultStreaming.final_output)
    *   [is\_complete](#agents.result.RunResultStreaming.is_complete)
    *   [last\_agent](#agents.result.RunResultStreaming.last_agent)
    *   [final\_output\_as](#agents.result.RunResultStreaming.final_output_as)
    *   [to\_input\_list](#agents.result.RunResultStreaming.to_input_list)
    *   [stream\_events](#agents.result.RunResultStreaming.stream_events)
    

`Results`
=========

### RunResultBase `dataclass`

Bases: `ABC`

Source code in `src/agents/result.py`



#### input `instance-attribute`

```
input: str | list[TResponseInputItem]

```


The original input items i.e. the items before run() was called. This may be a mutated version of the input, if there are handoff input filters that mutate the input.

#### new\_items `instance-attribute`

```
new_items: list[RunItem]

```


The new items generated during the agent run. These include things like new messages, tool calls and their outputs, etc.

#### raw\_responses `instance-attribute`

```
raw_responses: list[ModelResponse]

```


The raw LLM responses generated by the model during the agent run.

#### final\_output `instance-attribute`

```
final_output: Any

```


The output of the last agent.

#### input\_guardrail\_results `instance-attribute`

```
input_guardrail_results: list[InputGuardrailResult]

```


Guardrail results for the input messages.

#### output\_guardrail\_results `instance-attribute`

```
output_guardrail_results: list[OutputGuardrailResult]

```


Guardrail results for the final output of the agent.

#### last\_agent `abstractmethod` `property`

```
last_agent: Agent[Any]

```


The last agent that was run.

#### final\_output\_as

```
final_output_as(
    cls: type[T], raise_if_incorrect_type: bool = False
) -> T

```


A convenience method to cast the final output to a specific type. By default, the cast is only for the typechecker. If you set `raise_if_incorrect_type` to True, we'll raise a TypeError if the final output is not of the given type.

Parameters:



* Name:                 cls            
  * Type:                   type[T]            
  * Description:                               The type to cast the final output to.                          
  * Default:                 required            
* Name:                 raise_if_incorrect_type            
  * Type:                   bool            
  * Description:                               If True, we'll raise a TypeError if the final output is not ofthe given type.                          
  * Default:                   False            


Returns:



* Type:                   T            
  * Description:                               The final output casted to the given type.                          


Source code in `src/agents/result.py`



#### to\_input\_list

```
to_input_list() -> list[TResponseInputItem]

```


Creates a new input list, merging the original input with all the new items generated.

Source code in `src/agents/result.py`



### RunResult `dataclass`

Bases: `[RunResultBase](#agents.result.RunResultBase "RunResultBase dataclass (agents.result.RunResultBase)")`

Source code in `src/agents/result.py`



#### input `instance-attribute`

```
input: str | list[TResponseInputItem]

```


The original input items i.e. the items before run() was called. This may be a mutated version of the input, if there are handoff input filters that mutate the input.

#### new\_items `instance-attribute`

```
new_items: list[RunItem]

```


The new items generated during the agent run. These include things like new messages, tool calls and their outputs, etc.

#### raw\_responses `instance-attribute`

```
raw_responses: list[ModelResponse]

```


The raw LLM responses generated by the model during the agent run.

#### final\_output `instance-attribute`

```
final_output: Any

```


The output of the last agent.

#### input\_guardrail\_results `instance-attribute`

```
input_guardrail_results: list[InputGuardrailResult]

```


Guardrail results for the input messages.

#### output\_guardrail\_results `instance-attribute`

```
output_guardrail_results: list[OutputGuardrailResult]

```


Guardrail results for the final output of the agent.

#### last\_agent `property`

```
last_agent: Agent[Any]

```


The last agent that was run.

#### final\_output\_as

```
final_output_as(
    cls: type[T], raise_if_incorrect_type: bool = False
) -> T

```


A convenience method to cast the final output to a specific type. By default, the cast is only for the typechecker. If you set `raise_if_incorrect_type` to True, we'll raise a TypeError if the final output is not of the given type.

Parameters:



* Name:                 cls            
  * Type:                   type[T]            
  * Description:                               The type to cast the final output to.                          
  * Default:                 required            
* Name:                 raise_if_incorrect_type            
  * Type:                   bool            
  * Description:                               If True, we'll raise a TypeError if the final output is not ofthe given type.                          
  * Default:                   False            


Returns:



* Type:                   T            
  * Description:                               The final output casted to the given type.                          


Source code in `src/agents/result.py`



#### to\_input\_list

```
to_input_list() -> list[TResponseInputItem]

```


Creates a new input list, merging the original input with all the new items generated.

Source code in `src/agents/result.py`



### RunResultStreaming `dataclass`

Bases: `[RunResultBase](#agents.result.RunResultBase "RunResultBase dataclass (agents.result.RunResultBase)")`

The result of an agent run in streaming mode. You can use the `stream_events` method to receive semantic events as they are generated.

The streaming method will raise: - A MaxTurnsExceeded exception if the agent exceeds the max\_turns limit. - A GuardrailTripwireTriggered exception if a guardrail is tripped.

Source code in `src/agents/result.py`



#### input `instance-attribute`

```
input: str | list[TResponseInputItem]

```


The original input items i.e. the items before run() was called. This may be a mutated version of the input, if there are handoff input filters that mutate the input.

#### new\_items `instance-attribute`

```
new_items: list[RunItem]

```


The new items generated during the agent run. These include things like new messages, tool calls and their outputs, etc.

#### raw\_responses `instance-attribute`

```
raw_responses: list[ModelResponse]

```


The raw LLM responses generated by the model during the agent run.

#### input\_guardrail\_results `instance-attribute`

```
input_guardrail_results: list[InputGuardrailResult]

```


Guardrail results for the input messages.

#### output\_guardrail\_results `instance-attribute`

```
output_guardrail_results: list[OutputGuardrailResult]

```


Guardrail results for the final output of the agent.

#### current\_agent `instance-attribute`

```
current_agent: Agent[Any]

```


The current agent that is running.

#### current\_turn `instance-attribute`

```
current_turn: int

```


The current turn number.

#### max\_turns `instance-attribute`

```
max_turns: int

```


The maximum number of turns the agent can run for.

#### final\_output `instance-attribute`

```
final_output: Any

```


The final output of the agent. This is None until the agent has finished running.

#### is\_complete `class-attribute` `instance-attribute`

```
is_complete: bool = False

```


Whether the agent has finished running.

#### last\_agent `property`

```
last_agent: Agent[Any]

```


The last agent that was run. Updates as the agent run progresses, so the true last agent is only available after the agent run is complete.

#### final\_output\_as

```
final_output_as(
    cls: type[T], raise_if_incorrect_type: bool = False
) -> T

```


A convenience method to cast the final output to a specific type. By default, the cast is only for the typechecker. If you set `raise_if_incorrect_type` to True, we'll raise a TypeError if the final output is not of the given type.

Parameters:



* Name:                 cls            
  * Type:                   type[T]            
  * Description:                               The type to cast the final output to.                          
  * Default:                 required            
* Name:                 raise_if_incorrect_type            
  * Type:                   bool            
  * Description:                               If True, we'll raise a TypeError if the final output is not ofthe given type.                          
  * Default:                   False            


Returns:



* Type:                   T            
  * Description:                               The final output casted to the given type.                          


Source code in `src/agents/result.py`



#### to\_input\_list

```
to_input_list() -> list[TResponseInputItem]

```


Creates a new input list, merging the original input with all the new items generated.

Source code in `src/agents/result.py`



#### stream\_events `async`

```
stream_events() -> AsyncIterator[StreamEvent]

```


Stream deltas for new items as they are generated. We're using the types from the OpenAI Responses API, so these are semantic events: each event has a `type` field that describes the type of the event, along with the data for that event.

This will raise: - A MaxTurnsExceeded exception if the agent exceeds the max\_turns limit. - A GuardrailTripwireTriggered exception if a guardrail is tripped.

Source code in `src/agents/result.py`



Table of contents

*   [stream\_events](#agents.stream_events)
*   [StreamEvent](#agents.stream_events.StreamEvent)
*   [RawResponsesStreamEvent](#agents.stream_events.RawResponsesStreamEvent)
    
    *   [data](#agents.stream_events.RawResponsesStreamEvent.data)
    *   [type](#agents.stream_events.RawResponsesStreamEvent.type)
    
*   [RunItemStreamEvent](#agents.stream_events.RunItemStreamEvent)
    
    *   [name](#agents.stream_events.RunItemStreamEvent.name)
    *   [item](#agents.stream_events.RunItemStreamEvent.item)
    
*   [AgentUpdatedStreamEvent](#agents.stream_events.AgentUpdatedStreamEvent)
    
    *   [new\_agent](#agents.stream_events.AgentUpdatedStreamEvent.new_agent)
    

`Streaming events`
==================

### StreamEvent `module-attribute`

```
StreamEvent: TypeAlias = Union[
    RawResponsesStreamEvent,
    RunItemStreamEvent,
    AgentUpdatedStreamEvent,
]

```


A streaming event from an agent.

### RawResponsesStreamEvent `dataclass`

Streaming event from the LLM. These are 'raw' events, i.e. they are directly passed through from the LLM.

Source code in `src/agents/stream_events.py`



#### data `instance-attribute`

```
data: TResponseStreamEvent

```


The raw responses streaming event from the LLM.

#### type `class-attribute` `instance-attribute`

```
type: Literal['raw_response_event'] = 'raw_response_event'

```


The type of the event.

### RunItemStreamEvent `dataclass`

Streaming events that wrap a `RunItem`. As the agent processes the LLM response, it will generate these events for new messages, tool calls, tool outputs, handoffs, etc.

Source code in `src/agents/stream_events.py`



#### name `instance-attribute`

```
name: Literal[
    "message_output_created",
    "handoff_requested",
    "handoff_occured",
    "tool_called",
    "tool_output",
    "reasoning_item_created",
]

```


The name of the event.

#### item `instance-attribute`

```
item: RunItem

```


The item that was created.

### AgentUpdatedStreamEvent `dataclass`

Event that notifies that there is a new agent running.

Source code in `src/agents/stream_events.py`



#### new\_agent `instance-attribute`

```
new_agent: Agent[Any]

```


The new agent.




Table of contents

*   [handoffs](#agents.handoffs)
*   [HandoffInputFilter](#agents.handoffs.HandoffInputFilter)
*   [HandoffInputData](#agents.handoffs.HandoffInputData)
    
    *   [input\_history](#agents.handoffs.HandoffInputData.input_history)
    *   [pre\_handoff\_items](#agents.handoffs.HandoffInputData.pre_handoff_items)
    *   [new\_items](#agents.handoffs.HandoffInputData.new_items)
    
*   [Handoff](#agents.handoffs.Handoff)
    
    *   [tool\_name](#agents.handoffs.Handoff.tool_name)
    *   [tool\_description](#agents.handoffs.Handoff.tool_description)
    *   [input\_json\_schema](#agents.handoffs.Handoff.input_json_schema)
    *   [on\_invoke\_handoff](#agents.handoffs.Handoff.on_invoke_handoff)
    *   [agent\_name](#agents.handoffs.Handoff.agent_name)
    *   [input\_filter](#agents.handoffs.Handoff.input_filter)
    *   [strict\_json\_schema](#agents.handoffs.Handoff.strict_json_schema)
    
*   [handoff](#agents.handoffs.handoff)

`Handoffs`
==========

### HandoffInputFilter `module-attribute`

```
HandoffInputFilter: TypeAlias = Callable[
    [HandoffInputData], HandoffInputData
]

```


A function that filters the input data passed to the next agent.

### HandoffInputData `dataclass`

Source code in `src/agents/handoffs.py`



#### input\_history `instance-attribute`

```
input_history: str | tuple[TResponseInputItem, ...]

```


The input history before `Runner.run()` was called.

#### pre\_handoff\_items `instance-attribute`

```
pre_handoff_items: tuple[RunItem, ...]

```


The items generated before the agent turn where the handoff was invoked.

#### new\_items `instance-attribute`

```
new_items: tuple[RunItem, ...]

```


The new items generated during the current agent turn, including the item that triggered the handoff and the tool output message representing the response from the handoff output.

### Handoff `dataclass`

Bases: `Generic[TContext]`

A handoff is when an agent delegates a task to another agent. For example, in a customer support scenario you might have a "triage agent" that determines which agent should handle the user's request, and sub-agents that specialize in different areas like billing, account management, etc.

Source code in `src/agents/handoffs.py`



#### tool\_name `instance-attribute`

```
tool_name: str

```


The name of the tool that represents the handoff.

#### tool\_description `instance-attribute`

```
tool_description: str

```


The description of the tool that represents the handoff.

#### input\_json\_schema `instance-attribute`

```
input_json_schema: dict[str, Any]

```


The JSON schema for the handoff input. Can be empty if the handoff does not take an input.

#### on\_invoke\_handoff `instance-attribute`

```
on_invoke_handoff: Callable[
    [RunContextWrapper[Any], str],
    Awaitable[Agent[TContext]],
]

```


The function that invokes the handoff. The parameters passed are: 1. The handoff run context 2. The arguments from the LLM, as a JSON string. Empty string if input\_json\_schema is empty.

Must return an agent.

#### agent\_name `instance-attribute`

```
agent_name: str

```


The name of the agent that is being handed off to.

#### input\_filter `class-attribute` `instance-attribute`

```
input_filter: HandoffInputFilter | None = None

```


A function that filters the inputs that are passed to the next agent. By default, the new agent sees the entire conversation history. In some cases, you may want to filter inputs e.g. to remove older inputs, or remove tools from existing inputs.

The function will receive the entire conversation history so far, including the input item that triggered the handoff and a tool call output item representing the handoff tool's output.

You are free to modify the input history or new items as you see fit. The next agent that runs will receive `handoff_input_data.all_items`.

IMPORTANT: in streaming mode, we will not stream anything as a result of this function. The items generated before will already have been streamed.

#### strict\_json\_schema `class-attribute` `instance-attribute`

```
strict_json_schema: bool = True

```


Whether the input JSON schema is in strict mode. We **strongly** recommend setting this to True, as it increases the likelihood of correct JSON input.

### handoff

```
handoff(
    agent: Agent[TContext],
    *,
    tool_name_override: str | None = None,
    tool_description_override: str | None = None,
    input_filter: Callable[
        [HandoffInputData], HandoffInputData
    ]
    | None = None,
) -> Handoff[TContext]

```


```
handoff(
    agent: Agent[TContext],
    *,
    on_handoff: OnHandoffWithInput[THandoffInput],
    input_type: type[THandoffInput],
    tool_description_override: str | None = None,
    tool_name_override: str | None = None,
    input_filter: Callable[
        [HandoffInputData], HandoffInputData
    ]
    | None = None,
) -> Handoff[TContext]

```


```
handoff(
    agent: Agent[TContext],
    *,
    on_handoff: OnHandoffWithoutInput,
    tool_description_override: str | None = None,
    tool_name_override: str | None = None,
    input_filter: Callable[
        [HandoffInputData], HandoffInputData
    ]
    | None = None,
) -> Handoff[TContext]

```


```
handoff(
    agent: Agent[TContext],
    tool_name_override: str | None = None,
    tool_description_override: str | None = None,
    on_handoff: OnHandoffWithInput[THandoffInput]
    | OnHandoffWithoutInput
    | None = None,
    input_type: type[THandoffInput] | None = None,
    input_filter: Callable[
        [HandoffInputData], HandoffInputData
    ]
    | None = None,
) -> Handoff[TContext]

```


Create a handoff from an agent.

Parameters:



* Name:                 agent            
  * Type:                   Agent[TContext]            
  * Description:                               The agent to handoff to, or a function that returns an agent.                          
  * Default:                 required            
* Name:                 tool_name_override            
  * Type:                   str | None            
  * Description:                               Optional override for the name of the tool that represents the handoff.                          
  * Default:                   None            
* Name:                 tool_description_override            
  * Type:                   str | None            
  * Description:                               Optional override for the description of the tool thatrepresents the handoff.                          
  * Default:                   None            
* Name:                 on_handoff            
  * Type:                   OnHandoffWithInput[THandoffInput] | OnHandoffWithoutInput | None            
  * Description:                               A function that runs when the handoff is invoked.                          
  * Default:                   None            
* Name:                 input_type            
  * Type:                   type[THandoffInput] | None            
  * Description:                               the type of the input to the handoff. If provided, the input will be validatedagainst this type. Only relevant if you pass a function that takes an input.                          
  * Default:                   None            
* Name:                 input_filter            
  * Type:                   Callable[[HandoffInputData], HandoffInputData] | None            
  * Description:                               a function that filters the inputs that are passed to the next agent.                          
  * Default:                   None            


Source code in `src/agents/handoffs.py`




Table of contents

*   [lifecycle](#agents.lifecycle)
*   [RunHooks](#agents.lifecycle.RunHooks)
    
    *   [on\_agent\_start](#agents.lifecycle.RunHooks.on_agent_start)
    *   [on\_agent\_end](#agents.lifecycle.RunHooks.on_agent_end)
    *   [on\_handoff](#agents.lifecycle.RunHooks.on_handoff)
    *   [on\_tool\_start](#agents.lifecycle.RunHooks.on_tool_start)
    *   [on\_tool\_end](#agents.lifecycle.RunHooks.on_tool_end)
    
*   [AgentHooks](#agents.lifecycle.AgentHooks)
    
    *   [on\_start](#agents.lifecycle.AgentHooks.on_start)
    *   [on\_end](#agents.lifecycle.AgentHooks.on_end)
    *   [on\_handoff](#agents.lifecycle.AgentHooks.on_handoff)
    *   [on\_tool\_start](#agents.lifecycle.AgentHooks.on_tool_start)
    *   [on\_tool\_end](#agents.lifecycle.AgentHooks.on_tool_end)
    

`Lifecycle`
===========

### RunHooks

Bases: `Generic[TContext]`

A class that receives callbacks on various lifecycle events in an agent run. Subclass and override the methods you need.

#### on\_agent\_start `async`

```
on_agent_start(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
) -> None

```


Called before the agent is invoked. Called each time the current agent changes.

#### on\_agent\_end `async`

```
on_agent_end(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    output: Any,
) -> None

```


Called when the agent produces a final output.

#### on\_handoff `async`

```
on_handoff(
    context: RunContextWrapper[TContext],
    from_agent: Agent[TContext],
    to_agent: Agent[TContext],
) -> None

```


Called when a handoff occurs.

#### on\_tool\_start `async`

```
on_tool_start(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    tool: Tool,
) -> None

```


Called before a tool is invoked.

#### on\_tool\_end `async`

```
on_tool_end(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    tool: Tool,
    result: str,
) -> None

```


Called after a tool is invoked.

### AgentHooks

Bases: `Generic[TContext]`

A class that receives callbacks on various lifecycle events for a specific agent. You can set this on `agent.hooks` to receive events for that specific agent.

Subclass and override the methods you need.

#### on\_start `async`

```
on_start(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
) -> None

```


Called before the agent is invoked. Called each time the running agent is changed to this agent.

#### on\_end `async`

```
on_end(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    output: Any,
) -> None

```


Called when the agent produces a final output.

#### on\_handoff `async`

```
on_handoff(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    source: Agent[TContext],
) -> None

```


Called when the agent is being handed off to. The `source` is the agent that is handing off to this agent.

#### on\_tool\_start `async`

```
on_tool_start(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    tool: Tool,
) -> None

```


Called before a tool is invoked.

#### on\_tool\_end `async`

```
on_tool_end(
    context: RunContextWrapper[TContext],
    agent: Agent[TContext],
    tool: Tool,
    result: str,
) -> None

```


Called after a tool is invoked.



Table of contents

*   [items](#agents.items)
*   [TResponse](#agents.items.TResponse)
*   [TResponseInputItem](#agents.items.TResponseInputItem)
*   [TResponseOutputItem](#agents.items.TResponseOutputItem)
*   [TResponseStreamEvent](#agents.items.TResponseStreamEvent)
*   [ToolCallItemTypes](#agents.items.ToolCallItemTypes)
*   [RunItem](#agents.items.RunItem)
*   [RunItemBase](#agents.items.RunItemBase)
    
    *   [agent](#agents.items.RunItemBase.agent)
    *   [raw\_item](#agents.items.RunItemBase.raw_item)
    *   [to\_input\_item](#agents.items.RunItemBase.to_input_item)
    
*   [MessageOutputItem](#agents.items.MessageOutputItem)
    
    *   [agent](#agents.items.MessageOutputItem.agent)
    *   [raw\_item](#agents.items.MessageOutputItem.raw_item)
    *   [to\_input\_item](#agents.items.MessageOutputItem.to_input_item)
    
*   [HandoffCallItem](#agents.items.HandoffCallItem)
    
    *   [agent](#agents.items.HandoffCallItem.agent)
    *   [raw\_item](#agents.items.HandoffCallItem.raw_item)
    *   [to\_input\_item](#agents.items.HandoffCallItem.to_input_item)
    
*   [HandoffOutputItem](#agents.items.HandoffOutputItem)
    
    *   [agent](#agents.items.HandoffOutputItem.agent)
    *   [raw\_item](#agents.items.HandoffOutputItem.raw_item)
    *   [source\_agent](#agents.items.HandoffOutputItem.source_agent)
    *   [target\_agent](#agents.items.HandoffOutputItem.target_agent)
    *   [to\_input\_item](#agents.items.HandoffOutputItem.to_input_item)
    
*   [ToolCallItem](#agents.items.ToolCallItem)
    
    *   [agent](#agents.items.ToolCallItem.agent)
    *   [raw\_item](#agents.items.ToolCallItem.raw_item)
    *   [to\_input\_item](#agents.items.ToolCallItem.to_input_item)
    
*   [ToolCallOutputItem](#agents.items.ToolCallOutputItem)
    
    *   [agent](#agents.items.ToolCallOutputItem.agent)
    *   [raw\_item](#agents.items.ToolCallOutputItem.raw_item)
    *   [output](#agents.items.ToolCallOutputItem.output)
    *   [to\_input\_item](#agents.items.ToolCallOutputItem.to_input_item)
    
*   [ReasoningItem](#agents.items.ReasoningItem)
    
    *   [agent](#agents.items.ReasoningItem.agent)
    *   [raw\_item](#agents.items.ReasoningItem.raw_item)
    *   [to\_input\_item](#agents.items.ReasoningItem.to_input_item)
    
*   [ModelResponse](#agents.items.ModelResponse)
    
    *   [output](#agents.items.ModelResponse.output)
    *   [usage](#agents.items.ModelResponse.usage)
    *   [referenceable\_id](#agents.items.ModelResponse.referenceable_id)
    *   [to\_input\_items](#agents.items.ModelResponse.to_input_items)
    
*   [ItemHelpers](#agents.items.ItemHelpers)
    
    *   [extract\_last\_content](#agents.items.ItemHelpers.extract_last_content)
    *   [extract\_last\_text](#agents.items.ItemHelpers.extract_last_text)
    *   [input\_to\_new\_input\_list](#agents.items.ItemHelpers.input_to_new_input_list)
    *   [text\_message\_outputs](#agents.items.ItemHelpers.text_message_outputs)
    *   [text\_message\_output](#agents.items.ItemHelpers.text_message_output)
    *   [tool\_call\_output\_item](#agents.items.ItemHelpers.tool_call_output_item)
    

`Items`
=======

### TResponse `module-attribute`

```
TResponse = Response

```


A type alias for the Response type from the OpenAI SDK.

### TResponseInputItem `module-attribute`

```
TResponseInputItem = ResponseInputItemParam

```


A type alias for the ResponseInputItemParam type from the OpenAI SDK.

### TResponseOutputItem `module-attribute`

```
TResponseOutputItem = ResponseOutputItem

```


A type alias for the ResponseOutputItem type from the OpenAI SDK.

### TResponseStreamEvent `module-attribute`

```
TResponseStreamEvent = ResponseStreamEvent

```


A type alias for the ResponseStreamEvent type from the OpenAI SDK.

### ToolCallItemTypes `module-attribute`

```
ToolCallItemTypes: TypeAlias = Union[
    ResponseFunctionToolCall,
    ResponseComputerToolCall,
    ResponseFileSearchToolCall,
    ResponseFunctionWebSearch,
]

```


A type that represents a tool call item.

### RunItem `module-attribute`

```
RunItem: TypeAlias = Union[
    MessageOutputItem,
    HandoffCallItem,
    HandoffOutputItem,
    ToolCallItem,
    ToolCallOutputItem,
    ReasoningItem,
]

```


An item generated by an agent.

### RunItemBase `dataclass`

Bases: `Generic[T]`, `ABC`

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: T

```


The raw Responses item from the run. This will always be a either an output item (i.e. `openai.types.responses.ResponseOutputItem` or an input item (i.e. `openai.types.responses.ResponseInputItemParam`).

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### MessageOutputItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[ResponseOutputMessage]`

Represents a message from the LLM.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: ResponseOutputMessage

```


The raw response output message.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### HandoffCallItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[ResponseFunctionToolCall]`

Represents a tool call for a handoff from one agent to another.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: ResponseFunctionToolCall

```


The raw response function tool call that represents the handoff.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### HandoffOutputItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[[TResponseInputItem](#agents.items.TResponseInputItem "TResponseInputItem module-attribute (agents.items.TResponseInputItem)")]`

Represents the output of a handoff.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: TResponseInputItem

```


The raw input item that represents the handoff taking place.

#### source\_agent `instance-attribute`

```
source_agent: Agent[Any]

```


The agent that made the handoff.

#### target\_agent `instance-attribute`

```
target_agent: Agent[Any]

```


The agent that is being handed off to.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### ToolCallItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[[ToolCallItemTypes](#agents.items.ToolCallItemTypes "ToolCallItemTypes module-attribute (agents.items.ToolCallItemTypes)")]`

Represents a tool call e.g. a function call or computer action call.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: ToolCallItemTypes

```


The raw tool call item.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### ToolCallOutputItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[Union[FunctionCallOutput, ComputerCallOutput]]`

Represents the output of a tool call.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: FunctionCallOutput | ComputerCallOutput

```


The raw item from the model.

#### output `instance-attribute`

```
output: Any

```


The output of the tool call. This is whatever the tool call returned; the `raw_item` contains a string representation of the output.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### ReasoningItem `dataclass`

Bases: `[RunItemBase](#agents.items.RunItemBase "RunItemBase dataclass (agents.items.RunItemBase)")
[ResponseReasoningItem]`

Represents a reasoning item.

Source code in `src/agents/items.py`



#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent whose run caused this item to be generated.

#### raw\_item `instance-attribute`

```
raw_item: ResponseReasoningItem

```


The raw reasoning item.

#### to\_input\_item

```
to_input_item() -> TResponseInputItem

```


Converts this item into an input item suitable for passing to the model.

Source code in `src/agents/items.py`



### ModelResponse `dataclass`

Source code in `src/agents/items.py`



#### output `instance-attribute`

```
output: list[TResponseOutputItem]

```


A list of outputs (messages, tool calls, etc) generated by the model

#### usage `instance-attribute`

```
usage: Usage

```


The usage information for the response.

#### referenceable\_id `instance-attribute`

```
referenceable_id: str | None

```


An ID for the response which can be used to refer to the response in subsequent calls to the model. Not supported by all model providers.

#### to\_input\_items

```
to_input_items() -> list[TResponseInputItem]

```


Convert the output into a list of input items suitable for passing to the model.

Source code in `src/agents/items.py`



### ItemHelpers

Source code in `src/agents/items.py`



#### extract\_last\_content `classmethod`

```
extract_last_content(message: TResponseOutputItem) -> str

```


Extracts the last text content or refusal from a message.

Source code in `src/agents/items.py`



#### extract\_last\_text `classmethod`

```
extract_last_text(
    message: TResponseOutputItem,
) -> str | None

```


Extracts the last text content from a message, if any. Ignores refusals.

Source code in `src/agents/items.py`



#### input\_to\_new\_input\_list `classmethod`

```
input_to_new_input_list(
    input: str | list[TResponseInputItem],
) -> list[TResponseInputItem]

```


Converts a string or list of input items into a list of input items.

Source code in `src/agents/items.py`



#### text\_message\_outputs `classmethod`

```
text_message_outputs(items: list[RunItem]) -> str

```


Concatenates all the text content from a list of message output items.

Source code in `src/agents/items.py`



#### text\_message\_output `classmethod`

```
text_message_output(message: MessageOutputItem) -> str

```


Extracts all the text content from a single message output item.

Source code in `src/agents/items.py`



#### tool\_call\_output\_item `classmethod`

```
tool_call_output_item(
    tool_call: ResponseFunctionToolCall, output: str
) -> FunctionCallOutput

```


Creates a tool call output item from a tool call and its output.

Source code in `src/agents/items.py`



Table of contents

*   [run\_context](#agents.run_context)
*   [RunContextWrapper](#agents.run_context.RunContextWrapper)
    
    *   [context](#agents.run_context.RunContextWrapper.context)
    *   [usage](#agents.run_context.RunContextWrapper.usage)
    

`Run context`
=============

### RunContextWrapper `dataclass`

Bases: `Generic[TContext]`

This wraps the context object that you passed to `Runner.run()`. It also contains information about the usage of the agent run so far.

NOTE: Contexts are not passed to the LLM. They're a way to pass dependencies and data to code you implement, like tool functions, callbacks, hooks, etc.

Source code in `src/agents/run_context.py`



#### context `instance-attribute`

```
context: TContext

```


The context object (or None), passed by you to `Runner.run()`

#### usage `class-attribute` `instance-attribute`

```
usage: Usage = field(default_factory=Usage)

```


The usage of the agent run so far. For streamed responses, the usage will be stale until the last chunk of the stream is processed.


Table of contents

*   [usage](#agents.usage)
*   [Usage](#agents.usage.Usage)
    
    *   [requests](#agents.usage.Usage.requests)
    *   [input\_tokens](#agents.usage.Usage.input_tokens)
    *   [output\_tokens](#agents.usage.Usage.output_tokens)
    *   [total\_tokens](#agents.usage.Usage.total_tokens)
    

`Usage`
=======

### Usage `dataclass`

Source code in `src/agents/usage.py`



#### requests `class-attribute` `instance-attribute`

```
requests: int = 0

```


Total requests made to the LLM API.

#### input\_tokens `class-attribute` `instance-attribute`

```
input_tokens: int = 0

```


Total input tokens sent, across all requests.

#### output\_tokens `class-attribute` `instance-attribute`

```
output_tokens: int = 0

```


Total output tokens received, across all requests.

#### total\_tokens `class-attribute` `instance-attribute`

```
total_tokens: int = 0

```


Total tokens sent and received, across all requests.


Table of contents

*   [exceptions](#agents.exceptions)
*   [AgentsException](#agents.exceptions.AgentsException)
*   [MaxTurnsExceeded](#agents.exceptions.MaxTurnsExceeded)
*   [ModelBehaviorError](#agents.exceptions.ModelBehaviorError)
*   [UserError](#agents.exceptions.UserError)
*   [InputGuardrailTripwireTriggered](#agents.exceptions.InputGuardrailTripwireTriggered)
    
    *   [guardrail\_result](#agents.exceptions.InputGuardrailTripwireTriggered.guardrail_result)
    
*   [OutputGuardrailTripwireTriggered](#agents.exceptions.OutputGuardrailTripwireTriggered)
    
    *   [guardrail\_result](#agents.exceptions.OutputGuardrailTripwireTriggered.guardrail_result)
    

`Exceptions`
============

### AgentsException

Bases: `Exception`

Base class for all exceptions in the Agents SDK.

Source code in `src/agents/exceptions.py`



### MaxTurnsExceeded

Bases: `[AgentsException](#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when the maximum number of turns is exceeded.

Source code in `src/agents/exceptions.py`



### ModelBehaviorError

Bases: `[AgentsException](#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when the model does something unexpected, e.g. calling a tool that doesn't exist, or providing malformed JSON.

Source code in `src/agents/exceptions.py`



### UserError

Bases: `[AgentsException](#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when the user makes an error using the SDK.

Source code in `src/agents/exceptions.py`



### InputGuardrailTripwireTriggered

Bases: `[AgentsException](#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when a guardrail tripwire is triggered.

Source code in `src/agents/exceptions.py`



#### guardrail\_result `instance-attribute`

```
guardrail_result: InputGuardrailResult = guardrail_result

```


The result data of the guardrail that was triggered.

### OutputGuardrailTripwireTriggered

Bases: `[AgentsException](#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when a guardrail tripwire is triggered.

Source code in `src/agents/exceptions.py`



#### guardrail\_result `instance-attribute`

```
guardrail_result: OutputGuardrailResult = guardrail_result

```


The result data of the guardrail that was triggered.



Table of contents

*   [guardrail](#agents.guardrail)
*   [GuardrailFunctionOutput](#agents.guardrail.GuardrailFunctionOutput)
    
    *   [output\_info](#agents.guardrail.GuardrailFunctionOutput.output_info)
    *   [tripwire\_triggered](#agents.guardrail.GuardrailFunctionOutput.tripwire_triggered)
    
*   [InputGuardrailResult](#agents.guardrail.InputGuardrailResult)
    
    *   [guardrail](#agents.guardrail.InputGuardrailResult.guardrail)
    *   [output](#agents.guardrail.InputGuardrailResult.output)
    
*   [OutputGuardrailResult](#agents.guardrail.OutputGuardrailResult)
    
    *   [guardrail](#agents.guardrail.OutputGuardrailResult.guardrail)
    *   [agent\_output](#agents.guardrail.OutputGuardrailResult.agent_output)
    *   [agent](#agents.guardrail.OutputGuardrailResult.agent)
    *   [output](#agents.guardrail.OutputGuardrailResult.output)
    
*   [InputGuardrail](#agents.guardrail.InputGuardrail)
    
    *   [guardrail\_function](#agents.guardrail.InputGuardrail.guardrail_function)
    *   [name](#agents.guardrail.InputGuardrail.name)
    
*   [OutputGuardrail](#agents.guardrail.OutputGuardrail)
    
    *   [guardrail\_function](#agents.guardrail.OutputGuardrail.guardrail_function)
    *   [name](#agents.guardrail.OutputGuardrail.name)
    
*   [input\_guardrail](#agents.guardrail.input_guardrail)
*   [output\_guardrail](#agents.guardrail.output_guardrail)

`Guardrails`
============

### GuardrailFunctionOutput `dataclass`

The output of a guardrail function.

Source code in `src/agents/guardrail.py`



#### output\_info `instance-attribute`

```
output_info: Any

```


Optional information about the guardrail's output. For example, the guardrail could include information about the checks it performed and granular results.

#### tripwire\_triggered `instance-attribute`

```
tripwire_triggered: bool

```


Whether the tripwire was triggered. If triggered, the agent's execution will be halted.

### InputGuardrailResult `dataclass`

The result of a guardrail run.

Source code in `src/agents/guardrail.py`



#### guardrail `instance-attribute`

```
guardrail: InputGuardrail[Any]

```


The guardrail that was run.

#### output `instance-attribute`

```
output: GuardrailFunctionOutput

```


The output of the guardrail function.

### OutputGuardrailResult `dataclass`

The result of a guardrail run.

Source code in `src/agents/guardrail.py`



#### guardrail `instance-attribute`

```
guardrail: OutputGuardrail[Any]

```


The guardrail that was run.

#### agent\_output `instance-attribute`

```
agent_output: Any

```


The output of the agent that was checked by the guardrail.

#### agent `instance-attribute`

```
agent: Agent[Any]

```


The agent that was checked by the guardrail.

#### output `instance-attribute`

```
output: GuardrailFunctionOutput

```


The output of the guardrail function.

### InputGuardrail `dataclass`

Bases: `Generic[TContext]`

Input guardrails are checks that run in parallel to the agent's execution. They can be used to do things like: - Check if input messages are off-topic - Take over control of the agent's execution if an unexpected input is detected

You can use the `@input_guardrail()` decorator to turn a function into an `InputGuardrail`, or create an `InputGuardrail` manually.

Guardrails return a `GuardrailResult`. If `result.tripwire_triggered` is `True`, the agent execution will immediately stop and a `InputGuardrailTripwireTriggered` exception will be raised

Source code in `src/agents/guardrail.py`



#### guardrail\_function `instance-attribute`

```
guardrail_function: Callable[
    [
        RunContextWrapper[TContext],
        Agent[Any],
        str | list[TResponseInputItem],
    ],
    MaybeAwaitable[GuardrailFunctionOutput],
]

```


A function that receives the agent input and the context, and returns a `GuardrailResult`. The result marks whether the tripwire was triggered, and can optionally include information about the guardrail's output.

#### name `class-attribute` `instance-attribute`

```
name: str | None = None

```


The name of the guardrail, used for tracing. If not provided, we'll use the guardrail function's name.

### OutputGuardrail `dataclass`

Bases: `Generic[TContext]`

Output guardrails are checks that run on the final output of an agent. They can be used to do check if the output passes certain validation criteria

You can use the `@output_guardrail()` decorator to turn a function into an `OutputGuardrail`, or create an `OutputGuardrail` manually.

Guardrails return a `GuardrailResult`. If `result.tripwire_triggered` is `True`, a `OutputGuardrailTripwireTriggered` exception will be raised.

Source code in `src/agents/guardrail.py`



#### guardrail\_function `instance-attribute`

```
guardrail_function: Callable[
    [RunContextWrapper[TContext], Agent[Any], Any],
    MaybeAwaitable[GuardrailFunctionOutput],
]

```


A function that receives the final agent, its output, and the context, and returns a `GuardrailResult`. The result marks whether the tripwire was triggered, and can optionally include information about the guardrail's output.

#### name `class-attribute` `instance-attribute`

```
name: str | None = None

```


The name of the guardrail, used for tracing. If not provided, we'll use the guardrail function's name.

### input\_guardrail

```
input_guardrail(
    func: _InputGuardrailFuncSync[TContext_co],
) -> InputGuardrail[TContext_co]

```


```
input_guardrail(
    func: _InputGuardrailFuncAsync[TContext_co],
) -> InputGuardrail[TContext_co]

```


```
input_guardrail(
    *, name: str | None = None
) -> Callable[
    [
        _InputGuardrailFuncSync[TContext_co]
        | _InputGuardrailFuncAsync[TContext_co]
    ],
    InputGuardrail[TContext_co],
]

```


```
input_guardrail(
    func: _InputGuardrailFuncSync[TContext_co]
    | _InputGuardrailFuncAsync[TContext_co]
    | None = None,
    *,
    name: str | None = None,
) -> (
    InputGuardrail[TContext_co]
    | Callable[
        [
            _InputGuardrailFuncSync[TContext_co]
            | _InputGuardrailFuncAsync[TContext_co]
        ],
        InputGuardrail[TContext_co],
    ]
)

```


Decorator that transforms a sync or async function into an `InputGuardrail`. It can be used directly (no parentheses) or with keyword args, e.g.:

```
@input_guardrail
def my_sync_guardrail(...): ...

@input_guardrail(name="guardrail_name")
async def my_async_guardrail(...): ...

```


Source code in `src/agents/guardrail.py`



### output\_guardrail

```
output_guardrail(
    func: _OutputGuardrailFuncSync[TContext_co],
) -> OutputGuardrail[TContext_co]

```


```
output_guardrail(
    func: _OutputGuardrailFuncAsync[TContext_co],
) -> OutputGuardrail[TContext_co]

```


```
output_guardrail(
    *, name: str | None = None
) -> Callable[
    [
        _OutputGuardrailFuncSync[TContext_co]
        | _OutputGuardrailFuncAsync[TContext_co]
    ],
    OutputGuardrail[TContext_co],
]

```


```
output_guardrail(
    func: _OutputGuardrailFuncSync[TContext_co]
    | _OutputGuardrailFuncAsync[TContext_co]
    | None = None,
    *,
    name: str | None = None,
) -> (
    OutputGuardrail[TContext_co]
    | Callable[
        [
            _OutputGuardrailFuncSync[TContext_co]
            | _OutputGuardrailFuncAsync[TContext_co]
        ],
        OutputGuardrail[TContext_co],
    ]
)

```


Decorator that transforms a sync or async function into an `OutputGuardrail`. It can be used directly (no parentheses) or with keyword args, e.g.:

```
@output_guardrail
def my_sync_guardrail(...): ...

@output_guardrail(name="guardrail_name")
async def my_async_guardrail(...): ...

```


Source code in `src/agents/guardrail.py`


Table of contents

*   [model\_settings](#agents.model_settings)
*   [ModelSettings](#agents.model_settings.ModelSettings)
    
    *   [temperature](#agents.model_settings.ModelSettings.temperature)
    *   [top\_p](#agents.model_settings.ModelSettings.top_p)
    *   [frequency\_penalty](#agents.model_settings.ModelSettings.frequency_penalty)
    *   [presence\_penalty](#agents.model_settings.ModelSettings.presence_penalty)
    *   [tool\_choice](#agents.model_settings.ModelSettings.tool_choice)
    *   [parallel\_tool\_calls](#agents.model_settings.ModelSettings.parallel_tool_calls)
    *   [truncation](#agents.model_settings.ModelSettings.truncation)
    *   [max\_tokens](#agents.model_settings.ModelSettings.max_tokens)
    *   [store](#agents.model_settings.ModelSettings.store)
    *   [resolve](#agents.model_settings.ModelSettings.resolve)
    

`Model settings`
================

### ModelSettings `dataclass`

Settings to use when calling an LLM.

This class holds optional model configuration parameters (e.g. temperature, top\_p, penalties, truncation, etc.).

Not all models/providers support all of these parameters, so please check the API documentation for the specific model and provider you are using.

Source code in `src/agents/model_settings.py`



#### temperature `class-attribute` `instance-attribute`

```
temperature: float | None = None

```


The temperature to use when calling the model.

#### top\_p `class-attribute` `instance-attribute`

```
top_p: float | None = None

```


The top\_p to use when calling the model.

#### frequency\_penalty `class-attribute` `instance-attribute`

```
frequency_penalty: float | None = None

```


The frequency penalty to use when calling the model.

#### presence\_penalty `class-attribute` `instance-attribute`

```
presence_penalty: float | None = None

```


The presence penalty to use when calling the model.

#### tool\_choice `class-attribute` `instance-attribute`

```
tool_choice: (
    Literal["auto", "required", "none"] | str | None
) = None

```


The tool choice to use when calling the model.

#### parallel\_tool\_calls `class-attribute` `instance-attribute`

```
parallel_tool_calls: bool | None = None

```


Whether to use parallel tool calls when calling the model. Defaults to False if not provided.

#### truncation `class-attribute` `instance-attribute`

```
truncation: Literal['auto', 'disabled'] | None = None

```


The truncation strategy to use when calling the model.

#### max\_tokens `class-attribute` `instance-attribute`

```
max_tokens: int | None = None

```


The maximum number of output tokens to generate.

#### store `class-attribute` `instance-attribute`

```
store: bool | None = None

```


Whether to store the generated model response for later retrieval. Defaults to True if not provided.

#### resolve

```
resolve(override: ModelSettings | None) -> ModelSettings

```


Produce a new ModelSettings by overlaying any non-None values from the override on top of this instance.

Source code in `src/agents/model_settings.py`



    

Table of contents

*   [agent\_output](#agents.agent_output)
*   [AgentOutputSchema](#agents.agent_output.AgentOutputSchema)
    
    *   [output\_type](#agents.agent_output.AgentOutputSchema.output_type)
    *   [strict\_json\_schema](#agents.agent_output.AgentOutputSchema.strict_json_schema)
    *   [\_\_init\_\_](#agents.agent_output.AgentOutputSchema.__init__)
    *   [is\_plain\_text](#agents.agent_output.AgentOutputSchema.is_plain_text)
    *   [json\_schema](#agents.agent_output.AgentOutputSchema.json_schema)
    *   [validate\_json](#agents.agent_output.AgentOutputSchema.validate_json)
    *   [output\_type\_name](#agents.agent_output.AgentOutputSchema.output_type_name)
    

`Agent output`
==============

### AgentOutputSchema `dataclass`

An object that captures the JSON schema of the output, as well as validating/parsing JSON produced by the LLM into the output type.

Source code in `src/agents/agent_output.py`



#### output\_type `instance-attribute`

```
output_type: type[Any] = output_type

```


The type of the output.

#### strict\_json\_schema `instance-attribute`

```
strict_json_schema: bool = strict_json_schema

```


Whether the JSON schema is in strict mode. We **strongly** recommend setting this to True, as it increases the likelihood of correct JSON input.

#### \_\_init\_\_

```
__init__(
    output_type: type[Any], strict_json_schema: bool = True
)

```


Parameters:



* Name:                 output_type            
  * Type:                   type[Any]            
  * Description:                               The type of the output.                          
  * Default:                 required            
* Name:                 strict_json_schema            
  * Type:                   bool            
  * Description:                               Whether the JSON schema is in strict mode. We strongly recommendsetting this to True, as it increases the likelihood of correct JSON input.                          
  * Default:                   True            


Source code in `src/agents/agent_output.py`



#### is\_plain\_text

```
is_plain_text() -> bool

```


Whether the output type is plain text (versus a JSON object).

Source code in `src/agents/agent_output.py`



#### json\_schema

```
json_schema() -> dict[str, Any]

```


The JSON schema of the output type.

Source code in `src/agents/agent_output.py`



#### validate\_json

```
validate_json(json_str: str, partial: bool = False) -> Any

```


Validate a JSON string against the output type. Returns the validated object, or raises a `ModelBehaviorError` if the JSON is invalid.

Source code in `src/agents/agent_output.py`



#### output\_type\_name

```
output_type_name() -> str

```


The name of the output type.

Source code in `src/agents/agent_output.py`


Table of contents

*   [function\_schema](#agents.function_schema)
*   [FuncSchema](#agents.function_schema.FuncSchema)
    
    *   [name](#agents.function_schema.FuncSchema.name)
    *   [description](#agents.function_schema.FuncSchema.description)
    *   [params\_pydantic\_model](#agents.function_schema.FuncSchema.params_pydantic_model)
    *   [params\_json\_schema](#agents.function_schema.FuncSchema.params_json_schema)
    *   [signature](#agents.function_schema.FuncSchema.signature)
    *   [takes\_context](#agents.function_schema.FuncSchema.takes_context)
    *   [strict\_json\_schema](#agents.function_schema.FuncSchema.strict_json_schema)
    *   [to\_call\_args](#agents.function_schema.FuncSchema.to_call_args)
    
*   [FuncDocumentation](#agents.function_schema.FuncDocumentation)
    
    *   [name](#agents.function_schema.FuncDocumentation.name)
    *   [description](#agents.function_schema.FuncDocumentation.description)
    *   [param\_descriptions](#agents.function_schema.FuncDocumentation.param_descriptions)
    
*   [generate\_func\_documentation](#agents.function_schema.generate_func_documentation)
*   [function\_schema](#agents.function_schema.function_schema)

`Function schema`
=================

### FuncSchema `dataclass`

Captures the schema for a python function, in preparation for sending it to an LLM as a tool.

Source code in `src/agents/function_schema.py`



#### name `instance-attribute`

```
name: str

```


The name of the function.

#### description `instance-attribute`

```
description: str | None

```


The description of the function.

#### params\_pydantic\_model `instance-attribute`

```
params_pydantic_model: type[BaseModel]

```


A Pydantic model that represents the function's parameters.

#### params\_json\_schema `instance-attribute`

```
params_json_schema: dict[str, Any]

```


The JSON schema for the function's parameters, derived from the Pydantic model.

#### signature `instance-attribute`

```
signature: Signature

```


The signature of the function.

#### takes\_context `class-attribute` `instance-attribute`

```
takes_context: bool = False

```


Whether the function takes a RunContextWrapper argument (must be the first argument).

#### strict\_json\_schema `class-attribute` `instance-attribute`

```
strict_json_schema: bool = True

```


Whether the JSON schema is in strict mode. We **strongly** recommend setting this to True, as it increases the likelihood of correct JSON input.

#### to\_call\_args

```
to_call_args(
    data: BaseModel,
) -> tuple[list[Any], dict[str, Any]]

```


Converts validated data from the Pydantic model into (args, kwargs), suitable for calling the original function.

Source code in `src/agents/function_schema.py`



### FuncDocumentation `dataclass`

Contains metadata about a python function, extracted from its docstring.

Source code in `src/agents/function_schema.py`



#### name `instance-attribute`

```
name: str

```


The name of the function, via `__name__`.

#### description `instance-attribute`

```
description: str | None

```


The description of the function, derived from the docstring.

#### param\_descriptions `instance-attribute`

```
param_descriptions: dict[str, str] | None

```


The parameter descriptions of the function, derived from the docstring.

### generate\_func\_documentation

```
generate_func_documentation(
    func: Callable[..., Any],
    style: DocstringStyle | None = None,
) -> FuncDocumentation

```


Extracts metadata from a function docstring, in preparation for sending it to an LLM as a tool.

Parameters:



* Name:                 func            
  * Type:                   Callable[..., Any]            
  * Description:                               The function to extract documentation from.                          
  * Default:                 required            
* Name:                 style            
  * Type:                   DocstringStyle | None            
  * Description:                               The style of the docstring to use for parsing. If not provided, we will attempt toauto-detect the style.                          
  * Default:                   None            


Returns:



* Type:                   FuncDocumentation            
  * Description:                               A FuncDocumentation object containing the function's name, description, and parameter                          
* Type:                   FuncDocumentation            
  * Description:                               descriptions.                          


Source code in `src/agents/function_schema.py`



### function\_schema

```
function_schema(
    func: Callable[..., Any],
    docstring_style: DocstringStyle | None = None,
    name_override: str | None = None,
    description_override: str | None = None,
    use_docstring_info: bool = True,
    strict_json_schema: bool = True,
) -> FuncSchema

```


Given a python function, extracts a `FuncSchema` from it, capturing the name, description, parameter descriptions, and other metadata.

Parameters:



* Name:                 func            
  * Type:                   Callable[..., Any]            
  * Description:                               The function to extract the schema from.                          
  * Default:                 required            
* Name:                 docstring_style            
  * Type:                   DocstringStyle | None            
  * Description:                               The style of the docstring to use for parsing. If not provided, we willattempt to auto-detect the style.                          
  * Default:                   None            
* Name:                 name_override            
  * Type:                   str | None            
  * Description:                               If provided, use this name instead of the function's __name__.                          
  * Default:                   None            
* Name:                 description_override            
  * Type:                   str | None            
  * Description:                               If provided, use this description instead of the one derived from thedocstring.                          
  * Default:                   None            
* Name:                 use_docstring_info            
  * Type:                   bool            
  * Description:                               If True, uses the docstring to generate the description and parameterdescriptions.                          
  * Default:                   True            
* Name:                 strict_json_schema            
  * Type:                   bool            
  * Description:                               Whether the JSON schema is in strict mode. If True, we'll ensure thatthe schema adheres to the "strict" standard the OpenAI API expects. We stronglyrecommend setting this to True, as it increases the likelihood of the LLM providingcorrect JSON input.                          
  * Default:                   True            


Returns:



* Type:                   FuncSchema            
  * Description:                               A FuncSchema object containing the function's name, description, parameter descriptions,                          
* Type:                   FuncSchema            
  * Description:                               and other metadata.                          


Source code in `src/agents/function_schema.py`


Table of contents

*   [interface](#agents.models.interface)
*   [ModelTracing](#agents.models.interface.ModelTracing)
    
    *   [DISABLED](#agents.models.interface.ModelTracing.DISABLED)
    *   [ENABLED](#agents.models.interface.ModelTracing.ENABLED)
    *   [ENABLED\_WITHOUT\_DATA](#agents.models.interface.ModelTracing.ENABLED_WITHOUT_DATA)
    
*   [Model](#agents.models.interface.Model)
    
    *   [get\_response](#agents.models.interface.Model.get_response)
    *   [stream\_response](#agents.models.interface.Model.stream_response)
    
*   [ModelProvider](#agents.models.interface.ModelProvider)
    
    *   [get\_model](#agents.models.interface.ModelProvider.get_model)
    

`Model interface`
=================

### ModelTracing

Bases: `Enum`

Source code in `src/agents/models/interface.py`



#### DISABLED `class-attribute` `instance-attribute`

```
DISABLED = 0

```


Tracing is disabled entirely.

#### ENABLED `class-attribute` `instance-attribute`

```
ENABLED = 1

```


Tracing is enabled, and all data is included.

#### ENABLED\_WITHOUT\_DATA `class-attribute` `instance-attribute`

```
ENABLED_WITHOUT_DATA = 2

```


Tracing is enabled, but inputs/outputs are not included.

### Model

Bases: `ABC`

The base interface for calling an LLM.

Source code in `src/agents/models/interface.py`



#### get\_response `abstractmethod` `async`

```
get_response(
    system_instructions: str | None,
    input: str | list[TResponseInputItem],
    model_settings: ModelSettings,
    tools: list[Tool],
    output_schema: AgentOutputSchema | None,
    handoffs: list[Handoff],
    tracing: ModelTracing,
) -> ModelResponse

```


Get a response from the model.

Parameters:



* Name:                 system_instructions            
  * Type:                   str | None            
  * Description:                               The system instructions to use.                          
  * Default:                 required            
* Name:                 input            
  * Type:                   str | list[TResponseInputItem]            
  * Description:                               The input items to the model, in OpenAI Responses format.                          
  * Default:                 required            
* Name:                 model_settings            
  * Type:                   ModelSettings            
  * Description:                               The model settings to use.                          
  * Default:                 required            
* Name:                 tools            
  * Type:                   list[Tool]            
  * Description:                               The tools available to the model.                          
  * Default:                 required            
* Name:                 output_schema            
  * Type:                   AgentOutputSchema | None            
  * Description:                               The output schema to use.                          
  * Default:                 required            
* Name:                 handoffs            
  * Type:                   list[Handoff]            
  * Description:                               The handoffs available to the model.                          
  * Default:                 required            
* Name:                 tracing            
  * Type:                   ModelTracing            
  * Description:                               Tracing configuration.                          
  * Default:                 required            


Returns:



* Type:                   ModelResponse            
  * Description:                               The full model response.                          


Source code in `src/agents/models/interface.py`



#### stream\_response `abstractmethod`

```
stream_response(
    system_instructions: str | None,
    input: str | list[TResponseInputItem],
    model_settings: ModelSettings,
    tools: list[Tool],
    output_schema: AgentOutputSchema | None,
    handoffs: list[Handoff],
    tracing: ModelTracing,
) -> AsyncIterator[TResponseStreamEvent]

```


Stream a response from the model.

Parameters:



* Name:                 system_instructions            
  * Type:                   str | None            
  * Description:                               The system instructions to use.                          
  * Default:                 required            
* Name:                 input            
  * Type:                   str | list[TResponseInputItem]            
  * Description:                               The input items to the model, in OpenAI Responses format.                          
  * Default:                 required            
* Name:                 model_settings            
  * Type:                   ModelSettings            
  * Description:                               The model settings to use.                          
  * Default:                 required            
* Name:                 tools            
  * Type:                   list[Tool]            
  * Description:                               The tools available to the model.                          
  * Default:                 required            
* Name:                 output_schema            
  * Type:                   AgentOutputSchema | None            
  * Description:                               The output schema to use.                          
  * Default:                 required            
* Name:                 handoffs            
  * Type:                   list[Handoff]            
  * Description:                               The handoffs available to the model.                          
  * Default:                 required            
* Name:                 tracing            
  * Type:                   ModelTracing            
  * Description:                               Tracing configuration.                          
  * Default:                 required            


Returns:



* Type:                   AsyncIterator[TResponseStreamEvent]            
  * Description:                               An iterator of response stream events, in OpenAI Responses format.                          


Source code in `src/agents/models/interface.py`



### ModelProvider

Bases: `ABC`

The base interface for a model provider.

Model provider is responsible for looking up Models by name.

Source code in `src/agents/models/interface.py`



#### get\_model `abstractmethod`

```
get_model(model_name: str | None) -> Model

```


Get a model by name.

Parameters:



* Name:                 model_name            
  * Type:                   str | None            
  * Description:                               The name of the model to get.                          
  * Default:                 required            


Returns:



* Type:                   Model            
  * Description:                               The model.                          


Source code in `src/agents/models/interface.py`

   

Table of contents

*   [openai\_chatcompletions](#agents.models.openai_chatcompletions)
*   [OpenAIChatCompletionsModel](#agents.models.openai_chatcompletions.OpenAIChatCompletionsModel)
    
    *   [stream\_response](#agents.models.openai_chatcompletions.OpenAIChatCompletionsModel.stream_response)
    

`OpenAI Chat Completions model`
===============================

### OpenAIChatCompletionsModel

Bases: `[Model](../interface/#agents.models.interface.Model "Model (agents.models.interface.Model)")`

Source code in `src/agents/models/openai_chatcompletions.py`



#### stream\_response `async`

```
stream_response(
    system_instructions: str | None,
    input: str | list[TResponseInputItem],
    model_settings: ModelSettings,
    tools: list[Tool],
    output_schema: AgentOutputSchema | None,
    handoffs: list[Handoff],
    tracing: ModelTracing,
) -> AsyncIterator[TResponseStreamEvent]

```


Yields a partial message as it is generated, as well as the usage information.

Source code in `src/agents/models/openai_chatcompletions.py`


Table of contents

*   [openai\_responses](#agents.models.openai_responses)
*   [OpenAIResponsesModel](#agents.models.openai_responses.OpenAIResponsesModel)
    
    *   [stream\_response](#agents.models.openai_responses.OpenAIResponsesModel.stream_response)
    
*   [Converter](#agents.models.openai_responses.Converter)

`OpenAI Responses model`
========================

### OpenAIResponsesModel

Bases: `[Model](../interface/#agents.models.interface.Model "Model (agents.models.interface.Model)")`

Implementation of `Model` that uses the OpenAI Responses API.

Source code in `src/agents/models/openai_responses.py`



#### stream\_response `async`

```
stream_response(
    system_instructions: str | None,
    input: str | list[TResponseInputItem],
    model_settings: ModelSettings,
    tools: list[Tool],
    output_schema: AgentOutputSchema | None,
    handoffs: list[Handoff],
    tracing: ModelTracing,
) -> AsyncIterator[ResponseStreamEvent]

```


Yields a partial message as it is generated, as well as the usage information.

Source code in `src/agents/models/openai_responses.py`



### Converter

Source code in `src/agents/models/openai_responses.py`


Table of contents

*   [server](#agents.mcp.server)
*   [MCPServer](#agents.mcp.server.MCPServer)
    
    *   [name](#agents.mcp.server.MCPServer.name)
    *   [connect](#agents.mcp.server.MCPServer.connect)
    *   [cleanup](#agents.mcp.server.MCPServer.cleanup)
    *   [list\_tools](#agents.mcp.server.MCPServer.list_tools)
    *   [call\_tool](#agents.mcp.server.MCPServer.call_tool)
    
*   [MCPServerStdioParams](#agents.mcp.server.MCPServerStdioParams)
    
    *   [command](#agents.mcp.server.MCPServerStdioParams.command)
    *   [args](#agents.mcp.server.MCPServerStdioParams.args)
    *   [env](#agents.mcp.server.MCPServerStdioParams.env)
    *   [cwd](#agents.mcp.server.MCPServerStdioParams.cwd)
    *   [encoding](#agents.mcp.server.MCPServerStdioParams.encoding)
    *   [encoding\_error\_handler](#agents.mcp.server.MCPServerStdioParams.encoding_error_handler)
    
*   [MCPServerStdio](#agents.mcp.server.MCPServerStdio)
    
    *   [name](#agents.mcp.server.MCPServerStdio.name)
    *   [connect](#agents.mcp.server.MCPServerStdio.connect)
    *   [cleanup](#agents.mcp.server.MCPServerStdio.cleanup)
    *   [list\_tools](#agents.mcp.server.MCPServerStdio.list_tools)
    *   [call\_tool](#agents.mcp.server.MCPServerStdio.call_tool)
    *   [invalidate\_tools\_cache](#agents.mcp.server.MCPServerStdio.invalidate_tools_cache)
    *   [\_\_init\_\_](#agents.mcp.server.MCPServerStdio.__init__)
    *   [create\_streams](#agents.mcp.server.MCPServerStdio.create_streams)
    
*   [MCPServerSseParams](#agents.mcp.server.MCPServerSseParams)
    
    *   [url](#agents.mcp.server.MCPServerSseParams.url)
    *   [headers](#agents.mcp.server.MCPServerSseParams.headers)
    *   [timeout](#agents.mcp.server.MCPServerSseParams.timeout)
    *   [sse\_read\_timeout](#agents.mcp.server.MCPServerSseParams.sse_read_timeout)
    
*   [MCPServerSse](#agents.mcp.server.MCPServerSse)
    
    *   [name](#agents.mcp.server.MCPServerSse.name)
    *   [connect](#agents.mcp.server.MCPServerSse.connect)
    *   [cleanup](#agents.mcp.server.MCPServerSse.cleanup)
    *   [list\_tools](#agents.mcp.server.MCPServerSse.list_tools)
    *   [call\_tool](#agents.mcp.server.MCPServerSse.call_tool)
    *   [invalidate\_tools\_cache](#agents.mcp.server.MCPServerSse.invalidate_tools_cache)
    *   [\_\_init\_\_](#agents.mcp.server.MCPServerSse.__init__)
    *   [create\_streams](#agents.mcp.server.MCPServerSse.create_streams)
    

`MCP Servers`
=============

### MCPServer

Bases: `ABC`

Base class for Model Context Protocol servers.

Source code in `src/agents/mcp/server.py`



#### name `abstractmethod` `property`

```
name: str

```


A readable name for the server.

#### connect `abstractmethod` `async`

```
connect()

```


Connect to the server. For example, this might mean spawning a subprocess or opening a network connection. The server is expected to remain connected until `cleanup()` is called.

Source code in `src/agents/mcp/server.py`



#### cleanup `abstractmethod` `async`

```
cleanup()

```


Cleanup the server. For example, this might mean closing a subprocess or closing a network connection.

Source code in `src/agents/mcp/server.py`



#### list\_tools `abstractmethod` `async`

```
list_tools() -> list[Tool]

```


List the tools available on the server.

Source code in `src/agents/mcp/server.py`



#### call\_tool `abstractmethod` `async`

```
call_tool(
    tool_name: str, arguments: dict[str, Any] | None
) -> CallToolResult

```


Invoke a tool on the server.

Source code in `src/agents/mcp/server.py`



### MCPServerStdioParams

Bases: `TypedDict`

Mirrors `mcp.client.stdio.StdioServerParameters`, but lets you pass params without another import.

Source code in `src/agents/mcp/server.py`



#### command `instance-attribute`

```
command: str

```


The executable to run to start the server. For example, `python` or `node`.

#### args `instance-attribute`

```
args: NotRequired[list[str]]

```


Command line args to pass to the `command` executable. For example, `['foo.py']` or `['server.js', '--port', '8080']`.

#### env `instance-attribute`

```
env: NotRequired[dict[str, str]]

```


The environment variables to set for the server. .

#### cwd `instance-attribute`

```
cwd: NotRequired[str | Path]

```


The working directory to use when spawning the process.

#### encoding `instance-attribute`

```
encoding: NotRequired[str]

```


The text encoding used when sending/receiving messages to the server. Defaults to `utf-8`.

#### encoding\_error\_handler `instance-attribute`

```
encoding_error_handler: NotRequired[
    Literal["strict", "ignore", "replace"]
]

```


The text encoding error handler. Defaults to `strict`.

See https://docs.python.org/3/library/codecs.html#codec-base-classes for explanations of possible values.

### MCPServerStdio

Bases: `_MCPServerWithClientSession`

MCP server implementation that uses the stdio transport. See the \[spec\] (https://spec.modelcontextprotocol.io/specification/2024-11-05/basic/transports/#stdio) for details.

Source code in `src/agents/mcp/server.py`



#### name `property`

```
name: str

```


A readable name for the server.

#### connect `async`

```
connect()

```


Connect to the server.

Source code in `src/agents/mcp/server.py`



#### cleanup `async`

```
cleanup()

```


Cleanup the server.

Source code in `src/agents/mcp/server.py`



#### list\_tools `async`

```
list_tools() -> list[Tool]

```


List the tools available on the server.

Source code in `src/agents/mcp/server.py`



#### call\_tool `async`

```
call_tool(
    tool_name: str, arguments: dict[str, Any] | None
) -> CallToolResult

```


Invoke a tool on the server.

Source code in `src/agents/mcp/server.py`



#### invalidate\_tools\_cache

```
invalidate_tools_cache()

```


Invalidate the tools cache.

Source code in `src/agents/mcp/server.py`



#### \_\_init\_\_

```
__init__(
    params: MCPServerStdioParams,
    cache_tools_list: bool = False,
    name: str | None = None,
)

```


Create a new MCP server based on the stdio transport.

Parameters:



* Name:                 params            
  * Type:                   MCPServerStdioParams            
  * Description:                               The params that configure the server. This includes the command to run tostart the server, the args to pass to the command, the environment variables toset for the server, the working directory to use when spawning the process, andthe text encoding used when sending/receiving messages to the server.                          
  * Default:                 required            
* Name:                 cache_tools_list            
  * Type:                   bool            
  * Description:                               Whether to cache the tools list. If True, the tools list will becached and only fetched from the server once. If False, the tools list will befetched from the server on each call to list_tools(). The cache can beinvalidated by calling invalidate_tools_cache(). You should set this to Trueif you know the server will not change its tools list, because it can drasticallyimprove latency (by avoiding a round-trip to the server every time).                          
  * Default:                   False            
* Name:                 name            
  * Type:                   str | None            
  * Description:                               A readable name for the server. If not provided, we'll create one from thecommand.                          
  * Default:                   None            


Source code in `src/agents/mcp/server.py`



#### create\_streams

```
create_streams() -> AbstractAsyncContextManager[
    tuple[
        MemoryObjectReceiveStream[
            JSONRPCMessage | Exception
        ],
        MemoryObjectSendStream[JSONRPCMessage],
    ]
]

```


Create the streams for the server.

Source code in `src/agents/mcp/server.py`



### MCPServerSseParams

Bases: `TypedDict`

Mirrors the params in`mcp.client.sse.sse_client`.

Source code in `src/agents/mcp/server.py`



#### url `instance-attribute`

```
url: str

```


The URL of the server.

#### headers `instance-attribute`

```
headers: NotRequired[dict[str, str]]

```


The headers to send to the server.

#### timeout `instance-attribute`

```
timeout: NotRequired[float]

```


The timeout for the HTTP request. Defaults to 5 seconds.

#### sse\_read\_timeout `instance-attribute`

```
sse_read_timeout: NotRequired[float]

```


The timeout for the SSE connection, in seconds. Defaults to 5 minutes.

### MCPServerSse

Bases: `_MCPServerWithClientSession`

MCP server implementation that uses the HTTP with SSE transport. See the \[spec\] (https://spec.modelcontextprotocol.io/specification/2024-11-05/basic/transports/#http-with-sse) for details.

Source code in `src/agents/mcp/server.py`



#### name `property`

```
name: str

```


A readable name for the server.

#### connect `async`

```
connect()

```


Connect to the server.

Source code in `src/agents/mcp/server.py`



#### cleanup `async`

```
cleanup()

```


Cleanup the server.

Source code in `src/agents/mcp/server.py`



#### list\_tools `async`

```
list_tools() -> list[Tool]

```


List the tools available on the server.

Source code in `src/agents/mcp/server.py`



#### call\_tool `async`

```
call_tool(
    tool_name: str, arguments: dict[str, Any] | None
) -> CallToolResult

```


Invoke a tool on the server.

Source code in `src/agents/mcp/server.py`



#### invalidate\_tools\_cache

```
invalidate_tools_cache()

```


Invalidate the tools cache.

Source code in `src/agents/mcp/server.py`



#### \_\_init\_\_

```
__init__(
    params: MCPServerSseParams,
    cache_tools_list: bool = False,
    name: str | None = None,
)

```


Create a new MCP server based on the HTTP with SSE transport.

Parameters:



* Name:                 params            
  * Type:                   MCPServerSseParams            
  * Description:                               The params that configure the server. This includes the URL of the server,the headers to send to the server, the timeout for the HTTP request, and thetimeout for the SSE connection.                          
  * Default:                 required            
* Name:                 cache_tools_list            
  * Type:                   bool            
  * Description:                               Whether to cache the tools list. If True, the tools list will becached and only fetched from the server once. If False, the tools list will befetched from the server on each call to list_tools(). The cache can beinvalidated by calling invalidate_tools_cache(). You should set this to Trueif you know the server will not change its tools list, because it can drasticallyimprove latency (by avoiding a round-trip to the server every time).                          
  * Default:                   False            
* Name:                 name            
  * Type:                   str | None            
  * Description:                               A readable name for the server. If not provided, we'll create one from theURL.                          
  * Default:                   None            


Source code in `src/agents/mcp/server.py`



#### create\_streams

```
create_streams() -> AbstractAsyncContextManager[
    tuple[
        MemoryObjectReceiveStream[
            JSONRPCMessage | Exception
        ],
        MemoryObjectSendStream[JSONRPCMessage],
    ]
]

```


Create the streams for the server.

Source code in `src/agents/mcp/server.py`



Table of contents

*   [util](#agents.mcp.util)
*   [MCPUtil](#agents.mcp.util.MCPUtil)
    
    *   [get\_all\_function\_tools](#agents.mcp.util.MCPUtil.get_all_function_tools)
    *   [get\_function\_tools](#agents.mcp.util.MCPUtil.get_function_tools)
    *   [to\_function\_tool](#agents.mcp.util.MCPUtil.to_function_tool)
    *   [invoke\_mcp\_tool](#agents.mcp.util.MCPUtil.invoke_mcp_tool)
    

`MCP Util`
==========

### MCPUtil

Set of utilities for interop between MCP and Agents SDK tools.

Source code in `src/agents/mcp/util.py`



#### get\_all\_function\_tools `async` `classmethod`

```
get_all_function_tools(
    servers: list[MCPServer],
) -> list[Tool]

```


Get all function tools from a list of MCP servers.

Source code in `src/agents/mcp/util.py`



#### get\_function\_tools `async` `classmethod`

```
get_function_tools(server: MCPServer) -> list[Tool]

```


Get all function tools from a single MCP server.

Source code in `src/agents/mcp/util.py`



#### to\_function\_tool `classmethod`

```
to_function_tool(
    tool: Tool, server: MCPServer
) -> FunctionTool

```


Convert an MCP tool to an Agents SDK function tool.

Source code in `src/agents/mcp/util.py`



#### invoke\_mcp\_tool `async` `classmethod`

```
invoke_mcp_tool(
    server: MCPServer,
    tool: Tool,
    context: RunContextWrapper[Any],
    input_json: str,
) -> str

```


Invoke an MCP tool and return the result as a string.

Source code in `src/agents/mcp/util.py`

## OpenAI Agents Voice SDK



Table of contents

*   [pipeline](#agents.voice.pipeline)
*   [VoicePipeline](#agents.voice.pipeline.VoicePipeline)
    
    *   [\_\_init\_\_](#agents.voice.pipeline.VoicePipeline.__init__)
    *   [run](#agents.voice.pipeline.VoicePipeline.run)
    

`Pipeline`
==========

### VoicePipeline

An opinionated voice agent pipeline. It works in three steps: 1. Transcribe audio input into text. 2. Run the provided `workflow`, which produces a sequence of text responses. 3. Convert the text responses into streaming audio output.

Source code in `src/agents/voice/pipeline.py`



#### \_\_init\_\_

```
__init__(
    *,
    workflow: VoiceWorkflowBase,
    stt_model: STTModel | str | None = None,
    tts_model: TTSModel | str | None = None,
    config: VoicePipelineConfig | None = None,
)

```


Create a new voice pipeline.

Parameters:



* Name:                 workflow            
  * Type:                   VoiceWorkflowBase            
  * Description:                               The workflow to run. See VoiceWorkflowBase.                          
  * Default:                 required            
* Name:                 stt_model            
  * Type:                   STTModel | str | None            
  * Description:                               The speech-to-text model to use. If not provided, a default OpenAImodel will be used.                          
  * Default:                   None            
* Name:                 tts_model            
  * Type:                   TTSModel | str | None            
  * Description:                               The text-to-speech model to use. If not provided, a default OpenAImodel will be used.                          
  * Default:                   None            
* Name:                 config            
  * Type:                   VoicePipelineConfig | None            
  * Description:                               The pipeline configuration. If not provided, a default configuration will beused.                          
  * Default:                   None            


Source code in `src/agents/voice/pipeline.py`



#### run `async`

```
run(
    audio_input: AudioInput | StreamedAudioInput,
) -> StreamedAudioResult

```


Run the voice pipeline.

Parameters:



* Name:                 audio_input            
  * Type:                   AudioInput | StreamedAudioInput            
  * Description:                               The audio input to process. This can either be an AudioInput instance,which is a single static buffer, or a StreamedAudioInput instance, which is astream of audio data that you can append to.                          
  * Default:                 required            


Returns:



* Type:                   StreamedAudioResult            
  * Description:                               A StreamedAudioResult instance. You can use this object to stream audio events and                          
* Type:                   StreamedAudioResult            
  * Description:                               play them out.                          


Source code in `src/agents/voice/pipeline.py`




Table of contents

*   [workflow](#agents.voice.workflow)
*   [VoiceWorkflowBase](#agents.voice.workflow.VoiceWorkflowBase)
    
    *   [run](#agents.voice.workflow.VoiceWorkflowBase.run)
    
*   [VoiceWorkflowHelper](#agents.voice.workflow.VoiceWorkflowHelper)
    
    *   [stream\_text\_from](#agents.voice.workflow.VoiceWorkflowHelper.stream_text_from)
    
*   [SingleAgentWorkflowCallbacks](#agents.voice.workflow.SingleAgentWorkflowCallbacks)
    
    *   [on\_run](#agents.voice.workflow.SingleAgentWorkflowCallbacks.on_run)
    
*   [SingleAgentVoiceWorkflow](#agents.voice.workflow.SingleAgentVoiceWorkflow)
    
    *   [\_\_init\_\_](#agents.voice.workflow.SingleAgentVoiceWorkflow.__init__)
    

`Workflow`
==========

### VoiceWorkflowBase

Bases: `ABC`

A base class for a voice workflow. You must implement the `run` method. A "workflow" is any code you want, that receives a transcription and yields text that will be turned into speech by a text-to-speech model. In most cases, you'll create `Agent`s and use `Runner.run_streamed()` to run them, returning some or all of the text events from the stream. You can use the `VoiceWorkflowHelper` class to help with extracting text events from the stream. If you have a simple workflow that has a single starting agent and no custom logic, you can use `SingleAgentVoiceWorkflow` directly.

Source code in `src/agents/voice/workflow.py`



#### run `abstractmethod`

```
run(transcription: str) -> AsyncIterator[str]

```


Run the voice workflow. You will receive an input transcription, and must yield text that will be spoken to the user. You can run whatever logic you want here. In most cases, the final logic will involve calling `Runner.run_streamed()` and yielding any text events from the stream.

Source code in `src/agents/voice/workflow.py`



### VoiceWorkflowHelper

Source code in `src/agents/voice/workflow.py`



#### stream\_text\_from `async` `classmethod`

```
stream_text_from(
    result: RunResultStreaming,
) -> AsyncIterator[str]

```


Wraps a `RunResultStreaming` object and yields text events from the stream.

Source code in `src/agents/voice/workflow.py`



### SingleAgentWorkflowCallbacks

Source code in `src/agents/voice/workflow.py`



#### on\_run

```
on_run(
    workflow: SingleAgentVoiceWorkflow, transcription: str
) -> None

```


Called when the workflow is run.

Source code in `src/agents/voice/workflow.py`



### SingleAgentVoiceWorkflow

Bases: `[VoiceWorkflowBase](#agents.voice.workflow.VoiceWorkflowBase "VoiceWorkflowBase (agents.voice.workflow.VoiceWorkflowBase)")`

A simple voice workflow that runs a single agent. Each transcription and result is added to the input history. For more complex workflows (e.g. multiple Runner calls, custom message history, custom logic, custom configs), subclass `VoiceWorkflowBase` and implement your own logic.

Source code in `src/agents/voice/workflow.py`



#### \_\_init\_\_

```
__init__(
    agent: Agent[Any],
    callbacks: SingleAgentWorkflowCallbacks | None = None,
)

```


Create a new single agent voice workflow.

Parameters:



* Name:                 agent            
  * Type:                   Agent[Any]            
  * Description:                               The agent to run.                          
  * Default:                 required            
* Name:                 callbacks            
  * Type:                   SingleAgentWorkflowCallbacks | None            
  * Description:                               Optional callbacks to call during the workflow.                          
  * Default:                   None            


Source code in `src/agents/voice/workflow.py`



Table of contents

*   [input](#agents.voice.input)
*   [AudioInput](#agents.voice.input.AudioInput)
    
    *   [buffer](#agents.voice.input.AudioInput.buffer)
    *   [frame\_rate](#agents.voice.input.AudioInput.frame_rate)
    *   [sample\_width](#agents.voice.input.AudioInput.sample_width)
    *   [channels](#agents.voice.input.AudioInput.channels)
    *   [to\_audio\_file](#agents.voice.input.AudioInput.to_audio_file)
    *   [to\_base64](#agents.voice.input.AudioInput.to_base64)
    
*   [StreamedAudioInput](#agents.voice.input.StreamedAudioInput)
    
    *   [add\_audio](#agents.voice.input.StreamedAudioInput.add_audio)
    

`Input`
=======

### AudioInput `dataclass`

Static audio to be used as input for the VoicePipeline.

Source code in `src/agents/voice/input.py`



#### buffer `instance-attribute`

```
buffer: NDArray[int16 | float32]

```


A buffer containing the audio data for the agent. Must be a numpy array of int16 or float32.

#### frame\_rate `class-attribute` `instance-attribute`

```
frame_rate: int = DEFAULT_SAMPLE_RATE

```


The sample rate of the audio data. Defaults to 24000.

#### sample\_width `class-attribute` `instance-attribute`

```
sample_width: int = 2

```


The sample width of the audio data. Defaults to 2.

#### channels `class-attribute` `instance-attribute`

```
channels: int = 1

```


The number of channels in the audio data. Defaults to 1.

#### to\_audio\_file

```
to_audio_file() -> tuple[str, BytesIO, str]

```


Returns a tuple of (filename, bytes, content\_type)

Source code in `src/agents/voice/input.py`



#### to\_base64

```
to_base64() -> str

```


Returns the audio data as a base64 encoded string.

Source code in `src/agents/voice/input.py`



### StreamedAudioInput

Audio input represented as a stream of audio data. You can pass this to the `VoicePipeline` and then push audio data into the queue using the `add_audio` method.

Source code in `src/agents/voice/input.py`



#### add\_audio `async`

```
add_audio(audio: NDArray[int16 | float32])

```


Adds more audio data to the stream.

Parameters:



* Name:                 audio            
  * Type:                   NDArray[int16 | float32]            
  * Description:                               The audio data to add. Must be a numpy array of int16 or float32.                          
  * Default:                 required            


Source code in `src/agents/voice/input.py`


Table of contents

*   [result](#agents.voice.result)
*   [StreamedAudioResult](#agents.voice.result.StreamedAudioResult)
    
    *   [\_\_init\_\_](#agents.voice.result.StreamedAudioResult.__init__)
    *   [stream](#agents.voice.result.StreamedAudioResult.stream)
    

`Result`
========

### StreamedAudioResult

The output of a `VoicePipeline`. Streams events and audio data as they're generated.

Source code in `src/agents/voice/result.py`



#### \_\_init\_\_

```
__init__(
    tts_model: TTSModel,
    tts_settings: TTSModelSettings,
    voice_pipeline_config: VoicePipelineConfig,
)

```


Create a new `StreamedAudioResult` instance.

Parameters:



* Name:                 tts_model            
  * Type:                   TTSModel            
  * Description:                               The TTS model to use.                          
  * Default:                 required            
* Name:                 tts_settings            
  * Type:                   TTSModelSettings            
  * Description:                               The TTS settings to use.                          
  * Default:                 required            
* Name:                 voice_pipeline_config            
  * Type:                   VoicePipelineConfig            
  * Description:                               The voice pipeline config to use.                          
  * Default:                 required            


Source code in `src/agents/voice/result.py`



#### stream `async`

```
stream() -> AsyncIterator[VoiceStreamEvent]

```


Stream the events and audio data as they're generated.

Source code in `src/agents/voice/result.py`

# Pipeline Config - OpenAI Agents SDK
      Pipeline Config - OpenAI Agents SDK        

[Skip to content](#pipeline-config)

[![logo](../../../assets/logo.svg)](../../.. "OpenAI Agents SDK")

OpenAI Agents SDK

Pipeline Config

Initializing search

 [![logo](../../../assets/logo.svg)](../../.. "OpenAI Agents SDK")OpenAI Agents SDK

*   [Intro](../../..)
*   [Quickstart](../../../quickstart/)
*   [Examples](../../../examples/)
*    Documentation
    
    Documentation
    
    *   [Agents](../../../agents/)
    *   [Running agents](../../../running_agents/)
    *   [Results](../../../results/)
    *   [Streaming](../../../streaming/)
    *   [Tools](../../../tools/)
    *   [Model context protocol (MCP)](../../../mcp/)
    *   [Handoffs](../../../handoffs/)
    *   [Tracing](../../../tracing/)
    *   [Context management](../../../context/)
    *   [Guardrails](../../../guardrails/)
    *   [Orchestrating multiple agents](../../../multi_agent/)
    *   [Models](../../../models/)
    *   [Configuring the SDK](../../../config/)
    *   [Agent Visualization](../../../visualization/)
    *    Voice agents
        
        Voice agents
        
        *   [Quickstart](../../../voice/quickstart/)
        *   [Pipelines and workflows](../../../voice/pipeline/)
        *   [Tracing](../../../voice/tracing/)
        
    
*    API Reference
    
    API Reference
    
    *    Agents
        
        Agents
        
        *   [Agents module](../../)
        *   [Agents](../../agent/)
        *   [Runner](../../run/)
        *   [Tools](../../tool/)
        *   [Results](../../result/)
        *   [Streaming events](../../stream_events/)
        *   [Handoffs](../../handoffs/)
        *   [Lifecycle](../../lifecycle/)
        *   [Items](../../items/)
        *   [Run context](../../run_context/)
        *   [Usage](../../usage/)
        *   [Exceptions](../../exceptions/)
        *   [Guardrails](../../guardrail/)
        *   [Model settings](../../model_settings/)
        *   [Agent output](../../agent_output/)
        *   [Function schema](../../function_schema/)
        *   [Model interface](../../models/interface/)
        *   [OpenAI Chat Completions model](../../models/openai_chatcompletions/)
        *   [OpenAI Responses model](../../models/openai_responses/)
        *   [MCP Servers](../../mcp/server/)
        *   [MCP Util](../../mcp/util/)
        
    *    Tracing
        
        Tracing
        
        *   [Tracing module](../../tracing/)
        *   [Creating traces/spans](../../tracing/create/)
        *   [Traces](../../tracing/traces/)
        *   [Spans](../../tracing/spans/)
        *   [Processor interface](../../tracing/processor_interface/)
        *   [Processors](../../tracing/processors/)
        *   [Scope](../../tracing/scope/)
        *   [Setup](../../tracing/setup/)
        *   [Span data](../../tracing/span_data/)
        *   [Util](../../tracing/util/)
        
    *    Voice
        
        Voice
        
        *   [Pipeline](../pipeline/)
        *   [Workflow](../workflow/)
        *   [Input](../input/)
        *   [Result](../result/)
        *    Pipeline Config [Pipeline Config](./)
            
            Table of contents
            
            *   [pipeline\_config](#agents.voice.pipeline_config)
            *   [VoicePipelineConfig](#agents.voice.pipeline_config.VoicePipelineConfig)
                
                *   [model\_provider](#agents.voice.pipeline_config.VoicePipelineConfig.model_provider)
                *   [tracing\_disabled](#agents.voice.pipeline_config.VoicePipelineConfig.tracing_disabled)
                *   [trace\_include\_sensitive\_data](#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_data)
                *   [trace\_include\_sensitive\_audio\_data](#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_audio_data)
                *   [workflow\_name](#agents.voice.pipeline_config.VoicePipelineConfig.workflow_name)
                *   [group\_id](#agents.voice.pipeline_config.VoicePipelineConfig.group_id)
                *   [trace\_metadata](#agents.voice.pipeline_config.VoicePipelineConfig.trace_metadata)
                *   [stt\_settings](#agents.voice.pipeline_config.VoicePipelineConfig.stt_settings)
                *   [tts\_settings](#agents.voice.pipeline_config.VoicePipelineConfig.tts_settings)
                
            
        *   [Events](../events/)
        *   [Exceptions](../exceptions/)
        *   [Model](../model/)
        *   [Utils](../utils/)
        *   [OpenAIVoiceModelProvider](../models/openai_provider/)
        *   [OpenAI STT](../models/openai_stt/)
        *   [OpenAI TTS](../models/openai_tts/)
        
    *    Extensions
        
        Extensions
        
        *   [Handoff filters](../../extensions/handoff_filters/)
        *   [Handoff prompt](../../extensions/handoff_prompt/)
        
    

Table of contents

*   [pipeline\_config](#agents.voice.pipeline_config)
*   [VoicePipelineConfig](#agents.voice.pipeline_config.VoicePipelineConfig)
    
    *   [model\_provider](#agents.voice.pipeline_config.VoicePipelineConfig.model_provider)
    *   [tracing\_disabled](#agents.voice.pipeline_config.VoicePipelineConfig.tracing_disabled)
    *   [trace\_include\_sensitive\_data](#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_data)
    *   [trace\_include\_sensitive\_audio\_data](#agents.voice.pipeline_config.VoicePipelineConfig.trace_include_sensitive_audio_data)
    *   [workflow\_name](#agents.voice.pipeline_config.VoicePipelineConfig.workflow_name)
    *   [group\_id](#agents.voice.pipeline_config.VoicePipelineConfig.group_id)
    *   [trace\_metadata](#agents.voice.pipeline_config.VoicePipelineConfig.trace_metadata)
    *   [stt\_settings](#agents.voice.pipeline_config.VoicePipelineConfig.stt_settings)
    *   [tts\_settings](#agents.voice.pipeline_config.VoicePipelineConfig.tts_settings)
    

`Pipeline Config`
=================

### VoicePipelineConfig `dataclass`

Configuration for a `VoicePipeline`.

Source code in `src/agents/voice/pipeline_config.py`



#### model\_provider `class-attribute` `instance-attribute`

```
model_provider: VoiceModelProvider = field(
    default_factory=OpenAIVoiceModelProvider
)

```


The voice model provider to use for the pipeline. Defaults to OpenAI.

#### tracing\_disabled `class-attribute` `instance-attribute`

```
tracing_disabled: bool = False

```


Whether to disable tracing of the pipeline. Defaults to `False`.

#### trace\_include\_sensitive\_data `class-attribute` `instance-attribute`

```
trace_include_sensitive_data: bool = True

```


Whether to include sensitive data in traces. Defaults to `True`. This is specifically for the voice pipeline, and not for anything that goes on inside your Workflow.

#### trace\_include\_sensitive\_audio\_data `class-attribute` `instance-attribute`

```
trace_include_sensitive_audio_data: bool = True

```


Whether to include audio data in traces. Defaults to `True`.

#### workflow\_name `class-attribute` `instance-attribute`

```
workflow_name: str = 'Voice Agent'

```


The name of the workflow to use for tracing. Defaults to `Voice Agent`.

#### group\_id `class-attribute` `instance-attribute`

```
group_id: str = field(default_factory=gen_group_id)

```


A grouping identifier to use for tracing, to link multiple traces from the same conversation or process. If not provided, we will create a random group ID.

#### trace\_metadata `class-attribute` `instance-attribute`

```
trace_metadata: dict[str, Any] | None = None

```


An optional dictionary of additional metadata to include with the trace.

#### stt\_settings `class-attribute` `instance-attribute`

```
stt_settings: STTModelSettings = field(
    default_factory=STTModelSettings
)

```


The settings to use for the STT model.

#### tts\_settings `class-attribute` `instance-attribute`

```
tts_settings: TTSModelSettings = field(
    default_factory=TTSModelSettings
)

```


The settings to use for the TTS model.


Table of contents

*   [events](#agents.voice.events)
*   [VoiceStreamEvent](#agents.voice.events.VoiceStreamEvent)
*   [VoiceStreamEventAudio](#agents.voice.events.VoiceStreamEventAudio)
    
    *   [data](#agents.voice.events.VoiceStreamEventAudio.data)
    *   [type](#agents.voice.events.VoiceStreamEventAudio.type)
    
*   [VoiceStreamEventLifecycle](#agents.voice.events.VoiceStreamEventLifecycle)
    
    *   [event](#agents.voice.events.VoiceStreamEventLifecycle.event)
    *   [type](#agents.voice.events.VoiceStreamEventLifecycle.type)
    
*   [VoiceStreamEventError](#agents.voice.events.VoiceStreamEventError)
    
    *   [error](#agents.voice.events.VoiceStreamEventError.error)
    *   [type](#agents.voice.events.VoiceStreamEventError.type)
    

`Events`
========

### VoiceStreamEvent `module-attribute`

```
VoiceStreamEvent: TypeAlias = Union[
    VoiceStreamEventAudio,
    VoiceStreamEventLifecycle,
    VoiceStreamEventError,
]

```


An event from the `VoicePipeline`, streamed via `StreamedAudioResult.stream()`.

### VoiceStreamEventAudio `dataclass`

Streaming event from the VoicePipeline

Source code in `src/agents/voice/events.py`



#### data `instance-attribute`

```
data: NDArray[int16 | float32] | None

```


The audio data.

#### type `class-attribute` `instance-attribute`

```
type: Literal["voice_stream_event_audio"] = (
    "voice_stream_event_audio"
)

```


The type of event.

### VoiceStreamEventLifecycle `dataclass`

Streaming event from the VoicePipeline

Source code in `src/agents/voice/events.py`



#### event `instance-attribute`

```
event: Literal[
    "turn_started", "turn_ended", "session_ended"
]

```


The event that occurred.

#### type `class-attribute` `instance-attribute`

```
type: Literal["voice_stream_event_lifecycle"] = (
    "voice_stream_event_lifecycle"
)

```


The type of event.

### VoiceStreamEventError `dataclass`

Streaming event from the VoicePipeline

Source code in `src/agents/voice/events.py`



#### error `instance-attribute`

```
error: Exception

```


The error that occurred.

#### type `class-attribute` `instance-attribute`

```
type: Literal["voice_stream_event_error"] = (
    "voice_stream_event_error"
)

```


The type of event.



Table of contents

*   [exceptions](#agents.voice.exceptions)
*   [STTWebsocketConnectionError](#agents.voice.exceptions.STTWebsocketConnectionError)

`Exceptions`
============

### STTWebsocketConnectionError

Bases: `[AgentsException](../../exceptions/#agents.exceptions.AgentsException "AgentsException (agents.exceptions.AgentsException)")`

Exception raised when the STT websocket connection fails.

Source code in `src/agents/voice/exceptions.py`


Table of contents

*   [model](#agents.voice.model)
*   [TTSModelSettings](#agents.voice.model.TTSModelSettings)
    
    *   [voice](#agents.voice.model.TTSModelSettings.voice)
    *   [buffer\_size](#agents.voice.model.TTSModelSettings.buffer_size)
    *   [dtype](#agents.voice.model.TTSModelSettings.dtype)
    *   [transform\_data](#agents.voice.model.TTSModelSettings.transform_data)
    *   [instructions](#agents.voice.model.TTSModelSettings.instructions)
    *   [text\_splitter](#agents.voice.model.TTSModelSettings.text_splitter)
    *   [speed](#agents.voice.model.TTSModelSettings.speed)
    
*   [TTSModel](#agents.voice.model.TTSModel)
    
    *   [model\_name](#agents.voice.model.TTSModel.model_name)
    *   [run](#agents.voice.model.TTSModel.run)
    
*   [StreamedTranscriptionSession](#agents.voice.model.StreamedTranscriptionSession)
    
    *   [transcribe\_turns](#agents.voice.model.StreamedTranscriptionSession.transcribe_turns)
    *   [close](#agents.voice.model.StreamedTranscriptionSession.close)
    
*   [STTModelSettings](#agents.voice.model.STTModelSettings)
    
    *   [prompt](#agents.voice.model.STTModelSettings.prompt)
    *   [language](#agents.voice.model.STTModelSettings.language)
    *   [temperature](#agents.voice.model.STTModelSettings.temperature)
    *   [turn\_detection](#agents.voice.model.STTModelSettings.turn_detection)
    
*   [STTModel](#agents.voice.model.STTModel)
    
    *   [model\_name](#agents.voice.model.STTModel.model_name)
    *   [transcribe](#agents.voice.model.STTModel.transcribe)
    *   [create\_session](#agents.voice.model.STTModel.create_session)
    
*   [VoiceModelProvider](#agents.voice.model.VoiceModelProvider)
    
    *   [get\_stt\_model](#agents.voice.model.VoiceModelProvider.get_stt_model)
    *   [get\_tts\_model](#agents.voice.model.VoiceModelProvider.get_tts_model)
    

`Model`
=======

### TTSModelSettings `dataclass`

Settings for a TTS model.

Source code in `src/agents/voice/model.py`



#### voice `class-attribute` `instance-attribute`

```
voice: (
    Literal[
        "alloy",
        "ash",
        "coral",
        "echo",
        "fable",
        "onyx",
        "nova",
        "sage",
        "shimmer",
    ]
    | None
) = None

```


The voice to use for the TTS model. If not provided, the default voice for the respective model will be used.

#### buffer\_size `class-attribute` `instance-attribute`

```
buffer_size: int = 120

```


The minimal size of the chunks of audio data that are being streamed out.

#### dtype `class-attribute` `instance-attribute`

```
dtype: DTypeLike = int16

```


The data type for the audio data to be returned in.

#### transform\_data `class-attribute` `instance-attribute`

```
transform_data: (
    Callable[
        [NDArray[int16 | float32]], NDArray[int16 | float32]
    ]
    | None
) = None

```


A function to transform the data from the TTS model. This is useful if you want the resulting audio stream to have the data in a specific shape already.

#### instructions `class-attribute` `instance-attribute`

```
instructions: str = "You will receive partial sentences. Do not complete the sentence just read out the text."

```


The instructions to use for the TTS model. This is useful if you want to control the tone of the audio output.

#### text\_splitter `class-attribute` `instance-attribute`

```
text_splitter: Callable[[str], tuple[str, str]] = (
    get_sentence_based_splitter()
)

```


A function to split the text into chunks. This is useful if you want to split the text into chunks before sending it to the TTS model rather than waiting for the whole text to be processed.

#### speed `class-attribute` `instance-attribute`

```
speed: float | None = None

```


The speed with which the TTS model will read the text. Between 0.25 and 4.0.

### TTSModel

Bases: `ABC`

A text-to-speech model that can convert text into audio output.

Source code in `src/agents/voice/model.py`



#### model\_name `abstractmethod` `property`

```
model_name: str

```


The name of the TTS model.

#### run `abstractmethod`

```
run(
    text: str, settings: TTSModelSettings
) -> AsyncIterator[bytes]

```


Given a text string, produces a stream of audio bytes, in PCM format.

Parameters:



* Name:                 text            
  * Type:                   str            
  * Description:                               The text to convert to audio.                          
  * Default:                 required            


Returns:



* Type:                   AsyncIterator[bytes]            
  * Description:                               An async iterator of audio bytes, in PCM format.                          


Source code in `src/agents/voice/model.py`



### StreamedTranscriptionSession

Bases: `ABC`

A streamed transcription of audio input.

Source code in `src/agents/voice/model.py`



#### transcribe\_turns `abstractmethod`

```
transcribe_turns() -> AsyncIterator[str]

```


Yields a stream of text transcriptions. Each transcription is a turn in the conversation.

This method is expected to return only after `close()` is called.

Source code in `src/agents/voice/model.py`



#### close `abstractmethod` `async`

```
close() -> None

```


Closes the session.

Source code in `src/agents/voice/model.py`



### STTModelSettings `dataclass`

Settings for a speech-to-text model.

Source code in `src/agents/voice/model.py`



#### prompt `class-attribute` `instance-attribute`

```
prompt: str | None = None

```


Instructions for the model to follow.

#### language `class-attribute` `instance-attribute`

```
language: str | None = None

```


The language of the audio input.

#### temperature `class-attribute` `instance-attribute`

```
temperature: float | None = None

```


The temperature of the model.

#### turn\_detection `class-attribute` `instance-attribute`

```
turn_detection: dict[str, Any] | None = None

```


The turn detection settings for the model when using streamed audio input.

### STTModel

Bases: `ABC`

A speech-to-text model that can convert audio input into text.

Source code in `src/agents/voice/model.py`



#### model\_name `abstractmethod` `property`

```
model_name: str

```


The name of the STT model.

#### transcribe `abstractmethod` `async`

```
transcribe(
    input: AudioInput,
    settings: STTModelSettings,
    trace_include_sensitive_data: bool,
    trace_include_sensitive_audio_data: bool,
) -> str

```


Given an audio input, produces a text transcription.

Parameters:



* Name:                 input            
  * Type:                   AudioInput            
  * Description:                               The audio input to transcribe.                          
  * Default:                 required            
* Name:                 settings            
  * Type:                   STTModelSettings            
  * Description:                               The settings to use for the transcription.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive data in traces.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_audio_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive audio data in traces.                          
  * Default:                 required            


Returns:



* Type:                   str            
  * Description:                               The text transcription of the audio input.                          


Source code in `src/agents/voice/model.py`



#### create\_session `abstractmethod` `async`

```
create_session(
    input: StreamedAudioInput,
    settings: STTModelSettings,
    trace_include_sensitive_data: bool,
    trace_include_sensitive_audio_data: bool,
) -> StreamedTranscriptionSession

```


Creates a new transcription session, which you can push audio to, and receive a stream of text transcriptions.

Parameters:



* Name:                 input            
  * Type:                   StreamedAudioInput            
  * Description:                               The audio input to transcribe.                          
  * Default:                 required            
* Name:                 settings            
  * Type:                   STTModelSettings            
  * Description:                               The settings to use for the transcription.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive data in traces.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_audio_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive audio data in traces.                          
  * Default:                 required            


Returns:



* Type:                   StreamedTranscriptionSession            
  * Description:                               A new transcription session.                          


Source code in `src/agents/voice/model.py`



### VoiceModelProvider

Bases: `ABC`

The base interface for a voice model provider.

A model provider is responsible for creating speech-to-text and text-to-speech models, given a name.

Source code in `src/agents/voice/model.py`



#### get\_stt\_model `abstractmethod`

```
get_stt_model(model_name: str | None) -> STTModel

```


Get a speech-to-text model by name.

Parameters:



* Name:                 model_name            
  * Type:                   str | None            
  * Description:                               The name of the model to get.                          
  * Default:                 required            


Returns:



* Type:                   STTModel            
  * Description:                               The speech-to-text model.                          


Source code in `src/agents/voice/model.py`



#### get\_tts\_model `abstractmethod`

```
get_tts_model(model_name: str | None) -> TTSModel

```


Get a text-to-speech model by name.

Source code in `src/agents/voice/model.py`



Table of contents

*   [utils](#agents.voice.utils)
*   [get\_sentence\_based\_splitter](#agents.voice.utils.get_sentence_based_splitter)

`Utils`
=======

### get\_sentence\_based\_splitter

```
get_sentence_based_splitter(
    min_sentence_length: int = 20,
) -> Callable[[str], tuple[str, str]]

```


Returns a function that splits text into chunks based on sentence boundaries.

Parameters:



* Name:                 min_sentence_length            
  * Type:                   int            
  * Description:                               The minimum length of a sentence to be included in a chunk.                          
  * Default:                   20            


Returns:



* Type:                   Callable[[str], tuple[str, str]]            
  * Description:                               A function that splits text into chunks based on sentence boundaries.                          


Source code in `src/agents/voice/utils.py`



Table of contents

*   [openai\_model\_provider](#agents.voice.models.openai_model_provider)
*   [OpenAIVoiceModelProvider](#agents.voice.models.openai_model_provider.OpenAIVoiceModelProvider)
    
    *   [\_\_init\_\_](#agents.voice.models.openai_model_provider.OpenAIVoiceModelProvider.__init__)
    *   [get\_stt\_model](#agents.voice.models.openai_model_provider.OpenAIVoiceModelProvider.get_stt_model)
    *   [get\_tts\_model](#agents.voice.models.openai_model_provider.OpenAIVoiceModelProvider.get_tts_model)
    

`OpenAIVoiceModelProvider`
==========================

### OpenAIVoiceModelProvider

Bases: `[VoiceModelProvider](../../model/#agents.voice.model.VoiceModelProvider "VoiceModelProvider (agents.voice.model.VoiceModelProvider)")`

A voice model provider that uses OpenAI models.

Source code in `src/agents/voice/models/openai_model_provider.py`



#### \_\_init\_\_

```
__init__(
    *,
    api_key: str | None = None,
    base_url: str | None = None,
    openai_client: AsyncOpenAI | None = None,
    organization: str | None = None,
    project: str | None = None,
) -> None

```


Create a new OpenAI voice model provider.

Parameters:



* Name:                 api_key            
  * Type:                   str | None            
  * Description:                               The API key to use for the OpenAI client. If not provided, we will use thedefault API key.                          
  * Default:                   None            
* Name:                 base_url            
  * Type:                   str | None            
  * Description:                               The base URL to use for the OpenAI client. If not provided, we will use thedefault base URL.                          
  * Default:                   None            
* Name:                 openai_client            
  * Type:                   AsyncOpenAI | None            
  * Description:                               An optional OpenAI client to use. If not provided, we will create a newOpenAI client using the api_key and base_url.                          
  * Default:                   None            
* Name:                 organization            
  * Type:                   str | None            
  * Description:                               The organization to use for the OpenAI client.                          
  * Default:                   None            
* Name:                 project            
  * Type:                   str | None            
  * Description:                               The project to use for the OpenAI client.                          
  * Default:                   None            


Source code in `src/agents/voice/models/openai_model_provider.py`



#### get\_stt\_model

```
get_stt_model(model_name: str | None) -> STTModel

```


Get a speech-to-text model by name.

Parameters:



* Name:                 model_name            
  * Type:                   str | None            
  * Description:                               The name of the model to get.                          
  * Default:                 required            


Returns:



* Type:                   STTModel            
  * Description:                               The speech-to-text model.                          


Source code in `src/agents/voice/models/openai_model_provider.py`



#### get\_tts\_model

```
get_tts_model(model_name: str | None) -> TTSModel

```


Get a text-to-speech model by name.

Parameters:



* Name:                 model_name            
  * Type:                   str | None            
  * Description:                               The name of the model to get.                          
  * Default:                 required            


Returns:



* Type:                   TTSModel            
  * Description:                               The text-to-speech model.                          


Source code in `src/agents/voice/models/openai_model_provider.py`



Table of contents

*   [openai\_stt](#agents.voice.models.openai_stt)
*   [OpenAISTTTranscriptionSession](#agents.voice.models.openai_stt.OpenAISTTTranscriptionSession)
*   [OpenAISTTModel](#agents.voice.models.openai_stt.OpenAISTTModel)
    
    *   [\_\_init\_\_](#agents.voice.models.openai_stt.OpenAISTTModel.__init__)
    *   [transcribe](#agents.voice.models.openai_stt.OpenAISTTModel.transcribe)
    *   [create\_session](#agents.voice.models.openai_stt.OpenAISTTModel.create_session)
    

`OpenAI STT`
============

### OpenAISTTTranscriptionSession

Bases: `[StreamedTranscriptionSession](../../model/#agents.voice.model.StreamedTranscriptionSession "StreamedTranscriptionSession (agents.voice.model.StreamedTranscriptionSession)")`

A transcription session for OpenAI's STT model.

Source code in `src/agents/voice/models/openai_stt.py`



### OpenAISTTModel

Bases: `[STTModel](../../model/#agents.voice.model.STTModel "STTModel (agents.voice.model.STTModel)")`

A speech-to-text model for OpenAI.

Source code in `src/agents/voice/models/openai_stt.py`



#### \_\_init\_\_

```
__init__(model: str, openai_client: AsyncOpenAI)

```


Create a new OpenAI speech-to-text model.

Parameters:



* Name:                 model            
  * Type:                   str            
  * Description:                               The name of the model to use.                          
  * Default:                 required            
* Name:                 openai_client            
  * Type:                   AsyncOpenAI            
  * Description:                               The OpenAI client to use.                          
  * Default:                 required            


Source code in `src/agents/voice/models/openai_stt.py`



#### transcribe `async`

```
transcribe(
    input: AudioInput,
    settings: STTModelSettings,
    trace_include_sensitive_data: bool,
    trace_include_sensitive_audio_data: bool,
) -> str

```


Transcribe an audio input.

Parameters:



* Name:                 input            
  * Type:                   AudioInput            
  * Description:                               The audio input to transcribe.                          
  * Default:                 required            
* Name:                 settings            
  * Type:                   STTModelSettings            
  * Description:                               The settings to use for the transcription.                          
  * Default:                 required            


Returns:



* Type:                   str            
  * Description:                               The transcribed text.                          


Source code in `src/agents/voice/models/openai_stt.py`



#### create\_session `async`

```
create_session(
    input: StreamedAudioInput,
    settings: STTModelSettings,
    trace_include_sensitive_data: bool,
    trace_include_sensitive_audio_data: bool,
) -> StreamedTranscriptionSession

```


Create a new transcription session.

Parameters:



* Name:                 input            
  * Type:                   StreamedAudioInput            
  * Description:                               The audio input to transcribe.                          
  * Default:                 required            
* Name:                 settings            
  * Type:                   STTModelSettings            
  * Description:                               The settings to use for the transcription.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive data in traces.                          
  * Default:                 required            
* Name:                 trace_include_sensitive_audio_data            
  * Type:                   bool            
  * Description:                               Whether to include sensitive audio data in traces.                          
  * Default:                 required            


Returns:



* Type:                   StreamedTranscriptionSession            
  * Description:                               A new transcription session.                          


Source code in `src/agents/voice/models/openai_stt.py`



Table of contents

*   [openai\_tts](#agents.voice.models.openai_tts)
*   [OpenAITTSModel](#agents.voice.models.openai_tts.OpenAITTSModel)
    
    *   [\_\_init\_\_](#agents.voice.models.openai_tts.OpenAITTSModel.__init__)
    *   [run](#agents.voice.models.openai_tts.OpenAITTSModel.run)
    

`OpenAI TTS`
============

### OpenAITTSModel

Bases: `[TTSModel](../../model/#agents.voice.model.TTSModel "TTSModel (agents.voice.model.TTSModel)")`

A text-to-speech model for OpenAI.

Source code in `src/agents/voice/models/openai_tts.py`



#### \_\_init\_\_

```
__init__(model: str, openai_client: AsyncOpenAI)

```


Create a new OpenAI text-to-speech model.

Parameters:



* Name:                 model            
  * Type:                   str            
  * Description:                               The name of the model to use.                          
  * Default:                 required            
* Name:                 openai_client            
  * Type:                   AsyncOpenAI            
  * Description:                               The OpenAI client to use.                          
  * Default:                 required            


Source code in `src/agents/voice/models/openai_tts.py`



#### run `async`

```
run(
    text: str, settings: TTSModelSettings
) -> AsyncIterator[bytes]

```


Run the text-to-speech model.

Parameters:



* Name:                 text            
  * Type:                   str            
  * Description:                               The text to convert to speech.                          
  * Default:                 required            
* Name:                 settings            
  * Type:                   TTSModelSettings            
  * Description:                               The settings to use for the text-to-speech model.                          
  * Default:                 required            


Returns:



* Type:                   AsyncIterator[bytes]            
  * Description:                               An iterator of audio chunks.                          


Source code in `src/agents/voice/models/openai_tts.py`


    

Table of contents

*   [handoff\_filters](#agents.extensions.handoff_filters)
*   [remove\_all\_tools](#agents.extensions.handoff_filters.remove_all_tools)

`Handoff filters`
=================

### remove\_all\_tools

```
remove_all_tools(
    handoff_input_data: HandoffInputData,
) -> HandoffInputData

```


Filters out all tool items: file search, web search and function calls+output.

Source code in `src/agents/extensions/handoff_filters.py`



Table of contents

*   [handoff\_prompt](#agents.extensions.handoff_prompt)
*   [RECOMMENDED\_PROMPT\_PREFIX](#agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX)
*   [prompt\_with\_handoff\_instructions](#agents.extensions.handoff_prompt.prompt_with_handoff_instructions)

`Handoff prompt`
================

### RECOMMENDED\_PROMPT\_PREFIX `module-attribute`

```
RECOMMENDED_PROMPT_PREFIX = "# System context\nYou are part of a multi-agent system called the Agents SDK, designed to make agent coordination and execution easy. Agents uses two primary abstraction: **Agents** and **Handoffs**. An agent encompasses instructions and tools and can hand off a conversation to another agent when appropriate. Handoffs are achieved by calling a handoff function, generally named `transfer_to_<agent_name>`. Transfers between agents are handled seamlessly in the background; do not mention or draw attention to these transfers in your conversation with the user.\n"

```


### prompt\_with\_handoff\_instructions

```
prompt_with_handoff_instructions(prompt: str) -> str

```


Add recommended instructions to the prompt for agents that use handoffs.

Source code in `src/agents/extensions/handoff_prompt.py`

