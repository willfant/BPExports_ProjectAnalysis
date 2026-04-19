# Sistema de Debug — Echoes Between Worlds

## Objetivo

Este documento descreve o **sistema de debug global** do projeto **Echoes Between Worlds**:
- arquitetura,
- fluxo completo da mensagem,
- funções públicas de uso diário,
- perfis,
- feeds/widget,
- boas práticas,
- troubleshooting,
- inventário de funções,
- e sugestões de expansão.

O foco deste sistema é simples:

1. qualquer Blueprint pode emitir mensagens de debug de forma consistente;
2. o `GI_Echoes` centraliza regras e runtime caches;
3. o `BP_DebugManager` trata a camada visual;
4. as mensagens podem ser filtradas, deduplicadas, limitadas por throttle e organizadas por contexto.

---

# 1. Visão geral da arquitetura

## 1.1 Componentes principais

### `BFL_Debug`
É a **API pública** do sistema.

É aqui que os outros Blueprints devem chamar:
- `DebugMessage`
- `DebugMessageAdvanced`
- `DebugInfo`
- `DebugWarning`
- `DebugCritical`
- `DebugOnce`
- `DebugThrottle`
- `DebugLocation`
- `DebugStateChange_Name`

### `GI_Echoes`
É o **hub central** do sistema.

Responsabilidades:
- receber a mensagem final;
- aplicar regras globais;
- enriquecer contexto;
- decidir se a mensagem entra ou não;
- guardar histórico;
- despachar para output visual/log;
- guardar estado/configuração do debug.

### `BP_DebugManager`
É a **camada visual**.

Responsabilidades:
- registar-se no `GI_Echoes`;
- criar widgets de debug;
- ouvir os dispatchers do GI;
- encaminhar mensagens para os painéis certos;
- refrescar widgets a partir do histórico quando entra em jogo.

### `WBP_DebugFeedPanel`
É o widget/painel visual que recebe mensagens.

Pelo que o sistema mostra atualmente, este widget expõe pelo menos:
- `InitFeedPanel`
- `PushMessage`

---

# 2. Fluxo completo da mensagem

```text
Blueprint qualquer
-> BFL_Debug (wrapper)
-> DebugMessageAdvanced
-> Get Game Instance
-> Cast to GI_Echoes
-> SubmitDebugMessage
-> IsDebugMessageAllowed
-> StampDebugContext
-> ProcessOnceAndThrottle
-> AppendDebugHistory
-> RouteDebugMessage
-> dispatcher/evento no GI
-> BP_DebugManager
-> ForwardMessageToWidgets
-> WBP_DebugFeedPanel(s)
```

---

# 3. Funções públicas da BFL_Debug

## `DebugMessage`
Wrapper simples para envio base.

## `DebugMessageAdvanced`
Função pública mais importante da library:
- constrói a `S_DebugMessage`;
- faz `GetGameInstance`;
- faz cast para `GI_Echoes`;
- chama `SubmitDebugMessage`;
- se o cast falhar, faz fallback com `PrintString`.

## `DebugInfo`
Wrapper rápido para mensagens normais.

## `DebugWarning`
Wrapper para situações suspeitas mas não fatais.

## `DebugCritical`
Wrapper para situações graves.

## `DebugOnce`
Wrapper para mensagens que só devem aparecer uma vez por `MessageKey`.

## `DebugThrottle`
Wrapper para mensagens que podem repetir, mas com limite temporal.

## `DebugLocation`
Wrapper para mensagens com contexto espacial.

## `DebugStateChange_Name`
Wrapper mais semântico para mudanças de estado por `Name`.

---

# 4. O que entra numa S_DebugMessage

Campos relevantes:
- `Category`
- `Severity`
- `Verbosity`
- `SenderName`
- `Message`
- `MessageKey`
- `Duration`
- `bPrintToScreen`
- `bPrintToLog`
- `bShowInOverlay`
- `bOnceOnly`
- `ThrottleSeconds`
- `bUseLocation`
- `WorldLocation`
- `RepeatCount`
- `LevelName`
- `WorldContextName`
- `ContextGhostId`
- `ContextPuzzleId`
- `ContextStepId`
- `ContextTag`
- `DisplayColor`

---

# 5. O que o GI_Echoes faz à mensagem

## Pipeline principal
- `IsDebugMessageAllowed`
- `StampDebugContext`
- `ProcessOnceAndThrottle`
- `AppendDebugHistory`
- `RouteDebugMessage`
- `SubmitDebugMessage`

## Gestão/configuração do sistema de debug
- `SetInitialVarsDebug`
- `LoadDefaultDebugProfile`
- `ApplyDebugProfileAsset`
- `SetDebugSystemEnabled`
- `MuteDebugSender`
- `UnmuteDebugSender`
- `ClearDebugHistory`
- `ClearDebugRuntimeCaches`
- `SetCategoryEnabled`
- `RegisterDebugManager`
- `UnregisterDebugManager`

## Funções auxiliares de apresentação/apoio
- `GetDebugColorForSeverity`
- `DebugSeverityToInt`
- `DebugVerbosityToInt`
- `BuildFormattedDebugString`

---

# 6. BP_DebugManager

## Funções principais
- `CreateDebugWidgets`
- `RefreshWidgetsFromHistory`
- `ForwardMessageToWidgets`
- `ToggleMainDebugFeed`

## Papel
- criar feeds;
- bindar-se aos delegates do GI;
- encaminhar mensagens aceites para os painéis;
- reconstruir o UI a partir do histórico.

---

# 7. Porque é que existem só Main, Scare e Voice

O sistema atual foi desenhado com três feeds:
- `MainFeedWidget`
- `ScareFeedWidget`
- `VoiceFeedWidget`

## Porque faz sentido
### `Main`
Feed geral para tudo.

### `Scare`
Sistema muito dinâmico, cheio de condições, cooldowns, contexto espacial e decisão runtime.

### `Voice`
Sistema com input externo, estados de escuta/intenção/confirmação e debugging difícil quando misturado com tudo o resto.

## Porque não existem ainda outros feeds
Porque a primeira segmentação foi feita nas áreas mais ruidosas e sensíveis:
- global,
- sustos,
- voz.

### Expansões naturais futuras
- `PuzzleFeedWidget`
- `ObjectiveFeedWidget`
- `SaveFeedWidget`
- `GhostFeedWidget`

---

# 8. Como usar o sistema no dia a dia

## Regra número 1
Na maioria dos Blueprints, usa wrappers da `BFL_Debug`.

Não chames o `GI_Echoes` diretamente.

## Casos típicos
- mensagem simples -> `DebugInfo`
- aviso recuperável -> `DebugWarning`
- erro grave -> `DebugCritical`
- evitar spam -> `DebugOnce` ou `DebugThrottle`
- contexto espacial -> `DebugLocation`
- controlo total -> `DebugMessageAdvanced`

---

# 9. Boas práticas

## Usa sempre `SenderName` consistente
Exemplos:
- `BP_PuzzleController`
- `BP_SaveManager`
- `BP_GhostManager`
- `BP_ScareManager`

## Usa `MessageKey` quando a mensagem é importante
Se usa `Once`, `Throttle` ou representa um evento lógico importante, define uma key clara.

## Mantém a mensagem legível
Exemplo bom:
```text
[Puzzle] Step accepted -> Rune_02
```

Exemplo fraco:
```text
entrou aqui
```

## Usa categorias com disciplina
As categorias são importantes para routing e filtros.

---

# 10. Testes recomendados

## Teste mínimo
Num actor qualquer no level:

```text
BeginPlay
-> DebugInfo
```

Campos:
- `WorldContextObject = self`
- `Category = General`
- `SenderName = BP_Test`
- `Message = DEBUG SYSTEM ONLINE`

## Teste de Once
Chamar `DebugOnce` duas vezes com a mesma `MessageKey`.

## Teste de Throttle
Chamar `DebugThrottle` várias vezes por segundo.

## Teste de widgets
Confirmar:
- `BP_DebugManager` no level;
- `RegisterDebugManager` chamado;
- widgets criados;
- `ForwardMessageToWidgets` sem erros;
- painéis recebem as mensagens certas.

---

# 11. Troubleshooting

## “Não aparece nada”
Verificar:
1. `GI_Echoes` está definido em Project Settings;
2. `BP_DebugManager` está no level;
3. `bDebugSystemEnabled = true`;
4. `DebugMessageAdvanced` consegue cast para `GI_Echoes`;
5. `SubmitDebugMessage` não está a rejeitar;
6. `ForwardMessageToWidgets` não tem nodes partidos.

## “Só aparece que foi registado”
Normalmente significa:
- o manager entrou,
- mas a pipeline ou o output visual ainda estavam a falhar.

## “O feed geral aparece mas os feeds especializados não”
Olhar para:
- `ForwardMessageToWidgets`
- `PushMessage` dos painéis especializados
- widgets `ScareFeedWidget` / `VoiceFeedWidget`
- filtros por categoria

---

# 12. Inventário completo das funções relevantes

## 12.1 BFL_Debug — funções públicas do sistema
- `DebugMessage`
- `DebugMessageAdvanced`
- `DebugInfo`
- `DebugWarning`
- `DebugCritical`
- `DebugOnce`
- `DebugThrottle`
- `DebugLocation`
- `DebugStateChange_Name`

## 12.2 GI_Echoes — funções de debug que interessam para documentação
- `SetInitialVarsDebug`
- `LoadDefaultDebugProfile`
- `ApplyDebugProfileAsset`
- `RegisterDebugManager`
- `UnregisterDebugManager`
- `GetDebugColorForSeverity`
- `DebugSeverityToInt`
- `DebugVerbosityToInt`
- `IsDebugMessageAllowed`
- `StampDebugContext`
- `ProcessOnceAndThrottle`
- `AppendDebugHistory`
- `BuildFormattedDebugString`
- `RouteDebugMessage`
- `SubmitDebugMessage`
- `SetDebugSystemEnabled`
- `MuteDebugSender`
- `UnmuteDebugSender`
- `ClearDebugHistory`
- `ClearDebugRuntimeCaches`
- `SetCategoryEnabled`

## 12.3 BP_DebugManager — funções relevantes
- `CreateDebugWidgets`
- `RefreshWidgetsFromHistory`
- `ForwardMessageToWidgets`
- `ToggleMainDebugFeed`

---

# 13. O que ficou de fora de propósito

O `GI_Echoes` tem outras funções que aparecem no export, como:
- boot/listener de voice,
- gestão de sessão,
- world mode,
- ghost/quest/objective refs,
- save refs,
- pending transition.

Essas funções existem, mas **não são parte do núcleo do sistema de debug**.
Por isso, para uma documentação focada no debug system, elas não entram como “funções do sistema de debug”, embora vivam no mesmo blueprint.

---

# 14. Resumo executivo

Este sistema já não é um conjunto de `Print String` soltos.

Ele já está estruturado como um sistema real:
- **API pública** (`BFL_Debug`)
- **hub central de controlo** (`GI_Echoes`)
- **camada visual desacoplada** (`BP_DebugManager`)
- **painéis filtrados** (`WBP_DebugFeedPanel`)
- **histórico**
- **filtros**
- **contexto**
- **once**
- **throttle**
- **routing por categorias**

---

# 15. Regra prática final

Se tiveres dúvida sobre qual função usar:

- quero só uma mensagem normal -> `DebugInfo`
- isto cheira mal mas não morreu -> `DebugWarning`
- isto está grave -> `DebugCritical`
- está a spammer -> `DebugThrottle`
- só quero uma vez -> `DebugOnce`
- tem posição no mundo -> `DebugLocation`
- quero controlo total -> `DebugMessageAdvanced`
