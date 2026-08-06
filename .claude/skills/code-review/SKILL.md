---
name: code-review
description: Revisa mudanças de código comparando o branch atual com a main/master, seja antes de abrir um Pull Request ou ao revisar o PR de um colega. Use sempre que o usuário pedir para "revisar", "dar uma olhada no PR", "revisar antes de abrir PR", "code review", ou mencionar comparar branch com main/master antes de subir código. Foca só no que mudou (diff), separa bugs reais de preferência de estilo, cita arquivo e linha em cada achado, e reporta no fim o que foi de fato executado (comandos) vs. apenas lido.
---

# Code Review

Revisão de código focada no diff entre o branch atual e a main, aplicável a
qualquer linguagem. O objetivo é achar problemas reais — coisas que quebram,
comportam-se errado, ou têm risco de segurança/dado — sem virar bikeshedding
de estilo.

## Quando usar

- Antes de abrir um PR (usuário quer revisar o próprio branch).
- Ao revisar o PR de outra pessoa (branch de um colega, ou um PR já aberto no GitHub).

## Regras inegociáveis

1. **Revisar só o diff.** Nunca opinar sobre código que não mudou nesta
   branch/PR, mesmo que pareça ruim. Se algo fora do diff for relevante para
   entender um bug (ex.: uma função chamada pelo código novo), pode ser lido
   como contexto, mas não vira um "achado" — só serve para embasar um achado
   dentro do diff.
2. **Todo achado tem arquivo + linha + como reproduzir.** Nunca reportar "há
   um problema de validação nesse módulo" sem apontar `arquivo:linha` e um
   passo a passo (input, chamada, ou cenário) que mostra o problema
   acontecendo. Se não for possível descrever como reproduzir, não é um
   achado — no máximo uma pergunta de esclarecimento.
3. **Separar bugs de preferência, sempre em seções distintas.** Nunca
   misturar as duas coisas no mesmo item. Ver critério de classificação
   abaixo.
4. **Nunca reclamar de formatação.** Espaçamento, ponto e vírgula, ordem de
   imports, chaves na mesma linha ou não — isso é trabalho de linter/formatter
   automático, não de revisão humana. Ignorar completamente, mesmo que o
   próprio diff mude só formatação.
5. **Nunca sugerir refatoração sem dizer que problema concreto ela resolve.**
   "Isso poderia ser mais limpo" não é uma sugestão válida. Se não há um bug,
   risco, ou dor de manutenção específica e nomeável, não sugerir mudança de
   estrutura.
6. **Rodar comandos (testes, build, lint) só se o usuário pedir
   explicitamente.** Por padrão, a revisão é estática (leitura do diff e do
   código ao redor). Se o usuário disser algo como "roda os testes também" ou
   "confere se o lint passa", aí sim executar. Nunca assumir que pode
   executar side effects (ex.: `git push`, instalar dependências) sem pedir.
7. **No fim, declarar o que foi verificado executando comando vs. o que foi
   só lido.** Isso é obrigatório mesmo quando nada foi executado — nesse caso
   a seção diz explicitamente "nenhum comando executado, revisão só de
   leitura".

## Passo a passo

### 1. Descobrir o branch/PR e o diff

Primeiro, entender o contexto:

- Se o usuário já disse qual é o branch de comparação (ex.: "compara com
  `develop`"), usar esse.
- Senão, assumir `main` e cair para `master` se `main` não existir.
- Se o usuário mencionou um PR do GitHub (número ou link) e o `gh` CLI está
  disponível, usar `gh pr diff <numero>` ou `gh pr view` para pegar o diff e
  metadados do PR. Se `gh` não estiver disponível ou autenticado, cair para
  git local.
- Caso contrário (branch local), usar git puro:

```bash
# nome do branch de base (main ou master)
git rev-parse --verify main 2>/dev/null && BASE=main || BASE=master

# ponto comum entre o branch atual e a base
MERGE_BASE=$(git merge-base $BASE HEAD)

# lista de arquivos alterados
git diff --name-status $MERGE_BASE HEAD

# diff completo com contexto e números de linha
git diff $MERGE_BASE HEAD
```

Usar `git diff`, não `git diff main...HEAD` sem cuidado — o `merge-base`
explícito evita incluir mudanças que já estão na main mas ainda não chegaram
no branch local desatualizado.

Se o repositório tiver múltiplos módulos/linguagens, não se preocupar em
tratar cada um diferente nessa etapa — o diff já isola naturalmente os
arquivos tocados.

### 2. Ler cada arquivo alterado com contexto suficiente

O diff sozinho mostra as linhas mudadas, mas muitas vezes falta contexto para
julgar se algo é um bug real. Para cada arquivo no diff:

- Ler o arquivo completo (ou a função/classe inteira ao redor do trecho
  alterado) via `view`, não confiar só nas poucas linhas de contexto do diff.
- Se a mudança chama ou depende de outra função/módulo não alterado nesse
  diff, ler essa dependência também — só para entender o comportamento, sem
  comentar sobre ela como achado (regra 1).

### 3. Classificar cada achado: Bug ou Preferência

Usar este critério, não instinto:

**É Bug (quebra o programa)** se pelo menos um destes for verdade:
- Muda o comportamento observável de forma incorreta (retorno errado,
  exceção não tratada, estado inconsistente).
- Introduz ou piora uma falha de segurança (injeção, dado sensível exposto,
  falta de validação de entrada não confiável).
- Causa perda ou corrupção de dado.
- Quebra um contrato existente (assinatura de API, formato de resposta,
  tipo) sem atualizar quem consome.
- Condição de corrida, vazamento de recurso (memória, conexão, arquivo não
  fechado), ou loop infinito/travamento sob certas entradas.
- Edge case não tratado que vai ocorrer em uso normal (não hipotético
  extremo) — ex.: lista vazia, null/None, divisão por zero, índice fora do
  limite.

**É Preferência (não é achado obrigatório, mas pode ser mencionado à parte)**
se:
- Funciona corretamente, mas há uma forma mais idiomática/clara na
  linguagem do projeto.
- Nome de variável/função poderia ser mais descritivo, sem gerar confusão
  real.
- Duplicação de lógica que não causa bug hoje, mas custa manutenção
  amanhã — só entra aqui se vier acompanhada do problema concreto que
  resolve (regra 5); senão, nem menciona.

Nunca colocar os dois tipos na mesma lista ou no mesmo item. Usar seções
separadas na saída (ver formato abaixo).

### 4. (Opcional, só se pedido) Rodar verificações

Só executar se o usuário pedir explicitamente:

- Testes: detectar o comando pelo ecossistema do projeto (`package.json` →
  `npm test`; `pytest.ini`/`pyproject.toml` → `pytest`; `go.mod` → `go test
  ./...`; etc.) e rodar só nos módulos afetados pelo diff quando for viável
  isolar.
- Lint/typecheck: mesma lógica, comando específico do projeto.

Nunca rodar comandos que alterem o estado do repositório remoto (push, PR
merge, etc.) — a skill é só de leitura/verificação.

### 5. Montar a saída

Formato obrigatório (Markdown):

```markdown
## Revisão: <branch atual> → <base>

### 🐞 Bugs (quebram o programa)
1. **`caminho/arquivo.ext:42`** — <descrição objetiva do problema>
   **Como reproduzir:** <passo a passo ou cenário de input que dispara o problema>

(se não houver nenhum: "Nenhum bug encontrado no diff revisado.")

### 💭 Preferência (não bloqueante)
1. **`caminho/arquivo.ext:17`** — <sugestão> — **resolve:** <problema concreto que essa mudança evita>

(se não houver nenhum, omitir a seção ou dizer "Nada além de preferência pessoal a registrar.")

### ✅ O que foi verificado
- **Executado:** <lista de comandos rodados, ex.: `npm test -- src/foo.test.js`> (ou "nenhum comando executado")
- **Só lido:** <lista de arquivos/trechos lidos estaticamente para embasar a revisão>
```

Regras de conteúdo dessa saída:
- Todo item de Bug tem link `arquivo:linha` real (confirmado no diff/arquivo,
  nunca inventado) e reprodução concreta.
- Nunca incluir item de formatação em nenhuma seção.
- Se o diff inteiro estiver limpo, dizer isso claramente, sem forçar achados
  para preencher a resposta.
- A seção "O que foi verificado" é sempre a última e sempre presente, mesmo
  numa revisão 100% estática.

## Erros comuns a evitar

- Comentar sobre código fora do diff só porque "already tava ruim antes".
- Misturar bug com preferência num único bullet ("isso está errado e
  também poderia ser mais elegante").
- Apontar um "possível problema" sem conseguir descrever como reproduzir —
  nesse caso, formular como pergunta ao autor, não como achado.
- Sugerir troca de biblioteca/arquitetura sem um bug ou risco nomeado que
  justifique.
- Rodar testes/lint sem o usuário ter pedido.
- Esquecer a seção final de "o que foi verificado".