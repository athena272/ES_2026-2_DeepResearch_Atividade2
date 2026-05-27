# ES_2026-2_DeepResearch_Atividade2

Repositório de trabalho da **Equipe 1** para a **Atividade 2** (Auditoria Forense de Software e Plano de Resgate Técnico) — projeto analisado: [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch).

Atividade 1 (contexto): [ES_2026-2_DeepResearch](https://github.com/athena272/ES_2026-2_DeepResearch).

---

## Índice das partes (Atividade 2)

| Parte | Eixo | Responsável | Documento |
|-------|------|-------------|-----------|
| **1** | GPR — Arqueologia de issues | Guilherme Rosário Alves | [docs/parte1-arqueologia-issues.md](docs/parte1-arqueologia-issues.md) · [trecho PDF](docs/relatorio-pdf-parte1.md) |
| 2 | GPR — Ritmo de entrega e PRs | *(equipe)* | *(a publicar)* |
| 3 | SOLID — Teste da troca (DIP) | *(equipe)* | *(a publicar)* |
| 4 | SOLID — SRP e DRY | *(equipe)* | *(a publicar)* |
| **5** | GoF — Agente e fronteiras | *(colega)* | Seção abaixo neste README |
| 6 | Plano de resgate + MPS.BR G | *(equipe)* | *(a publicar)* |

### Evidências visuais (Parte 1)

Prints do GitHub: pasta [assets/parte1/](assets/parte1/) — ver [checklist de capturas](assets/parte1/README.md).

---

# Parte 5 — GoF no agente e nas fronteiras (criacional, estrutural, comportamental)

Este documento analisa o núcleo de inferência e a orquestração do agente (`inference/react_agent.py`) sob a ótica dos padrões de projeto GoF, identificando como as dependências são gerenciadas, como o sistema se comunica com o mundo exterior e como o fluxo de decisão interno é estruturado.

---

## 5.1 Criacional – Instanciação de dependências caras

### Evidência

A classe `MultiTurnReactAgent` herda de `FnCallAgent` (pacote `qwen_agent`). O cliente LLM é criado via configuração passada ao construtor da superclasse. As ferramentas são registradas em um dicionário global `TOOL_MAP`:

```python
# inference/react_agent.py (linhas 31-38)
TOOL_CLASS = [
    FileParser(),
    Scholar(),
    Visit(),
    Search(),
    PythonInterpreter(),
]
TOOL_MAP = {tool.name: tool for tool in TOOL_CLASS}
```

A instanciação real ocorre em `MultiTurnReactAgent._init_tools()`, que percorre `TOOL_MAP` e cria objetos chamando os construtores das classes. O cliente LLM (OpenAI SDK) é criado dentro de `call_server()` diretamente:

```python
client = OpenAI(base_url=endpoint, api_key=api_key)
```

### Diagnóstico

| Problema | Descrição |
|---|---|
| ❌ Ausência de Factory Method / Abstract Factory | A criação de ferramentas é centralizada em um dicionário, mas ainda é uma fábrica ingênua (mapeamento `string → classe`). Não há separação entre a lógica de criação e o uso. |
| ❌ Singleton não utilizado | O cliente HTTP para a LLM é recriado a cada chamada de `call_server()`, gerando desperdício de conexões e sem reuso de pools. |
| ❌ Acoplamento a construtores concretos | Se uma ferramenta precisar de parâmetros extras (ex: timeout diferente), o dicionário simples não suporta injeção de dependências. |

### Recomendação

- Aplicar **Factory Method** para criar ferramentas: cada ferramenta implementa um método `create(config)`, ou usar uma `ToolFactory` com registro dinâmico (padrão Registry).
- Usar **Singleton** ou, preferencialmente, **injeção de dependência** para o cliente LLM — um `LLMClient` único criado no início e passado ao agente, reduzindo overhead e facilitando testes.
- Adotar um **Builder** para configurar o agente com diferentes conjuntos de ferramentas (ex: agente só com `search+visit`, outro com `python+file`).

### Risco
**Alto** – A criação dispersa e sem controle do cliente LLM pode levar a estouro de conexões e vazamento de recursos. A ausência de uma fábrica extensível torna a adição de novas ferramentas rígida e propensa a erros no carregamento.

---

## 5.2 Estrutural – Padronização de acesso a backends (Adapter)

### Evidência

O repositório usa uma estratégia de **Adapter** via protocolo OpenAI-compatible. A função `call_server` é responsável por comunicar com o backend de inferência, mas apresenta valores fixados (*hardcoded*):

```python
# inference/react_agent.py (linhas 59-66)
def call_server(self, msgs, planning_port, max_tries=10):
        
    openai_api_key = "EMPTY"
    openai_api_base = f"http://127.0.0.1:{planning_port}/v1"

    client = OpenAI(
        api_key=openai_api_key,
        base_url=openai_api_base,
        timeout=600.0,
    )
```

O servidor pode ser **vLLM local**, **OpenRouter**, ou qualquer provedor que implemente o formato `/v1/chat/completions`. As variáveis `BASE_URL` e `API_KEY` no `.env.example` permitem a troca de backend.

### Diagnóstico

| Aspecto | Status | Descrição |
|---|---|---|
| Adapter presente | ✅ | O formato OpenAI atua como interface comum (Target). Diferentes backends são adaptados para o mesmo contrato. |
| Abstração própria ausente e Hardcoding: | ❌ | O código depende diretamente do SDK da OpenAI e fixa a base da URL (`127.0.0.1`) dentro do próprio método, violando a injeção de dependências e dificultando a conteinerização (Docker) em redes complexas.|
| Adaptadores para ferramentas | ❌ | `tool_search.py` chama diretamente a API da Serper (`requests.post`), sem camada unificada para provedores de busca. |

### Recomendação

- Extrair uma interface `LLMClient` (método `chat(messages)`) e implementar adaptadores concretos: `OpenAIClient`, `VLLMClient`, `AnthropicClient`.
- Aplicar o mesmo padrão **Adapter** para ferramentas de busca/visita: criar uma interface `SearchProvider` com implementações `SerperSearch`, `GoogleSearch`, `BingSearch`.
- Usar **Bridge** se existirem múltiplas dimensões de variação (ex: LLM provider + formato de tool calling).

### Risco

**Médio** – O *Adapter* via OpenAI-compatible mitiga problemas imediatos, mas o acoplamento direto ao SDK oficial cria engessamento. A ausência de adaptadores para as ferramentas de busca torna a troca de provedor (ex: sair do Serper para o Google) custosa, exigindo alteração em vários arquivos e falhando no "Teste da Troca".
---

## 5.3 Comportamental – Fluxo de decisão do agente (Chain vs. condicionais)

### Evidência

O fluxo principal do agente ReAct está no método `_run` (herdado de `FnCallAgent`), que forma um bloco monolítico de mais de 100 linhas:

```python
# inference/react_agent.py (linhas 120-226)
def _run(self, data: str, model: str, **kwargs) -> List[List[Message]]:
    # ... (inicialização de variáveis e prompt)
    
    while num_llm_calls_available > 0:
        # 1. Chama o modelo (linha 153)
        content = self.call_server(messages, planning_port)

        # 2. Verifica as actions (linha 159)
        if '<tool_call>' in content and '</tool_call>' in content:
            tool_call = content.split('<tool_call>')[1].split('</tool_call>')[0]
            
            # 3. Executa a ferramenta com emaranhado condicional (linhas 161-176)
            try:
                if "python" in tool_call.lower():
                    # ... lógica hardcoded para python
                else:
                    # ... lógica para parse_file, scholar, search via custom_call_tool
            except:
                result = 'Error: Tool call is not a valid JSON...'
                
        # 4. Verifica encerramento (linha 180)
        if '<answer>' in content and '</answer>' in content:
            break
            
    # ... (formata o result final)
    return result
```

Há uma longa cadeia de `if/elif` para decidir qual ferramenta executar, sem padrão Chain of Responsibility ou dispatcher desacoplado.

### Diagnóstico

| Problema | Descrição |
|---|---|
| ❌ Violação do Open/Closed | Adicionar uma nova ferramenta exige modificar `_run` (adicionar mais um `elif`). |
| ❌ Ausência de Chain of Responsibility / Command | Uma cadeia de handlers permitiria adicionar novas ferramentas sem tocar no loop principal. |
| ❌ Emaranhado de condicionais | O código mistura lógica de parsing, execução e atualização de estado, dificultando testes unitários. |
| ❌ Ausência de Strategy para Validação | Não há um `AnswerValidatorStrategy` para auditar o retorno antes de finalizar o turno — a lógica de validação fica espalhada e relegada a scripts externos. |
| ✅ Template Method presente (parcialmente) | `FnCallAgent` define o esqueleto do agente e `MultiTurnReactAgent` sobrescreve alguns passos corretamente, mas o dispatcher ainda é condicional. |

### Recomendação

Substituir a cadeia de `if/elif` por um padrão **Registry + Command**:

```python
tool = self.tools.get(action)
result = tool.call(action_input) if tool else "Unknown action"
```

- Extrair a lógica de "pensar → agir → observar" para uma classe separada `ReactLoop` (padrão **Strategy**), permitindo variações do loop (ex: ReAct com reflexão, ReAct com memória).
- Implementar um `ValidatorStrategy` injetável dentro do laço principal, criando uma barreira ativa contra alucinações durante a inferência — não apenas no evaluation.

### Risco
**Alto** – O *dispatcher* condicional é frágil e eleva a complexidade ciclomática a cada nova ferramenta. Além disso, a ausência de um mecanismo de validação nativo no fluxo comportamental compromete a confiabilidade geral da IA, sendo o principal vetor para as alucinações e falhas identificadas na Atividade 1.

---
