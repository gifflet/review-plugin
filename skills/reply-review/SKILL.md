---
name: reply-review
description: Compose a one-sentence, natural reply to a PR request-change comment, mirroring the reviewer's language and tone. Use when the user pastes review feedback or selects it in the IDE and wants a grounded, human-sounding response ready to paste on GitHub.
argument-hint: [texto do request-change | vazio para input interativo]
allowed-tools: Read Grep Glob Bash(git log:*) Bash(git blame:*) Bash(git show:*) Bash(git diff:*)
disable-model-invocation: true
---

# Reply to PR Request-Change

Sua entrega é **uma única frase** pronta para colar em um comentário de Pull Request no GitHub — natural, conversacional, ancorada no código. Nada mais.

## Texto do request-change

$ARGUMENTS

---

## Instruções

### 1. Validar entrada

Se `$ARGUMENTS` estiver vazio, peça ao usuário o texto do request-change (ou a seleção na IDE). **Não invente** o conteúdo.

### 2. Detectar idioma e tom

- **Idioma**: responda no mesmo idioma do comentário (pt, en, es, etc.). Sem traduzir, sem misturar.
- **Tom**: espelhe o tom do revisor (técnico, direto, didático, informal, áspero, etc.) sem amplificar, suavizar ou revidar.

### 3. Mapear referências de código

Localize no repositório qualquer pista — caminhos, símbolos, funções, comportamentos descritos — usando `Glob`, `Grep`, `Read` e `git log` / `git blame` / `git show` / `git diff`. **Nunca chute.** Se não conseguir confirmar uma referência, omita-a ou diga isso na frase de forma natural.

### 4. Decidir a postura

Classifique internamente (não escreva o rótulo):

1. **Procedente** — concorde e diga o que vai mudar.
2. **Procedente parcialmente** — reconheça o que faz sentido, contraponha o resto com evidência.
3. **Improcedente** — esclareça com referência direta ao código, sem defensiva.
4. **Ambíguo** — peça um esclarecimento específico.

Honestidade: nem concordar por gentileza, nem rejeitar por orgulho.

### 5. Escrever a frase

**Regra-mestra: uma única sentença.** A saída termina em **exatamente um** sinal de fim de sentença (`.`, `?` ou `!`). Pontos em caminhos (`src/foo.ts`) ou versões (`1.2.3`) não contam. Quebras de linha, parágrafos, listas, bullets, headers, blocos de código multi-linha e múltiplas sentenças estão **proibidos**.

Para encadear ideias, use `,`, `—`, `;`, `:` ou parênteses. Mesmo quando o revisor levanta vários pontos, condense tudo em uma só frase (separe cláusulas com `;` ou `—`).

**Estilo:**

- Conversacional, como uma resposta no Slack para um colega — contrações, voz ativa, direto.
- Sem cerimônia: nada de "Olá!", "Obrigado pelo feedback", "Resumo:", "Conclusão:", "Qualquer dúvida me avise".
- Sem clichês de LLM: "Excelente ponto!", "Você está absolutamente correto", "Aprecio sua observação", "Vamos abordar isso", "É importante notar que".
- Variação no começo da frase — não sempre "Faz sentido,...".
- Referências de código no fluxo da frase: "isso já é tratado em `src/foo.ts:42`", não "Conforme verificado em `src/foo.ts:42`...".
- Markdown permitido: apenas `código inline` para símbolos e `caminho/arquivo.ext:linha` para localização. Sem negrito, sem títulos, sem blocos.
- Calibração: se vai mudar, diga o que muda; se não vai, diga o porquê com o ponteiro pro código. Sem hedging ("talvez possa ser que..."), sem arrogância.

**Antes de devolver, conte os pontos finais.** Se houver mais de um, reescreva juntando as ideias em uma só frase.

### Exemplos

> Faz sentido — troco pelo `lodash.debounce` que já é usado em `src/hooks/useSearch.ts:18` e empurro no próximo commit.

> Esse caminho já passa pelo `validateInput` em `src/api/handlers.ts:54` antes de chegar no banco, então o cast direto é seguro aqui.

> Concordo em extrair pra um helper no primeiro ponto, mas no segundo o `useEffect` precisa rodar depois do mount por causa do `ref` em `Modal.tsx:30`, então mantenho como está.

> Pode esclarecer se a validação deve falhar logo na entrada ou só no momento do submit? — o fluxo atual em `forms/validate.ts:22` faz no submit, e quero confirmar antes de mexer.

Cada exemplo é **uma frase**, mesmo cobrindo múltiplos pontos.

### Saída

Apenas a frase, pronta para colar no PR. Se precisar pedir esclarecimento ao usuário (input vazio, referência não localizável), entregue essa pergunta **separada** do texto-resposta e deixe explícito que é uma pergunta para o usuário, não para o revisor.
