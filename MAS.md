```py
from dotenv import load_dotenv
from pydantic_ai import Agent

load_dotenv()

agent_1 = Agent("groq:openai/gpt-oss-120b",instructions="Just reply with one sentence")
agent_2 = Agent("groq:openai/gpt-oss-120b",instructions="Just reply with one sentence and it should be a question")
agent_3 = Agent("groq:openai/gpt-oss-120b",instructions="Just reply with one sentence")


result = agent_1.run_sync("Explain About AI").output

print(f"A1 :{result}")
for q in range(2):
    result = agent_2.run_sync(result).output
    print(f"A2 :{result}")
    result = agent_3.run_sync(result).output
    print(f"A3 :{result}")
    result = agent_1.run_sync(result).output
    print(f"A1 :{result}")
```


-----

```py
from dotenv import load_dotenv
from pydantic_ai import Agent

load_dotenv()

agent_r = Agent("groq:openai/gpt-oss-120b",instructions="Research the toipic and return 5 main points.")
agent_w = Agent("groq:openai/gpt-oss-120b",instructions="Create max 300 word article using given main points")
agent_re = Agent("groq:openai/gpt-oss-120b",instructions="Read the input and fix issues and make sure it is readable")

task = "Explain what is AI and How it works"

print(f"Task >> {task}")

points = agent_r.run_sync(task).output
print(f"\nResearch Points >> {points}")


article = agent_w.run_sync(points).output
print(f"\nWriter Article >> {article}")


final_article = agent_re.run_sync(article).output
print(f"\nFinal Article >> {final_article}")



```


----------------------

```py
import asyncio

from dotenv import load_dotenv
from pydantic_ai import Agent

load_dotenv()

agent_r = Agent(
    "groq:openai/gpt-oss-120b",
    instructions="Research the toipic and return 5 main points.",
)
agent_w_1 = Agent(
    "groq:openai/gpt-oss-120b",
    instructions="Create max 100 word summary note using given main points and foucs about technical examples",
)
agent_w_2 = Agent(
    "groq:openai/gpt-oss-120b",
    instructions="Create max 100 word summary note  using given main points and foucs about day today examples",
)
agent_final_writer = Agent(
    "groq:openai/gpt-oss-120b", instructions="Create Final Article"
)

task = "Explain what is AI and How it works"

print(f"Task >> {task}")


async def main():
    points = (await agent_r.run(task)).output
    print(f"\nResearch Points >> {points}")

    results = await asyncio.gather(
        agent_w_1.run(points),
        agent_w_2.run(points)
    )

    final_article = (await agent_final_writer.run(f"{results[0].output} - {results[1].output}")).output
    print(f"\nFinal Article >> {final_article}")


if "__main__" == __name__:
    asyncio.run(main())
```
