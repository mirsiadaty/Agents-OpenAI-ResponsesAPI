# Agents-OpenAI-ResponsesAPI
multi-agent-system MAS via OpenAI's python Agents framework and Responses API

The OpenAI Agents SDK and Responses API are new tools designed to simplify the creation of AI agents, which are systems that can independently perform tasks on your behalf. The Responses API (https://platform.openai.com/docs/api-reference/responses) is a foundational API for building agentic experiences, offering built-in tools and streamlined functionality, while the Agents SDK (https://openai.github.io/openai-agents-python/) provides a framework for orchestrating multiple agents to work together (https://openai.github.io/openai-agents-python/quickstart/). This combination allows developers to build complex, modular AI applications with less complexity.

![agent_graph_250627](https://github.com/user-attachments/assets/a52ad106-cbcb-4082-a07f-767e6f0a0ea1)



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

## MAS response to the first question: history

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

## MAS response to the second question: math

And here our MAS reasons and solves the algebraic formula given:

```
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"

Sure, let's solve the equation \(2x + 5 = 11\).

### Step 1: Isolate the term with \(x\)
First, we want to isolate the term that contains \(x\). To do this, we need to get rid of the constant term on the left side of the equation.

\[
2x + 5 = 11
\]

Subtract 5 from both sides:

\[
2x + 5 - 5 = 11 - 5
\]

This simplifies to:

\[
2x = 6
\]

### Step 2: Solve for \(x\)
Now, divide both sides by 2 to solve for \(x\):

\[
\frac{2x}{2} = \frac{6}{2}
\]

This simplifies to:

\[
x = 3
\]

### Checking the solution
To ensure our solution is correct, substitute \(x = 3\) back into the original equation:

\[
2(3) + 5 = 11
\]

This simplifies to:

\[
6 + 5 = 11
\]

Since both sides of the equation are equal, our solution \(x = 3\) is correct.

### Example
If we had a similar problem, such as \(3x + 4 = 13\), we would follow the same process:

1. Subtract 4 from both sides:

   \[
   3x + 4 - 4 = 13 - 4
   \]
   
   This simplifies to:
   
   \[
   3x = 9
   \]

2. Divide both sides by 3:

   \[
   \frac{3x}{3} = \frac{9}{3}
   \]
   
   This simplifies to:
   
   \[
   x = 3
   \]

The solution, \(x = 3\), works when substituted back into the original equation.

Let me know if you need further clarification!
```

## MAS response to the third question: out of scope

The third question "what is life" is outside the scope of this MAS which covers history and math questions. We expect the guardrail agent to kick in, and trigger the tripwire, hence stopping the processing of user's question.

The following python excpetion trace verifies this: "InputGuardrailTripwireTriggered: Guardrail InputGuardrail triggered tripwire"

```
INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"

---------------------------------------------------------------------------
InputGuardrailTripwireTriggered           Traceback (most recent call last)
Cell In[9], line 6
      1 print('22' , time.asctime( time.localtime( time.time() ) ))
      5 if __name__ == "__main__":
----> 6     await main()
      7 #
     11 '''
     12 if __name__ == "__main__":
     13     asyncio.run(main())
   (...)
     22     await my_async_function()
     23 '''

Cell In[8], line 8, in main()
      5 result = await Runner.run(triage_agent, "Can you help me solve for x: 2x + 5 = 11")
      6 print(result.final_output)
----> 8 result = await Runner.run(triage_agent, "what is life")
      9 print(result.final_output)

File ~/virenv20250624/lib/python3.10/site-packages/agents/run.py:199, in Runner.run(cls, starting_agent, input, context, max_turns, hooks, run_config, previous_response_id)
    172 """Run a workflow starting at the given agent. The agent will run in a loop until a final
    173 output is generated. The loop runs like so:
    174 1. The agent is invoked with the given input.
   (...)
    196     agent. Agents may perform handoffs, so we don't know the specific type of the output.
    197 """
    198 runner = DEFAULT_AGENT_RUNNER
--> 199 return await runner.run(
    200     starting_agent,
    201     input,
    202     context=context,
    203     max_turns=max_turns,
    204     hooks=hooks,
    205     run_config=run_config,
    206     previous_response_id=previous_response_id,
    207 )

File ~/virenv20250624/lib/python3.10/site-packages/agents/run.py:395, in AgentRunner.run(self, starting_agent, input, **kwargs)
    390 logger.debug(
    391     f"Running agent {current_agent.name} (turn {current_turn})",
    392 )
    394 if current_turn == 1:
--> 395     input_guardrail_results, turn_result = await asyncio.gather(
    396         self._run_input_guardrails(
    397             starting_agent,
    398             starting_agent.input_guardrails
    399             + (run_config.input_guardrails or []),
    400             copy.deepcopy(input),
    401             context_wrapper,
    402         ),
    403         self._run_single_turn(
    404             agent=current_agent,
    405             all_tools=all_tools,
    406             original_input=original_input,
    407             generated_items=generated_items,
    408             hooks=hooks,
    409             context_wrapper=context_wrapper,
    410             run_config=run_config,
    411             should_run_agent_start_hooks=should_run_agent_start_hooks,
    412             tool_use_tracker=tool_use_tracker,
    413             previous_response_id=previous_response_id,
    414         ),
    415     )
    416 else:
    417     turn_result = await self._run_single_turn(
    418         agent=current_agent,
    419         all_tools=all_tools,
   (...)
    427         previous_response_id=previous_response_id,
    428     )

File ~/virenv20250624/lib/python3.10/site-packages/agents/run.py:1003, in AgentRunner._run_input_guardrails(cls, agent, guardrails, input, context)
    996         t.cancel()
    997     _error_tracing.attach_error_to_current_span(
    998         SpanError(
    999             message="Guardrail tripwire triggered",
   1000             data={"guardrail": result.guardrail.get_name()},
   1001         )
   1002     )
-> 1003     raise InputGuardrailTripwireTriggered(result)
   1004 else:
   1005     guardrail_results.append(result)

InputGuardrailTripwireTriggered: Guardrail InputGuardrail triggered tripwire

INFO:httpx:HTTP Request: POST https://api.openai.com/v1/responses "HTTP/1.1 200 OK"
```


