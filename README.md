# Como Criar um Agente com Claude

Este guia mostra como criar agentes de IA usando a API da Anthropic e o Agent SDK do Claude.

---

## O que é um Agente?

Um agente é um programa que usa um modelo de linguagem para tomar decisões e executar ações de forma autônoma. Em vez de apenas responder a uma pergunta, um agente pode usar **ferramentas** (buscar na web, ler arquivos, executar código) e iterar até completar uma tarefa complexa.

---

## Quando usar cada abordagem?

| Situação | Abordagem |
|---|---|
| Pergunta simples / extração / classificação | Claude API (chamada única) |
| Pipeline com lógica controlada por código | Claude API + tool use |
| Agente com ferramentas próprias | Claude API + loop agêntico |
| Agente com acesso a ficheiros, web, terminal | Agent SDK |

---

## Opção 1 — Claude API com Tool Use (Python)

Ideal para agentes com ferramentas personalizadas, onde você controla o loop.

### Instalação

```bash
pip install anthropic
```

### Agente básico (sem ferramentas)

```python
import anthropic

client = anthropic.Anthropic()  # lê ANTHROPIC_API_KEY do ambiente

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Olá! O que você pode fazer?"}]
)

print(response.content[0].text)
```

### Agente com ferramentas (tool use)

O Claude chama as ferramentas que você define. O SDK repete o loop automaticamente até o agente terminar.

```python
import anthropic
from anthropic import beta_tool

client = anthropic.Anthropic()

@beta_tool
def get_weather(location: str) -> str:
    """Retorna o clima atual de uma cidade.

    Args:
        location: Nome da cidade, ex: Lisboa, Portugal.
    """
    # Substitua por uma chamada real a uma API de clima
    return f"O clima em {location} é ensolarado, 22°C"

# O tool runner gere o loop agêntico automaticamente
runner = client.beta.messages.tool_runner(
    model="claude-opus-4-6",
    max_tokens=4096,
    tools=[get_weather],
    messages=[{"role": "user", "content": "Como está o tempo em Lisboa?"}],
)

for message in runner:
    for block in message.content:
        if hasattr(block, "text"):
            print(block.text)
```

### Loop manual (controlo total)

Use quando precisar de aprovação humana antes de executar ferramentas, logging personalizado, etc.

```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Retorna o clima de uma cidade",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "Nome da cidade"}
            },
            "required": ["city"]
        }
    }
]

def handle_tool(name: str, inputs: dict) -> str:
    if name == "get_weather":
        return f"O clima em {inputs['city']} é ensolarado, 22°C"
    return "Ferramenta não encontrada"

def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            # Agente terminou — retornar resposta final
            return next(b.text for b in response.content if b.type == "text")

        # Processar chamadas de ferramentas
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = handle_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result
                })

        # Adicionar resposta do assistente e resultados ao histórico
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

print(run_agent("Como está o tempo em Lisboa e no Porto?"))
```

---

## Opção 2 — Agent SDK (Python)

O Agent SDK é ideal quando o agente precisa de acesso a ficheiros, web, terminal — sem reimplementar essas ferramentas.

### Instalação

```bash
pip install claude-agent-sdk
```

### Agente básico

```python
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    async for message in query(
        prompt="Explica o que este repositório faz",
        options=ClaudeAgentOptions(
            cwd="/caminho/para/o/projeto",
            allowed_tools=["Read", "Glob", "Grep"]
        )
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

anyio.run(main)
```

### Ferramentas disponíveis no Agent SDK

| Ferramenta | Descrição |
|---|---|
| `Read` | Lê ficheiros |
| `Write` | Cria ficheiros |
| `Edit` | Edita ficheiros existentes |
| `Bash` | Executa comandos no terminal |
| `Glob` | Encontra ficheiros por padrão |
| `Grep` | Pesquisa conteúdo em ficheiros |
| `WebSearch` | Pesquisa na web |
| `WebFetch` | Obtém e analisa páginas web |
| `Agent` | Lança subagentes |

### Agente com ferramentas personalizadas

```python
import anyio
from claude_agent_sdk import (
    tool,
    create_sdk_mcp_server,
    ClaudeSDKClient,
    ClaudeAgentOptions,
    AssistantMessage,
    TextBlock,
)

@tool("get_weather", "Retorna o clima atual de uma cidade", {"city": str})
async def get_weather(args):
    city = args["city"]
    return {"content": [{"type": "text", "text": f"O clima em {city} é ensolarado, 22°C"}]}

server = create_sdk_mcp_server("weather-tools", tools=[get_weather])

async def main():
    options = ClaudeAgentOptions(mcp_servers={"weather": server})
    async with ClaudeSDKClient(options=options) as client:
        await client.query("Como está o tempo em Lisboa?")
        async for message in client.receive_response():
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, TextBlock):
                        print(block.text)

anyio.run(main)
```

---

## Configurar a chave de API

Nunca coloque a chave diretamente no código. Use uma variável de ambiente:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

Obtenha a sua chave em: [console.anthropic.com](https://console.anthropic.com)

---

## Modelos disponíveis

| Modelo | ID | Melhor para |
|---|---|---|
| Claude Opus 4.6 | `claude-opus-4-6` | Agentes complexos, raciocínio avançado |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | Equilíbrio entre velocidade e inteligência |
| Claude Haiku 4.5 | `claude-haiku-4-5` | Tarefas simples, baixo custo |

---

## Recursos adicionais

- [Documentação da API](https://platform.claude.com/docs)
- [Anthropic SDK Python](https://github.com/anthropics/anthropic-sdk-python)
- [Claude Agent SDK Python](https://github.com/anthropics/claude-agent-sdk-python)
