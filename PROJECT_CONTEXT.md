# AdTurbo Refactor Engine — PROJECT_CONTEXT.md

**Versão:** 0.12 (atualizar ao final de cada sessão)
**Última atualização:** 2026-07-04
**SPEC.md:** contrato técnico imutável da arquitetura — este arquivo não o substitui.
**Propósito deste arquivo:** memória do estado real do projeto para permitir que uma
nova sessão continue exatamente de onde a anterior parou, sem precisar reler toda a
conversa.

---

## 1. Contexto e origem do projeto

A Engine surgiu de um incidente real: um script Bash (`adturbo-fix-auth.sh`) usou regex
para remover funções duplicadas de routes Next.js no projeto AdTurbo e corrompeu
`app/api/criativos/route.js`, deixando um `);` solto que quebrou o build do Next.js.

A partir desse incidente, foi acordado construir uma engine de refatoração profissional,
auditável, transacional e segura — em vez de continuar com scripts Bash ad-hoc.

O AdTurbo é um projeto Next.js (App Router) rodando em Termux (Android ARM64), com
um bot WhatsApp integrado. O ambiente tem limitações conhecidas: sem `/tmp` garantido,
`next build` instável por limitações de wasm no ARM64.

---

## 2. Objetivos da Engine

- **Segura:** nunca deixa o projeto em estado pior do que encontrou.
- **Auditável:** toda execução é rastreável meses depois via artefatos em `.adturbo-engine/runs/`.
- **Idempotente:** rodar o mesmo patch N vezes produz o mesmo resultado da primeira vez.
- **Conservadora:** na dúvida, para e pede intervenção humana. Nunca "adivinha".
- **Incremental:** cada versão entregue é utilizável sozinha, mesmo com escopo reduzido.

**Princípio fundamental (inegociável):**
> A Engine pode diagnosticar, validar e recomendar ações baseadas em evidências
> objetivas. Ela nunca deve tomar decisões baseadas em heurísticas quando houver
> mais de uma interpretação possível do estado do projeto.

---

## 3. Arquitetura

### Três camadas

```
Camada 1 — Bash Engine
  Orquestração pura. Nunca edita JS. Nunca conhece regras do AdTurbo.
  Responsabilidades: inventário, backup, rollback, ambiente, relatórios,
  invocar a Engine Node, interpretar resultado.

Camada 2 — Node Engine
  Transformações estruturais via AST (Babel/Recast). Uma responsabilidade
  por transformador. Cada transformador: verifica pré-condições, verifica
  idempotência, transforma, valida, retorna relatório estruturado.

Camada 3 — Biblioteca de transformadores
  Composições de transformadores já individualmente validados ("recipes").
  Só existe depois que os transformadores usados passaram por validação.
```

### Onde fica o código

```
adturbo-engine/           — raiz da Engine (separada do projeto AdTurbo)
  src/
    config/
      constants.js        — ✅ APROVADO. Única fonte de verdade para
                             ENGINE_DIR_NAME, RUNS_DIR_NAME, LOCK_FILE_NAME,
                             TMP_DIR_NAME, BACKUP_DIR_NAME,
                             ARTIFACT_FILE_NAMES, RUN_ID_PATTERN,
                             RUN_SEQUENCE_MIN/MAX — o contrato exato já
                             consumido por paths.js. DT-006 resolvida (ver
                             seção 15/16).
      constants.draft.js  — ⚠️ RASCUNHO TEMPORÁRIO, não arquitetura definitiva.
                             Contém KNOWN_BINARIES (environment.js) e
                             CATEGORIES/SUBSYSTEMS/RISK_LEVELS
                             (classifier.js). Migração para constants.js
                             segue pendente — ver DT-007 (deliberadamente
                             não resolvida nesta sessão; escopo distinto
                             de DT-006).
    modules/
      paths.js            — ✅ APROVADO
      hash.js             — ✅ APROVADO
      logger.js           — ✅ APROVADO
      environment.js       — ✅ APROVADO
      classifier.js        — ✅ APROVADO (ver seção 13/14)
      inventory.js          — ✅ APROVADO (ver seção 17)
      fingerprint.js        — ✅ APROVADO (ver seção 18)
      fingerprint-worker.js — ✅ APROVADO (script interno, ver seção 18)
      manifest.js          — não iniciado
      plan.js               — não iniciado
      report.js             — não iniciado
      lock.js                — não iniciado
      fs-helpers.js          — não iniciado
      analyze.js             — não iniciado
      index.js                — não iniciado
      __tests__/
        paths.test.js     — ✅ 30/30 passando (destravado — DT-006
                               resolvida, ver seção 16)
        hash.test.js      — ✅ 15/15 passando
        logger.test.js    — ✅ 25/25 passando
        environment.test.js — ✅ 27/27 passando (timeout testado com
                               fixture Node puro, sem dependência de
                               `sleep`; import de constants.draft.js
                               corrigido nesta sessão, ver seção 16)
        inventory.test.js   — ✅ 25/25 passando (ver seção 17)
        fingerprint.test.js — ✅ 23/23 passando (ver seção 18)
```

### Onde ficam os artefatos gerados (em runtime)

```
<PROJECT_ROOT>/
  .adturbo-engine/
    engine.lock           — lockfile de execução concorrente
    tmp/                  — temporários (fallback quando /tmp indisponível)
    runs/
      YYYYMMDD-NNN/
        manifest.json
        inventory.json
        environment.json
        hashes.json
        fingerprint.json
        plan.json
        report.md
        rollback.json
        backup/           — cópias dos arquivos-alvo pré-alteração
        full-backup.tar.gz  — snapshot completo (só quando --full-backup)
```

---

## 4. Regras que nunca podem ser quebradas

1. **Engine Bash é agnóstica de domínio.** Não pode conter nenhuma string,
   caminho, nome de função ou regra de negócio do AdTurbo. Tudo isso fica
   nos transformadores (Camada 2).

2. **Nenhuma transformação estrutural em JS/TS pode usar regex ou substituição
   de texto para localizar código.** Toda modificação estrutural ocorre via AST.
   Se a AST não puder ser construída, a transformação falha explicitamente e
   solicita intervenção humana.

3. **Nenhum módulo imprime no console exceto `logger.js` e `index.js`.**
   Todos os outros módulos retornam objetos estruturados ou lançam erros.

4. **Arquivos classificados como `secret` nunca têm seu conteúdo aberto nem
   recebem hash.** A Engine apenas registra que existem.

5. **Fail-fast por padrão:** se qualquer arquivo-alvo já tem sintaxe inválida
   na verificação inicial, a Engine aborta sem modificar nada. O modo
   `--continue-on-invalid` é opt-in explícito a cada invocação.

6. **Um transformador = uma responsabilidade.** Transformadores não fazem
   operações implícitas. Se precisam de outro transformador, declaram a
   dependência explicitamente.

7. **Toda run gera artefatos completos em `.adturbo-engine/runs/<run-id>/`.**
   Não existe execução sem rastro auditável.

8. **`paths.js` é dependência estável.** Qualquer mudança nele é tratada como
   revisão de arquitetura, não alteração incidental.

---

## 5. Decisões técnicas já tomadas e seus motivos

| Decisão | Motivo |
|---|---|
| `PROJECT_ROOT` é sempre argumento explícito, nunca `process.cwd()` | Evita dependência de estado do processo; módulos são testáveis de forma isolada |
| `paths.js` não conhece `PROJECT_ROOT` da Engine em si | Engine pode estar instalada em qualquer lugar sem alterar paths.js |
| `hash.js` não conhece `PROJECT_ROOT` | Primitiva pura; quem chama passa caminho já validado via `resolveWithinProject` |
| Erros customizados com `code` estável (`PathsError`, `HashError`, `LoggerError`) | Permite tratamento programático sem parsing de string de mensagem |
| `validateProjectRoot` → `assertProjectRootExists` (lança em vez de retornar `{valid,reason}`) | API de erros unificada: sucesso silencioso ou exceção, nunca dois estilos |
| `constants.js` centraliza todos os nomes de artefatos | Uma mudança de nome de arquivo = uma linha alterada em um lugar |
| `isValidRunId` (predicado booleano) + `assertValidRunId` (lança) coexistem | Predicado para quem precisa checar sem try/catch; assert para pontos onde o run-id inválido é sempre erro |
| Hash SHA-256 obrigatório, streaming com `READ_CHUNK_SIZE = 64KB` | Memória constante independente do tamanho do arquivo |
| Erros de I/O mapeados: ENOENT→FILE_NOT_FOUND, EACCES/EPERM→FILE_NOT_READABLE, EISDIR→PATH_IS_DIRECTORY, demais→IO_ERROR com `cause` | Apenas erros com semântica acionável recebem código específico; `cause` preserva o original |
| Backup híbrido: só arquivos-alvo por padrão; `--full-backup` sob demanda ou quando >N arquivos modificados | Velocidade no dia a dia, snapshot completo quando realmente necessário |
| `--recover` é apenas diagnóstico, não toma decisões automáticas | Princípio fundamental: Engine não decide diante de ambiguidade |
| Fail-fast (abort total) quando arquivo-alvo já tem sintaxe inválida | Projeto em estado desconhecido; aplicar mudanças parciais piora o diagnóstico |
| Rollback granular durante transformação (não abort total) | A falha é da transformação que acabamos de tentar; sabemos a causa; abort total seria excesso |
| `next build` é validador "Importante" (avisa), não "Crítico" (bloqueia) | No Termux ARM64, build falha por razões de ambiente não relacionadas ao patch |
| Casos de teste são parte obrigatória do contrato de cada transformador | Sem testes = não pode ser referenciado em plano de transformação real |
| Compatibilidade do transformador: `engineVersion` (semver range, obrigatório), `runtime` e `target` (informativos, futuros) | `engineVersion` é verificável automaticamente; os demais crescem sem quebrar versões anteriores |
| Run-id formato `YYYYMMDD-NNN`: paths.js sabe gerar, não decide o próximo NNN | Decidir o próximo NNN requer ler o disco — responsabilidade de manifest.js |
| `logger.js` é factory (`createLogger`), não singleton global | Sem estado module-level; múltiplas instâncias configuradas de forma independente (ex.: logger silencioso em teste vs. verboso na CLI); consistente com "nunca dependência implícita de estado do processo" |
| Filtragem de nível por comparação numérica direta, sem buffer/fila | Fase 1 é console-only; buffer/assincronia seriam complexidade sem caso de uso real |
| ERROR/WARN → stderr (`console.error`/`console.warn`); DEBUG/INFO → stdout (`console.log`) | Convenção Unix; permite à Camada 1 redirecionar stderr separadamente sem a engine saber disso |
| API do logger nasce mínima: só `debug/info/warn/error` — sem `isLevelEnabled`, sem output em arquivo, sem formato JSON | Nenhum desses tem consumidor real hoje; API cresce só quando houver caso de uso concreto (ajuste arquitetural aprovado nesta sessão) |
| `clock` injetável em `createLogger` (usado só quando `withTimestamp: true`) | Mantém a API determinística e testável sem mockar `Date` globalmente |
| Log em si (`logger.info(...)` etc.) nunca lança; só `createLogger(...)` com config inválida lança `LoggerError` | Infraestrutura transversal usada por quase todo módulo não pode virar novo ponto de falha em uso normal; erro de configuração deve falhar alto e cedo |
| `environment.js` detecta binários via `spawnSync(name, [flag])` real, não via `which`/`command -v` nem parsing manual de `PATH` | `which` é ele mesmo um binário externo não garantido (Termux/BusyBox minimalista); `spawnSync` é Node core puro; `ENOENT` no resultado é sinal confiável de ausência |
| `environment.js` trata qualquer resposta do processo (mesmo exit code ≠ 0) como "binário existe"; só `ENOENT`/timeout são "ausente" | Binários minimalistas podem não suportar a flag de versão testada e retornar erro sem estarem ausentes |
| `environment.js` nunca lança por binário ausente; só por parâmetros de entrada inválidos (`EnvironmentError`) | Binário ausente é resultado de domínio válido e esperado, não erro de uso da API — mesmo padrão de hash.js/paths.js/logger.js |
| `environment.js` não decide fallback nem aborta — apenas relata fatos (`available`, `version`, `error`) | Decisão de abortar/fallback pertence à camada de orquestração (`index.js`); preserva responsabilidade única e o Princípio Fundamental (Engine não decide diante de ambiguidade) |
| `environment.js` não expõe campo `path` do binário resolvido na v1 | Exigiria mecanismo adicional (parsing de PATH ou `which`), sem consumidor real hoje — YAGNI, mesmo princípio aplicado a `logger.js` |
| `sha256sum`/`shasum` marcados como `required: false` em `KNOWN_BINARIES`, divergindo do texto da SPEC v1.0 | Camada 2 já usa `node:crypto` puro (`hash.js`) para hashing; binários de sistema tornaram-se informativos, não obrigatórios — continuidade de DT-003 (SPEC.md desatualizada), não é dívida técnica nova |
| `ETIMEDOUT` checado antes do branch genérico de `result.error` em `detectBinary` | `spawnSync` popula `error.code === "ETIMEDOUT"` E `signal === "SIGTERM"` simultaneamente em timeout real; ordem errada classificava timeout como `SPAWN_ERROR` genérico (bug encontrado e corrigido durante os testes desta sessão) |

---

## 6. Estado atual de implementação

### ✅ Concluído e aprovado

### ✅ `src/config/constants.js` — aprovado (resolve DT-006)

- Única fonte de verdade para: `ENGINE_DIR_NAME`, `RUNS_DIR_NAME`, `LOCK_FILE_NAME`,
  `TMP_DIR_NAME`, `BACKUP_DIR_NAME`, `ARTIFACT_FILE_NAMES` (todos os artefatos de run),
  `RUN_ID_PATTERN`, `RUN_SEQUENCE_MIN/MAX`.
- Puramente declarativo, sem lógica, sem imports.
- Escopo desta versão contém **exclusivamente** o bloco já contratado por
  `paths.js` — todos os valores conferidos contra `paths.js` (destructuring
  do import) e contra `paths.test.js` (valores literais esperados:
  `.adturbo-engine`, `runs`, `engine.lock`, `tmp`, `backup`, nomes exatos
  de artefato, `RUN_ID_PATTERN = /^\d{8}-\d{3}$/`, sequência 1–999).
- **Deliberadamente NÃO contém** `KNOWN_BINARIES`
  (`environment.js`/DT-007) nem `CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS`
  (`classifier.js`/DT-007) — permanecem em `constants.draft.js`. Decisão
  já tomada e não reaberta nesta sessão: nenhum módulo depende dessas
  constantes estarem em `constants.js` hoje, e migrá-las sem necessidade
  concreta fundiria DT-006 (resolvida) com DT-007 (ainda aberta) no mesmo
  commit. Ver seção 15 para o raciocínio completo dessa decisão.
- **DT-006 resolvida nesta sessão** (2026-07-03) — ver seção 16 para o
  registro completo da sessão e evidência de execução.

**`src/modules/paths.js`** — aprovado formalmente pelo usuário
- API pública: `PathsError`, `isValidRunId`, `assertValidRunId`, `formatRunId`,
  `assertProjectRootExists`, `engineDir`, `runsDir`, `runDir`, `runArtifactPaths`,
  `lockFilePath`, `tmpDir`, `resolveWithinProject`.
- 30 testes passando em `__tests__/paths.test.js`.
- Única função que toca o disco: `assertProjectRootExists` (leitura, nunca escrita).
- Proteção contra path traversal em `resolveWithinProject`.

### ✅ `src/modules/hash.js` — aprovado

API pública: `HashError`, `hashFile(filePath)`, `hashBuffer(buffer)`, `HASH_ALGORITHM`.
- `hashFile` — SHA-256 via streaming (`createReadStream`, `highWaterMark: 64KB`).
  Lança `HashError` com codes: `INVALID_INPUT`, `FILE_NOT_FOUND`,
  `FILE_NOT_READABLE`, `PATH_IS_DIRECTORY`, `IO_ERROR` (com `cause`).
- `hashBuffer` — SHA-256 síncrono de um `Buffer` já em memória.
  Lança `HashError` `INVALID_INPUT` para não-Buffer.
- 15/15 testes passando em `__tests__/hash.test.js`.

### ✅ `src/modules/logger.js` — aprovado

API pública: `LoggerError`, `LOG_LEVELS`, `createLogger(options)`.
- `createLogger(options?)` — factory pura, sem estado module-level. Retorna
  `{ debug, info, warn, error }` (objeto congelado via `Object.freeze`).
- `options`: `minLevel` (default `"INFO"`), `prefix` (opcional), `withTimestamp`
  (default `false`), `clock` (opcional, `() => Date`, injetável para testes).
- Filtragem por nível: comparação numérica direta (`LOG_LEVELS[level] >= minLevel`),
  sem buffer/fila.
- Roteamento de stream: DEBUG/INFO → `console.log`; WARN → `console.warn`;
  ERROR → `console.error`.
- Lança `LoggerError` apenas na criação (`INVALID_LOG_LEVEL`, `INVALID_PREFIX`,
  `INVALID_CLOCK`). Métodos de log em si nunca lançam.
- API deliberadamente mínima nesta fase — sem `isLevelEnabled`, sem output em
  arquivo, sem formato estruturado (JSON). Cresce só com caso de uso real futuro.
- 25/25 testes passando em `__tests__/logger.test.js`.

### ✅ `src/modules/environment.js` — aprovado (definitivo)

API pública: `EnvironmentError`, `detectBinary(name, options?)`, `detectEnvironment(options?)`.
- `detectBinary(name, { versionFlag?, timeoutMs? })` — detecta um único
  binário via `spawnSync(name, [versionFlag])` real (não `which`, não
  parsing de `PATH`). Retorna `{ name, available, version, error }`.
  `available: true` para qualquer resposta do processo, mesmo exit code
  ≠ 0; só `ENOENT` (ausência real) e `ETIMEDOUT` (timeout) resultam em
  `available: false`. Versão extraída best-effort (primeira linha não
  vazia de stdout, senão stderr) — nunca tratado como erro se ausente.
- `detectEnvironment({ binaries? })` — detecta o conjunto de
  `KNOWN_BINARIES` (default) ou uma lista customizada. Retorna
  `{ platform, arch, nodeVersion, detectedAt, binaries: {...} }`.
  Sequencial (não paralelo) por padrão — lista pequena, roda uma vez por
  execução da Engine, não é hot path.
- Nunca lança por binário ausente — só por parâmetros de entrada
  inválidos (`INVALID_BINARY_NAME`, `INVALID_TIMEOUT`,
  `INVALID_VERSION_FLAG`, `INVALID_BINARIES_LIST`).
- **Não decide fallback nem aborta** — só relata fatos. Decisão de
  abortar (ex.: `node` obrigatório ausente) pertence a `index.js`
  (futuro), que consome o relatório. Campo `required` em
  `KNOWN_BINARIES` é puramente informativo, não usado por este módulo.
- **Não persiste nada em disco** — grava-lo como `environment.json` é
  responsabilidade futura de `manifest.js`.
- Divergência conhecida com SPEC.md v1.0 (seção 6): `sha256sum`/`shasum`
  são descritos como "obrigatórios" na SPEC, mas tratados como
  `required: false`/informativos aqui, pois `hash.js` já resolve hashing
  via `node:crypto` puro. Tratada como **continuidade de DT-003**
  (SPEC.md desatualizada), não como dívida técnica própria.
- Bug real encontrado e corrigido durante a sessão: `ETIMEDOUT` precisa
  ser checado antes do branch genérico de `result.error`, pois
  `spawnSync` popula `error.code === "ETIMEDOUT"` e
  `signal === "SIGTERM"` simultaneamente em timeout real.
- 27/27 testes passando em `__tests__/environment.test.js`, incluindo
  teste de timeout com fixture 100% Node puro (script temporário com
  `setTimeout`, invocado como `node <arquivo>`) — sem dependência de
  `sleep` nem de nenhum outro binário externo além do próprio `node`,
  que já é pré-requisito de todo o projeto. Arquivo temporário é criado
  e removido dentro do próprio teste (`try/finally`), sem deixar rastro.
- **`KNOWN_BINARIES` isolado em `src/config/constants.draft.js`**, um
  RASCUNHO explicitamente marcado como não-definitivo (path e conteúdo
  deixam isso inequívoco), não o `constants.js` real do projeto — que
  ainda não foi anexado a nenhuma sessão. `environment.js` importa de
  `../config/constants.draft.js`, com comentário no próprio import
  apontando para o procedimento de migração. **O arquivo de teste
  também importa `KNOWN_BINARIES` do rascunho** (para validar a lista
  padrão contra a fonte real, não uma cópia hardcoded) — a migração
  futura de DT-006 precisa atualizar **dois** imports, não um; ambos
  documentados com comentário explícito no ponto de import. Rastreado
  como DT-006 (renumerada nesta sessão a partir da antiga DT-007, após
  a fusão de DT-006 original em DT-003).

> **Atualização (sessão de 2026-07-03, seção 16):** `constants.js` real foi
> anexado e DT-006 foi resolvida — mas apenas para o bloco consumido por
> `paths.js`. `environment.js` **continua**, deliberadamente, importando
> `KNOWN_BINARIES` de `constants.draft.js` (não migrado; ver DT-007). O
> parágrafo abaixo ("Ação obrigatória pendente") descreve a migração de
> `KNOWN_BINARIES`, que **não** foi executada nesta sessão e permanece
> como ação futura sob DT-007.

**Ação obrigatória pendente (DT-007, ex-parte de DT-006):** quando a
migração de `KNOWN_BINARIES` for priorizada, mover o bloco de
`constants.draft.js` para dentro do `constants.js` real e atualizar o
import em `environment.js` para `../config/constants.js`. **Não apagar
`constants.draft.js`** nesse momento — o arquivo também contém
`CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS` (seção separada, mesma dívida
DT-007), usados por `classifier.js`; só pode ser removido quando ambos os
blocos tiverem migrado. Procedimento detalhado no cabeçalho do próprio
arquivo de rascunho.

### ✅ `src/modules/classifier.js` — aprovado

Ver seção 13 (decisão arquitetural) e seção 14 (implementação) para o
registro completo. Resumo: única autoridade para categoria/subsistema/
risco de um arquivo, via `classify(relativePath)` — pura, síncrona, sem
I/O, decide exclusivamente por path. 80/80 testes passando em
`__tests__/classifier.test.js`.

### ✅ `src/modules/inventory.js` — aprovado (seção 17)

API pública: `InventoryError`, `buildInventory(projectRoot, relativePaths)`.
- Descreve cada arquivo de uma lista de paths **já definida por quem
  chama** (não descobre/varre arquivos sozinho — nenhum documento do
  projeto fecha regras de include/exclude). Para cada path retorna
  `path`, `sizeBytes`, `sha256`, `lastModified`, `category`, `subsystem`,
  `risk`, `matchedRule`, `error`.
- **Escopo deliberadamente parcial** em relação ao `inventory.json` da
  SPEC.md §7: NÃO inclui `exports`/`imports`/`syntaxValid`/`parseError`
  — esses campos exigem AST e são reservados para `fingerprint.js`
  (fronteira já fixada em PROJECT_CONTEXT.md, seção 14). Decisão
  confirmada explicitamente nesta sessão, não presumida.
- Reaproveita `paths.js:resolveWithinProject` (proteção contra path
  traversal), `classifier.js:classify` (categoria/subsistema/risco) e
  `hash.js:hashFile` (SHA-256) — nenhuma lógica duplicada.
- Erro de **input** (`projectRoot`/`relativePaths` malformado) lança
  `InventoryError`. Erro por **arquivo individual** (ausente, sem
  permissão, diretório em vez de arquivo, fora do projeto) nunca lança —
  fica estruturado em `item.error = { code, message }`, mesmo espírito
  de `environment.js` (ausência é fato de domínio, não exceção).
- Classificação roda sempre, mesmo para arquivo inexistente — só
  `sizeBytes`/`sha256`/`lastModified` dependem do arquivo existir.
- Concorrência **sequencial** (`for...of` + `await`), decisão explícita
  do usuário nesta sessão — mesmo padrão conservador de
  `environment.js:detectEnvironment`.
- 25/25 testes passando em `__tests__/inventory.test.js`, incluindo hash
  verificado de forma independente via `node:crypto` puro (não apenas
  formato), path traversal, arquivo ausente, diretório em vez de
  arquivo, e confirmação de que os campos de escopo de `fingerprint.js`
  nunca aparecem no resultado.
- Bug real encontrado e corrigido durante a sessão (por execução, não
  revisão estática): erro de `resolveWithinProject` recebia prefixo
  `PATH_` duplicado sobre um `code` que já era auto-descritivo (ex.:
  `PATH_PATH_OUTSIDE_PROJECT`). Corrigido para usar `err.code`
  diretamente.
- **Nota (não é dívida técnica):** ao inventariar os próprios paths-fonte
  da Engine (ex.: `"src/modules/paths.js"`), `classify()` retorna
  `category: "unknown"` / `risk: "high"` para a maioria — confirmado
  contra `classifier.test.js` (80/80 passando) antes de decidir não
  tocar em `classifier.js`. O único caso coberto e testado é o prefixo
  literal `"adturbo-engine/"` (teste `"deriveSubsystem: adturbo-engine/**
  => engine"`), cenário de "Engine analisando a si mesma como
  projeto-alvo a partir de um `PROJECT_ROOT` pai" — diferente de rodar
  com `PROJECT_ROOT` já apontando para a raiz da própria Engine, onde
  esse prefixo nunca aparece. Nenhum documento do projeto exige que
  `classifier.js` reconheça os demais paths internos da Engine — é o
  fail-safe `unknown` já documentado, funcionando como projetado para
  qualquer path fora das convenções do Next.js App Router do AdTurbo
  (único domínio para o qual `classifier.js` foi construído).
  `classifier.js` **não foi modificado** nesta sessão.
- Ver seção 17 para o registro completo da sessão de implementação.

### ✅ `src/modules/fingerprint.js` — aprovado (seção 18)

API pública: `FingerprintError`, `async function fingerprintFile(absolutePath)`.
- Determina `syntaxValid`/`parseError` e a lista de módulos importados
  (`imports`, só `source`) de um arquivo já existente em disco — os
  campos de "o que o arquivo contém" que SPEC.md §7 atribui à análise
  via AST, complementares a `inventory.js` (metadados sem abrir
  conteúdo).
- **Divergência arquitetural real confirmada e registrada nesta sessão
  (DT-011):** SPEC.md descreve essa análise como "Camada 2 — Node
  Engine" usando Babel/Recast (dependências npm, `.mjs`/ESM), invocada
  por uma "Camada 1 — Bash Engine" via stdin/stdout JSON. Nenhum módulo
  do projeto é Bash; tudo é Node/CommonJS puro. Decisão explícita do
  usuário: **PROJECT_CONTEXT.md é a referência arquitetural quando
  diverge de SPEC.md** — a divergência foi registrada (DT-011), não
  "resolvida" introduzindo uma arquitetura nova.
- **Detecção de sintaxe:** `node --check` foi avaliado e **rejeitado**
  — confirmado por execução real que, em arquivo `.js` contendo
  `export`/`import` (padrão das routes do AdTurbo), `node --check` pode
  retornar exit 0 mesmo com erro de sintaxe real mais adiante no
  arquivo (falso negativo grave, reproduz o padrão do incidente
  fundador). Adotado `vm.SourceTextModule` (Node core) após protótipo
  isolado validado contra 4 requisitos (detecção correta, não-execução
  do código, Node Core puro, zero deps) — 3 de 4 limpos; o requisito de
  "Node Core puro" exige a flag `--experimental-vm-modules`, que não
  pode ser ativada seletivamente via `require()`. Mitigação: a checagem
  roda isolada em subprocesso (`fingerprint-worker.js`, via
  `child_process.spawnSync`), então o processo pai (testes, futuro
  `index.js`) não precisa da flag. Registrado como **DT-012** (decisão
  aceita, não bloqueante).
- **Escopo deliberadamente reduzido, decisão confirmada, não lacuna:**
  NÃO produz `exports` nem `imports[].names` — `vm.SourceTextModule`
  só expõe `moduleRequests` (specifier/source de cada import), sem
  nomes específicos nem lista de exports. Extrair isso exigiria AST
  completo (Babel, vetado). Alternativa sem dependências (regex sobre
  texto-fonte) foi avaliada e descartada por fragilidade (comentários,
  strings multilinha, imports quebrados em várias linhas) — exatamente
  o tipo de heurística que o projeto rejeita.
- Erro de input (`absolutePath` não-string/vazio/relativo) lança
  `FingerprintError`. Erro de leitura de arquivo ou de infraestrutura
  do subprocess (spawn falhou, timeout, saída não é JSON) nunca lança
  — fica estruturado em `result.error`, mesmo espírito de
  `inventory.js`.
- Segurança verificada por execução real: conteúdo do arquivo analisado
  vai via stdin para o subprocesso (`spawnSync` sem `shell: true`) —
  confirmado que uma string com sintaxe de injeção de shell no
  conteúdo-fonte não executa comando algum (tratada como dado puro).
  Confirmado também que o código analisado nunca é executado (só a
  construção de `SourceTextModule`, sem `link()`/`evaluate()`) — um
  side effect global de teste não disparou.
- 23/23 testes passando em `__tests__/fingerprint.test.js`, incluindo o
  caso crítico (arquivo com `)` órfão no padrão exato do incidente
  fundador, que `node --check` teria aprovado erroneamente), arquivo
  binário, diretório, arquivo inexistente, tentativa de injeção via
  conteúdo, e confirmação de que `exports`/`imports[].names` nunca
  aparecem no resultado.
- Ver seção 18 para o registro completo da sessão de implementação,
  incluindo o protótipo que validou `vm.SourceTextModule` antes de
  qualquer código de produção.

---

## 7. Próximas tarefas em ordem

1. ~~`src/modules/inventory.js`~~ — ✅ concluído (seção 17)
2. ~~`src/modules/fingerprint.js`~~ — ✅ concluído (seção 18)
3. **PRÓXIMO: `src/modules/manifest.js`**
4. **`src/modules/plan.js`**
5. **`src/modules/report.js`**
6. **`src/modules/lock.js`**
7. **`src/modules/fs-helpers.js`**
8. **`src/modules/analyze.js`**
9. **`src/index.js`** — CLI final, integração de todos os módulos

---

## 8. Convenções adotadas

### Nomenclatura de erros
- Classes de erro: `<Modulo>Error` (ex: `PathsError`, `HashError`, `LoggerError`, `EnvironmentError`).
- `code` sempre em `UPPER_SNAKE_CASE`, estável entre versões.
- Predicado booleano: `isValid*` — nunca lança.
- Assert que lança: `assert*` — lança `<Modulo>Error` ou subtipo.

### Estrutura de cada módulo
```
"use strict";
// JSDoc com decisões arquiteturais no cabeçalho
// imports (só Node core na Fase 1)
// classe de erro customizada
// funções internas (não exportadas)
// funções públicas (exportadas)
// module.exports = { ... } — explícito, sem * exports
```

### Testes
- Arquivo: `src/modules/__tests__/<modulo>.test.js`
- Zero dependências externas — só Node core (`assert`, `fs`, `os`, `path`, `crypto`).
- Runner manual mínimo com `test()` / `testSync()` + contador de passes/falhas.
- Exit code 1 se qualquer teste falhou.
- Rodar: `node src/modules/__tests__/<modulo>.test.js`
- Testes assíncronos em função `async main()` com `await` em cada `test()`.

### Processo de desenvolvimento
- Um módulo por vez, aprovado antes do próximo.
- Se surgir decisão arquitetural não coberta pela spec: interromper, documentar
  o trade-off, discutir antes de implementar.
- API de cada módulo nasce mínima; funcionalidades pensadas para "futuras fases"
  sem caso de uso real atual não entram no escopo até serem necessárias de fato
  (princípio reforçado explicitamente na sessão de `logger.js`).
- Ao final de cada sessão: atualizar este arquivo.

---

## 9. Dívidas técnicas e problemas conhecidos

| ID | Descrição | Severidade | Status |
|---|---|---|---|
| DT-001 | `hash.js`: `hashBuffer` ausente | Alta | ✅ Resolvido |
| DT-002 | `hash.js`: validação de input lançava `TypeError` em vez de `HashError` | Alta | ✅ Resolvido |
| DT-003 | SPEC.md não foi atualizada com as últimas mudanças acordadas (v1.1 incompleto). Inclui explicitamente: (a) SPEC.md seção 6 descreve `sha256sum`/`shasum` como binário obrigatório para hashing, mas `hash.js` já resolve hashing via `node:crypto` puro — `environment.js` reflete o estado real (`required: false`), divergindo do texto ainda não atualizado da SPEC | Média | Pendente (pode esperar Fase 2) |
| DT-004 | Teste de memória em `hash.test.js` (teste "não retém conteúdo em memória") usa threshold heurístico (2MB) — pode ter falsos positivos em ambientes com GC não determinístico | Baixa — documentado no próprio teste | Aceito |
| DT-005 | `logger.js`: parâmetro `cause` de `LoggerError` existe na API (consistente com `HashError`) mas não é exercitado por nenhum teste, pois nenhum dos três erros atuais (`INVALID_LOG_LEVEL`, `INVALID_PREFIX`, `INVALID_CLOCK`) se origina de uma exceção interna encadeada | Baixa — código morto potencial, mas mantido por consistência de API entre módulos de erro | Aceito, revisar se `cause` seguir sem uso após mais 2-3 módulos |
| DT-006 | `constants.js` real do projeto (contendo `ENGINE_DIR_NAME`, `RUNS_DIR_NAME`, `LOCK_FILE_NAME`, `TMP_DIR_NAME`, `BACKUP_DIR_NAME`, `ARTIFACT_FILE_NAMES`, `RUN_ID_PATTERN`, `RUN_SEQUENCE_MIN/MAX`, usados por `paths.js`) não havia sido anexado a nenhuma sessão | Alta — bloqueava `paths.test.js` (`MODULE_NOT_FOUND`) | ✅ **Resolvida** (sessão 2026-07-03, seção 16) — `src/config/constants.js` criado com o contrato exato exigido por `paths.js`, verificado contra `paths.test.js`. `paths.test.js`: 30/30 passando. **Nota de escopo:** esta resolução cobre apenas o bloco consumido por `paths.js`. A migração de `KNOWN_BINARIES`/`CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS` de `constants.draft.js` para `constants.js` **não** faz parte de DT-006 — é DT-007, que permanece pendente |
| DT-007 | `KNOWN_BINARIES` (usado por `environment.js`) e `CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS` (usados por `classifier.js`) permanecem em `src/config/constants.draft.js` — um RASCUNHO explícito, não o `constants.js` real (que já existe e está aprovado, mas contém apenas o bloco de DT-006). O rascunho mistura, no mesmo arquivo, constantes de **ambiente de execução** (`KNOWN_BINARIES`) com constantes de **domínio do projeto-alvo** (categorias/subsistemas/risco do AdTurbo) — naturezas diferentes de configuração | Média — risco de divergência ao migrar; migração terá que mover blocos heterogêneos, não um bloco único; `environment.js` e `classifier.js` continuam importando do caminho não-canônico | Pendente — ação obrigatória: migrar `KNOWN_BINARIES` e `CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS` para dentro de `constants.js`, cada seção com atenção para não fundir os dois conceitos no arquivo definitivo; atualizar os quatro pontos de import (`environment.js`, `environment.test.js`, `classifier.js`, `classifier.test.js`); só então apagar `constants.draft.js`. Deliberadamente **não** executada na sessão de DT-006 (2026-07-03) — decisão registrada na seção 15/16 para não fundir os dois escopos no mesmo commit |
| DT-008 | Bug estrutural de import pré-existente, encontrado por execução real (não suposição) nesta sessão: `src/__tests__/logger.test.js`, `environment.test.js` e `paths.test.js` fazem `require('../logger.js')`, `require('../environment.js')`, `require('../paths')` — mas os módulos reais vivem em `src/modules/`, não em `src/`. Os três falham com `MODULE_NOT_FOUND` ao rodar diretamente do ZIP fornecido nesta sessão. `classifier.test.js` tinha o mesmo bug e **foi corrigido nesta sessão** (escopo autorizado); os outros três **não foram tocados** por estarem fora do escopo desta sessão (só `classifier.js` foi autorizado) | Alta — três suítes de teste já "aprovadas" em sessões anteriores não executam no estado atual do ZIP | Pendente — ação: aplicar a mesma correção de path (`../logger.js` → `../modules/logger.js`, etc.) em sessão futura dedicada, ou explicitamente incluir no escopo da próxima sessão de cada módulo |
| DT-009 | A tabela de risco aprovada em PROJECT_CONTEXT.md (seção 13, decisão 3) não menciona a categoria `config` em nenhuma faixa (`secret`→critical; `route`/`auth`→high; `lib`/`component`/`page`/`whatsapp-bot`→medium; `test`/`script`→low; `unknown`→high). `deriveRisk()` trata `config` pelo fail-safe (`high`), coerente com o princípio "ausência de regra explícita = cautela", mas isso não foi uma decisão explícita de ninguém sobre `config` especificamente — é uma consequência do fail-safe genérico, não uma regra pensada para esse caso | Baixa/Média — comportamento é seguro (não subestima risco), mas pode não refletir a intenção real (ex.: `next.config.js` talvez devesse ser `medium`, não `high`, dependendo do apetite de risco do usuário) | Pendente — ação: confirmar explicitamente em sessão futura se `config` deveria ter uma faixa própria na tabela de risco, ou se o fail-safe `high` é de fato o comportamento desejado |
| DT-010 | `PROJECT_CONTEXT.md` (seção 6) documenta `src/modules/__tests__/hash.test.js` como "15/15 testes passando", mas o arquivo **não existe** no ZIP fornecido nesta sessão — confirmado por listagem real de disco (`find . -type f -name "*.js"`), não por suposição. `hash.js` em si existe e está sintaticamente correto, mas não há evidência executável de que os 15 testes documentados ainda passam neste estado do repositório | Média — mesma natureza do DT-008 (documentação descreve um estado que a execução real não confirma); diferente de DT-008 porque aqui o arquivo de teste está ausente, não só com import quebrado | Pendente — fora do escopo desta sessão (autorizada apenas para `inventory.js`). Ação futura: reconstruir `hash.test.js` em sessão dedicada, ou confirmar se foi removido intencionalmente em algum ponto não documentado |
| DT-011 | SPEC.md, seção 3, descreve uma arquitetura de 3 camadas onde a "Camada 1 — Bash Engine" é literalmente scripts Bash (`.sh`) e a "Camada 2 — Node Engine" usa AST via Babel/Recast (dependências npm, módulos `.mjs`/ESM), invocada via stdin/stdout JSON pela Camada 1. Na prática, **nenhum módulo implementado até hoje é Bash** — `paths.js`, `hash.js`, `logger.js`, `environment.js`, `classifier.js`, `inventory.js` são todos Node/CommonJS puro (`.js`, `require`/`module.exports`, zero dependências npm), incluindo responsabilidades que a SPEC atribui à Camada 1 (inventário, hash/backup, detecção de ambiente). Essa divergência nunca foi decidida explicitamente em nenhuma sessão anterior — foi confirmada e resolvida nesta sessão (ver seção 18): decisão do usuário é que **PROJECT_CONTEXT.md é a referência arquitetural principal quando há conflito com SPEC.md**, mantendo CommonJS, sem ESM, sem dependências npm | Média — não bloqueia nada tecnicamente (a implementação real funciona e é testada), mas SPEC.md permanece desatualizada como documento de referência externo; alguém lendo só a SPEC teria uma expectativa de arquitetura incorreta | Registrada, não "resolvida" com nova arquitetura (decisão explícita do usuário, seção 18) — ação futura opcional: atualizar o texto de SPEC.md para refletir a arquitetura real (Node/CommonJS unificado, sem camada Bash literal), ou manter SPEC.md como "visão original" histórica e PROJECT_CONTEXT.md como fonte de verdade corrente, deixando claro no topo de SPEC.md que diverge do estado real |
| DT-012 | `fingerprint.js` usa `vm.SourceTextModule` (Node core) para detectar `syntaxValid`/`parseError` de forma confiável em arquivos ESM — ver seção 18 para o protótipo que validou essa escolha. Essa API exige a flag `--experimental-vm-modules` no processo Node que a invoca; confirmado por execução que a flag não pode ser ativada seletivamente de dentro de um módulo `require()`'d por um processo já iniciado sem ela. Mitigação adotada: a chamada roda isolada em subprocesso via `child_process.spawnSync(process.execPath, ["--experimental-vm-modules", ...])`, mesmo padrão já aprovado em `environment.js` — o processo pai (testes, futuro `index.js`) não precisa da flag | Baixa/Média — funciona e foi validado por execução real, mas depende de uma API rotulada "experimental" pelo próprio Node (pode mudar de comportamento em versões futuras) e paga custo de spawn de processo por arquivo verificado (~dezenas de ms cada) | Aceita como decisão de design (não bloqueante). Ação futura: se o custo de spawn por arquivo se mostrar proibitivo em projetos muito grandes, avaliar processar múltiplos arquivos por invocação do subprocesso (batch via stdin com múltiplos documentos), em vez de um spawn por arquivo |

---

## 10. Arquivos e responsabilidades (referência rápida)

| Arquivo | Responsabilidade | Status |
|---|---|---|
| `src/config/constants.js` | Nomes, padrões, limites usados por `paths.js` — sem lógica | ✅ Aprovado (DT-006 resolvida, seção 16) |
| `src/config/constants.draft.js` | RASCUNHO temporário — `KNOWN_BINARIES` (`environment.js`) e `CATEGORIES`/`SUBSYSTEMS`/`RISK_LEVELS` (`classifier.js`), até migrarem para `constants.js` | ⚠️ Temporário, não é arquitetura definitiva — ver DT-007 |
| `src/modules/paths.js` | Resolver e validar caminhos | ✅ Aprovado |
| `src/modules/hash.js` | Hash SHA-256 via streaming + hash síncrono de Buffer | ✅ Aprovado |
| `src/modules/logger.js` | Único módulo (junto com index.js) autorizado a imprimir no console; níveis DEBUG/INFO/WARN/ERROR com filtragem | ✅ Aprovado |
| `src/modules/environment.js` | Detectar binários disponíveis no ambiente (via spawn real) e metadados de plataforma/runtime; não decide fallback, não persiste | ✅ Aprovado |
| `src/modules/classifier.js` | Única autoridade para categoria, subsistema e risco de um arquivo | ✅ Aprovado (seção 14) |
| `src/modules/inventory.js` | Descrever arquivos do projeto-alvo (metadados + classificação; exports/imports/syntaxValid ficam para `fingerprint.js`) | ✅ Aprovado (seção 17) |
| `src/modules/fingerprint.js` | syntaxValid/parseError (via vm.SourceTextModule em subprocess) + imports (só source); exports e imports.names fora do escopo (ver seção 18) | ✅ Aprovado (seção 18) |
| `src/modules/manifest.js` | Gerar run-id e escrever artefatos da run | Não iniciado |
| `src/modules/plan.js` | Gerar plano de transformação antes do backup | Não iniciado |
| `src/modules/report.js` | Gerar `report.md` legível por humano | Não iniciado |
| `src/modules/lock.js` | Lockfile de execução concorrente via `fs.open(..., "wx")` | Não iniciado |
| `src/modules/fs-helpers.js` | Operações de arquivo reutilizáveis (copiar, criar dir, etc.) | Não iniciado |
| `src/modules/analyze.js` | Orquestrar o modo `--analyze` (read-only) | Não iniciado |
| `src/index.js` | CLI: parse de argumentos, integração de módulos, único ponto de I/O de console | Não iniciado |

---

## 11. Modos de execução previstos

| Flag | Comportamento |
|---|---|
| `--analyze` | Read-only: inventário + relatório, sem backup, sem modificação |
| `--dry-run` | Mostra o que faria, sem aplicar |
| `--apply` | Aplica transformações com backup + validação + rollback automático |
| `--restore <run-id>` | Reverte arquivos tocados naquela run para o estado pré-alteração |
| `--recover` | Diagnóstico: compara hashes atuais com todos os runs conhecidos; não toma decisões |
| `--full-backup` | Força snapshot completo do projeto (além do backup padrão dos arquivos-alvo) |
| `--continue-on-invalid` | Opt-in: pula arquivos já inválidos em vez de abortar a run inteira |
| `--strict` | Torna `next build` um validador Crítico (bloqueia) em vez de Importante |

---

## 12. Informações de ambiente

- **Ambiente primário:** Termux (Android ARM64)
- **Node.js:** v22.22.2 (confirmado em sessão atual)
- **Limitações conhecidas:**
  - `/tmp` pode não existir → fallback: `.adturbo-engine/tmp/`
  - `mktemp` pode falhar → usar fallback de path próprio
  - `next build` instável no ARM64 (cache wasm) → não é gate crítico
  - `sha256sum` ou `shasum`: verificar qual está disponível no Termux antes de usar em scripts Bash

---

## 13. Log de sessões

### Sessão atual (2026-07-01)
- Implementado `src/modules/logger.js` completo: níveis DEBUG/INFO/WARN/ERROR,
  filtragem por `minLevel`, prefixo opcional, timestamp opcional com `clock`
  injetável, roteamento de stream (stdout para DEBUG/INFO, stderr para WARN/ERROR).
- Ajuste arquitetural aplicado durante a revisão: API cortada para o mínimo
  necessário — removidos `isLevelEnabled`/`getMinLevel` da proposta inicial por
  não terem consumidor real ainda.
- Testes implementados e **executados de fato** em `src/modules/__tests__/logger.test.js`:
  **25/25 passando**, exit code 0 (`node src/modules/__tests__/logger.test.js`).
- `logger.js` promovido a ✅ APROVADO nas seções 3, 6 e 10.
- Próximo módulo da fila: `src/modules/environment.js`.

### Sessão seguinte, mesma data (2026-07-01) — `environment.js`
- Implementado `src/modules/environment.js` completo: `detectBinary` (um
  binário via `spawnSync` real) e `detectEnvironment` (conjunto de
  `KNOWN_BINARIES` + metadados de plataforma/runtime). Módulo relata
  fatos, não decide fallback/abort, não persiste em disco.
- Criado `src/config/constants.js` **parcial**, contendo apenas
  `KNOWN_BINARIES` (necessário para `environment.js`) — demais
  constantes de `paths.js` não foram fornecidas e não foram
  reconstruídas nesta sessão. Ver DT-007.
- Identificada e documentada divergência real entre SPEC.md v1.0
  (`sha256sum`/`shasum` como obrigatórios) e o estado real do projeto
  (`hash.js` já usa `node:crypto` puro). Tratado como informativo em
  `KNOWN_BINARIES`, registrado como DT-006, não decidido unilateralmente
  sem sinalizar.
- Bug real encontrado durante a escrita dos testes: `ETIMEDOUT` caía no
  branch genérico de erro em vez de ser classificado como `TIMEOUT`,
  porque `spawnSync` popula `error.code` e `signal` simultaneamente em
  timeout. Corrigido no módulo (reordenação de branches), com teste de
  regressão usando timeout real via `sleep` (não simulado).
- Testes implementados e **executados de fato** em
  `src/modules/__tests__/environment.test.js`: **26/26 passando**, exit
  code 0. Suíte completa do projeto (logger + environment) também
  reexecutada: **51/51 passando**, sem regressão.
- `environment.js` submetido para aprovação nas seções 3, 6 e 10.
  **Correção de registro**: a aprovação desta sessão foi **parcial** —
  ver sessão seguinte. `constants.js` criado nesta sessão ocupava
  indevidamente o caminho canônico `src/config/constants.js` como se
  fosse definitivo, mesmo sendo parcial. Corrigido na sessão seguinte.
- Próximo módulo da fila (na época): `src/modules/classifier.js`.

### Sessão de correção (2026-07-01) — ajuste obrigatório em `environment.js`

Aprovação da sessão anterior foi **parcial**: módulo funcionalmente correto,
mas com três ajustes exigidos antes de considerar `environment.js` concluído.

1. **`constants.js` não pode ser arquivo paralelo parcial definitivo.**
   Corrigido: o `constants.js` parcial foi removido do caminho canônico
   `src/config/constants.js` e recriado como `src/config/constants.draft.js`
   — um rascunho explicitamente marcado como temporário tanto no nome do
   arquivo (sinal estrutural, não só comentário) quanto no conteúdo, com
   procedimento de migração documentado no próprio cabeçalho. `environment.js`
   atualizado para importar de `../config/constants.draft.js`, com comentário
   no import explicando a natureza temporária. Isso vira **DT-006**
   (renumerada; a antiga DT-007 tornou-se DT-006 após a fusão abaixo).

2. **DT-006 original (divergência `sha256sum`/`shasum` vs. SPEC) fundida em
   DT-003.** Mesma causa raiz (SPEC.md desatualizada em relação a decisões já
   tomadas) — não é dívida técnica nova, é continuidade de DT-003. Entrada
   DT-006 antiga removida da tabela; DT-003 expandida para citar o caso
   explicitamente. Referências cruzadas no restante do documento atualizadas
   (comentários em `constants.draft.js`, tabela de decisões arquiteturais).

3. **Teste de timeout tornado 100% portátil.** O teste anterior dependia do
   binário `sleep` (POSIX-padrão, mas ainda uma dependência externa de
   ambiente). Substituído por um fixture gerado dinamicamente dentro do
   próprio teste: um script Node temporário com `setTimeout`, invocado como
   `node <caminho-do-arquivo>` — path como único argumento, compatível com a
   API `detectBinary(name, { versionFlag })`. Mecanismo validado
   isoladamente (`spawnSync` fora do teste) antes de aplicar: confirmado que
   produz `error.code === "ETIMEDOUT"` e `signal === "SIGTERM"`, o mesmo
   padrão já tratado pelo módulo. Arquivo temporário limpo via
   `try/finally`, com verificação manual pós-execução de que não sobrou
   nenhum resíduo em `os.tmpdir()`.

**Validação real executada nesta sessão:**
```
node src/modules/__tests__/environment.test.js  → 26 passed, 0 failed, exit 0
node src/modules/__tests__/logger.test.js       → 25 passed, 0 failed, exit 0 (regressão)
```
Total do projeto: **51/51 testes passando**, sem regressão introduzida pelos
ajustes. Nenhum outro módulo depende de `constants.js`/`constants.draft.js`
ainda, então o impacto do rename foi isolado a `environment.js` e seu teste.

**Limitação declarada:** o `constants.js` real do projeto continua sem ter
sido anexado a nenhuma sessão. `constants.draft.js` é uma solução de
contorno deliberadamente temporária — DT-006 permanece **pendente** até que
o arquivo real seja fornecido e a migração seja executada.

- `environment.js` promovido a ✅ **APROVADO (definitivo)** nas seções 3, 6 e 10.
- Próximo módulo da fila: `src/modules/classifier.js`.

### Sessão de revisão de qualidade (2026-07-02) — melhorias não bloqueantes em `environment.test.js`

Revisão final de `environment.js`/testes/`constants.draft.js` aprovada sem
ressalvas arquiteturais. Três melhorias opcionais de qualidade nos testes
foram propostas, validadas empiricamente uma a uma antes de aplicar, e
implementadas — **nenhuma mudança em `environment.js`**, só no arquivo de
teste.

1. **Precisão do teste de versão.** `assert.ok(result.version.startsWith("v"))`
   trocado por `assert.strictEqual(result.version, process.version)`.
   Validado antes da troca: `detectBinary("node").version` bate
   exatamente com `process.version` neste ambiente (`"v22.22.2"` ===
   `"v22.22.2"`). O assert anterior aceitaria falsos positivos (qualquer
   string começando com "v").

2. **Cobertura de caso de borda `binaries: []`.** Array vazio explícito é
   semanticamente diferente de "options omitido" (que usa o default
   `KNOWN_BINARIES` completo) e não tinha teste próprio. Validado
   empiricamente antes de escrever o teste — `detectEnvironment({ binaries: [] })`
   retorna relatório válido com `binaries: {}`, sem lançar, com os
   demais campos (`platform`, `arch`, `nodeVersion`, `detectedAt`)
   preservados. Novo teste adicionado cobrindo isso.

3. **Teste de "lista padrão" validado contra a fonte real.** A lista de
   chaves esperadas estava hardcoded duas vezes (uma em
   `constants.draft.js`, outra copiada manualmente no teste) — um teste
   assim pode ficar silenciosamente errado (falso positivo) se alguém
   adicionar um binário novo e esquecer de atualizar a cópia no teste.
   Corrigido para comparar contra `Object.keys(KNOWN_BINARIES)` real,
   importado de `constants.draft.js`. **Efeito colateral identificado e
   documentado**: isso cria uma segunda referência ao rascunho temporário
   (além da já existente em `environment.js`) — a migração futura de
   DT-006 (quando o `constants.js` real for anexado) precisa atualizar
   **dois** imports, não um. Ambos os pontos de import têm comentário
   explícito apontando isso.

**Validação real executada nesta sessão:**
```
node src/modules/__tests__/environment.test.js  → 27 passed, 0 failed, exit 0
node src/modules/__tests__/logger.test.js       → 25 passed, 0 failed, exit 0 (regressão)
```
Total do projeto: **52/52 testes passando**, sem regressão. Nenhuma
mudança de arquitetura, comportamento ou API pública de `environment.js`
— só reforço da qualidade e precisão da suíte de testes.

`environment.js` permanece ✅ **APROVADO (definitivo)**. Próximo módulo
da fila, sem alteração: `src/modules/classifier.js`.

### Sessão de decisão arquitetural (2026-07-02) — `classifier.js` (pré-implementação)

Antes de iniciar a implementação de `classifier.js`, cinco pendências
identificadas durante a análise arquitetural do módulo (Fase 1) foram
levadas para decisão explícita. Nenhum código foi escrito nesta sessão —
apenas o contrato e as regras foram fechados.

**Decisões aprovadas:**

1. **Layout do bot WhatsApp — NÃO CONFIRMADO, sem regra provisória.**
   Não existe, em nenhum documento do projeto, informação sobre o
   diretório real do bot WhatsApp dentro do AdTurbo. Decisão explícita
   do usuário: **não implementar nenhum pattern provisório** (nem lista
   de prefixos candidatos) para evitar classificar arquivos do bot de
   forma potencialmente errada e silenciosa. Até que a árvore de
   diretórios real do AdTurbo seja fornecida, **qualquer arquivo do
   subsistema do bot é classificado como `category: "unknown"` /
   `subsystem: "unknown"`** — o que já implica `risk: "high"` pela
   regra do item 3. A categoria `bot` e o subsistema `whatsapp-bot`
   permanecem definidos no enum (`CATEGORIES`/`SUBSYSTEMS`), mas **sem
   nenhuma regra de path associada** até nova evidência ser anexada a
   uma sessão futura.

2. **Categorias e subsistemas propostos (Fase 1, seções 1.4/1.5) —
   APROVADOS**, com base em convenções padrão do Next.js App Router
   (`route`, `page`, `component`, `lib`, `config`, `secret`, `script`,
   `test`, `unknown`; subsistemas `api`, `frontend`, `auth`,
   `build-config`, `engine`, `unknown`). `bot`/`whatsapp-bot`
   permanecem no enum, inativos, conforme item 1.

3. **Risco como função derivada de categoria+subsistema — APROVADO.**
   `unknown` (categoria) mapeia diretamente para `RISK_LEVELS.HIGH` —
   não foi criado um quinto nível de risco separado, para não obrigar
   todo consumidor futuro (`plan.js`, `report.js`) a tratar um valor
   quase-redundante. Tabela completa: `secret` → `critical`; `route`
   ou subsistema `auth` → `high`; `lib`/`component`/`page`/
   `whatsapp-bot` → `medium`; `test`/`script` → `low`; `unknown` →
   `high` (fail-safe: ausência de informação é tratada com cautela,
   nunca como neutra ou como baixo risco).

4. **Campo `matchedRule` no resultado de `classify()` — APROVADO.**
   Não é requisito de nenhum documento anterior; é inferência a partir
   do Princípio Fundamental (Engine auditável). Mesmo padrão já usado
   em `environment.js`, que retorna fatos + evidência (`available`,
   `version`, `error`), nunca um resultado opaco.

5. **Local das novas constantes — `constants.draft.js` único,
   organizado por seções.** Decisão do usuário: reverte a proposta
   original (arquivo de rascunho isolado `classifier-constants.draft.js`)
   para **não multiplicar arquivos de rascunho temporário**.
   `CATEGORIES`, `SUBSYSTEMS`, `RISK_LEVELS` entram no mesmo
   `src/config/constants.draft.js` já usado por `KNOWN_BINARIES`, como
   uma seção separada e comentada. Trade-off aceito e registrado como
   **DT-007** (seção 9): o arquivo de rascunho passa a misturar
   constantes de ambiente de execução com constantes de domínio do
   projeto-alvo, o que torna a futura migração para `constants.js`
   real mais heterogênea. Mitigação: manter separação clara por
   comentário de seção dentro do próprio `constants.draft.js` até a
   migração acontecer.

6. **Ordem de precedência das regras de classificação — APROVADA.**
   Definida explicitamente para eliminar ambiguidade nos casos em que
   um mesmo path poderia, em teoria, casar com mais de uma regra de
   categoria. `classify()` testa as regras nesta ordem exata e retorna
   no primeiro match — a implementação e os testes devem seguir
   estritamente esta sequência:

   ```
   1. secret
   2. config
   3. route
   4. page
   5. component
   6. lib
   7. script
   8. test
   9. unknown   (fallback — nenhuma regra anterior casou)
   ```

   `bot` não aparece nesta ordem porque, conforme item 1, não possui
   nenhum pattern de path ativo — não há regra para testar, logo não
   compete por precedência com as demais. Quando a estrutura real do
   bot WhatsApp for confirmada em sessão futura e uma regra de path
   for adicionada para `bot`, sua posição nesta ordem precisará ser
   decidida explicitamente naquele momento (não assumir uma posição
   por omissão).

   Justificativa da ordem escolhida (nota: o usuário forneceu a ordem
   pronta; a racionalização abaixo é minha leitura de por que essa
   sequência faz sentido tecnicamente, não uma justificativa que ele
   tenha declarado — deve ser validada, não tratada como fato
   registrado por ele): `secret` primeiro por ser a categoria com
   maior consequência de erro (Regra 4 — nunca abrir/hashear);
   `config` antes de `route`/`page`/`component` porque arquivos de
   configuração na raiz (ex.: `next.config.js`) têm nome fixo e não
   devem ser capturados por regras de diretório mais genéricas;
   `route` antes de `page` porque ambos vivem sob `app/**` e
   `route.{js,ts}` é um nome de arquivo mais específico que precisa
   ser checado antes de qualquer regra estrutural mais ampla de
   `app/**`; `page` antes de `component` pelo mesmo motivo (nome de
   arquivo específico antes de diretório genérico); `lib` depois, por
   ser um diretório próprio sem sobreposição típica com as anteriores;
   `script` e `test` por último entre as regras "ativas" porque seus
   patterns (extensão/sufixo) são os mais amplos e poderiam, em tese,
   casar acidentalmente com arquivos já cobertos pelas regras
   anteriores se testados antes delas; `unknown` sempre por último,
   como fallback puro.

**Contrato de `classify()` fechado para implementação:**
```js
function classify(relativePath) → {
  path: string,
  category: string,       // um de CATEGORIES
  subsystem: string,      // um de SUBSYSTEMS
  risk: string,            // um de RISK_LEVELS
  matchedRule: string | null,
}
```
Pura, síncrona, sem I/O. Recebe path já relativo e normalizado
(responsabilidade de quem chama, tipicamente `inventory.js` via
`paths.js`). Não resolve filesystem, não checa existência do arquivo.
Separador `/` assumido sempre (ambiente Termux/POSIX); sem suporte a
`\`, documentado como limitação conhecida.

**Arquitetura da Engine com `classifier.js`, definida:** módulo classifica
exclusivamente por path (nunca por conteúdo) — fronteira com
`fingerprint.js` (que decide *o que o arquivo contém*, via AST) mantida
explícita para evitar duplicação de lógica de análise entre os dois
módulos quando `fingerprint.js` for implementado.

**Pendência real que permanece aberta:** estrutura de diretórios real do
bot WhatsApp no AdTurbo. Sem essa informação, todo arquivo do bot será
classificado como `unknown`/`high` até nova sessão trazer evidência
concreta (ex.: listagem real de diretórios do projeto AdTurbo).

`classifier.js` está **aprovado para implementação** com o contrato
acima. Próximo passo: implementar o módulo, seguindo o mesmo processo
já usado nos módulos anteriores (código + testes reais executados +
atualização desta seção ao final da sessão).


---

## 14. Sessão de implementação (2026-07-02) — `classifier.js` (código real)

Implementação completa a partir de um ZIP fornecido pelo usuário contendo o
estado real do repositório da Engine (não do AdTurbo — o ZIP **não contém**
o projeto AdTurbo real, apenas a própria Engine). O ZIP continha uma
implementação parcial de `classifier.js`, interrompida deliberadamente em
duas frentes ("Interrupção 1": `constants.draft.js` sem `CATEGORIES`/
`SUBSYSTEMS`/`RISK_LEVELS`; "Interrupção 2": nenhum critério real de path
confirmado). Ambas as interrupções foram investigadas contra o material
real antes de decidir como proceder — não presumidas.

**Descoberta relevante**: as duas interrupções registradas no código
estavam **desatualizadas** em relação ao próprio `PROJECT_CONTEXT.md` do
ZIP — a seção 13 (linhas 555–694) já continha uma sessão de decisão
arquitetural completa e aprovada, fechando exatamente as cinco pendências
que motivaram a implementação parcial anterior (layout do bot, categorias/
subsistemas, fórmula de risco, campo `matchedRule`, local das constantes,
e a ordem de precedência exata). A implementação desta sessão seguiu essa
decisão já registrada, sem reabri-la.

### Evidência real vs. convenção de framework

Cada regra de path implementada foi marcada no código-fonte com sua
proveniência:
- **[PROJETO]** — confirmado por exemplo literal em SPEC.md/PROJECT_CONTEXT.md:
  `app/api/**/route.js` (categoria `route`, citado repetidamente:
  `app/api/produtos/route.js`, `app/api/mensagens/route.js`, e o arquivo do
  incidente fundador `app/api/criativos/route.js`); `lib/**` (categoria
  `lib`, citado: `lib/auth.js`, `lib/db.js`, `lib/apiClient.js`).
- **[CONVENÇÃO]** — padrão estável do Next.js App Router / Node, não citado
  literalmente no projeto, mas já coberto pela aprovação da decisão 2 da
  seção 13 ("com base em convenções padrão do Next.js App Router"):
  `page`/`layout` sob `app/**`; `components/**`; `.env*`/`*.pem`/`*.key`/
  `*credentials*` para `secret`; basenames fixos de config
  (`next.config.js`, `package.json`, `tsconfig.json` etc.); `scripts/**`
  e `*.sh` para `script`; `__tests__/` e `*.test.js`/`*.spec.js` para `test`.
- **[BLOQUEADO]** — bot WhatsApp: **nenhum pattern de path implementado**,
  por decisão 1 já aprovada. Qualquer arquivo candidato a bot (`whatsapp/**`,
  `bot/**`, `whatsapp-bot/**`) cai em `category: unknown`, `subsystem: unknown`,
  `risk: high`. Testado explicitamente com múltiplos nomes de diretório
  plausíveis para garantir que nenhum caia acidentalmente em outra
  categoria.

### API implementada

`src/modules/classifier.js` exporta: `ClassifierError`, `RULES` (array de
regras com `{ name, category, test() }`), `RULE_PRECEDENCE` (array de nomes,
**derivado de `RULES.map(r => r.name)`**, nunca duplicado manualmente, para
que as duas representações nunca possam divergir — mantém compatibilidade
retroativa com a API já testada na implementação parcial anterior),
`matchRule(relativePath)`, `deriveSubsystem(normalizedPath, category)`,
`deriveRisk(category, subsystem)`, `classify(relativePath)`.

`constants.draft.js` estendido com `CATEGORIES`, `SUBSYSTEMS`, `RISK_LEVELS`
como nova seção comentada e claramente separada de `KNOWN_BINARIES`
(decisão 5/DT-007, já aprovadas — mesmo arquivo de rascunho, não um novo
arquivo).

### Bugs reais encontrados e corrigidos durante a sessão

1. **Import quebrado em `classifier.test.js`** (pré-existente no ZIP):
   `require('../classifier.js')` a partir de `src/__tests__/` resolve para
   `src/classifier.js`, que não existe — o módulo real está em
   `src/modules/classifier.js`. Confirmado por execução real
   (`MODULE_NOT_FOUND`) antes de corrigir. **O mesmo bug existe em
   `logger.test.js`, `environment.test.js` e `paths.test.js`** — confirmado
   por execução, mas **não corrigido** por estar fora do escopo desta
   sessão. Registrado como **DT-008**.
2. **Erro de design em `deriveSubsystem` para categoria `lib` genérica**:
   a primeira versão retornava `SUBSYSTEMS.FRONTEND` como fallback para
   qualquer arquivo `lib/*` sem "auth" no nome — semanticamente incorreto
   (`lib/db.js` não é frontend). Corrigido para `SUBSYSTEMS.UNKNOWN`,
   honesto sobre ausência de informação suficiente para decidir o
   subsistema real. Descoberto ao inspecionar manualmente a saída de
   `classify("lib/db.js")` antes de formalizar os testes — não foi
   revelado pela suíte automatizada, foi encontrado por revisão crítica
   deliberada da própria saída.
3. **Expectativa errada no próprio teste** (não no módulo): um teste
   assumia que `app/whatsapp/route.js` deveria cair em `subsystem: unknown`
   por conter "whatsapp" no path — mas esse path casa legitimamente com a
   regra `route` (nome de arquivo `route.js` sob `app/`, regra ativa e
   confirmada), e o subsistema correto é `frontend` (via regra `app/**`
   não-api), não `unknown`. Este NÃO é o cenário da decisão 1 (bot sem
   regra) — é a regra `route` genérica funcionando como projetado. O teste
   foi corrigido para refletir o comportamento correto; o código já estava
   certo. Encontrado pela suíte real (1 falha em 80 na primeira execução).

### Lacuna real identificada na decisão já aprovada

A tabela de risco aprovada (seção 13, decisão 3) não cobre a categoria
`config` em nenhuma faixa. `deriveRisk()` trata isso via fail-safe
(`high`), coerente com o princípio já registrado, mas essa não foi uma
decisão pensada especificamente para `config` — é consequência do
fail-safe genérico. Registrado como **DT-009**, não decidido
silenciosamente como se fosse óbvio.

### Validação real executada nesta sessão

```
node src/modules/classifier.js --check   (checagem de sintaxe, node --check)
node src/config/constants.draft.js --check
node src/__tests__/classifier.test.js
```
Resultado real: **80 passed, 0 failed, exit code 0** (após corrigir a falha
real de 1/80 encontrada na primeira execução, descrita no item 3 acima).

Os outros três testes do projeto (`logger.test.js`, `environment.test.js`,
`paths.test.js`) **não foram executados com sucesso** neste ZIP — todos
falham por `MODULE_NOT_FOUND` (DT-008), bug pré-existente não introduzido
nesta sessão e fora do escopo autorizado (só `classifier.js`).

### Estado final

`classifier.js` está ✅ **APROVADO** (implementação completa, 80/80 testes
reais passando). Nenhuma funcionalidade determinável foi substituída por
comentário — o único ponto genuinamente bloqueado (bot WhatsApp) permanece
bloqueado por decisão explícita já registrada, não por nova cautela desta
sessão.

**Pendências reais para sessões futuras:**
- DT-008: corrigir imports quebrados em `logger.test.js`,
  `environment.test.js`, `paths.test.js`.
- DT-009: confirmar explicitamente a faixa de risco de `config`.
- Estrutura real de diretórios do bot WhatsApp no AdTurbo, para ativar
  as regras `bot`/`whatsapp-bot` (item 1 da decisão da seção 13,
  continua aberto).

Próximo módulo da fila: `src/modules/inventory.js`.

---

## 15. Sessão de correção — DT-008 (imports quebrados de teste) (2026-07-03)

Escopo: corrigir os imports quebrados registrados em DT-008 (item da seção 14)
em `logger.test.js`, `environment.test.js` e `paths.test.js`. Cada correção
foi verificada por execução real (`node <arquivo>.test.js`), não presumida.

### Correções aplicadas

| Arquivo | Linha | Antes | Depois |
|---|---|---|---|
| `src/__tests__/paths.test.js` | 14 | `require("..modules/paths")` | `require("../modules/paths")` |
| `src/__tests__/logger.test.js` | 4 | `require("..modules/logger.js")` | `require("../modules/logger.js")` |
| `src/__tests__/environment.test.js` | 7 | `require("../environment.js")` | `require("../modules/environment.js")` |
| `src/__tests__/environment.test.js` | 13 | `require("../../config/constants.draft.js")` | `require("../config/constants.draft.js")` |

**Divergência encontrada:** o comentário original (agora desatualizado) nas
linhas 8–13 de `environment.test.js` afirmava que a migração futura do DT-006
trocaria o import para `"../../config/constants.js"` (dois níveis). A
estrutura real do projeto mostra que um nível (`../config/...`) é o correto
a partir de `src/__tests__/`. O comentário deve ser corrigido na próxima
sessão que tocar esse arquivo.

### Resultado de execução real

- `logger.test.js` — ✅ **25/25 passed**
- `environment.test.js` — ✅ **27/27 passed**
- `paths.test.js` — ❌ **continua bloqueado**, agora por um motivo diferente
  e mais profundo do que o typo original de DT-008: `src/modules/paths.js`
  (linha 65) faz `require("../config/constants")`, que não existe. O
  `constants.js` real (com `ENGINE_DIR_NAME`, `RUNS_DIR_NAME`,
  `LOCK_FILE_NAME`, `TMP_DIR_NAME`, `BACKUP_DIR_NAME`,
  `ARTIFACT_FILE_NAMES`, `RUN_ID_PATTERN`, `RUN_SEQUENCE_MIN/MAX`) nunca foi
  anexado a nenhuma sessão (DT-006, já registrado na seção 3).

### Decisão tomada nesta sessão: não criar proxy/stub para `constants.js`

Foi cogitado criar `src/config/constants.js` como arquivo-proxy re-exportando
o conteúdo de `constants.draft.js`, apenas para destravar `paths.test.js`.
**Decisão: rejeitada.** Motivos:
1. `constants.draft.js` documenta explicitamente que sua localização fora do
   caminho canônico é uma garantia estrutural deliberada, não um acidente.
2. Um proxy que só repassa `KNOWN_BINARIES` não resolveria a dependência real
   de `paths.js`, que espera campos que não existem em `constants.draft.js`
   (`ENGINE_DIR_NAME` etc.) — o teste continuaria falhando, só que de forma
   mais difícil de diagnosticar.
3. Criar o arquivo no caminho canônico antes da hora arrisca conflito quando
   o `constants.js` real for anexado em sessão futura.

`paths.test.js` permanece **bloqueado por design**, como consequência
legítima e documentada do DT-006 — não é uma regressão desta sessão.

### DT-008: estado

**Parcialmente resolvido.** Os três imports com typo/profundidade errada
foram corrigidos e verificados (`logger.test.js`, `environment.test.js`).
`paths.test.js` não pode ser considerado "resolvido" via DT-008 porque seu
bloqueio real é o DT-006 (dependência de `constants.js` ainda não anexado),
não mais um problema de path relativo.

### Pendências reais para sessões futuras

- **DT-006** (não novo, reafirmado): anexar/migrar `constants.js` real para
  `src/config/constants.js`. É o único bloqueador confirmado de
  `paths.test.js`.
- Corrigir comentário desatualizado sobre profundidade de path em
  `environment.test.js` (linhas 8–13) na próxima vez que o arquivo for
  tocado.
- DT-009 e a pendência de estrutura do bot WhatsApp (seção 14) seguem
  abertas, inalteradas por esta sessão.

---

## 16. Sessão de resolução — DT-006, `src/config/constants.js` (2026-07-03)

Escopo: criar o `constants.js` real do projeto, resolvendo o bloqueio
confirmado de `paths.test.js` (DT-006), sem tocar em nenhuma decisão
arquitetural já aprovada. Análise feita **antes** de qualquer edição:
leitura completa de SPEC.md, de todo o PROJECT_CONTEXT.md (incluindo o
histórico de sessões), do destructuring exato em `paths.js` (linhas
55–65) e dos valores literais esperados em `paths.test.js`. Suíte
completa executada como baseline **antes** de qualquer mudança, para
confirmar o estado real (não presumido) do projeto.

### Baseline real (antes da mudança)

```
node src/__tests__/paths.test.js         → MODULE_NOT_FOUND ('../config/constants')
node src/__tests__/environment.test.js   → MODULE_NOT_FOUND ('../../config/constants.draft.js')
node src/__tests__/logger.test.js        → 25/25 passed, exit 0
node src/__tests__/classifier.test.js    → 80/80 passed, exit 0
```
Confirma exatamente o que já estava documentado nas seções 9 e 15 — nenhuma
surpresa na baseline.

### Decisão: escopo de `constants.js` limitado ao contrato de `paths.js`

Avaliado explicitamente se `KNOWN_BINARIES`, `CATEGORIES`, `SUBSYSTEMS` e
`RISK_LEVELS` deveriam migrar de `constants.draft.js` para `constants.js`
nesta mesma sessão. **Decisão: não migrar.** Motivos:

1. Nenhum desses quatro tem, hoje, um contrato de consumo que exija estar
   em `constants.js` — `environment.js` e `classifier.js` já funcionam
   corretamente importando do rascunho, com 27/27 e 80/80 testes passando
   respectivamente antes e depois desta sessão.
2. A seção 15 (sessão anterior) já havia avaliado e **rejeitado** uma
   proposta de proxy/atalho envolvendo `constants.js`, justamente para
   não misturar esse escopo com a resolução pontual de DT-006. Reabrir
   essa decisão sem fato novo contradiria a própria seção 15.
3. Migrar tudo de uma vez fundiria DT-006 (bloco de `paths.js`, com
   contrato fechado e não-ambíguo) com DT-007 (blocos de `environment.js`
   e `classifier.js`, que exigem atualizar quatro pontos de import
   diferentes — `environment.js`, `environment.test.js`, `classifier.js`,
   `classifier.test.js` — e cuja migração o próprio DT-007 já descreve
   como "heterogênea", não um bloco único).
4. Princípio já adotado no projeto (ver decisão sobre API mínima em
   `logger.js`, seção 5): não fazer mudança especulativa sem caso de uso
   real e imediato.

Consequência: DT-006 tratada como **resolvida** (bloco de `paths.js`).
DT-007 **permanece pendente**, sem alteração de escopo.

### Implementação

**Arquivo novo: `src/config/constants.js`**
- Exporta exatamente `ENGINE_DIR_NAME`, `RUNS_DIR_NAME`, `LOCK_FILE_NAME`,
  `TMP_DIR_NAME`, `BACKUP_DIR_NAME`, `ARTIFACT_FILE_NAMES`,
  `RUN_ID_PATTERN`, `RUN_SEQUENCE_MIN`, `RUN_SEQUENCE_MAX` — o
  destructuring exato de `paths.js` linha 55–65, nem mais nem menos.
- Valores conferidos um a um contra `paths.test.js`: `ENGINE_DIR_NAME =
  ".adturbo-engine"`, `RUNS_DIR_NAME = "runs"`, `LOCK_FILE_NAME =
  "engine.lock"`, `TMP_DIR_NAME = "tmp"`, `BACKUP_DIR_NAME = "backup"`,
  `ARTIFACT_FILE_NAMES` com as 9 chaves usadas em
  `runArtifactPaths` (`MANIFEST`, `INVENTORY`, `ENVIRONMENT`, `HASHES`,
  `FINGERPRINT`, `PLAN`, `REPORT`, `ROLLBACK`, `FULL_BACKUP_ARCHIVE`),
  `RUN_ID_PATTERN = /^\d{8}-\d{3}$/`, `RUN_SEQUENCE_MIN = 1`,
  `RUN_SEQUENCE_MAX = 999`.
- Puramente declarativo: sem lógica, sem imports, sem I/O. Todos os
  objetos com `Object.freeze`, mesma convenção de `constants.draft.js`.
- JSDoc no cabeçalho documenta explicitamente por que `KNOWN_BINARIES` e
  as constantes de `classifier.js` **não** estão neste arquivo (aponta
  para DT-007), para que uma sessão futura não presuma omissão acidental.

**Arquivo modificado: `src/__tests__/environment.test.js`**
- Único import corrigido: `require("../../config/constants.draft.js")`
  (dois níveis, quebrado — bug de profundidade já apontado como pendência
  na seção 15) → `require("../config/constants.draft.js")` (um nível,
  correto). Comentário do import atualizado para não confundir esta
  correção com a migração de DT-007 (o import continua apontando para o
  rascunho, não para `constants.js` — `KNOWN_BINARIES` não migrou).
- **Nenhuma outra linha do arquivo foi tocada.**

**Nenhum outro arquivo foi modificado.** Confirmado por diff binário
contra o ZIP original desta sessão: `paths.js`, `environment.js`,
`classifier.js`, `logger.js`, `constants.draft.js`, `paths.test.js`,
`logger.test.js` e `classifier.test.js` permanecem byte-a-byte
idênticos.

### Validação real executada nesta sessão

```
node --check src/config/constants.js              → OK
node --check src/__tests__/environment.test.js     → OK
node src/__tests__/paths.test.js                   → 30 passed, 0 failed, exit 0
node src/__tests__/environment.test.js             → 27 passed, 0 failed, exit 0
node src/__tests__/logger.test.js      (regressão) → 25 passed, 0 failed, exit 0
node src/__tests__/classifier.test.js  (regressão) → 80 passed, 0 failed, exit 0
```

**Total real: 162/162 testes passando, exit code 0 em todos os arquivos.**
Nenhuma regressão introduzida em `logger.js`/`classifier.js` (números
idênticos aos da baseline, antes e depois).

### Estado final

- **DT-006 — ✅ Resolvida.** `paths.test.js` destravado (30/30). `constants.js`
  aprovado nas seções 3, 6 e 10.
- **DT-007 — pendente, sem alteração de escopo.** `KNOWN_BINARIES`,
  `CATEGORIES`, `SUBSYSTEMS`, `RISK_LEVELS` continuam em
  `constants.draft.js`, com decisão explícita de não migrar nesta sessão
  (ver acima).
- DT-003, DT-004, DT-005, DT-009 e a pendência de estrutura do bot
  WhatsApp — inalteradas.
- DT-008: nenhuma mudança de status adicional além da já registrada na
  seção 15 (o item de `environment.test.js` que restava ali era
  justamente esta correção de path, agora aplicada e verificada).

Engine considerada pronta para iniciar `src/modules/inventory.js`: suas
dependências diretas (`paths.js`, `hash.js`, `classifier.js`) estão todas
✅ aprovadas, com testes reais passando, e nenhuma delas depende da
migração pendente de DT-007.

Próximo módulo da fila: `src/modules/inventory.js`.

---

## 17. Sessão de implementação — `src/modules/inventory.js` (2026-07-03)

Escopo: implementar `inventory.js` seguindo o processo padrão do
projeto — análise de SPEC.md + PROJECT_CONTEXT.md, definição de
arquitetura antes de código, levantamento de decisões pendentes ao
usuário, só então implementação e testes. Nenhuma alteração autorizada
em módulos já aprovados, salvo bug real comprovado por execução.

### Fase 1 — Análise

Releitura completa de SPEC.md §7 (formato de `inventory.json`), do
destructuring exato de `paths.js` (reaproveitado, não modificado), da
API pública de `hash.js` (`hashFile`, assíncrona) e `classifier.js`
(`classify`, síncrona/pura). Identificado um ponto de tensão real entre
documentos: SPEC.md §7 descreve `inventory.json` incluindo
`exports`/`imports`/`syntaxValid`/`parseError`, mas PROJECT_CONTEXT.md
(seção 14) já reserva explicitamente "decidir o que o arquivo contém,
via AST" para `fingerprint.js`, ainda não implementado. Não presumido:
levado ao usuário como decisão explícita antes de codar.

Confirmado por listagem real de disco (`find . -type f -name "*.js"`)
que `src/modules/__tests__/hash.test.js`, documentado em
PROJECT_CONTEXT.md como "15/15 passando", **não existe** no ZIP desta
sessão. Registrado como **DT-010** (nova), fora do escopo autorizado
desta sessão (só `inventory.js`) — não reconstruído.

### Fase 2 — Decisões levantadas ao usuário (3 perguntas, todas respondidas)

1. **Fronteira SPEC §7 vs. fingerprint.js:** decidido que `inventory.js`
   emite apenas os campos que não exigem AST (`path`, `sizeBytes`,
   `sha256`, `lastModified`, `category`/`subsystem`/`risk` via
   `classifier.js`) e **omite** `exports`/`imports`/`syntaxValid`/
   `parseError`, documentando a lacuna até `fingerprint.js` existir.
2. **Concorrência do walk + hash:** decidido processamento **sequencial**
   (`for...of` + `await`), mesmo padrão conservador de
   `environment.js:detectEnvironment`.
3. **`hash.test.js` ausente:** decidido registrar como dívida técnica
   nova (DT-010) e seguir com `inventory.js` sem recriar o arquivo de
   teste ausente (fora do escopo desta sessão).

### Fase 3 — Implementação

**Arquivo novo: `src/modules/inventory.js`**
- API pública: `InventoryError`, `async function buildInventory(projectRoot, relativePaths)`.
- Recebe a lista de arquivos-alvo já resolvida por quem chama — não faz
  walk recursivo nem decide regras de include/exclude (nenhum documento
  do projeto fecha essas regras; decisão arquitetural não autorizada
  nesta sessão).
- Reaproveita, sem duplicar lógica: `paths.js:resolveWithinProject`
  (proteção contra path traversal), `classifier.js:classify`
  (categoria/subsistema/risco), `hash.js:hashFile` (SHA-256).
- Erro de input (`projectRoot`/`relativePaths` malformado) lança
  `InventoryError` com `code` estável (`INVALID_PROJECT_ROOT`,
  `INVALID_RELATIVE_PATHS`). Erro por arquivo individual (ausente, sem
  permissão, diretório em vez de arquivo, fora do projeto) nunca lança —
  fica estruturado em `item.error = { code, message }`, mesmo espírito
  de `environment.js` (ausência é fato de domínio, não exceção da API).
- Classificação roda sempre, mesmo para arquivo inexistente no disco —
  só `sizeBytes`/`sha256`/`lastModified` dependem da existência real do
  arquivo.

**Arquivo novo: `src/__tests__/inventory.test.js`**
- Mesma convenção de runner manual do projeto (sem dependências
  externas, `process.exit(1)` em falha).
- Projeto-alvo fake criado em `os.tmpdir()`/`fs.mkdtempSync` (mesmo
  padrão de `paths.test.js`), com limpeza (`fs.rmSync`) ao final.
- 25 testes cobrindo: validação de input, formato do resultado, metadados
  de arquivo existente (incluindo hash **verificado de forma
  independente** via `node:crypto` puro, não apenas formato/regex),
  classificação integrada, erros por arquivo (ausente, diretório,
  path traversal) sem exceção lançada, processamento misto
  sucesso/erro sem abortar, ausência confirmada dos campos reservados a
  `fingerprint.js`, e não-mutação do array de entrada.

**Nenhum outro arquivo foi modificado.** `classifier.js`, `hash.js`,
`paths.js`, `environment.js`, `logger.js`, `constants.js`,
`constants.draft.js` e todos os testes pré-existentes permanecem
byte-a-byte idênticos — confirmado por diff contra o ZIP original desta
sessão.

### Observação levantada durante a Fase 3 (comportamento de `classifier.js`, sem alteração)

Ao rodar um smoke-test inicial contra os próprios paths-fonte da Engine
(ex.: `"src/modules/paths.js"`), `classify()` retornou
`category: "unknown"` / `risk: "high"`. Investigado **antes de qualquer
alteração**, conforme instrução explícita do usuário nesta sessão:

- `SPEC.md`: nenhuma menção a classificar arquivos-fonte da própria
  Engine — todas as referências a `.adturbo-engine/` tratam do diretório
  de artefatos gerados em runtime, não do código-fonte da Engine.
- `PROJECT_CONTEXT.md`: nenhuma seção exige que `classifier.js`
  reconheça paths internos da Engine além do prefixo `"adturbo-engine/"`
  já implementado.
- `classifier.test.js` (80/80 passando, não alterado): o único caso
  testado e aprovado é `deriveSubsystem("adturbo-engine/src/modules/
  paths.js", ...) === SUBSYSTEMS.ENGINE` — path prefixado literalmente
  por `"adturbo-engine/"`, cenário de "Engine analisando a si mesma como
  projeto-alvo a partir de um `PROJECT_ROOT` pai". Já
  `"src/modules/__tests__/paths.test.js"` (linha 237 do teste) cai na
  regra `test` por conter `__tests__/`, sem relação com o subsistema
  `engine`.
- Um smoke-test com `PROJECT_ROOT` já apontando para a raiz da própria
  Engine nunca produz esse prefixo nos paths relativos — logo o
  resultado `unknown/high` é o fail-safe já documentado funcionando como
  projetado para qualquer path fora das convenções do Next.js App Router
  do AdTurbo, não uma lacuna.

**Conclusão: não há contrato explícito que exija a mudança. `classifier.js`
não foi modificado nesta sessão.** Registrado como nota (não como DT
nova, por não haver lacuna real) no cabeçalho de `inventory.js` e na
seção 6 deste documento, para que sessões futuras não confundam esse
comportamento com bug.

### Bug real corrigido (encontrado por execução, dentro do código novo desta sessão)

Erro retornado por `resolveWithinProject` (ex.: `code:
"PATH_OUTSIDE_PROJECT"`) estava recebendo um prefixo `"PATH_"` adicional
em `describeFile`, produzindo códigos redundantes como
`"PATH_PATH_OUTSIDE_PROJECT"`. Corrigido para usar `err.code`
diretamente. **Este bug estava no código escrito nesta própria sessão**
(não em módulo já aprovado) — não se enquadra na restrição de "só
corrigir módulos aprovados com bug comprovado", mas é registrado aqui
para rastreabilidade.

### Validação real executada nesta sessão

```
node --check src/modules/inventory.js               → OK
node --check src/__tests__/inventory.test.js         → OK
node src/__tests__/inventory.test.js                 → 25 passed, 0 failed, exit 0

Regressão completa da suíte (nenhum módulo aprovado tocado):
node src/__tests__/paths.test.js                     → 30 passed, 0 failed, exit 0
node src/__tests__/environment.test.js               → 27 passed, 0 failed, exit 0
node src/__tests__/logger.test.js                    → 25 passed, 0 failed, exit 0
node src/__tests__/classifier.test.js                → 80 passed, 0 failed, exit 0
node src/__tests__/inventory.test.js                 → 25 passed, 0 failed, exit 0
```

**Total real: 187/187 testes passando, exit code 0 em todos os arquivos.**
Nenhuma regressão. Confirmado por diff binário contra o ZIP original:
`paths.js`, `hash.js`, `environment.js`, `classifier.js`, `logger.js`,
`constants.draft.js` e os testes pré-existentes de `paths`, `logger` e
`classifier` permanecem idênticos.

### Estado final

- **`src/modules/inventory.js` — ✅ Aprovado.**
- **DT-010 — nova, pendente.** `hash.test.js` ausente do ZIP; fora do
  escopo desta sessão.
- DT-006 — permanece ✅ Resolvida (sessão anterior, inalterada).
- DT-007, DT-008, DT-009 — permanecem pendentes, sem alteração de
  escopo nesta sessão.
- `classifier.js` — permanece ✅ Aprovado, **não modificado** nesta
  sessão; comportamento observado documentado como nota, não como bug.

Próximo módulo da fila: `src/modules/fingerprint.js`.

**Sessão 17 consolidada e encerrada em 2026-07-03.** `inventory.js`
permanece ✅ Aprovado e não deve ser reaberto sem bug real comprovado por
execução. DT-010 permanece pendente, sem ação nesta consolidação.


---

## 18. Sessão de implementação — `src/modules/fingerprint.js` (2026-07-04)

Escopo: implementar `fingerprint.js` seguindo a mesma disciplina de
sessões anteriores — ler SPEC.md + PROJECT_CONTEXT.md, fechar o contrato
antes de codificar, levantar decisões pendentes ao usuário, só então
implementar e validar. Nenhuma alteração em módulos já aprovados
(`inventory.js` explicitamente confirmado como não-reaberto).

### Fase 1 — Análise: divergência arquitetural real encontrada

Releitura de SPEC.md §3 (arquitetura) revelou uma tensão não presumível:
a SPEC descreve 3 camadas — "Camada 1: Bash Engine" (inventário, backup,
ambiente, relatórios, orquestração pura, nunca edita JS) e "Camada 2:
Node Engine" (transformações via AST, Babel/Recast, um transformador por
arquivo `.mjs`, invocada por stdin/stdout JSON pela Camada 1). Na
prática, **nenhum módulo implementado até hoje é Bash** —
`paths.js`, `hash.js`, `logger.js`, `environment.js`, `classifier.js`,
`inventory.js` são todos Node/CommonJS puro, incluindo responsabilidades
que a SPEC atribui à Camada 1. Confirmado por grep em todo o
PROJECT_CONTEXT.md que essa divergência nunca havia sido decidida
explicitamente antes. Registrada como **DT-011** e levada ao usuário
antes de prosseguir — não presumida nem resolvida por conta própria.

**Decisão do usuário:** PROJECT_CONTEXT.md é a referência arquitetural
principal quando há conflito com SPEC.md. Manter CommonJS
(`require`/`module.exports`), não introduzir ESM (`.mjs`), não adicionar
dependências npm sem necessidade comprovada. Divergências devem ser
registradas no PROJECT_CONTEXT.md, não resolvidas criando uma
arquitetura nova. `fingerprint.js` implementado compatível com a
arquitetura atual da Engine (Node/CommonJS).

### Fase 1 (continuação) — Achado técnico crítico: `node --check` não confiável

Antes de desenhar a detecção de `syntaxValid`, testado por execução real
se `node --check` (usado em sessões anteriores como ferramenta de dev)
seria adequado para este contrato formal. Resultado, confirmado com
casos progressivamente mais realistas:

- `node --check arquivo.jsx` → falha com `ERR_UNKNOWN_FILE_EXTENSION`
  (falso positivo de "inválido" para um arquivo `.jsx` sintaticamente
  correto — Node não reconhece a extensão).
- `node --check arquivo.js` contendo `export`/`import` E um erro de
  sintaxe real mais adiante (`);` órfão) → **retornou exit 0 (válido)**.
  Confirmado isolando a causa: é a combinação específica de extensão
  `.js` + presença de `export`, que faz `wrapSafe` (carregador CJS do
  Node) tratar o arquivo de um jeito que não alcança o erro adiante no
  arquivo. `require()` real do mesmo arquivo lança `SyntaxError`
  corretamente, mas rodar `require()` executaria o módulo (efeito
  colateral inaceitável).
- Reproduzido com um caso **estruturalmente idêntico** a
  `app/api/produtos/route.js` (citado na própria SPEC §7): múltiplos
  imports, dois `export async function`, `);` órfão no final — mesmo
  padrão do incidente fundador da Engine (`adturbo-fix-auth.sh`) que
  motivou a existência do projeto. `node --check` aprovou esse arquivo
  como válido.

Esse achado foi levado ao usuário antes de prosseguir, com o exato
comando/saída que o comprova.

### Fase 2 — Protótipo isolado de `vm.SourceTextModule` (autorizado pelo usuário antes de qualquer código de produção)

Por instrução explícita: prototipar `vm.SourceTextModule` isoladamente e
validar 4 requisitos antes de adotá-lo, com a condição de que, se
qualquer requisito falhasse, o resultado seria uma DT nova — não uma
heurística improvisada.

**Resultado do protótipo (`/tmp/vm-proto`, descartado após a sessão):**

| Requisito | Resultado |
|---|---|
| 1. Detecta erros de sintaxe ESM corretamente (incluindo o caso que `node --check` errou) | ✅ Passou — reproduziu e detectou corretamente o `);` órfão no caso realista de `route.js`; detectou chave não fechada; ESM válido passou como válido |
| 2. Não executa o código do módulo | ✅ Passou — construir `SourceTextModule` sem chamar `.link()`/`.evaluate()` só faz parsing; um side effect global de teste não disparou |
| 3. Funciona apenas com Node Core | ⚠️ Parcial — a API é 100% Node Core (`node:vm`), mas **exige a flag `--experimental-vm-modules`** no processo que a invoca; confirmado por execução que a flag não pode ser ativada seletivamente via `require()` de um processo já iniciado sem ela (`require('vm').SourceTextModule` é `undefined` sem a flag no processo) |
| 4. Sem dependências npm | ✅ Passou — nenhum `package.json`/`node_modules` envolvido |

Por não ser 100% limpo (requisito 3), essa lacuna foi levada ao usuário
com a mitigação já testada (isolar em subprocesso via
`child_process.spawnSync`, mesmo padrão já aprovado em `environment.js`
para detecção de binários — confirmado por execução que o processo pai
não precisa da flag quando o subprocesso a recebe na sua própria
invocação). **Decisão do usuário:** adotar `vm.SourceTextModule` via
subprocesso isolado, aceitando o custo de spawn por arquivo. Registrado
como **DT-012** (decisão de design aceita, não bloqueante).

Também confirmado no protótipo: `vm.SourceTextModule` expõe
`mod.moduleRequests` (specifier/source de cada import), mas **não**
expõe nomes específicos importados nem lista de exports. Extrair
`imports[].names`/`exports` exigiria AST completo (Babel, vetado por
zero dependências). Regex sobre texto-fonte foi avaliada e descartada
por fragilidade real (comentários contendo "import", strings
multilinha, imports quebrados em várias linhas) — o próprio tipo de
heurística improvisada que o usuário instruiu a evitar. **Decisão do
usuário:** escopo reduzido — `fingerprint.js` produz `syntaxValid`,
`parseError` e `imports[].source`; **não** produz `exports` nem
`imports[].names`, documentado como decisão de escopo, não lacuna.

### Fase 3 — Implementação

**Arquivo novo: `src/modules/fingerprint-worker.js`**
- Script interno, não é API pública (não deve ser `require()`'d por
  outros módulos). Roda como processo filho isolado, invocado por
  `fingerprint.js` via `spawnSync` sempre com
  `--experimental-vm-modules`.
- Lê o código-fonte via stdin, constrói `vm.SourceTextModule` (sem
  `link()`/`evaluate()`), escreve um único JSON em stdout:
  `{ syntaxValid, importSources }` ou `{ syntaxValid: false,
  importSources: [], parseError }`.

**Arquivo novo: `src/modules/fingerprint.js`**
- API pública: `FingerprintError`, `async function
  fingerprintFile(absolutePath)`.
- Recebe path absoluto já resolvido por quem chama (mesmo padrão de
  `paths.js`/`hash.js` — não faz resolução de path relativo nem
  verificação de path traversal, isso é responsabilidade de
  `paths.js`).
- Lê o arquivo com `fs.readFileSync`, passa o conteúdo via stdin ao
  subprocesso (`fingerprint-worker.js`), mapeia o resultado.
- Erro de input lança `FingerprintError`. Erro de leitura de arquivo ou
  de infraestrutura do subprocess (spawn falhou, timeout via
  `SUBPROCESS_TIMEOUT_MS = 10_000`, saída não é JSON válido, processo
  morto por sinal) nunca lança — fica estruturado em `result.error`.

**Arquivo novo: `src/__tests__/fingerprint.test.js`**
- Mesma convenção de runner manual do projeto.
- Fixtures reais em `os.tmpdir()`/`fs.mkdtempSync` (mesmo padrão de
  `inventory.test.js`), com limpeza ao final.
- 23 testes cobrindo: validação de input, sintaxe válida com imports,
  o caso crítico do `);` órfão no padrão exato do incidente fundador,
  chave não fechada, arquivo vazio, CommonJS puro (sem import/export
  ESM), arquivo binário, arquivo inexistente, diretório em vez de
  arquivo, confirmação de não-execução do código analisado (side
  effect global não dispara), confirmação de que não há RCE via
  conteúdo com sintaxe de injeção de shell (tratado como dado puro via
  stdin, sem `shell: true`), e confirmação de que `exports`/
  `imports[].names` nunca aparecem no resultado (escopo reduzido).

**Nenhum outro arquivo foi modificado.** `paths.js`, `hash.js`,
`environment.js`, `classifier.js`, `logger.js`, `inventory.js`,
`constants.js`, `constants.draft.js` e todos os testes pré-existentes
permanecem byte-a-byte idênticos — confirmado por diff contra o ZIP
original desta sessão. `inventory.js`, especificamente, não foi
reaberto, conforme instrução explícita do usuário.

### Verificações de segurança executadas (não apenas assumidas)

- **Injeção de comando via conteúdo analisado:** testado com uma string
  no código-fonte contendo sintaxe de injeção de shell
  (`` `; touch marker; ` ``) — confirmado que nenhum comando foi
  executado, porque `spawnSync` recebe o conteúdo via `input` (stdin),
  sem `shell: true`, então a string nunca é interpretada por um shell.
- **Execução não intencional do código analisado:** testado com um
  arquivo que, se executado, setaria uma variável global — confirmado
  que a variável nunca foi setada, porque `vm.SourceTextModule` sem
  `link()`/`evaluate()` só faz parsing.
- **Timeout do subprocesso:** testado isoladamente que
  `spawnSync(..., { timeout })` realmente mata o processo filho
  (`result.error.code === "ETIMEDOUT"` e/ou `result.signal ===
  "SIGTERM"`), confirmando que os dois ramos de tratamento de erro no
  código (`WORKER_TIMEOUT`/`WORKER_KILLED`) são alcançáveis e corretos.

### Validação real executada nesta sessão

```
node --check src/modules/fingerprint-worker.js       → OK
node --check src/modules/fingerprint.js               → OK
node --check src/__tests__/fingerprint.test.js         → OK
node src/__tests__/fingerprint.test.js                 → 23 passed, 0 failed, exit 0 (656ms)

Regressão completa da suíte (nenhum módulo aprovado tocado):
node src/__tests__/paths.test.js                       → 30 passou(aram), 0 falhou(aram), exit 0
node src/__tests__/environment.test.js                 → 27 passed, 0 failed, exit 0
node src/__tests__/logger.test.js                      → 25 passed, 0 failed, exit 0
node src/__tests__/classifier.test.js                  → 80 passed, 0 failed, exit 0
node src/__tests__/inventory.test.js                   → 25 passed, 0 failed, exit 0
node src/__tests__/fingerprint.test.js                 → 23 passed, 0 failed, exit 0
```

**Total real: 210/210 testes passando, exit code 0 em todos os arquivos.**
Nenhuma regressão. `node --check` válido nos 10 arquivos de
módulo/config do projeto.

### Estado final

- **`src/modules/fingerprint.js` — ✅ Aprovado.**
- **`src/modules/fingerprint-worker.js` — ✅ Aprovado** (script interno,
  não é API pública).
- **DT-011 — nova, registrada, não bloqueante.** Divergência SPEC.md
  (arquitetura 3 camadas, Bash + Babel/ESM) vs. implementação real
  (Node/CommonJS unificado). Decisão do usuário: PROJECT_CONTEXT.md é a
  referência corrente; SPEC.md permanece como documento historicamente
  desatualizado, sem reescrita da arquitetura real para "encaixar" nela.
- **DT-012 — nova, aceita como decisão de design, não bloqueante.**
  `vm.SourceTextModule` exige `--experimental-vm-modules`; mitigado via
  subprocesso isolado. Ação futura opcional: batelar múltiplos arquivos
  por invocação do subprocesso, se o custo de spawn por arquivo se
  mostrar proibitivo em projetos grandes.
- DT-006, DT-007, DT-008, DT-009, DT-010 — sem alteração de status
  nesta sessão.
- `inventory.js`, `classifier.js` e os demais módulos aprovados —
  **não modificados**, conforme instrução explícita.

Próximo módulo da fila: `src/modules/manifest.js`.
