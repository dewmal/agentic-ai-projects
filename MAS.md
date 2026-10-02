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
