# Tarefas: Pipeline de Refatoração Atômica com LLM (Tier C)

Base: main
Branch: tasks/pipeline-tier-c-refactoring

Legenda: [ ] aberta | [~] aguardando verificação | [x] verificada | [!] falhou

## T1 — Enriquecer detectores com evidência de localização do smell (linhas e nós)
- [ ] Status
- Arquivos: `detectores/deep_nesting.py`, `detectores/long_method.py`, `tests/test_deep_nesting.py`, `tests/test_long_method.py`
- Descrição: Adicionar ao dicionário `evidence` dos detectores as linhas exatas e nós da AST responsáveis pelo smell (para deep nesting, apontar linha/bloco que excede max_depth; para long method, detalhar limites de statements e blocos), alinhando com magic_numbers e dead_code que já retornam linhas.
- Check rápido: `python -m pytest tests/test_deep_nesting.py tests/test_long_param_list.py`
- Critérios:
  - `python -m pytest tests/test_deep_nesting.py` passa e valida que `result.evidence` contém chave `violating_lines` (lista de inteiros com as linhas de profundidade excessiva).
  - `python -m pytest tests/test_long_method.py` passa e confirma que `evidence` expõe detalhes métricos e linhas de início/fim da função.
- Notas do executor:
- Notas do verificador:

## T2 — Implementar cliente de inferência local Ollama com fallback e extração limpa de código
- [ ] Status
- Arquivos: `extracao/llm/ollama_client.py`, `tests/test_ollama_client.py`
- Descrição: Criar módulo cliente HTTP para comunicação com o Ollama local (`/api/chat`), com tratamento de exceção para servidor offline, extração determinística de blocos de código Python (removendo tags markdown/explicações) e parametrização de modelo e temperatura.
- Check rápido: `python -m pytest tests/test_ollama_client.py`
- Critérios:
  - Testes unitários com mock HTTP passam validando envio de mensagens e extração de código puro sem formatação markdown.
  - Função levanta exceção customizada `OllamaConnectionError` com mensagem explicativa quando o Ollama não estiver respondendo na porta configurada.
- Notas do executor:
- Notas do verificador:

## T3 — Implementar construtor de prompts atômicos guiados por evidência de smell
- [ ] Status
- Arquivos: `extracao/refatoracao/prompt_builder.py`, `tests/test_prompt_builder.py`
- Descrição: Integrar as regras de `trilha_b/training/instruction_templates.py` com as evidências do detector para gerar prompts de refatoração atômica estrita, instruindo o LLM a modificar exclusivamente o trecho apontado pelo smell alvo sem alterar assinaturas ou adicionar features.
- Check rápido: `python -m pytest tests/test_prompt_builder.py`
- Critérios:
  - `python -m pytest tests/test_prompt_builder.py` passa validando que para cada um dos 5 smells (R1 a R5) o prompt gerado contém o código original, a regra específica do smell e a indicação explícita dos trechos/linhas reportados no `evidence`.
  - Prompt inclui instrução explícita de atomicidade ("do not touch other code or smells").
- Notas do executor:
- Notas do verificador:

## T4 — Implementar validador duplo de refatoração (Filtro Estático AST + Juiz LLM com feedback)
- [ ] Status
- Arquivos: `extracao/refatoracao/validador.py`, `tests/test_validador_refatoracao.py`
- Descrição: Criar orquestrador de validação em 3 etapas: (1) validação sintática via `ast.parse`; (2) validação estática via `detectores/` confirmando que o smell alvo sumiu e nenhum smell novo foi introduzido; (3) avaliação semântica pelo Juiz LLM (Gemma) analisando o código completo com geração de feedback caso haja divergência.
- Check rápido: `python -m pytest tests/test_validador_refatoracao.py`
- Critérios:
  - `python -m pytest tests/test_validador_refatoracao.py` passa com testes cobrindo: descarte imediato de código com SyntaxError; descarte de código que não eliminou o smell alvo; descarte de código que introduziu outro smell (regressão); e aprovação de código válido pelo juiz mockado.
  - Validador retorna objeto estruturado com status (`APROVADO`, `REJEITADO_ESTATICO`, `REJEITADO_JUIZ`) e justificativa textual para viabilizar re-tentativa.
- Notas do executor:
- Notas do verificador:

## T5 — Implementar o runner sequencial da esteira Tier C (`generate_tier_c.py`)
- [ ] Status
- Arquivos: `scripts/generate_tier_c.py`, `tests/test_generate_tier_c.py`
- Descrição: Criar o script CLI principal que lê os pares de candidatos do dataset (código `before`), detecta os smells presentes, orquestra a refatoração atômica sequencial (um smell por vez em cadeia), gerencia re-tentativas com feedback do juiz, e salva as tuplas atômicas resultantes (`before`, `after`, `smell_type`, `evidence`, `verdict`) no formato JSONL padrão do projeto.
- Check rápido: `python scripts/generate_tier_c.py --help`
- Critérios:
  - `python scripts/generate_tier_c.py --help` executa com código de saída 0 e exibe argumentos (`--input`, `--output`, `--generator-model`, `--judge-model`, `--max-retries`).
  - Execução do runner em uma fixture com função contendo 2 smells gera 2 registros atômicos sequenciais independentes e conformes ao schema do projeto.
- Notas do executor:
- Notas do verificador:
