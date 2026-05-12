---
name: reply-review
description: Compose a grounded reply to a PR request-change comment, mirroring the original language and tone. Use when the user pastes review feedback or selects it in the IDE and wants a response backed by the actual codebase.
argument-hint: [texto do request-change | vazio para input interativo]
allowed-tools: Read Grep Glob Bash(git log:*) Bash(git blame:*) Bash(git show:*) Bash(git diff:*)
disable-model-invocation: true
---

# Reply to PR Request-Change

Você está respondendo a um comentário de "request change" de Pull Request. A saída final é o **texto da resposta pronto para ser colado no PR** — sem cabeçalhos, sem rótulos meta, sem narrativa de bastidores.

## Texto do request-change

$ARGUMENTS

---

## Instruções

### 1. Validar entrada

Se a seção acima estiver vazia (sem `$ARGUMENTS`), peça ao usuário que cole o texto do request-change ou selecione o trecho na IDE e reinvoque. **Não invente** o conteúdo do comentário nem assuma do contexto anterior.

### 2. Detectar idioma e tom

- **Idioma**: identifique o idioma predominante do texto (pt, en, es, etc.). Sua resposta DEVE estar no mesmo idioma — sem traduzir, sem misturar.
- **Tom**: classifique o tom do revisor (técnico/formal, direto/seco, didático, informal, conciliador, áspero, irônico, etc.). Espelhe esse tom na resposta — sem inflar formalidade, sem amenizar críticas, sem reproduzir hostilidade. Se o tom é áspero, mantenha-se profissional e direto sem revidar.

### 3. Mapear referências de código

Extraia todas as pistas que apontam para o codebase:

- **Diretas**: caminhos de arquivo, nomes de funções/classes/métodos, símbolos, números de linha, IDs de commit, nomes de variáveis ou constantes.
- **Indiretas**: descrições de comportamento, conceitos de domínio, nomes de feature/rota, padrões arquiteturais que possam ser localizados via busca.

Localize os trechos usando as ferramentas disponíveis:

- `Glob` para encontrar arquivos por padrão.
- `Grep` para buscar símbolos, strings ou trechos.
- `Read` para inspecionar o código encontrado.
- `git log` / `git blame` / `git show` / `git diff` quando precisar de contexto histórico, autoria ou diff.

**Nunca chute caminhos ou trechos.** Confirme antes de citar. Se uma referência não puder ser localizada no repositório, declare isso explicitamente na resposta em vez de improvisar.

### 4. Avaliar o mérito do pedido

Antes de redigir, classifique internamente o pedido em uma das categorias abaixo. Não escreva o rótulo na resposta — ele só guia a estrutura.

1. **Procedente**: o revisor tem razão. Concorde, descreva brevemente o que será alterado e cite onde.
2. **Procedente parcialmente**: parte é válida, parte não. Reconheça o que faz sentido, contraponha o resto com evidência do código.
3. **Improcedente**: o código já cumpre o pedido, ou ele contradiz uma restrição/decisão do projeto. Esclareça com referência direta ao código, sem defensiva.
4. **Ambíguo**: não há informação suficiente. Peça esclarecimento específico, sugerindo o que falta.

Seja honesto: não concorde por gentileza nem rejeite por orgulho. Cada afirmação se apoia em código real.

### 5. Compor a resposta

Estrutura sugerida (adapte ao tamanho do comentário original):

1. Uma linha reconhecendo o ponto levantado, no tom do revisor.
2. Posicionamento ancorado em código (concordar / contrapor / esclarecer / pedir mais contexto).
3. Referências concretas no formato `caminho/do/arquivo.ext:linha` quando aplicável.
4. Se houver mudança a ser feita: plano curto. Se não houver: justificativa baseada no código.
5. Fechamento compatível com o tom (sem floreio em tom direto, sem secura em tom cordial).

**Restrições rígidas:**

- **Idioma**: igual ao do input.
- **Tom**: espelhado, sem amplificar nem suavizar artificialmente.
- **Tamanho**: proporcional ao comentário original. Comentário curto → resposta curta. Comentário longo → resposta detalhada.
- **Verificação**: toda referência a código precisa ter sido confirmada nesta sessão. Se não confirmou, não cite.
- **Sem meta**: não escreva "Resposta:", não explique seu raciocínio na saída final, não comente sobre a skill.

### Saída

Apenas o texto da resposta, pronto para colar no PR. Se precisar pedir esclarecimento ao usuário (input vazio, referência não localizável), entregue essa pergunta de forma clara e **separada** do texto-resposta — deixe explícito que é uma pergunta para o usuário, não para o revisor.
