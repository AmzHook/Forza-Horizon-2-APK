# Forza Horizon 2 Recomp — Port Android (RexGlue)

> [!WARNING]
> **Este repositório NÃO contém nenhum arquivo do jogo.** Nenhum binário
> `.xex`, ISO, textura, áudio, carro ou mapa de Forza Horizon 2 é distribuído
> aqui. O projeto assume que **você possui uma cópia legalmente adquirida do
> jogo** e fornece os seus próprios arquivos em runtime (via Storage Access
> Framework, na primeira execução do app).

Port Android (arm64-v8a) de **Forza Horizon 2** (Xbox 360) via **recompilação
estática** com o [rexglue-SDK](https://github.com/rexglue/rexglue-sdk) — o
código PowerPC do `default.xex` (+ `SpeechFacade_default.xex` e
`XMediaFacade_default.xex`) é convertido para C++ nativo em tempo de build e
executado diretamente no Android, sem emulação interpretada.

Homebrew/educacional. **Não é afiliado à Microsoft, Playground Games, Turn 10
Studios ou Xbox.** Todas as marcas e direitos autorais pertencem aos seus
respectivos proprietários.

## Status do projeto

| Área | Estado |
|---|---|
| Pipeline de recompilação (host) | ✅ Funcional — 643 arquivos C++ gerados dos 3 binários |
| Build CI (APK) | ✅ Automática por push (GitHub Actions) |
| Runtime Android (janela, input, áudio, FS, Vulkan) | ✅ Infra herdada do modelo de port comprovado |
| **Execução do jogo de ponta a ponta** | ⚠️ **Em progresso** — exige testes em device real e iteração de kernel HLE (ver `docs/BACKLOG.md`) |

Este é um port em fase inicial: a **infraestrutura inteira está pronta e o
build de CI gera APKs assinados**, mas o comportamento do jogo em dispositivo
real (boot do título, streaming de mundo aberto, física de veículos, clima
dinâmico) depende de iteração de runtime que só acontece com testes em
hardware. Acompanhe a aba **Actions** e o `docs/BACKLOG.md`.

## Requisitos de dados do jogo (assets)

O port **não inclui** nenhum arquivo do jogo. Em primeira execução o app pede:

| Arquivo | Para quê | De onde |
|---|---|---|
| ISO do Forza Horizon 2 (ou pasta extraída com `default.xex`) | Runtime (conteúdo do jogo) | **você fornece** no app, via seletor de arquivos |

Os binários `.xex` usados pelo **codegen** (build do recomp) são baixados do
repositório privado do portador pelo CI, via `secrets.GH_PAT` — nunca são
commitados nem empacotados no APK (ver `.gitignore`).

## Downloads

O APK é construído automaticamente pelo GitHub Actions em cada push:
aba **Actions** → build → artefato `forzahorizon2-recomp-apk`
(`ForzaHorizon2-Recomp-arm64-release.apk` / `-debug.apk`).

Requisitos de device: **Android 8.0+ (API 26+)**, arm64-v8a, GPU Vulkan
(Adreno/Mali/Turnip). Gamepads Bluetooth/USB suportados (mapeamento Xbox 360
padrão, triggers analógicos para acelerador/freio).

## Recursos do port

- **Renderer Vulkan nativo** do SDK rexglue (Xenia-derived) —
  `VK_KHR_android_surface` sobre a janela SDL3
- **Virtual gamepad translúcido** (padrão Xbox 360) com ajuste de
  opacidade/tamanho e haptics — layout pensado para direção: **LT/RT
  analógicos no touch** (rampa ao longo da pílula, com feedback visual de
  preenchimento); analógicos de verdade em gamepad físico
- **Gamepads Bluetooth/USB** via pipeline HID do SDL3
- **Boot in-place do ISO** (modelo XenDroid): a imagem é montada onde está,
  sem cópia e sem extração (zero espaço adicional); pasta extraída também é
  aceita
- **Driver Turnip customizado (AdrenoTools)**: carregado via `libadrenotools`
  — falha de carregamento cai no driver do sistema sem crash
- **Log detalhado persistido** em
  `/storage/emulated/0/Forza Horizon 2 Recomp/logs/` (fallback no storage
  privado) + crash handler nativo com backtrace em `last_crash.txt`
- **Caches de shader/pipeline** no storage privado do app
- **Limiter de FPS real** (`fps_cap`): pacing por software no presenter
  (30/60/90/120/∞), aplicado ao vivo pelo painel rápido (4 dedos)
- **Refresh do modo de vídeo guest** (`video_mode_refresh_rate`, cvar do
  SDK) nas Configurações
- **Painel rápido (4 dedos)**: FPS cap, contador de FPS, controles, sair
- Tela de seleção de dados com parallax/partículas (paleta FH2) e créditos

## Arquitetura

```
app/                      Kotlin + Compose (tela inicial, settings, SAF, HUD)
native/CMakeLists.txt     libfh2.so (entrypoint+glue+JNI) + 2 facades .so
native/jni/               SDL_main → cvar::Init → SDLWindowedAppContext → fh2
native/overlay/           arquivos Android sobrepostos ao submódulo do SDK
recomp/                   PROJETO DO JOGO: fh2_manifest.toml + src/ + assets/
recomp/generated/         saída do codegen (gerada no host/CI, não versionada)
rexglue-sdk/              (submódulo) SDK rexglue v0.10.0 (Xenia-derived, BSD-3)
native/thirdparty/libadrenotools  (submódulo) driver Turnip
scripts/                  fetch_xex + codegen host + build NDK + overlays
docs/                     decisões técnicas, pipeline, backlog
```

Os módulos facade são `.so` separados (`libfh2_SpeechFacade_default.so`,
`libfh2_XMediaFacade_default.so`) — o runtime os carrega via `dlopen`
(`KernelState::LoadUserModule`), espelhando o carregamento por módulo do
Xbox 360; o bionic resolve os nomes no `nativeLibraryDir` do app
(`useLegacyPackaging=true`).

### Build local (Linux)

```bash
git clone --recurse-submodules https://github.com/deivid22srk/ForzaHorizon2Recomp-Android-RexGlue
cd ForzaHorizon2Recomp-Android-RexGlue

# 1) .xex do jogo (requer GH_TOKEN com acesso ao repo privado)
export GH_TOKEN=<PAT com escopo repo>
bash scripts/fetch_xex.sh

# 2) Codegen no host (build do CLI rexglue + conversão PPC → C++)
bash scripts/build_host_codegen.sh

# 3) Build nativo Android (NDK arm64; ANDROID_NDK_HOME apontando para o NDK)
bash scripts/apply_overlays.sh
bash scripts/build_android_native.sh

# 4) APK final
./gradlew :app:assembleDebug
```

Variáveis: `ANDROID_NDK_HOME` (ou `ANDROID_HOME`), `ABI` (padrão arm64-v8a),
`API` (padrão 26), `FORCE_CODEGEN=1` para regenerar o recomp.

O codegen é executado **automaticamente no CI** a cada push — builds locais
são opcionais.

## CI/CD (`build.yml`)

1. Checkout com submódulos (`rexglue-sdk`, `libadrenotools`)
2. Clang 19 no host (requisito do SDK) + JDK 17 + NDK 27.2
3. Overlays Android sobre o SDK
4. Testes unitários Kotlin (falham rápido)
5. Download dos `.xex` do repo privado via `secrets.GH_PAT` + codegen
6. Build nativo (libs → `jniLibs/`) + APKs assinados (keystore versionado no
   repo, padrão do modelo de port — não usar na Play Store)
7. Verificação fail-fast das libs no APK + upload dos artefatos e símbolos

## Créditos

- **Tom (crack) e comunidade** — [rexglue-SDK](https://github.com/rexglue/rexglue-sdk)
- **Xenia** e **XenonRecomp** — fundamentos do recomp Xbox 360
- **SDL3 (Simple DirectMedia Layer)** — janela/input no Android; as classes
  Java `org.libsdl.app` são vendadas sob a **zlib License** (© SDL contributors)
- **libadrenotools (bylaws)** — carregamento de driver Turnip custom
- **Hailgames (deivid22srk)** — port Android, template de interface e pipeline
  (baseado no modelo comprovado do port de referência do mesmo autor)
- Arte de fundo e marca: originais do port (sem material do jogo)

## FAQ

**O jogo roda de verdade?** O pipeline de recompilação está completo e os
APKs carregam o motor com o código do jogo recompilado. O ajuste fino de
runtime (kernel HLE do FH2, streaming, performance) é um trabalho contínuo —
acompanhe `docs/BACKLOG.md` e as issues.

**Preciso desinstalar entre versões?** Não — todas as builds saem com a
mesma assinatura (keystore versionado no repo); upgrades são in-place.

**Vou ser banido?** Este projeto não conecta a serviços online da Microsoft.

**Como confiro a integridade do APK?** Cada run de CI imprime o `SHA-256`
dos APKs no log (step "Print APK SHA-256") e empacota o `SHA256SUMS.txt`
junto do artefato. Como o keystore é público (modelo de distribuição do
port), confira o hash antes de instalar um APK de terceiros.

**Posso contribuir?** Sim — issues de "Unimplemented instruction"/kernel
export do log são o melhor ponto de partida (ver `docs/RECOMPILATION.md`).
