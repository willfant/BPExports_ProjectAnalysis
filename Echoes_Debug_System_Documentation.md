# Echoes Between Worlds — Debug System Documentation

## Objetivo

Este documento descreve o novo sistema global de debug do projeto **Echoes Between Worlds**. O objetivo é substituir `Print String` soltos por uma pipeline profissional, centralizada e reutilizável.

O sistema permite:

- enviar mensagens de debug a partir de qualquer Blueprint;
- identificar automaticamente quem enviou a mensagem;
- filtrar por categoria;
- controlar severidade e verbosity;
- evitar spam com `Once` e `Throttle`;
- guardar histórico;
- mostrar mensagens em widgets/overlay;
- trocar perfis de debug através de Data Assets;
- manter os Blueprints mais limpos através de macros `DBG_*`.

---

# 1. Arquitetura geral

A pipeline principal é:

```text
Blueprint qualquer
-> BFL_Debug ou Macro DBG_*
-> GI_Echoes
-> BP_DebugManager
-> WBP_DebugFeedPanel
-> WBP_DebugLine
```

## 1.1 `BFL_Debug`

É a fachada pública do sistema. Contém as funções que os Blueprints usam para enviar debug:

- `DebugMessageAdvanced`
- `DebugMessage`
- `DebugInfo`
- `DebugWarning`
- `DebugCritical`
- `DebugOnce`
- `DebugThrottle`
- `DebugLocation`
- `DebugStateChange_Name`
- `GetSenderName`

Regra prática: os Blueprints normais devem chamar a `BFL_Debug` ou as macros `DBG_*`, não a lógica interna do `GI_Echoes`.

## 1.2 `GI_Echoes`

É o cérebro do sistema. Recebe mensagens, aplica filtros, preenche contexto, controla once/throttle, guarda histórico e envia a mensagem para log, screen e overlay.

Funções principais:

- `SetInitialVarsDebug`
- `LoadDefaultDebugProfile`
- `ApplyDebugProfileAsset`
- `RegisterDebugManager`
- `UnregisterDebugManager`
- `IsDebugMessageAllowed`
- `StampDebugContext`
- `ProcessOnceAndThrottle`
- `AppendDebugHistory`
- `RouteDebugMessage`
- `SubmitDebugMessage`
- `SetDebugSystemEnabled`
- `SetCategoryEnabled`
- `MuteDebugSender`
- `UnmuteDebugSender`
- `ClearDebugHistory`
- `ClearDebugRuntimeCaches`
- `BuildFormattedDebugString`
- `GetDebugColorForSeverity`

## 1.3 `BP_DebugManager`

É a camada visual. Vive no level e recebe mensagens aceites pelo `GI_Echoes`.

Responsabilidades:

- fazer cast para `GI_Echoes`;
- registar-se no `GI_Echoes`;
- fazer bind a `OnDebugMessageAccepted`;
- fazer bind a `OnDebugSettingsChanged`;
- criar widgets de debug;
- encaminhar mensagens para os feeds;
- reconstruir os feeds a partir do histórico;
- mostrar/esconder painéis.

Funções principais:

- `EnsureWidgetsCreated`
- `CreateMainFeedWidget`
- `CreateScareFeedWidget`
- `CreateVoiceFeedWidget`
- `ApplyFeedViewportLayout`
- `HandleDebugMessageAccepted`
- `ForwardMessageToWidgets`
- `RefreshWidgetsFromHistory`
- `ClearAllFeeds`
- `ToggleMainDebugFeed`

## 1.4 `WBP_DebugFeedPanel`

É o painel de mensagens. Cada feed tem título e categorias permitidas.

Exemplos:

- `Debug All`
- `Debug Scares`
- `Debug Voice`

Funções principais:

- `InitFeedPanel`
- `PushMessage`
- `ClearFeed`
- `FormatDebugTime`

## 1.5 `WBP_DebugLine`

É uma linha individual de mensagem.

Função principal:

- `SetupRow`

Mostra:

- sender;
- hora;
- mensagem;
- cor/barra por severidade.

---

# 2. Fluxo completo da mensagem

## 2.1 Fluxo com função normal

```text
BP_PuzzleController
-> DebugInfo
-> DebugMessageAdvanced
-> Get Game Instance
-> Cast to GI_Echoes
-> SubmitDebugMessage
-> IsDebugMessageAllowed
-> StampDebugContext
-> ProcessOnceAndThrottle
-> AppendDebugHistory
-> RouteDebugMessage
-> OnDebugMessageAccepted
-> BP_DebugManager.HandleDebugMessageAccepted
-> ForwardMessageToWidgets
-> WBP_DebugFeedPanel.PushMessage
-> WBP_DebugLine.SetupRow
```

## 2.2 Fluxo com macro nova

```text
BP_PuzzleController
-> DBG_Info
-> Self automático dentro da macro
-> DebugInfo
-> DebugMessageAdvanced
-> GI_Echoes
-> BP_DebugManager
-> Widgets
```

A macro existe para não teres de ligar manualmente:

```text
Reference to Self -> WorldContextObject
Reference to Self -> GetSenderName -> SenderName
```

---

# 3. Estrutura `S_DebugMessage`

A mensagem de debug é uma struct com contexto completo.

Campos principais:

| Campo | Para que serve |
|---|---|
| `Category` | Categoria do sistema, por exemplo Puzzle, Save, Scare, Voice |
| `Severity` | Info, Warning ou Critical |
| `Verbosity` | Nível de detalhe |
| `SenderName` | Quem enviou a mensagem |
| `Message` | Texto principal |
| `MessageKey` | Chave para Once/Throttle |
| `Duration` | Duração no ecrã |
| `bPrintToScreen` | Print no ecrã |
| `bPrintToLog` | Print no Output Log |
| `bShowInOverlay` | Mostrar nos widgets |
| `bOnceOnly` | Mostrar só uma vez |
| `ThrottleSeconds` | Intervalo mínimo entre repetições |
| `bUseLocation` | Usar posição no mundo |
| `WorldLocation` | Posição 3D |
| `ContextGhostId` | Ghost atual |
| `ContextPuzzleId` | Puzzle atual |
| `ContextStepId` | Step atual |
| `ContextTag` | Tag extra |
| `LevelName` | Nome do level |
| `WorldContextName` | Nome do contexto |
| `TimeSeconds` | Tempo da mensagem |
| `DisplayColor` | Cor final |
| `RepeatCount` | Repetições registadas |

---

# 4. Severidade

## 4.1 `Info`

Usa para fluxo normal.

Exemplos:

```text
[Puzzle] Started puzzle PZ_Isabel_MR_RuneSequence
[Save] Save loaded successfully
[Ghost] Active ghost changed to Isabel
```

## 4.2 `Warning`

Usa quando algo está estranho, mas o sistema continua.

Exemplos:

```text
[Audio] Audio state row not found, using fallback
[Scare] No valid anchor found, using fallback actor
[Puzzle] Broadcast had zero receivers
```

## 4.3 `Critical`

Usa quando algo obrigatório falhou.

Exemplos:

```text
[Save] SaveManagerRef invalid
[Transition] Target level invalid
[Puzzle] CurrentPuzzleId is None during completion
```

---

# 5. Verbosity

A verbosity controla o detalhe.

| Verbosity | Uso |
|---|---|
| Basic | Mensagens importantes e curtas |
| Detailed | Fluxo normal com contexto |
| VeryVerbose | Debug profundo, loops, validações internas |

Regra prática:

- `Basic` para alto nível;
- `Detailed` para managers;
- `VeryVerbose` para loops, timers, traces e debugging profundo.

---

# 6. Categorias

As categorias permitem filtrar e encaminhar mensagens.

Exemplos típicos:

- `General`
- `Puzzle`
- `Save`
- `Ghost`
- `Objective`
- `Scare`
- `Voice`
- `Audio`
- `Transition`
- `WorldMode`
- `UI`
- `Interaction`

Usa a categoria do sistema que está a emitir a mensagem.

Exemplos:

```text
BP_SaveManager -> Save
BP_ScareManager -> Scare
BP_VoiceListener -> Voice
BP_PuzzleController -> Puzzle
BP_ObjectiveManager -> Objective
```

---

# 7. Funções da `BFL_Debug`

## 7.1 `DebugMessageAdvanced`

Função mais completa. Usa quando precisas de controlar tudo.

Controla:

- categoria;
- severidade;
- verbosity;
- sender;
- mensagem;
- message key;
- duração;
- print screen;
- print log;
- overlay;
- once;
- throttle;
- localização;
- contexto extra.

Usa diretamente só quando os wrappers não chegam.

## 7.2 `DebugMessage`

Wrapper intermédio. Usa quando queres mais controlo do que `DebugInfo`, mas não precisas de tudo o que existe no Advanced.

## 7.3 `DebugInfo`

Mensagem normal.

Usa para:

- inicializações;
- estados esperados;
- confirmação de ações;
- início/fim de fluxo importante.

Exemplos:

```text
Puzzle started
Save loaded
Ghost changed
Audio state applied
Transition started
```

## 7.4 `DebugWarning`

Mensagem de aviso recuperável.

Usa para:

- fallback usado;
- referência opcional inválida;
- row ausente com alternativa;
- broadcast sem receivers;
- algo suspeito mas não fatal.

## 7.5 `DebugCritical`

Erro grave.

Usa para:

- referência obrigatória inválida;
- cast essencial falhado;
- estado impossível;
- falha que impede continuação correta.

## 7.6 `DebugOnce`

Mostra uma mensagem uma única vez por `MessageKey`.

Usa quando uma mensagem pode repetir muito mas só queres saber uma vez.

Exemplos:

```text
Missing component
Fallback initialized
No puzzle receiver found
```

Sempre define uma `MessageKey` clara.

## 7.7 `DebugThrottle`

Permite repetir uma mensagem, mas só de X em X segundos.

Usa em:

- Tick;
- Timers;
- Loops;
- Traces;
- Scare loop;
- Voice activity;
- Hand tracking;
- Proximity checks.

Exemplo:

```text
Category = Scare
MessageKey = Scare_TryTriggerNext
ThrottleSeconds = 2.0
Message = "Trying to trigger next scare"
```

## 7.8 `DebugLocation`

Mensagem com posição no mundo.

Usa para:

- spawn location;
- anchor location;
- trace hit;
- portal location;
- scare location;
- puzzle object location.

## 7.9 `DebugStateChange_Name`

Log padronizado para mudança de estado.

Usa para:

- `GhostState`;
- `WorldMode`;
- `ObjectiveStatus`;
- `PuzzleId`;
- `StepId`.

Exemplo conceptual:

```text
[GhostState] Dormant -> Manifesting
```

## 7.10 `GetSenderName`

Função auxiliar que recebe uma referência e devolve o nome do objecto.

Antes era usada assim:

```text
self -> GetSenderName -> SenderName
```

Agora continua útil para chamadas antigas, mas o objetivo é usar macros `DBG_*` para não ter de ligar isto manualmente.

---

# 8. Macros novas: `BML_DebugAuto_Actor`

## 8.1 Problema resolvido

Sem macros, cada debug precisava de:

```text
Self -> WorldContextObject
Self -> GetSenderName -> SenderName
```

Com macros, usas apenas:

```text
DBG_Info
Category = Puzzle
Message = "Puzzle started"
```

## 8.2 Macro Library correta

Para actors e managers:

```text
BML_DebugAuto_Actor
Parent Class = Actor
```

Usa em:

- `BP_DebugManager`
- `BP_SaveManager`
- `BP_PuzzleController`
- `BP_GhostManager`
- `BP_ScareManager`
- `BP_AudioManager`
- `BP_VoiceListener`
- `BP_TransitionManager`
- `BP_ObjectiveManager`
- `BP_PuzzleActorBase`
- `BP_ScareBase`
- actors de teste

## 8.3 Porque parent `Actor`

A macro precisa de `Self` com contexto de actor.

Com parent `Object`, o contexto pode não resolver como esperado. Com parent `Actor`, a macro é mais fiável para gameplay actors.

## 8.4 Macro `DBG_Info`

Inputs:

```text
Category : E_DebugCategory
Message  : Text
```

Internamente:

```text
Inputs Exec -> DebugInfo -> Outputs Exec
Inputs.Category -> DebugInfo.Category
Inputs.Message -> DebugInfo.Message
Self -> DebugInfo.WorldContextObject
Self -> DebugInfo.WorldContext
Self -> Get Display Name -> String To Name -> DebugInfo.SenderName
```

## 8.5 Macro `DBG_Warning`

Inputs:

```text
Category : E_DebugCategory
Message  : Text
```

Chama `DebugWarning` com o mesmo auto-contexto.

## 8.6 Macro `DBG_Critical`

Inputs:

```text
Category : E_DebugCategory
Message  : Text
```

Chama `DebugCritical` com o mesmo auto-contexto.

## 8.7 Macro `DBG_Once`

Inputs recomendados:

```text
Category   : E_DebugCategory
Message    : Text
MessageKey : Name
```

Chama `DebugOnce`.

## 8.8 Macro `DBG_Throttle`

Inputs recomendados:

```text
Category        : E_DebugCategory
Message         : Text
MessageKey      : Name
ThrottleSeconds : Float
```

Chama `DebugThrottle`.

## 8.9 Macro `DBG_Location`

Inputs recomendados:

```text
Category      : E_DebugCategory
Message       : Text
WorldLocation : Vector
```

Chama `DebugLocation`.

## 8.10 Macro `DBG_StateChange_Name`

Inputs recomendados:

```text
Category  : E_DebugCategory
StateName : Name
OldValue  : Name
NewValue  : Name
```

Chama `DebugStateChange_Name`.

---

# 9. C++ helper: `EchoesDebugAutoContextLibrary`

Foi tentada uma função C++ chamada `Get Auto Debug Context` para auto-preencher:

- `WorldContextObject`;
- `SenderName`.

A ideia era usar metadata:

```cpp
WorldContext="WorldContextObject"
DefaultToSelf="WorldContextObject"
HidePin="WorldContextObject"
```

Dentro de uma Macro Library, esse node pode não apanhar o caller real como esperado. Por isso, para macros Actor, a solução mais estável é usar `Self` dentro da própria macro.

O helper C++ ainda pode ser útil quando colocado diretamente num Blueprint normal.

## 9.1 Erro com `UUserWidget`

Se o C++ fizer cast para `UUserWidget`, precisas do módulo `UMG` no `.Build.cs`.

Erro típico:

```text
unresolved external symbol Z_Construct_UClass_UUserWidget_NoRegister
```

Soluções:

1. adicionar `UMG` ao `.Build.cs`;
2. ou remover a dependência de `UUserWidget` e usar fallback genérico por classe.

---

# 10. `GI_Echoes` em detalhe

## 10.1 `SetInitialVarsDebug`

Inicializa o estado base do sistema.

Deve correr no arranque do `GI_Echoes`.

## 10.2 `LoadDefaultDebugProfile`

Carrega o perfil default.

Usa o Data Asset default definido no projeto.

## 10.3 `ApplyDebugProfileAsset`

Aplica um Data Asset de perfil.

Exemplos:

```text
DA_DP_Quiet
DA_DP_Gameplay
DA_DP_AudioVoice
DA_DP_AllVerbose
```

Depois de aplicar, deve disparar atualização das settings para o `BP_DebugManager` refrescar UI.

## 10.4 `RegisterDebugManager`

Chamado pelo `BP_DebugManager` no `BeginPlay`.

Guarda a referência do manager visual.

## 10.5 `UnregisterDebugManager`

Remove a referência do manager.

## 10.6 `IsDebugMessageAllowed`

Decide se uma mensagem passa.

Verifica:

- sistema ligado;
- categoria ativa;
- sender mutado;
- verbosity permitida;
- perfil atual;
- regras globais.

## 10.7 `StampDebugContext`

Preenche contexto.

Pode preencher:

- `LevelName`;
- `WorldContextName`;
- `DisplayColor`;
- `TimeSeconds`;
- contexto ghost/puzzle/step.

## 10.8 `ProcessOnceAndThrottle`

Evita spam.

Controla:

- mensagens once;
- mensagens throttle;
- runtime caches;
- repeat count.

## 10.9 `AppendDebugHistory`

Guarda a mensagem no array `DebugHistory`.

Este histórico permite reconstruir os widgets.

## 10.10 `RouteDebugMessage`

Faz output final:

- screen;
- log;
- overlay;
- dispatcher.

## 10.11 `SubmitDebugMessage`

Entrada principal chamada pela `BFL_Debug`.

Pipeline esperada:

```text
SubmitDebugMessage
-> IsDebugMessageAllowed
-> StampDebugContext
-> ProcessOnceAndThrottle
-> AppendDebugHistory
-> RouteDebugMessage
```

## 10.12 `SetDebugSystemEnabled`

Liga/desliga o sistema global.

## 10.13 `SetCategoryEnabled`

Liga/desliga uma categoria.

## 10.14 `MuteDebugSender` / `UnmuteDebugSender`

Silencia ou reativa um sender.

## 10.15 `ClearDebugHistory`

Limpa histórico.

## 10.16 `ClearDebugRuntimeCaches`

Limpa caches de once/throttle.

Útil quando estás a testar e queres que mensagens once possam aparecer outra vez.

## 10.17 `BuildFormattedDebugString`

Constrói uma string final bonita para log/display.

## 10.18 `GetDebugColorForSeverity`

Retorna cor por severidade.

---

# 11. `BP_DebugManager` em detalhe

## 11.1 BeginPlay recomendado

```text
Event BeginPlay
-> Get Game Instance
-> Cast To GI_Echoes
-> Set CachedGI
-> IsValid(CachedGI)
-> Branch
True:
    RegisterDebugManager
    Bind Event to OnDebugMessageAccepted
    Bind Event to OnDebugSettingsChanged
    EnsureWidgetsCreated
    RefreshWidgetsFromHistory
```

## 11.2 `EnsureWidgetsCreated`

Função segura/idempotente.

Pode ser chamada várias vezes sem duplicar widgets.

Fluxo:

```text
se bCreateMainFeed e MainFeedWidget inválido -> CreateMainFeedWidget
se bCreateScareFeed e ScareFeedWidget inválido -> CreateScareFeedWidget
se bCreateVoiceFeed e VoiceFeedWidget inválido -> CreateVoiceFeedWidget
```

## 11.3 `CreateMainFeedWidget`

Cria feed geral.

Configuração:

```text
Title = Debug All
AllowedCategories = array vazio
Position = (30, 30)
Size = (560, 320)
```

Array vazio significa aceitar tudo.

## 11.4 `CreateScareFeedWidget`

Cria feed de scares.

Configuração:

```text
Title = Debug Scares
AllowedCategories = [Scare]
Position = (30, 370)
Size = (560, 320)
```

## 11.5 `CreateVoiceFeedWidget`

Cria feed de voice.

Configuração:

```text
Title = Debug Voice
AllowedCategories = [Voice]
Position = (610, 30)
Size = (560, 320)
```

## 11.6 `ApplyFeedViewportLayout`

Aplica layout visual:

```text
Set Desired Size in Viewport
Set Alignment in Viewport
Set Position in Viewport
Set Visibility
```

## 11.7 `HandleDebugMessageAccepted`

Chamado pelo dispatcher do GI.

Fluxo:

```text
HandleDebugMessageAccepted(Message)
-> EnsureWidgetsCreated
-> ForwardMessageToWidgets(Message)
```

## 11.8 `ForwardMessageToWidgets`

Não filtra categoria.

Apenas distribui:

```text
if MainFeedWidget valid -> PushMessage
if ScareFeedWidget valid -> PushMessage
if VoiceFeedWidget valid -> PushMessage
```

Quem filtra é o `WBP_DebugFeedPanel`.

## 11.9 `RefreshWidgetsFromHistory`

Reconstrói feeds.

Fluxo:

```text
EnsureWidgetsCreated
-> IsValid(CachedGI)
-> ClearAllFeeds
-> ForEach CachedGI.DebugHistory
-> ForwardMessageToWidgets(ArrayElement)
```

## 11.10 `ClearAllFeeds`

Limpa todos os feeds válidos.

## 11.11 `ToggleMainDebugFeed`

Alterna visibilidade do feed principal.

---

# 12. Widgets em detalhe

## 12.1 `WBP_DebugFeedPanel`

Variáveis importantes:

| Variável | Função |
|---|---|
| `FeedTitle` | Título |
| `AllowedCategories` | Categorias aceites |
| `DebugLineWidgetClass` | Classe da linha |
| `MaxVisibleRows` | Máximo de linhas |

## 12.2 `InitFeedPanel`

Inputs:

```text
InTitle
InAllowedCategories
```

Faz:

```text
Set FeedTitle
Set AllowedCategories
Set Text_Title
Set Text_Count = 0
```

## 12.3 `PushMessage`

Fluxo:

```text
Break S_DebugMessage
-> AllowedCategories vazio OR Contains(Category)
-> Create Widget WBP_DebugLine
-> SetupRow
-> Add Child to VerticalBox_MessageList
-> Update counter
-> Scroll To End
```

Regra importante:

```text
AllowedCategories vazio = aceitar tudo
```

## 12.4 `ClearFeed`

```text
VerticalBox_MessageList -> Clear Children
Text_Count -> SetText("0")
```

## 12.5 `FormatDebugTime`

Transforma segundos em texto.

Exemplo:

```text
83.4 -> 01:23
```

## 12.6 `WBP_DebugLine.SetupRow`

Inputs:

```text
InMessage
InFormattedText
InColor
InTimeText
```

Faz:

```text
Set Text_Sender
Set Text_Time
Set Text_Message
Set Brush Color da barra de severidade
```

---

# 13. Data Assets / Debug Profiles

## 13.1 Para que servem

Os Debug Profiles são Data Assets que alteram o comportamento do sistema sem mexer manualmente em várias variáveis.

Perfis criados:

```text
DA_DP_Quiet
DA_DP_Gameplay
DA_DP_AudioVoice
DA_DP_AllVerbose
```

Eles servem para mudar rapidamente o volume de debug.

## 13.2 Campos conceptuais de um perfil

A tua estrutura pode ter nomes ligeiramente diferentes, mas conceptualmente um perfil controla:

| Campo | Função |
|---|---|
| `bDebugSystemEnabled` | Liga/desliga debug |
| `bPrintToScreenDefault` | Default para screen |
| `bPrintToLogDefault` | Default para log |
| `bShowOverlayDefault` | Default para overlay |
| `MinVerbosity` | Verbosity mínima |
| `EnabledCategories` | Categorias ativas |
| `MutedSenders` | Senders silenciados |
| `HistoryLimit` | Limite de histórico |
| `DefaultDuration` | Duração default |
| `bEnableOnce` | Ativar Once |
| `bEnableThrottle` | Ativar Throttle |
| `bEnableOverlay` | Ativar widgets |

## 13.3 `DA_DP_Quiet`

Perfil silencioso.

Usa quando queres reduzir ruído.

Recomendado para:

- gravação;
- testes em Quest sem overlay;
- gameplay sem spam;
- sessões onde só queres warnings/criticals.

Configuração conceptual:

```text
DebugSystemEnabled = true
PrintToScreenDefault = false
PrintToLogDefault = true
ShowOverlayDefault = false
MinVerbosity = Basic
EnabledCategories = só principais / warnings / criticals
MutedSenders = sistemas ruidosos
```

## 13.4 `DA_DP_Gameplay`

Perfil normal de desenvolvimento.

Usa para:

- puzzle flow;
- objectives;
- ghost state;
- scare decisions principais;
- save/restore;
- transition flow.

Configuração conceptual:

```text
DebugSystemEnabled = true
PrintToScreenDefault = true
PrintToLogDefault = true
ShowOverlayDefault = true
MinVerbosity = Detailed
EnabledCategories = General, Puzzle, Save, Ghost, Objective, Scare, Transition, WorldMode
```

## 13.5 `DA_DP_AudioVoice`

Perfil focado em áudio e voz.

Usa para:

- `BP_AudioManager`;
- `BP_VoiceListener`;
- voice intents;
- shout/activity;
- stingers;
- audio states.

Configuração conceptual:

```text
DebugSystemEnabled = true
PrintToScreenDefault = true
PrintToLogDefault = true
ShowOverlayDefault = true
MinVerbosity = Detailed ou VeryVerbose
EnabledCategories = Audio, Voice
MutedSenders = sistemas não relacionados
```

## 13.6 `DA_DP_AllVerbose`

Perfil máximo.

Usa para:

- bugs difíceis;
- integração entre managers;
- perceber fluxo completo;
- debug profundo.

Configuração conceptual:

```text
DebugSystemEnabled = true
PrintToScreenDefault = true
PrintToLogDefault = true
ShowOverlayDefault = true
MinVerbosity = VeryVerbose
EnabledCategories = todas
MutedSenders = vazio
```

Cuidado: pode gerar muito spam.

## 13.7 Como aplicar um perfil

No `GI_Echoes`:

```text
ApplyDebugProfileAsset(ProfileAsset)
```

Exemplo:

```text
ApplyDebugProfileAsset(DA_DP_Gameplay)
```

Fluxo esperado:

```text
ApplyDebugProfileAsset
-> lê Data Asset
-> aplica variáveis globais
-> aplica categorias
-> aplica muted senders
-> atualiza defaults
-> dispara OnDebugSettingsChanged
```

## 13.8 Quando aplicar

No arranque:

```text
LoadDefaultDebugProfile
```

Durante teste:

```text
1 -> DA_DP_Quiet
2 -> DA_DP_Gameplay
3 -> DA_DP_AudioVoice
4 -> DA_DP_AllVerbose
```

---

# 14. Como usar no dia a dia

## 14.1 Actor/Manager normal

Usa macros `DBG_*`.

Exemplo:

```text
DBG_Info
Category = Puzzle
Message = "Started puzzle"
```

## 14.2 Warning

```text
DBG_Warning
Category = Puzzle
Message = "No receivers found for active step"
```

## 14.3 Critical

```text
DBG_Critical
Category = Save
Message = "SaveManagerRef invalid"
```

## 14.4 Once

```text
DBG_Once
Category = Audio
MessageKey = Audio_MissingStateFallback
Message = "Audio state not found, using fallback"
```

## 14.5 Throttle

```text
DBG_Throttle
Category = Scare
MessageKey = Scare_TryTriggerNext
ThrottleSeconds = 2.0
Message = "Trying to trigger next scare"
```

## 14.6 Location

```text
DBG_Location
Category = Scare
Message = "Selected anchor location"
WorldLocation = AnchorLocation
```

## 14.7 State change

```text
DBG_StateChange_Name
Category = Ghost
StateName = GhostState
OldValue = Dormant
NewValue = Manifesting
```

---

# 15. Quando usar cada coisa

| Situação | Usa |
|---|---|
| Mensagem normal | `DBG_Info` / `DebugInfo` |
| Aviso recuperável | `DBG_Warning` / `DebugWarning` |
| Erro grave | `DBG_Critical` / `DebugCritical` |
| Só mostrar uma vez | `DBG_Once` / `DebugOnce` |
| Loop/timer/tick | `DBG_Throttle` / `DebugThrottle` |
| Mensagem com posição | `DBG_Location` / `DebugLocation` |
| Mudança de estado | `DBG_StateChange_Name` |
| Controlo total | `DebugMessageAdvanced` |

---

# 16. Convenções

## 16.1 Mensagem

Formato recomendado:

```text
[Contexto] ação -> detalhe
```

Exemplos:

```text
[Puzzle] Start -> PZ_Isabel_MR_RuneSequence
[Puzzle] Step activated -> Rune_02
[Save] Restore -> CurrentPuzzleId=PZ_Isabel_MR_RuneSequence Step=1
[Scare] Selected -> Isabel_MR_WhisperBack_01
[Voice] Intent detected -> BanishingWord
```

## 16.2 SenderName

Com macros, vem de:

```text
Self -> Get Display Name -> String To Name
```

Se o sender aparecer feio, renomeia a instância do actor no level.

## 16.3 MessageKey

Formato recomendado:

```text
System_Event_Detail
```

Exemplos:

```text
Puzzle_NoReceivers
Puzzle_InvalidStepIndex
Save_AutoSaveDisabled
Scare_NoValidScare
Audio_StateMissing
Voice_IntentLowConfidence
```

---

# 17. Plano de testes

## 17.1 Teste macro

Num actor no level:

```text
BeginPlay -> DBG_Info
Category = General
Message = "DBG_INFO_AUTO_ACTOR TEST"
```

Esperado:

- mensagem aparece;
- sender automático correto;
- não ligaste self manualmente.

## 17.2 Teste warning

```text
DBG_Warning
Category = General
Message = "DBG_WARNING_AUTO TEST"
```

## 17.3 Teste critical

```text
DBG_Critical
Category = General
Message = "DBG_CRITICAL_AUTO TEST"
```

## 17.4 Teste Scare feed

```text
DBG_Info
Category = Scare
Message = "SCARE FEED TEST"
```

Esperado:

- aparece no `Debug All`;
- aparece no `Debug Scares`;
- não aparece no `Debug Voice`.

## 17.5 Teste Voice feed

```text
DBG_Info
Category = Voice
Message = "VOICE FEED TEST"
```

Esperado:

- aparece no `Debug All`;
- aparece no `Debug Voice`;
- não aparece no `Debug Scares`.

## 17.6 Teste throttle

Chamar várias vezes:

```text
DBG_Throttle
Category = Scare
MessageKey = Test_Throttle
ThrottleSeconds = 2.0
Message = "Throttle test"
```

Esperado: não aparece em spam.

## 17.7 Teste once

Chamar 3 vezes:

```text
DBG_Once
Category = Puzzle
MessageKey = Test_Once
Message = "Once test"
```

Esperado: aparece só uma vez.

## 17.8 Teste refresh

1. gerar 3 mensagens;
2. chamar `RefreshWidgetsFromHistory`.

Esperado:

- limpa;
- reconstrói;
- não duplica.

## 17.9 Teste perfis

Aplicar:

```text
DA_DP_Quiet
DA_DP_Gameplay
DA_DP_AudioVoice
DA_DP_AllVerbose
```

Confirmar que o volume de debug muda.

---

# 18. Troubleshooting

## 18.1 Macro não mostra nada

Verificar:

1. o actor está no level?
2. o `BeginPlay` corre?
3. mete `Print String` temporário dentro da macro antes do `DebugInfo`.
4. se esse print não aparece, a macro não está a ser executada.
5. se esse print aparece mas o debug não, problema está depois da macro.

## 18.2 `Get Auto Debug Context` não aparece

Só aparece se a classe C++ compilar.

Para macros Actor, usa `Self` dentro da macro em vez desse node.

## 18.3 Widget só mostra título pequeno

Verificar layout do `WBP_DebugFeedPanel`:

- root;
- `SizeBox`;
- `Canvas Slot Auto Size`;
- `Set Desired Size in Viewport`;
- fundo `Border_Background`.

## 18.4 Mensagens não aparecem no overlay

Verificar:

- `bShowInOverlay`;
- perfil ativo;
- `BP_DebugManager` no level;
- binds;
- `ForwardMessageToWidgets`;
- `PushMessage`.

## 18.5 Mensagem não aparece por categoria

Verificar:

- categoria enviada;
- `AllowedCategories` do feed;
- array vazio = accept all;
- categoria ativa no perfil.

## 18.6 Mensagem não repete

Verificar:

- `bOnceOnly`;
- `MessageKey`;
- `ThrottleSeconds`;
- caches;
- `ClearDebugRuntimeCaches`.

---

# 19. Estado atual

## Já existe / desenhado

- `BFL_Debug`;
- wrappers principais;
- `GI_Echoes` como hub;
- `DebugHistory`;
- perfis/data assets;
- `BP_DebugManager`;
- `EnsureWidgetsCreated`;
- `ForwardMessageToWidgets`;
- `RefreshWidgetsFromHistory`;
- `WBP_DebugFeedPanel`;
- `WBP_DebugLine`;
- feeds Main/Scare/Voice;
- abordagem com macros `DBG_*`.

## Ainda a validar / fechar

- layout final dos widgets;
- `FormatDebugTime`/timestamp;
- macros todas além de `DBG_Info`;
- comportamento final dos perfis;
- testes ponta-a-ponta;
- possível macro library específica para widgets se precisares.

---

# 20. Regras finais

1. Não uses `Print String` solto para debug normal.
2. Em actors/managers, usa `DBG_*`.
3. Em casos especiais, usa `DebugMessageAdvanced`.
4. Em loops, usa `DBG_Throttle`.
5. Em warnings repetidos, usa `DBG_Once`.
6. Define categorias corretamente.
7. Define `MessageKey` para Once/Throttle.
8. Usa perfis `DA_DP_*` para controlar volume de debug.

---

# 21. Resumo rápido

| Quero | Uso |
|---|---|
| Mensagem normal | `DBG_Info` |
| Aviso | `DBG_Warning` |
| Erro grave | `DBG_Critical` |
| Só uma vez | `DBG_Once` |
| Limitar spam | `DBG_Throttle` |
| Posição no mundo | `DBG_Location` |
| Mudança de estado | `DBG_StateChange_Name` |
| Controlo total | `DebugMessageAdvanced` |
| Menos ruído | `DA_DP_Quiet` |
| Debug gameplay normal | `DA_DP_Gameplay` |
| Debug áudio/voz | `DA_DP_AudioVoice` |
| Debug máximo | `DA_DP_AllVerbose` |

---

# 22. Próximo passo recomendado

Para fechar o sistema:

1. confirmar `DBG_Info` em `BML_DebugAuto_Actor`;
2. criar `DBG_Warning`;
3. criar `DBG_Critical`;
4. criar `DBG_Once`;
5. criar `DBG_Throttle`;
6. testar Main/Scare/Voice;
7. corrigir layout final do feed;
8. fechar `FormatDebugTime`;
9. validar perfis `DA_DP_*`.

---

# 23. Conclusão

Este sistema é a infraestrutura de debug global do projeto.

Regra principal:

```text
BFL_Debug / DBG_* recebem a mensagem.
GI_Echoes decide e guarda.
BP_DebugManager distribui.
WBP_DebugFeedPanel filtra.
WBP_DebugLine mostra.
Debug Profiles controlam o volume.
```
