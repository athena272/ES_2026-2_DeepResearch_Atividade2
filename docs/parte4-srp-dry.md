# Parte 4 — Coesão e repetição (SOLID: SRP + DRY)

Este documento analisa o núcleo de inferência e a orquestração do agente (`inference/react_agent.py`) sob a ótica de dois princípios fundamentais da orientação a objetos (SOLID). O objetivo é demonstrar como a centralização excessiva de responsabilidades (SRP) e a repetição de código (DRY) dificultam a manutenção e a evolução do DeepResearch.

---

## 4.1 SRP – O "God Module" na Orquestração

O **Princípio da Responsabilidade Única (SRP)** estabelece que uma classe deve ter apenas um motivo para ser modificada. Se uma classe faz muitas coisas diferentes, qualquer alteração no sistema exigirá mexer nela.

### Evidência

A classe `MultiTurnReactAgent` centraliza o comportamento do sistema, atuando como um "God Module" (Módulo Deus). O acúmulo de responsabilidades fica claro ao observarmos o método `call_server` e o método `_run`:

```python
# inference/react_agent.py
def call_server(self, msgs, planning_port, max_tries=10):
    # Responsabilidade 1: Infraestrutura de Rede
    client = OpenAI(api_key=openai_api_key, base_url=openai_api_base, timeout=600.0)
    
    for attempt in range(max_tries):
        try:
            chat_response = client.chat.completions.create(...)
        except (APIError, APIConnectionError, APITimeoutError) as e:
            time.sleep(2) # Responsabilidade 2: Gestão de falhas e backoff
```

Além da infraestrutura acima, dentro do laço principal do agente (método `_run`), o código realiza extração manual de dados fatiando texto bruto: `content.split('<tool_call>')[1]` (**Responsabilidade 3: Parsing**).

### Diagnóstico

A classe `MultiTurnReactAgent` possui três motivos distintos para mudar, o que quebra a coesão do código:

* **Regra de Negócio:** Se o ciclo de raciocínio da IA (ReAct) mudar.
* **Infraestrutura de Rede:** Se a forma de conectar à API da OpenAI mudar ou se a política de *retries* precisar de ajustes.
* **Formatação de Dados:** Se a IA passar a devolver dados em formato JSON em vez de tags XML `<tool_call>`.

### Recomendação de Refatoração

Para tornar o código mais coeso, a recomendação é dividir as tarefas:

1. **Isolar a Rede:** Criar uma classe separada (ex: `LLMNetworkClient`) que cuide apenas de enviar requisições HTTP e tratar os *timeouts*.
2. **Isolar o Parsing:** Criar uma classe utilitária (`ResponseParser`) que receba o texto bruto e devolva a informação limpa.
3. **Aliviar o Agente:** A `MultiTurnReactAgent` deve receber essas duas classes prontas, dedicando-se exclusivamente a controlar o fluxo de iterações do agente, sem precisar entender de redes ou manipulação de *strings*.

### Risco

**Médio para Alto.** O principal problema de manter um "God Module" é a rigidez. Se a equipe quiser adicionar suporte a outro provedor de Inteligência Artificial ou mudar a forma de conexão, terá que alterar o arquivo principal onde reside a lógica central do agente. Isso aumenta a chance de inserir pequenos erros (regressões) no loop principal de inteligência apenas por causa de uma atualização de infraestrutura, além de dificultar muito a criação de testes automatizados isolados.

---

## 4.2 DRY – Duplicação de Tratamento de Erros e Parsing

O princípio **DRY (*Don't Repeat Yourself* / Não se Repita)** orienta que cada pedaço de conhecimento ou lógica no sistema deve ter uma representação única. Copiar e colar a mesma estrutura em vários lugares gera dívida técnica.

### 🔍 Evidência

No DeepResearch, a forma como o sistema se defende de erros e lê as respostas da IA se repete em vários arquivos, tanto no orquestrador quanto dentro de cada ferramenta individual.

```python
# No orquestrador (inference/react_agent.py) - Proteção genérica
except Exception as e:
    result = 'Error: Tool call is not a valid JSON...'

# Repetido dentro das ferramentas (ex: inference/tool_python.py)
except Exception as e:
    return f"[Python Interpreter Error]: {str(e)}"
```

### Diagnóstico

* **Tratamento de Exceções Duplicado:** O sistema precisa transformar falhas de código (exceções do Python) em mensagens de texto amigáveis para a IA entender o que deu errado. Atualmente, cada desenvolvedor que cria uma ferramenta nova escreve o seu próprio bloco `try/except`, reinventando a forma de capturar o erro.
* **Lógica de Extração Repetida:** O código que procura as tags e corta a string (`split`) aparece espalhado em vez de estar contido em uma única função utilitária.

### Recomendação de Refatoração

A solução para o princípio DRY é a centralização:

1. **Decorador de Erros:** Criar um decorador Python (ex: `@safe_tool_execute`) ou um método unificado na classe base `BaseTool`. Assim, as ferramentas executam apenas o seu trabalho principal (buscar no Google, rodar Python). Se falharem, o decorador captura a falha e avisa a IA de forma padronizada.
2. **Utilitário de Texto:** Agrupar toda a manipulação de tags e conversão de JSON em um único arquivo de utilidades (ex: `parser_utils.py`).

### Risco

**Médio.** A repetição de lógica aumenta o custo de manutenção. Se a equipe decidir mudar o idioma das mensagens de erro ou melhorar o formato para a IA, terá que abrir arquivo por arquivo das ferramentas para alterar manualmente. Além disso, se um colaborador criar uma ferramenta nova e esquecer de adicionar o seu próprio bloco `try/except`, uma falha simples na ferramenta não será tratada, fazendo o programa fechar abruptamente (crash) em vez de o agente tentar contornar o problema.