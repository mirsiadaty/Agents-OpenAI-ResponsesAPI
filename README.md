# Agents-OpenAI-ResponsesAPI
multi-agent-system MAS via OpenAI's python Agents framework and Responses API

The OpenAI Agents SDK and Responses API are new tools designed to simplify the creation of AI agents, which are systems that can independently perform tasks on your behalf. The Responses API (https://platform.openai.com/docs/api-reference/responses) is a foundational API for building agentic experiences, offering built-in tools and streamlined functionality, while the Agents SDK (https://openai.github.io/openai-agents-python/) provides a framework for orchestrating multiple agents to work together (https://openai.github.io/openai-agents-python/quickstart/). This combination allows developers to build complex, modular AI applications with less complexity.


## creating specialized worker agents

The following code excerpt shows creating two agents that help with math versus history questions/homeworks:

```
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

```

## creating orchestrator agent

We then define a third agent, an orchestrator agent, which takes user's question, and decides which of the specialized worker agents to call next.

```
triage_agent = Agent(
    name="Triage Agent",
    instructions="You determine which agent to use based on the user's homework question",
    handoffs=[history_tutor_agent, math_tutor_agent],
    input_guardrails=[
        InputGuardrail(guardrail_function=homework_guardrail),
    ],
)
```

## creating guardrail

We create input guardrails to make sure the inputs to our agents are valid. For example, we might want to make sure that the user isn't trying to ask for help with problems that are not covered by our MAS multi agent system.

Hence this MAS has four agents, {orchestrator agent, math agent, history agent, guardrail agent}.

```
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

## test the MAS: three questions

We then send a set of questions to the MAS we just built. This set has three questions, where one is a history question, the second is a math, and the third one is a question that is beyond our MAS, hence we expect the guardrail to trip.

```
async def main():
    result = await Runner.run(triage_agent, "who was the first president of the united states?")
    print(result.final_output)
    
    result = await Runner.run(triage_agent, "Can you help me solve for x: 2x + 5 = 11")
    print(result.final_output)
    
    result = await Runner.run(triage_agent, "what is life")
    print(result.final_output)
```

## MAS response to the first question

The following shows that our MAS is using the latest OpenAI API called Responses, to generate the answer to the history question:

```
22 Thu Jun 26 12:45:43 2025

INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"

The first president of the United States was George Washington. He served from 1789 to 1797.
Washington was unanimously elected by the Electoral College and is often referred to as the \"Father of His Country\" for his leadership in the founding of the nation.
Before becoming president, he played a pivotal role as the commander-in-chief of the Continental Army during the American Revolutionary War.
Washington set many precedents for the new government, including the tradition of a two-term limit for presidents.
```



