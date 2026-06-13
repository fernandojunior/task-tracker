# Guia interno: Prompt Engineering

Resumo das técnicas aplicadas neste projeto.

> 📖 **Tempo estimado:** 25 min
> 🎯 **O que você vai aprender:**
> - As 6 técnicas mais alavancadas de prompt engineering segundo a Anthropic
> - Quando usar XML tags, exemplos, chain-of-thought, role prompting
> - Como escrever prompts que reduzem alucinação
> - Diferença entre prompts conversacionais e prompts de agente
> 🛠 **Pré-requisitos:** Capítulos 0-2
> 📝 **Exercícios:** 6 questões + 2 reescritas práticas
 
### 3.1 — Por que prompt engineering ainda importa
 
Em 2026, modelos melhoraram a ponto de "qualquer prompt funcionar mais
ou menos". Mas a diferença entre **funcionar** e **funcionar
consistentemente em produção** é enorme. Quem se destaca não é quem
escreve frases mais clever — é quem **estrutura o contexto** ao redor
da tarefa.
 
Prompt engineering é uma fração do AI engineering. As outras fatias
(spec, harness, skills) já vimos. Aqui focamos na arte de **uma
mensagem específica** ser eficaz.
 
### 3.2 — As 6 técnicas alavancadas
 
A documentação oficial da Anthropic destaca um conjunto de técnicas que
compõem o "núcleo" do prompt engineering. Vou organizá-las em ordem de
impacto que vejo na prática.
 
#### Técnica 1: Clareza e especificidade
 
**Anti-padrão:**
````
Faça uma função pra autenticar.
````
 
**Padrão:**
````
Escreva uma função Python `authenticate(email: str, password: str) -> User | None`
que:
- Normaliza o email (lowercase, strip).
- Busca o usuário via UserRepository (passado por DI).
- Verifica a senha com bcrypt.
- Retorna o User se válido, None caso contrário.
- Lança AuthenticationError em vez de None em caso de exceção (DB indisponível).
 
Convenções do projeto:
- Type hints estritos.
- Não logar a senha.
````
 
A primeira gera 30 implementações diferentes. A segunda gera previsível.
 
**Princípio:** trate o agente como um engenheiro talentoso recém-contratado.
Brilhante, mas zero contexto. Quanto mais preciso, melhor o output.
 
#### Técnica 2: Exemplos (few-shot)
 
Mostrar é mais eficaz que dizer. Especialmente para formato.
 
**Sem exemplo:**
````
Extraia as datas do texto.
````
 
**Com exemplo:**
````
Extraia as datas do texto e retorne como JSON.
 
Exemplo:
Texto: "Reunião em 15/03/2026 e entrega até 02/04/2026"
Saída: {"datas": ["2026-03-15", "2026-04-02"]}
 
Texto: "Lançamento previsto para Q4/2025"
Saída: {"datas": []}  # apenas datas exatas, não períodos
 
Agora processe:
Texto: <texto real>
Saída:
````
 
3-5 exemplos cobrindo casos diferentes (incluindo casos de borda) elevam
muito a consistência.
 
#### Técnica 3: XML tags para estruturar
 
Claude foi treinado especialmente bem para reconhecer estrutura XML em
prompts. Use tags para separar partes do prompt.
 
````
<task>
Refatorar esta função para Clean Architecture.
</task>
 
<code>
def create_user(name, email):
    db = sqlite3.connect("users.db")
    db.execute(f"INSERT INTO users VALUES ('{name}', '{email}')")
    return {"name": name, "email": email}
</code>
 
<rules>
- Domain deve ser puro Python.
- Repositório deve ser interface (Protocol) no domain.
- Implementação SQL fica em infrastructure.
- Use bcrypt para senhas; aqui não há senha.
- Validação de email no value object EmailAddress.
</rules>
 
<output_format>
Liste primeiro a estrutura de arquivos proposta, depois o código de cada um.
</output_format>
````
 
Estrutura clara → resposta estruturada. Em particular, isto facilita você
parsear a resposta programaticamente depois.
 
#### Técnica 4: Role prompting
 
Atribuir um papel via system prompt ajusta tom, profundidade e estilo
sem você precisar especificar cada um.
 
````
System: Você é um arquiteto de software sênior especializado em DDD
e Python. Sua função é revisar código apontando violações de princípios
de Clean Architecture. Seja direto, cite arquivo:linha, e proponha fix
específico.
````
 
vs.
 
````
System: Você é um tutor paciente ensinando programação Python para
iniciantes. Use analogias do dia-a-dia. Explique cada decisão.
````
 
O mesmo modelo, prompts iguais no `user`, geram respostas muito diferentes.
 
#### Técnica 5: Chain-of-thought (pensar passo a passo)
 
Para problemas que envolvem raciocínio (debug, refatoração complexa,
análise), peça explicitamente que o modelo "pense alto":
 
````
Antes de propor a refatoração, faça em <thinking>:
1. Liste as violações de Clean Arch que você vê.
2. Para cada uma, explique por que viola.
3. Proponha a refatoração mínima que corrige.
 
Depois, em <solution>, mostre o código final.
````
 
Em modelos com **extended thinking**, esse passo já é nativo — mas
mesmo sem isso, instruir o modelo a "explicitar o raciocínio antes da
resposta" reduz erros em problemas não-triviais.
 
⚠️ Use chain-of-thought para problemas complexos. Para perguntas
diretas ("qual o tipo de retorno desta função?"), forçar CoT só desperdiça
tokens.
 
#### Técnica 6: Pré-fill da resposta
 
Você pode "começar" a resposta do assistente, forçando um formato:
 
````
User: Liste 3 vantagens de Clean Architecture.
 
Assistant (pre-fill): Aqui estão 3 vantagens:
1.
````
 
O modelo continua daí. Útil para formato rígido (JSON, XML, listas).
 
````python
# Exemplo com API
response = client.messages.create(
    model="claude-sonnet-4-6",
    messages=[
        {"role": "user", "content": "Extraia nome e email do texto..."},
        {"role": "assistant", "content": "{\"name\": \""}  # pre-fill
    ],
    max_tokens=200,
)
# A resposta vem completando o JSON
````
 
### 3.3 — Anti-padrões comuns
 
**1. Negações soltas.**
- ❌ "Não use any."
- ✅ "Use tipos específicos. Se o tipo for desconhecido, use `object` ou
  defina um `TypeAlias`."
LLMs respondem melhor a "faça X" do que "não faça Y". Diga o que você
quer, não só o que você não quer.
 
**2. Pedidos vagos de "boa qualidade".**
- ❌ "Faça código bonito."
- ✅ "Aplique ruff format. Mantenha linhas ≤ 100 chars. Funções públicas
  com docstring."
"Bonito" e "limpo" são subjetivos. Critérios objetivos são executáveis.
 
**3. Múltiplos objetivos no mesmo prompt.**
- ❌ "Implemente o endpoint, escreva os testes, atualize o README, e
  abra um PR."
- ✅ Quebrar em 4 prompts sequenciais, validando cada um.
Múltiplos objetivos → atenção fragmentada → cada um sai medíocre.
 
**4. Confiança excessiva em "pense profundamente".**
- ❌ "Pense muito bem antes de responder. É importante. Eu acredito em você."
- ✅ Estruturar o problema com clareza. Mostrar exemplos.
Modelos modernos com extended thinking pensam por padrão. Frases
motivacionais não somam.
 
### 3.4 — Prompts para agentes (Claude Code) vs. prompts conversacionais
 
H� uma diferença importante.
 
**Conversacional (chat na web):**
- Você quer resposta em texto.
- Pode reformular se não gostar.
- Contexto curto (o prompt + memória de chat).
**Agente (Claude Code):**
- Modelo executa ações (ler arquivo, editar, rodar comandos).
- Erros têm efeito real (arquivo apagado, commit feito).
- Contexto envolve árvore de arquivos, CLAUDE.md, skills.
Para agentes, dois ajustes:
 
1. **Seja explícito sobre verificação.** "Verifique se o arquivo
   existe antes de editar." Sem isso, o agente pode editar arquivo
   errado/inexistente.
2. **Limite o escopo.** "Modifique APENAS `app/domain/task.py`." Sem
   isso, ele decide o escopo — e o que vier junto é seu problema.
### 3.5 — Template prático: prompt para feature
 
````
<context>
Estou implementando a spec docs/specs/003-create-task.md.
Já temos: Project entity, ProjectRepository, CreateProjectUseCase.
Plan aprovado em docs/plans/003-create-task.md.
</context>
 
<task>
Execute o passo 1 do plan: criar entidade Task, value objects e enums
em app/domain/task.py, com testes unitários em tests/unit/test_task.py.
 
NÃO toque em outros arquivos neste turno.
</task>
 
<rules>
- Siga CLAUDE.md (Clean Arch, type hints, naming).
- TDD: escreva test_task.py primeiro com casos da spec, rode pytest
  (vai falhar), aí implemente task.py, rode pytest de novo (deve passar).
- Reporte: arquivos criados, comando rodado, resultado.
</rules>
 
<output_format>
1. Diff dos arquivos novos.
2. Output do pytest.
3. Próximo passo sugerido (do plan).
</output_format>
````
 
Esse formato funciona em qualquer agente. Tem contexto, tarefa, regras,
formato. Sem ambiguidade.
 
### 3.6 — Iteração: refinar prompt vs. refinar resposta
 
Quando a resposta não veio boa, dois caminhos:
 
**Refinar resposta:** "Refaça aquela função, mas use injeção de
dependência." — Funciona pra ajuste pontual.
 
**Refinar prompt:** Volte ao prompt original, identifique o que faltou,
acrescente. Funciona pra padrões que vão se repetir.
 
Se você ajusta a mesma coisa 3+ vezes na mesma sessão, é sinal de que
o prompt original (ou o CLAUDE.md) tem uma lacuna. Refine na fonte.
 
### 3.7 — 💾 Commit
 
````bash
$ git checkout -b docs/prompt-engineering-guide
$ mkdir -p docs/guides
$ cat > docs/guides/prompt-engineering.md << 'EOF'
# Guia interno: Prompt Engineering
 
Resumo das técnicas aplicadas neste projeto.
[copie o conteúdo do Cap. 3 ou link para o handbook]
EOF
$ git add docs/
$ git commit -m "docs: add internal prompt engineering guide"
$ git push -u origin docs/prompt-engineering-guide
$ gh pr create --fill
````
 
