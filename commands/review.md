---
name: review
description: Interactive PR code reviewer that fetches PR details from GitHub, analyzes changes using the code-reviewer agent, and creates a GitHub review with user approval for each suggestion
allowed-tools:
  - Bash(git *)
  - Read
  - Grep
  - Glob
  - mcp__plugin_github_github__pull_request_read
  - mcp__plugin_github_github__get_file_contents
  - mcp__plugin_github_github__pull_request_review_write
  - mcp__plugin_github_github__add_comment_to_pending_review
  - Task
  - AskUserQuestion
---

# Interactive PR Code Reviewer

Este comando realiza uma revisão interativa de código de uma Pull Request do GitHub.

## Uso

```
/review <PR_NUMBER|PR_URL> [LANGUAGE]
```

Exemplos:
- `/review 123`
- `/review 123 pt`
- `/review https://github.com/org/repo/pull/123 portuguese`
- `/review https://github.com/org/repo/pull/123 en`

## Argumentos

ARGUMENTS: $ARGUMENTS

### Parsing de Argumentos:
1. **PR_IDENTIFIER** (obrigatório): Primeiro argumento - número ou URL da PR
2. **LANGUAGE** (opcional): Segundo argumento - idioma para mensagens e review
   - Valores aceitos: `english`, `en`, `portuguese`, `pt`
   - Default: `english`
   - Afeta: mensagens do comando E idioma do feedback do code-reviewer

---

## ⚠️ REGRA ABSOLUTA — NUNCA NEGOCIÁVEL

**NENHUM review ou comentário é submetido no GitHub sem confirmação explícita do usuário.**

- A **PHASE 6** é o **ÚNICO ponto de submissão** deste comando.
- TODA execução deve obrigatoriamente passar pela `AskUserQuestion` de PHASE 6 antes de chamar `submit_pending`.
- **NÃO HÁ EXCEÇÕES**: nem para reviews vazios, nem para aprovações automáticas, nem para qualquer outro caso.
- Se o fluxo salta para PHASE 6 a partir de qualquer fase anterior, a confirmação de PHASE 6 **sempre deve ser exibida** — os valores de `event` e `body` passados são apenas sugestões de padrão, nunca executados automaticamente.

---

## PHASE 1: Parse and Validate Input

**Objetivo**: Extrair owner, repo, pull_number e idioma do input do usuário

### Instruções:

1. **Parse $ARGUMENTS** para extrair PR identifier e idioma:
   - Split $ARGUMENTS por espaço
   - Primeiro argumento: PR_IDENTIFIER (obrigatório)
   - Segundo argumento: LANGUAGE (opcional, default: "english")

2. **Parse PR_IDENTIFIER**:

   **Se formato URL** (contém "github.com"):
   - Extrair usando regex: `https://github.com/([^/]+)/([^/]+)/pull/(\d+)`
   - Grupo 1: owner
   - Grupo 2: repo
   - Grupo 3: pull_number

   **Se formato numérico** (apenas dígitos):
   - pull_number = PR_IDENTIFIER
   - Obter owner/repo do git remote atual:
     ```bash
     git remote get-url origin
     ```
   - Parse URL do remote (formato: git@github.com:owner/repo.git ou https://github.com/owner/repo.git)
   - Extrair owner e repo

3. **Parse LANGUAGE**:
   - Se "portuguese" ou "pt": idioma = "portuguese"
   - Se "english" ou "en" ou vazio: idioma = "english" (default)
   - Qualquer outro valor: avisar e usar "english" como fallback

4. **Validação**:
   - Verifique que owner, repo e pull_number estão presentes
   - Se faltarem dados:
     * **English**: "❌ Error: Could not determine repository information. Please provide PR URL or run from a git repository."
     * **Portuguese**: "❌ Erro: Não foi possível determinar as informações do repositório. Forneça a URL da PR ou execute de um repositório git."
     * Saia do comando

5. **Exemplos de extração**:
   - `/review 123`:
     * PR_IDENTIFIER = "123"
     * LANGUAGE = "english" (default)
     * owner/repo = obtido do git remote

   - `/review 123 pt`:
     * PR_IDENTIFIER = "123"
     * LANGUAGE = "portuguese"
     * owner/repo = obtido do git remote

   - `/review https://github.com/shamanec/GADS/pull/123 en`:
     * owner = "shamanec"
     * repo = "GADS"
     * pull_number = 123
     * LANGUAGE = "english"

---

## PHASE 2: Fetch PR Details

**Objetivo**: Obter todas as informações da PR via GitHub MCP

### Instruções:

1. **Mostrar mensagem inicial** (adaptar conforme idioma):
   - **English**: `🔍 Analyzing PR #{pull_number}...`
   - **Portuguese**: `🔍 Analisando PR #{pull_number}...`

2. **Obter metadata da PR**:
   - Use: `mcp__plugin_github_github__pull_request_read`
   - Parameters:
     ```json
     {
       "method": "get",
       "owner": "{owner}",
       "repo": "{repo}",
       "pullNumber": {pull_number}
     }
     ```
   - Extraia: title, body (description), base.ref, head.ref, user.login (author), additions, deletions
   - **Error handling**: Se falhar (404, 403, etc.), mostre erro e saia

3. **Obter diff completo**:
   - Use: `mcp__plugin_github_github__pull_request_read`
   - Parameters:
     ```json
     {
       "method": "get_diff",
       "owner": "{owner}",
       "repo": "{repo}",
       "pullNumber": {pull_number}
     }
     ```
   - Armazene o diff para análise posterior

4. **Obter lista de arquivos modificados**:
   - Use: `mcp__plugin_github_github__pull_request_read`
   - Parameters:
     ```json
     {
       "method": "get_files",
       "owner": "{owner}",
       "repo": "{repo}",
       "pullNumber": {pull_number}
     }
     ```
   - Extraia para cada arquivo: filename, status, additions, deletions, patch

5. **Mostrar informações básicas** (adaptar conforme idioma):
   - **English**:
     ```
     ✓ PR found: "{title}"
       Author: @{author}
       Files changed: {count} (+{additions}, -{deletions})
       Base: {base_branch} ← Head: {head_branch}
     ```
   - **Portuguese**:
     ```
     ✓ PR encontrada: "{title}"
       Autor: @{author}
       Arquivos modificados: {count} (+{additions}, -{deletions})
       Base: {base_branch} ← Head: {head_branch}
     ```

6. **Edge case: PR muito grande**:
   - Se count_files > 100:
     * **English**: "⚠️ Warning: This PR has {count} modified files. Review may take a while."
     * **Portuguese**: "⚠️ Aviso: Esta PR tem {count} arquivos modificados. A revisão pode demorar."
     * Use AskUserQuestion: "Do you want to continue?" / "Deseja continuar?"
     * Se não: saia

---

## PHASE 3: Analyze Dependencies and Related Projects

**Objetivo**: Detectar quando mudanças na PR afetam outros projetos (hub ↔ hub-ui) e buscar contexto adicional

### Instruções:

1. **Mostrar mensagem inicial** (adaptar conforme idioma):
   - **English**: `🔍 Detecting cross-project dependencies...`
   - **Portuguese**: `🔍 Detectando dependências entre projetos...`

2. **Inicializar estrutura de dados**:
   - Criar lista `related_files` vazia para armazenar arquivos relacionados encontrados

3. **PARA CADA arquivo modificado**:

   **A. Detectar mudanças em API backend** (hub/routes/*, hub/handlers/*):

   - **Condição**: Se `filename` contém "hub/routes/" ou "hub/handlers/"

   - **Ação**:
     1. Extrair endpoints modificados do diff/patch:
        - Buscar padrões como: `"/api/..."`
        - Regex sugerido: `"(/api/[^"]+)"`
        - Armazenar lista de endpoints encontrados

     2. Para cada endpoint encontrado:
        - Use Grep para buscar no projeto hub-ui:
          ```
          Grep:
          pattern: "{endpoint}"
          path: "hub-ui/src"
          output_mode: "files_with_matches"
          ```

        3. Se encontrar arquivos no hub-ui:
           - Para cada arquivo encontrado (limitar a 3 arquivos por endpoint):
             * Use `mcp__plugin_github_github__get_file_contents`:
               ```json
               {
                 "owner": "{owner}",
                 "repo": "hub-ui",
                 "path": "{file_path}"
               }
               ```
             * Adicione à lista `related_files`:
               ```
               {
                 "file": "{file_path}",
                 "project": "hub-ui (Frontend)",
                 "reason": "Consome endpoint {endpoint} modificado na PR",
                 "content": "{file_content}"
               }
               ```

   **B. Detectar mudanças em chamadas de API frontend** (hub-ui/src/api/*, hub-ui/src/services/*):

   - **Condição**: Se `filename` contém "hub-ui/src/api/" ou "hub-ui/src/services/"

   - **Ação**:
     1. Extrair chamadas de API do diff/patch:
        - Buscar padrões como: fetch("/api/..."), axios.get("/api/...")
        - Regex sugerido: `[fetch|axios]\s*\(\s*["']([^"']+)["']`
        - Armazenar lista de endpoints chamados

     2. Para cada endpoint chamado:
        - Extrair path da API (ex: "/api/users" de "/api/users/${id}")
        - Use Grep para buscar handlers no backend:
          ```
          Grep:
          pattern: "\"/{api_path}\""
          path: "hub/routes"
          output_mode: "files_with_matches"
          ```

        3. Se encontrar arquivos no backend:
           - Para cada arquivo encontrado (limitar a 3 arquivos):
             * Use `mcp__plugin_github_github__get_file_contents`:
               ```json
               {
                 "owner": "{owner}",
                 "repo": "{repo}",
                 "path": "{file_path}"
               }
               ```
             * Adicione à lista `related_files`:
               ```
               {
                 "file": "{file_path}",
                 "project": "hub (Backend)",
                 "reason": "Implementa endpoint {endpoint} chamado no frontend",
                 "content": "{file_content}"
               }
               ```

4. **Compilar contexto adicional**:

   - Se `related_files` não está vazio:
     * **English**:
       ```
       ✓ Detected: {backend_count} backend changes → found {frontend_count} frontend consumers
       ✓ Detected: {frontend_count} frontend changes → found {backend_count} backend handlers
       ```
     * **Portuguese**:
       ```
       ✓ Detectado: {backend_count} mudanças no backend → encontrados {frontend_count} consumidores no frontend
       ✓ Detectado: {frontend_count} mudanças no frontend → encontrados {backend_count} handlers no backend
       ```

   - Criar seção formatada `related_files_context`:
     ```markdown
     ### Related Files from Other Projects

     **File**: {file_path} ({project})
     **Reason**: {reason}
     **Snippet** (primeiros 50 linhas):
     ```{language}
     {first_50_lines_of_content}
     ```

     ---

     [repetir para cada arquivo relacionado]
     ```

5. **Error handling**:
   - Se falhar ao acessar hub-ui ou outro projeto:
     * **English**: "⚠️ Warning: Could not access related project {project_name}"
     * **Portuguese**: "⚠️ Aviso: Não foi possível acessar projeto relacionado {project_name}"
     * Continue sem contexto adicional (não é erro crítico)

---

## PHASE 4: Invoke Code Reviewer Agent

**Objetivo**: Solicitar análise detalhada do code-reviewer agent

### Instruções:

1. **Mostrar mensagem inicial** (adaptar conforme idioma):
   - **English**: `📋 Invoking code-reviewer agent...`
   - **Portuguese**: `📋 Invocando code-reviewer agent...`

2. **Preparar contexto completo para o agente**:

   ```markdown
   ### PR Information
   - Repository: {owner}/{repo}
   - PR #{pull_number}: "{title}"
   - Author: @{author}
   - Base: {base_branch} ← Head: {head_branch}
   - Files Changed: {count_files} (+{additions}, -{deletions})

   ### Description
   {pr_description}

   ### Complete Diff
   {diff_content}

   ### Modified Files
   {file_list_with_stats}

   ### Related Files from Other Projects
   {related_files_context}

   ### Review Request
   Analise esta PR seguindo os princípios KISS e diretrizes do CLAUDE.md.
   Para cada issue encontrado, forneça feedback estruturado usando seu formato padrão.

   Foque especialmente em:
   - Simplicidade (KISS principle)
   - Over-engineering
   - SOLID principles
   - Segurança
   - Padrões do projeto GADS

   **IDIOMA DO FEEDBACK**: {language}
   - Se idioma = "english": Forneça TODO o feedback em inglês
   - Se idioma = "portuguese": Forneça TODO o feedback em português

   Estruture cada sugestão com:
   - Categoria e Severidade (High/Medium/Low para english, Alto/Médio/Baixo para portuguese)
   - File path e line number (quando identificável no diff)
   - Código original (se aplicável)
   - Sugestão de melhoria
   - Justificativa clara
   ```

3. **Invocar o agente**:
   - Use: Task tool
   - Parameters:
     ```json
     {
       "subagent_type": "code-reviewer",
       "description": "Review GitHub PR #{pull_number}",
       "prompt": "{contexto_preparado_acima}"
     }
     ```

4. **Aguardar resposta**:
   - Se demorar, mostrar mensagem de progresso:
     * **English**: `🔍 Analysis in progress... The code-reviewer is analyzing {count_files} modified files.`
     * **Portuguese**: `🔍 Análise em progresso... O code-reviewer está analisando {count_files} arquivos modificados.`

5. **Parse a resposta do agente**:
   - Extrair sugestões individuais da resposta
   - Para cada sugestão, identificar:
     * Categoria (ex: "Simplicity", "Security", "SOLID", etc.)
     * Severidade (High/Alto, Medium/Médio, Low/Baixo)
     * File path (se mencionado)
     * Line number (se mencionado)
     * Código original (se fornecido)
     * Código sugerido (se fornecido)
     * Justificativa
   - Armazenar em lista `suggestions`

6. **Mostrar resultado** (adaptar conforme idioma):
   - Se `suggestions` não está vazio:
     * **English**: `✓ Analysis complete! {count} suggestions found.`
     * **Portuguese**: `✓ Análise completa! {count} sugestões encontradas.`

   - Se `suggestions` está vazio:
     * **English**: `ℹ️ The code-reviewer did not find improvement suggestions in this PR.`
     * **Portuguese**: `ℹ️ O code-reviewer não encontrou sugestões de melhoria nesta PR.`
     * Use AskUserQuestion:
       - **English**: "No suggestions found. Do you want to proceed to submit a review anyway?"
       - **Portuguese**: "Nenhuma sugestão encontrada. Deseja prosseguir para submeter um review mesmo assim?"
       - Options: "Yes/Sim", "No/Não"
     * Se "Yes/Sim": prossiga para PHASE 6 normalmente (sem pending review criado ainda)
       - **IMPORTANTE**: NÃO defina `event` ou `body` pré-configurados. PHASE 6 fará a confirmação completa.
     * Se "No/Não": saia do comando

---

## PHASE 5: Interactive Review Approval

**Objetivo**: Solicitar aprovação do usuário para cada sugestão antes de adicionar ao review

**IMPORTANTE**: Todas as mensagens desta fase devem estar no idioma escolhido pelo usuário

### Instruções:

1. **Criar pending review no GitHub**:
   - Use: `mcp__plugin_github_github__pull_request_review_write`
   - Parameters:
     ```json
     {
       "method": "create",
       "owner": "{owner}",
       "repo": "{repo}",
       "pullNumber": {pull_number}
     }
     ```
   - **IMPORTANTE**: NÃO incluir parameter "event" (isso cria pending review, não submete)
   - **Error handling**: Se falhar:
     * **English**: "❌ Error creating pending review on GitHub: {error_message}"
     * **Portuguese**: "❌ Erro ao criar pending review no GitHub: {error_message}"
     * Saia do comando

2. **Inicializar contadores**:
   ```
   total_suggestions = len(suggestions)
   approved_count = 0
   edited_count = 0
   skipped_count = 0
   severity_counts = {"High": 0, "Medium": 0, "Low": 0}  # ou "Alto", "Médio", "Baixo"
   ```

3. **PARA CADA sugestão em `suggestions`**:

   **A. Mostrar ao usuário** (adaptar labels conforme idioma):

   Mostrar separador:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

   **Se idioma = "english"**:
   ```
   === Suggestion {current}/{total} ===

   Category: {category} - Severity: {severity}
   File: {file_path}:{line_number}

   Original Code:
   ```{language}
   {original_code}
   ```

   Suggestion:
   ```{language}
   {suggested_code}
   ```

   Rationale: {rationale}
   ```

   **Se idioma = "portuguese"**:
   ```
   === Sugestão {current}/{total} ===

   Categoria: {categoria} - Severidade: {severidade}
   File: {file_path}:{line_number}

   Código Original:
   ```{language}
   {codigo_original}
   ```

   Sugestão:
   ```{language}
   {codigo_sugerido}
   ```

   Justificativa: {justificativa}
   ```

   **B. Solicitar decisão com AskUserQuestion** (traduzir conforme idioma):

   **Se idioma = "english"**:
   - Use AskUserQuestion:
     ```json
     {
       "questions": [{
         "question": "What would you like to do with this suggestion?",
         "header": "Action",
         "multiSelect": false,
         "options": [
           {
             "label": "Approve and add to review",
             "description": "Add this comment to the pending review as-is"
           },
           {
             "label": "Edit comment before adding",
             "description": "Modify the comment text before adding to review"
           },
           {
             "label": "Skip this suggestion",
             "description": "Do not add this comment to the review"
           },
           {
             "label": "Cancel entire review",
             "description": "Delete pending review and exit"
           }
         ]
       }]
     }
     ```

   **Se idioma = "portuguese"**:
   - Use AskUserQuestion:
     ```json
     {
       "questions": [{
         "question": "O que deseja fazer com esta sugestão?",
         "header": "Ação",
         "multiSelect": false,
         "options": [
           {
             "label": "Aprovar e adicionar ao review",
             "description": "Adicionar este comentário ao pending review como está"
           },
           {
             "label": "Editar comentário antes de adicionar",
             "description": "Modificar o texto do comentário antes de adicionar ao review"
           },
           {
             "label": "Pular esta sugestão",
             "description": "Não adicionar este comentário ao review"
           },
           {
             "label": "Cancelar review completo",
             "description": "Deletar pending review e sair"
           }
         ]
       }]
     }
     ```

   **C. Processar resposta do usuário**:

   **CASO 1: "Aprovar e adicionar ao review" / "Approve and add to review"**

   1. Formatar o comentário:
      ```markdown
      ### {categoria} - {severidade}

      **Sugestão** / **Suggestion**:
      ```{language}
      {codigo_sugerido ou descrição}
      ```

      **Justificativa** / **Rationale**:
      {motivo}
      ```

   2. Determinar parâmetros para add_comment_to_pending_review:
      - **SEMPRE usar `subjectType = "LINE"`** quando houver `file_path` — NUNCA usar `subjectType = "FILE"`
      - Se `file_path` está disponível mas `line_number` não foi identificado pelo code-reviewer:
        * Use a ferramenta Read para abrir o arquivo e encontrar a linha exata do código em questão
        * O parâmetro `line` refere-se à linha no arquivo modificado (versão nova, `side = "RIGHT"`)
        * Não é o número da linha no diff — é o número real da linha no arquivo
      - Se `file_path` E `line_number` estão disponíveis:
        * path = file_path
        * line = line_number
        * subjectType = "LINE"
        * side = "RIGHT" (comentário no código novo)
      - Se nenhum está disponível:
        * Comentário geral (não específico de arquivo)
        * Use apenas `body` sem `path`

   3. Chamar `mcp__plugin_github_github__add_comment_to_pending_review`:
      ```json
      {
        "owner": "{owner}",
        "repo": "{repo}",
        "pullNumber": {pull_number},
        "body": "{comentario_formatado}",
        "path": "{file_path}",
        "line": {line_number},
        "side": "RIGHT",
        "subjectType": "LINE"
      }
      ```

   4. **Error handling**: Se falhar ao adicionar comentário:
      - **English**: "⚠️ Warning: Failed to add comment: {error_message}"
      - **Portuguese**: "⚠️ Aviso: Falha ao adicionar comentário: {error_message}"
      - Use AskUserQuestion: "Do you want to continue with the next suggestions?" / "Deseja continuar com as próximas sugestões?"
      - Se não: delete pending review e saia

   5. Se sucesso:
      - Incrementar `approved_count`
      - Incrementar `severity_counts[severity]`
      - **English**: "✓ Comment added to pending review."
      - **Portuguese**: "✓ Comentário adicionado ao pending review."

   **CASO 2: "Editar comentário antes de adicionar" / "Edit comment before adding"**

   1. Solicitar texto editado:
      - Use AskUserQuestion:
        ```json
        {
          "questions": [{
            "question": "Enter the edited comment text:" / "Digite o texto do comentário editado:",
            "header": "Edit",
            "multiSelect": false,
            "options": [
              {
                "label": "Continue with original",
                "description": "Use the original comment text"
              }
            ]
          }]
        }
        ```
      - A opção "Other" permitirá que o usuário digite texto customizado

   2. Se usuário fornecer texto editado:
      - Use o texto editado como `body`
      - Chamar `add_comment_to_pending_review` com mesmo processo do CASO 1
      - Incrementar `edited_count` (não `approved_count`)

   3. Se usuário escolher "Continue with original":
      - Processar como CASO 1

   **CASO 3: "Pular esta sugestão" / "Skip this suggestion"**

   - Incrementar `skipped_count`
   - **English**: "⊘ Suggestion skipped."
   - **Portuguese**: "⊘ Sugestão pulada."
   - Continue para próxima sugestão

   **CASO 4: "Cancelar review completo" / "Cancel entire review"**

   1. Deletar pending review:
      - Use: `mcp__plugin_github_github__pull_request_review_write`
      - Parameters:
        ```json
        {
          "method": "delete_pending",
          "owner": "{owner}",
          "repo": "{repo}",
          "pullNumber": {pull_number}
        }
        ```

   2. Mostrar mensagem:
      - **English**: "🚫 Review canceled. Pending review deleted."
      - **Portuguese**: "🚫 Review cancelado. Pending review deletado."

   3. Saia do comando

4. **Rastreamento de progresso**:
   - A cada sugestão processada, atualizar `current` counter
   - Mostrar progresso no início de cada sugestão (ex: "Suggestion 2/5")

---

## PHASE 6: Submit Review

**Objetivo**: Confirmar com usuário e submeter review final

**IMPORTANTE**: Todas as mensagens desta fase devem estar no idioma escolhido pelo usuário

> **⚠️ REGRA ABSOLUTA**: Esta é a única fase onde `submit_pending` pode ser chamado.
> A `AskUserQuestion` do Step 3 NUNCA pode ser pulada ou ter sua resposta pré-definida.
> O usuário DEVE selecionar explicitamente uma das opções de submissão.

### Instruções:

1. **Mostrar separador final**:
   ```
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ```

2. **Mostrar resumo** (adaptar labels conforme idioma):

   **Se idioma = "english"**:
   ```
   📊 Review Summary:

   Suggestions analyzed: {total_suggestions}
   Comments added: {approved_count + edited_count}
   Suggestions skipped: {skipped_count}

   Breakdown by severity:
   • High: {severity_counts['High']} comments
   • Medium: {severity_counts['Medium']} comments
   • Low: {severity_counts['Low']} comments

   Files with comments: {count_unique_files}
   ```

   **Se idioma = "portuguese"**:
   ```
   📊 Resumo do Review:

   Sugestões analisadas: {total_suggestions}
   Comentários adicionados: {approved_count + edited_count}
   Sugestões puladas: {skipped_count}

   Breakdown por severidade:
   • Alto: {severity_counts['Alto']} comentários
   • Médio: {severity_counts['Médio']} comentários
   • Baixo: {severity_counts['Baixo']} comentários

   Arquivos com comentários: {count_unique_files}
   ```

3. **Solicitar confirmação de submissão** (traduzir conforme idioma):

   **Se idioma = "english"**:
   - Use AskUserQuestion:
     ```json
     {
       "questions": [{
         "question": "Do you want to submit the review now?",
         "header": "Submit",
         "multiSelect": false,
         "options": [
           {
             "label": "Yes, submit as COMMENT",
             "description": "General feedback without approval/rejection"
           },
           {
             "label": "Yes, submit as REQUEST_CHANGES",
             "description": "Request changes before merging"
           },
           {
             "label": "Yes, submit as APPROVE",
             "description": "Approve the PR for merging"
           },
           {
             "label": "No, keep as pending review",
             "description": "Submit manually later via GitHub UI"
           }
         ]
       }]
     }
     ```

   **Se idioma = "portuguese"**:
   - Use AskUserQuestion:
     ```json
     {
       "questions": [{
         "question": "Deseja submeter o review agora?",
         "header": "Submeter",
         "multiSelect": false,
         "options": [
           {
             "label": "Sim, submeter como COMMENT",
             "description": "Comentários gerais sem aprovação/rejeição"
           },
           {
             "label": "Sim, submeter como REQUEST_CHANGES",
             "description": "Solicitar mudanças antes de merge"
           },
           {
             "label": "Sim, submeter como APPROVE",
             "description": "Aprovar a PR para merge"
           },
           {
             "label": "Não, manter como pending review",
             "description": "Submeter manualmente depois via GitHub UI"
           }
         ]
       }]
     }
     ```

4. **Processar escolha**:

   **CASO: Opção 1, 2 ou 3 (Submeter)**

   1. Determinar event type:
      - Opção 1: event = "COMMENT"
      - Opção 2: event = "REQUEST_CHANGES"
      - Opção 3: event = "APPROVE"

   2. Preparar body do review:
      - **English**:
        ```
        I've reviewed the changes and left some comments inline with suggestions for improvement.
        ```
      - **Portuguese**:
        ```
        Revisei as mudanças e deixei alguns comentários inline com sugestões de melhoria.
        ```

   3. Submeter review:
      - Use: `mcp__plugin_github_github__pull_request_review_write`
      - Parameters:
        ```json
        {
          "method": "submit_pending",
          "owner": "{owner}",
          "repo": "{repo}",
          "pullNumber": {pull_number},
          "event": "{event_type}",
          "body": "{review_body}"
        }
        ```

   4. Se sucesso:
      - **English**: "✓ Review submitted successfully!"
      - **Portuguese**: "✓ Review submetido com sucesso!"
      - Continue para mensagem final

   5. **Error handling**: Se falhar:
      - **English**: "❌ Error submitting review: {error_message}"
      - **Portuguese**: "❌ Erro ao submeter review: {error_message}"
      - Mostre: "The review is still pending. You can submit it manually via GitHub UI."
      - Mostre URL da PR

   **CASO: Opção 4 (Manter como pending)**

   - **English**:
     ```
     ℹ️ Review kept as pending. You can:
     - Submit manually via GitHub UI
     - Add more comments manually
     - Delete the review if needed

     PR URL: https://github.com/{owner}/{repo}/pull/{pull_number}
     ```
   - **Portuguese**:
     ```
     ℹ️ Review mantido como pending. Você pode:
     - Submeter manualmente via GitHub UI
     - Adicionar mais comentários manualmente
     - Deletar o review se necessário

     PR URL: https://github.com/{owner}/{repo}/pull/{pull_number}
     ```
   - Saia do comando (não mostrar mensagem final)

5. **Mensagem final de sucesso** (se review foi submetido):

   **Se idioma = "english"**:
   ```
   ✅ Review submitted successfully!

   PR: https://github.com/{owner}/{repo}/pull/{pull_number}
   ```

   **Se idioma = "portuguese"**:
   ```
   ✅ Review submetido com sucesso!

   PR: https://github.com/{owner}/{repo}/pull/{pull_number}
   ```

---

## Error Handling Summary

### Erros Críticos (Parar Execução):

1. **PR não encontrada** (404):
   - **English**: "❌ Error: PR #{pull_number} not found in repo {owner}/{repo}"
   - **Portuguese**: "❌ Erro: PR #{pull_number} não encontrada no repo {owner}/{repo}"
   - Saia do comando

2. **Sem permissão** (403):
   - **English**: "❌ Error: No permission to access this PR. Check your GitHub credentials."
   - **Portuguese**: "❌ Erro: Sem permissão para acessar esta PR. Verifique suas credenciais do GitHub."
   - Saia do comando

3. **Falha ao criar pending review**:
   - **English**: "❌ Error creating pending review on GitHub: {error_message}"
   - **Portuguese**: "❌ Erro ao criar pending review no GitHub: {error_message}"
   - Saia do comando

4. **Repository information missing**:
   - **English**: "❌ Error: Could not determine repository information. Please provide PR URL or run from a git repository."
   - **Portuguese**: "❌ Erro: Não foi possível determinar as informações do repositório. Forneça a URL da PR ou execute de um repositório git."
   - Saia do comando

### Erros Recuperáveis (Continuar com Avisos):

1. **Falha ao adicionar comentário individual**:
   - **English**: "⚠️ Warning: Failed to add comment: {error_message}"
   - **Portuguese**: "⚠️ Aviso: Falha ao adicionar comentário: {error_message}"
   - Perguntar se deseja continuar

2. **Projeto relacionado não encontrado**:
   - **English**: "⚠️ Warning: Could not access related project {project_name}"
   - **Portuguese**: "⚠️ Aviso: Não foi possível acessar projeto relacionado {project_name}"
   - Continue sem contexto adicional

3. **Code reviewer não retorna sugestões**:
   - **English**: "ℹ️ The code-reviewer did not find improvement suggestions in this PR."
   - **Portuguese**: "ℹ️ O code-reviewer não encontrou sugestões de melhoria nesta PR."
   - Perguntar se deseja criar review vazio com APPROVE

4. **Falha ao submeter review**:
   - **English**: "❌ Error submitting review: {error_message}. The review is still pending."
   - **Portuguese**: "❌ Erro ao submeter review: {error_message}. O review ainda está pending."
   - Mostrar URL da PR para submissão manual

### Edge Cases:

1. **PR muito grande** (>100 arquivos):
   - **English**: "⚠️ Warning: This PR has {count} modified files. Review may take a while."
   - **Portuguese**: "⚠️ Aviso: Esta PR tem {count} arquivos modificados. A revisão pode demorar."
   - Perguntar se deseja continuar

2. **Diff muito grande**:
   - Se diff > 1MB: truncar e avisar
   - **English**: "⚠️ Warning: Diff truncated due to size. Review may not be complete."
   - **Portuguese**: "⚠️ Aviso: Diff truncado devido ao tamanho. Revisão pode não ser completa."

3. **Code reviewer demora**:
   - **English**: "🔍 Analysis in progress... The code-reviewer is analyzing {count} modified files."
   - **Portuguese**: "🔍 Análise em progresso... O code-reviewer está analisando {count} arquivos modificados."

---

## Princípios de Design (KISS)

Este comando segue os princípios KISS:

1. **Solução direta**: Usa GitHub MCP tools nativos, sem abstrações desnecessárias
2. **Workflow linear**: 6 fases bem definidas, uma após a outra
3. **Interatividade simples**: AskUserQuestion com opções claras
4. **Sem cache ou otimizações prematuras**: Implementa apenas o necessário
5. **Error handling básico mas eficaz**: Trata erros críticos, avisa sobre recuperáveis
6. **Sem features especulativas**: Não implementa batch approval, auto-fix, templates customizados, etc.
7. **Submissão somente com aprovação explícita**: `submit_pending` é chamado EXCLUSIVAMENTE em PHASE 6, SEMPRE após `AskUserQuestion`. Nenhum fluxo bypassa esta confirmação.

**Features NÃO implementadas intencionalmente** (podem ser adicionadas DEPOIS se necessário):
- Batch approval de múltiplas sugestões
- Auto-fix de sugestões simples
- Templates customizados de comentários
- Cache de análises anteriores
- Integração com CI/CD
- Webhooks ou notificações
- Dashboard ou relatórios
- Métricas históricas

---

## Notas de Implementação

1. **Parsing de sugestões do code-reviewer**:
   - O agente code-reviewer retorna texto markdown livre
   - Use heurísticas para identificar blocos de sugestões:
     * Linhas começando com "###" ou "**" podem indicar início de sugestão
     * Blocos de código cercados por ``` são código
     * Palavras-chave como "Category", "Severity", "File", "Suggestion", "Rationale"
   - Se parsing falhar, trate toda a resposta como uma única sugestão geral

2. **File path e line number**:
   - Extrair do diff quando possível
   - Se code-reviewer mencionar "file.go:123", parse isso
   - **NUNCA usar `subjectType: "FILE"`** — sempre usar `subjectType: "LINE"` com número de linha real
   - Se o code-reviewer não informar a linha, usar a ferramenta Read para abrir o arquivo e encontrar a linha exata do trecho problemático antes de adicionar o comentário
   - O parâmetro `line` da API do GitHub é o número da linha no arquivo modificado (não no diff)
   - Se não for possível identificar arquivo nem linha, o comentário fica sem `path` (comentário geral na PR)

3. **Idioma do code-reviewer**:
   - O prompt enviado ao code-reviewer DEVE incluir instrução explícita de idioma
   - Exemplo: "**IDIOMA DO FEEDBACK**: portuguese - Forneça TODO o feedback em português"
   - Isso garante que sugestões, categorias e severidades estejam no idioma correto

4. **Related files limit**:
   - Limitar a 3 arquivos relacionados por endpoint detectado
   - Limitar snippet a primeiras 50 linhas de cada arquivo
   - Isso evita contexto excessivo para o code-reviewer

5. **Contadores de severidade**:
   - Mapear severidades em inglês e português:
     * "High" / "Alto"
     * "Medium" / "Médio"
     * "Low" / "Baixo"
   - Use normalização case-insensitive ao incrementar contadores
