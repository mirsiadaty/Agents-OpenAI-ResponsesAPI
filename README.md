# Agents-OpenAI-ResponsesAPI
multi-agent-system via OpenAI Agents framework and Responses API

The OpenAI Agents SDK and Responses API are new tools designed to simplify the creation of AI agents, which are systems that can independently perform tasks on your behalf. The Responses API is a foundational API for building agentic experiences, offering built-in tools and streamlined functionality, while the Agents SDK provides a framework for orchestrating multiple agents to work together. This combination allows developers to build complex, modular AI applications with less complexity.


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







