# Changelog — ForzaHorizon2Recomp-Android-RexGlue

## v0.1.6 (2026-09-17)

Ciclo de iteração de runtime #7 (sétima sessão em device, log via pastebin;
build da iteração 8, run #16).

- **Confirmação da iteração 8 em device**: 85.471 funções registradas; o
  boot passou da enumeração de conteúdo e o **pipeline de áudio entrou em
  operação real** — `AudioWorker` despachando o callback guest `831B1D50`
  continuamente (`XAudioSubmitRenderDriverFrame` por frame,
  `SDLAudioDriver::SubmitFrame` consumindo `queued_count` 1..8). Init de
  mídia avançou por `\Media\Audio\Cars\*`, fontes `\media\ui\fonts` e
  `XMPGetPlaybackController`.
- **Iteração 9 de codegen — SWEEP GLOBAL de 81 novos alvos** no
  `[entrypoint.functions]` (148 no total). Evidência: `[FATAL] Call to
  invalid or unregistered function at guest address 0x831E77E0` a partir de
  `sub_82C1FEB8` (+644) via `bctrl` em vtable (slot `0x822C1974`).
- **`tools/audit_vtable_family.py` v3 — auditor extents-aware reescrito**
  para tagging em massa seguro:
  - **extents exatos** das 85.061 funções registradas extraídos dos .cpp
    gerados (`scripts/extract_extents.py` → `extents_main.json`;
    validado contra casos conhecidos — o heuristic antecessor da v2
    aceitava bytes mid-function escondidos atrás de `b` forward
    intra-função e foi aposentado);
  - plausibilidade de primeira instrução (`bl`/`bctrl`/`bclr`/`mtctr`/
    branch condicional na entrada = sítio mid-function → rejeita);
  - aceitação em lote com detecção de overlap entre extents candidatos;
  - downgrade em ponto fixo de candidatos com `b`/`bl` direto a alvo não
    registrado dentro do extent (evita novos `UnresolvedCall`).
- **Scanner global**: 11 clusters de bancos de ponteiros
  `{funcptr,flag}`/vtables, 1.910 alvos alinhados não registrados → 102
  SAFE únicos → 81 novos tags após deduplicação com as iterações 1-8
  (o registry local ainda reflete o codegen base). Cada tag passou por:
  alinhamento 4, fora de todo extent registrado exato, entrada plausível,
  extent linear sem cruzar função registrada, sem chamada direta não
  resolvida, sem overlap entre candidatos. Mapa dos bancos restantes
  documentado em `docs/BACKLOG.md`.

## v0.1.5 (2026-09-17)

Ciclo de iteração de runtime #6 (evidências da sexta sessão em device,
`log5.zip`).

- **Confirmação da iteração 7**: 85.470 funções registradas (+21 exatas) e o
  boot passou da enumeração de conteúdo. **O pipeline de áudio entrou em
  operação pela primeira vez**: `XAudioRegisterRenderDriverClient`,
  `AudioSystem::RegisterClient` (callback 831B1D50), endpoint
  "System audio playback device" 6 ch @ 48 kHz (formato 0x8120) e
  `SDLCallback` ativo (silêncio inicial, sem frames enfileirados).
- **Iteração 8 de codegen — 2 novos alvos tagados** no
  `[entrypoint.functions]`. Evidência: `[FATAL] Call to invalid or
  unregistered function at guest address 0x830387E0` a partir de
  `sub_83046DA0` (+1680), na thread principal durante a init de
  áudio/mídia (após probes de `\Media\Audio\Placeholder`).
- **Auditoria extents-aware** (`tools/audit_vtable_family.py`, janela
  `0x82264000..0x82265000`): 3 alvos não registrados, verificados
  individualmente —
  - `0x830387E0` (alvo do crash; slot `0x82264688`): thunk de 2 instruções
    (`li r5,1; b 0x83033468`, registrada), gap exato entre thunks — TAGADO;
  - `0x8303EE88` (slot `0x8226493C`): função real de 28 bytes (carrega o
    campo `0x80A0` do objeto, null-check, erro `0x1F`, `blr`), sem overlap
    com a próxima registrada (`sub_8303EEA4`) — TAGADO;
  - `0x83045378` (slot `0x82264AB8`): **REJEITADO** pelo filtro — código no
    meio de função existente (a extensão linear cruza `0x830453EC`, já
    registrada). Tagar quebraria a tradução — o filtro extents-aware
    funcionando exatamente como projetado; documentado no manifest para
    investigação futura da função contenedora (possível over-extension da
    análise estática).

## v0.1.4 (2026-09-17)

Ciclo de iteração de runtime #5 (evidências da quinta sessão em device,
`log4.zip`).

- **Confirmação da correção do walker CRT do XMediaFacade (iteração 6)**: a
  sessão passou da inicialização CRT do módulo — o boot avançou para a fase
  de enumeração de conteúdo na thread principal (varredura de `\media`,
  `XamContentCreateEnumerator`).
- **Iteração 7 de codegen — 21 novos alvos tagados** no
  `[entrypoint.functions]` (módulo principal). Evidência:
  `[FATAL] Call to invalid or unregistered function at guest address
  0x8319D238` a partir de `sub_8319CAC8` (+4408 no backtrace nativo), logo
  após `XamContentCreateEnumerator: added 0 items to enumerator` — chamada
  virtual via vtable para função não descoberta pela análise estática.
- **Auditoria extents-aware da FAMÍLIA inteira de vtables** (método previsto
  no backlog desde a iteração 5 e agora executado pela primeira vez): os
  slots referenciados formam 5 cópias de vtables de classes de enumeração de
  conteúdo (dados em `0x8229E594..0x8229EA34`). A varredura da região
  contra a tabela de 85.449 funções registradas encontrou 21 alvos únicos
  não registrados, todos verificados endereço a endereço:
  - 18 thunks de 2 instruções (`addi r3,r3,-4; b <alvo>`), com todos os
    alvos de desvio já registrados e cada thunk em gap entre irmãos da
    mesma família de 8 bytes;
  - 3 funções reais (prólogo limpo, extensão confirmada até o terminador
    `bctr` — tail-call de método virtual com IID `0x1337F001`, família
    QueryInterface): `0x8319D238` (o alvo do crash), `0x8319E478` e
    `0x83198118`.
- Nenhum dos 21 endereços cai dentro da extensão de uma função registrada —
  o modo de falha da iteração 5 (2.528 "Unresolved conditional branch") está
  estruturalmente excluído. Tagar a família completa de uma vez elimina 2-3
  rodadas de device que crashariam nos slots irmãos (`D228`, `D230`,
  `E448`...) na mesma sequência de boot.

## v0.1.3 (2026-09-17)

Ciclo de iteração de runtime #4 (evidências da quarta sessão em device,
`log3.zip`).

- **Confirmação da correção do dlopen (run #12)**: o XMediaFacade agora é
  carregado e seu código executa — a sessão 3 de device chegava a
  "Failed to load shared library for module 'xmediafacade_default.xex'" e
  abortava; a sessão 4 passou desse ponto e registrou 3.196 funções do
  módulo.
- **Iteração 6 de codegen — 1 novo alvo tagado** no `[modules.functions]`
  (XMediaFacade): `0x882435D8`. O walker de construtores CRT do próprio
  módulo (`sub_881E8D88`, homólogo de `sub_82BFF9E8` do módulo principal)
  percorre tabelas de ponteiros em `0x88250000..0x88250028` e chama cada
  entrada via `bctrl`; `0x882435D8` era a única entrada não registrada
  (evidência: `[FATAL] Call to invalid or unregistered function at guest
  address 0x882435D8` durante o `LoadUserModule`, crash em
  `function_dispatcher.cpp:39` + SIGABRT).
- **Análise proativa do SpeechFacade** com o mesmo método (walkers
  `sub_891F0058`/`sub_891F0138`, tabelas em `0x89210000..0x89210094` +
  single-word `0x8900068C`): 32/32 alvos já registrados — nenhuma tag
  necessária. Evita uma 5ª sessão de device apenas para descobrir o mesmo
  crash no próximo facade a ser carregado.
- Segurança da tag verificada contra a lição da iteração 5: `0x882435D8`
  está em gap entre `sub_88242BC0` e `sub_882435E8`, é um thunk real de 3
  instruções (addis/addi/b) e o fluxo da função antecessora desvia antes
  dele (bc condicional + b incondicional) — não há risco de quebrar a
  tradução de função existente.

## v0.1.2 (2026-09-17)

Ciclo de iteração de runtime #3 (evidências da terceira sessão em device).

- **Correção de carregamento dos módulos facade**: o `module_registry`
  gerado registra `fh2_XMediaFacade_default` sem prefixo/sufixo e o bionic
  não normaliza nomes no `dlopen` — em device o boot chegava ao
  `XexLoadImage` dos facades e abortava com "Dirty Disc" (evidência:
  `log2.zip`, terceira sessão). O overlay
  `native/overlay/rexglue-sdk/src/core/dynlib_posix.cpp` agora retenta com
  `lib<nome>.so` (forma empacotada no jniLibs); validado em host com
  biblioteca de teste (5/5 checks).
- Rodada 2 da avaliação crítica independente: **9.2/10** — resíduos
  documentais corrigidos (`release.yml` citado 4x em comentários do
  `build.gradle.kts` + afirmação falsa de histórico de assinatura;
  `vblank_hz` em comentários do `PortSettings.kt`; `on_swap` no
  `NativeBridge`; `RestuffCrashHandler` no `android_main`; issues #1-#4
  citadas por número nos docs).
- Confirmação da iteração 5 em device: 85.449 funções registradas, walker
  de construtores CRT passou, chamadas de kernel reais executando.

## v0.1.1 (2026-09-17)

Ciclo de iteração de runtime #2 (evidências da segunda sessão em device).

- **Iteração 5 de codegen — 29 novos alvos tagados** no
  `recomp/fh2_manifest.toml`: todas as entradas não registradas das tabelas
  de construtores CRT percorridas pelo walker `sub_82BFF9E8` no xstart
  (evidência: crash `[FATAL] … 0x83243770` em device real). A análise foi
  feita com as novas ferramentas `tools/xex_extract.py` +
  `tools/scan_indirect_targets.py` — encerra a dinâmica de um endereço por
  rodada de device para esta classe de crash.
- `tools/` + `docs/RECOMPILATION.md §3.1`: método documentado para alvos de
  chamadas indiretas orientadas a dados (tabelas em DATA), com a lição do
  experimento rejeitado de tagamento em massa de vtables (2.528
  `Unresolved conditional branch` → plano extents-aware no backlog).
- `docs/BACKLOG.md`: evidências da segunda sessão (ISO montada in-place com
  2.996 arquivos, 85.420 funções registradas, imports krnl/xam patcheados),
  baseline de warnings de codegen e plano de cobertura extents-aware.

## v0.1.0 (2026-09-17)

Primeira versão pública do port Android.

- Projeto Android completo (Kotlin + Jetpack Compose + SDL3) adaptado do modelo
  de port comprovado do `naughtybear-restuff-android`.
- Pipeline de recompilação RexGlue end-to-end: `rexglue init`/`init module` +
  `codegen` do `default.xex` (entrypoint) + `SpeechFacade_default.xex` +
  `XMediaFacade_default.xex` — 643 arquivos C++ gerados, análise PowerPC
  validada (16 funções tagadas manualmente no `recomp/fh2_manifest.toml`,
  incluindo a evidência de device real 0x83243750).
- Build nativo Android (arm64-v8a): `libfh2.so` (entrypoint + glue + JNI),
  `libfh2_SpeechFacade_default.so`, `libfh2_XMediaFacade_default.so`,
  `librexruntime.so` e AdrenoTools (driver Turnip custom) — sem shaderc
  (o SDK de recompilação não usa GLSL em runtime).
- Limiter de FPS real: cvar `fps_cap` definido no overlay do presenter com
  pacing por software em `PaintAndPresentImpl` (30/60/90/120/∞ ao vivo);
  `video_mode_refresh_rate` (cvar do SDK) exposto como refresh do modo de
  vídeo guest; gatilhos LT/RT do HUD touch são analógicos (rampa ao longo
  da pílula).
- CI GitHub Actions: baixa os .xex do repositório privado via
  `secrets.GH_PAT`, roda codegen no host, compila o nativo e assina os APKs
  (release/debug) — artefatos por run.
- UI de primeira execução com seleção de ISO/pasta via SAF (boot in-place,
  modelo XenDroid), HUD de controles touch padrão Xbox 360 com
  opacidade/tamanho/haptics, gamepads Bluetooth/USB, painel rápido
  (4 dedos), diagnóstico e logs persistentes.
