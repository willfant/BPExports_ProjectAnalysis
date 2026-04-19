# Echoes_Architecture_Ownership.md

# Echoes Between Worlds — Architecture Ownership & Communication Constitution

Versão: 1.0  
Estado: Final de referência  
Objetivo: Definir a arquitetura-alvo oficial do projeto, os donos de cada estado global, as regras de comunicação entre systems, a ordem de bootstrap/restore, os fluxos oficiais de runtime e os critérios de aceitação para fechar a re-arquitetura.

---

## 1. Porque este documento existe

Este documento existe para acabar com ambiguidade arquitetural.

O projeto Echoes Between Worlds já tem uma base real de systems e managers, mas a análise do export atual mostra um padrão perigoso: variáveis core como ghost, world mode, objetivos e puzzle progress estão a ser escritas por vários Blueprints ao mesmo tempo, e os managers mais centrais estão demasiado ligados uns aos outros por chamadas diretas.

Quando isso acontece, o projeto parece avançar durante algum tempo, mas começa a ganhar estes problemas:

- saves que restauram “quase” certo mas não totalmente
- managers com cópias divergentes do mesmo estado
- transições que funcionam num contexto e falham noutro
- efeitos secundários difíceis de rastrear
- cada nova feature passa a custar mais porque toca em demasiados sítios

Este documento define a arquitetura-alvo final para impedir isso.

---

## 2. Diagnóstico que justifica esta re-arquitetura

A análise atual do projeto mostra:

- 22 Blueprints exportados
- 205 ligações cross-blueprint
- hubs fortes em `BP_PuzzleController`, `BP_SaveManager`, `BP_AudioManager`, `BP_TransitionManager`, `BP_GhostManager`, `BP_WorldModeSystem` e `GI_Echoes`
- múltiplos writers para variáveis globais como `CurrentGhostId`, `CurrentWorldMode`, `CurrentPuzzleId` e `CompletedPuzzleIds`
- um ciclo potencial detetado na arquitetura

Isto não significa que o projeto esteja perdido. Significa que o projeto já é suficientemente sério para exigir ownership rígido.

Conclusão operacional:

**o próximo salto de qualidade não vem de criar mais systems. Vem de clarificar autoridade, cortar acoplamento e estabilizar o fluxo de runtime/save/restore.**

---

## 3. Princípios finais da arquitetura

A arquitetura de Echoes Between Worlds passa a obedecer a estes princípios sem exceções.

### 3.1 Single Source of Truth por domínio
Cada domínio de estado tem um único dono.

Exemplos:
- ghost -> GhostManager
- world mode -> WorldModeSystem
- objetivos -> ObjectiveManager
- puzzle progress -> PuzzleController
- persistência -> SaveManager

### 3.2 Runtime state não é Save state
O runtime vive nos managers.
O save é um snapshot persistido.
O save não é a autoridade viva do jogo.

### 3.3 Event-driven sempre que possível
Managers anunciam mudanças.
Outros managers reagem.
Evitar “manager A manda diretamente em manager B” como padrão.

### 3.4 GI_Echoes não é um segundo gameplay manager
O GI serve para bootstrap, sessão, registry e helpers leves.
Não é o dono de ghost, objectives, puzzle ou world mode.

### 3.5 Consequências e ownership são coisas diferentes
Um system pode causar uma consequência sem ser dono do estado afetado.
Exemplo:
- puzzle completion pode causar uma mudança de objective
- mas o dono dos objectives continua a ser o ObjectiveManager

### 3.6 Restore é um fluxo formal
Restaurar estado não é “ir buscar valores aqui e ali”.
Restore tem ordem, protocolo e handoff entre systems.

### 3.7 Nada escreve estado de outro domínio sem contrato explícito
Se uma variável tem dono, os outros systems:
- ou leem
- ou pedem por evento
- ou reagem por dispatcher
- mas não a escrevem diretamente

---

## 4. Domínios oficiais de ownership

Esta secção é a lei principal do projeto.

## 4.1 Ghost Domain
**Dono único:** `BP_GhostManager`

### Variáveis deste domínio
- `CurrentGhostId`
- `CurrentGhostState`
- `CurrentGhostProfile`
- `CurrentGhostActor`
- qualquer estado runtime diretamente ligado ao fantasma ativo

### Só este BP pode escrever
- `BP_GhostManager`

### Outros systems podem
- ler ghost atual
- reagir a mudanças
- pedir mudança através de evento/fluxo legítimo

### Outros systems não podem
- escrever `CurrentGhostId`
- escrever `CurrentGhostState`
- manter uma cópia concorrente “oficial”

## 4.2 World Mode Domain
**Dono único:** `BP_WorldModeSystem`

### Variáveis deste domínio
- `CurrentWorldMode`
- estados auxiliares de mudança de modo
- flags runtime diretamente ligadas a MR / VR / VRMemory

### Só este BP pode escrever
- `BP_WorldModeSystem`

### Outros systems podem
- ler o modo atual
- reagir a `OnWorldModeChanged`

### Outros systems não podem
- escrever `CurrentWorldMode`
- usar o save como autoridade do modo atual durante runtime

## 4.3 Objective / Quest Domain
**Dono único:** `BP_ObjectiveManager`

### Variáveis deste domínio
- `CurrentQuestId`
- `CurrentObjectiveId`
- `ObjectiveStates`
- qualquer estado de objetivo ativo/concluído/falhado
- listas locais de progressão de objectives

### Só este BP pode escrever
- `BP_ObjectiveManager`

### Outros systems podem
- pedir início/conclusão através de API pública do ObjectiveManager
- reagir a `OnObjectiveStarted` e `OnObjectiveCompleted`

### Outros systems não podem
- escrever `CurrentObjectiveId`
- escrever `CurrentQuestId`
- manipular `ObjectiveStates` diretamente

## 4.4 Puzzle Progress Domain
**Dono único:** `BP_PuzzleController`

### Variáveis deste domínio
- `CurrentPuzzleId`
- `CurrentStepIndex`
- `CurrentPuzzleData`
- `CurrentStepData`
- `CompletedStepIds`
- `FailedStepIds`
- `StartedPuzzleIds`
- `CompletedPuzzleIds`
- flags de puzzle ativo / step ativo

### Só este BP pode escrever
- `BP_PuzzleController`

### Outros systems podem
- perguntar pelo puzzle atual
- reagir a eventos de puzzle
- solicitar arranque de puzzle por API controlada

### Outros systems não podem
- escrever `CurrentPuzzleId`
- escrever `CurrentStepIndex`
- escrever `CompletedPuzzleIds`

## 4.5 Save / Persistence Domain
**Dono único:** `BP_SaveManager`

### Variáveis deste domínio
- `CurrentSaveObject`
- `CurrentSaveData`
- slot name
- save flags
- estado do autosave
- snapshot persistido
- campos guardados em disco

### Só este BP pode escrever
- `BP_SaveManager`

### Outros systems podem
- fornecer snapshot
- receber restore
- pedir autosave
- pedir save manual

### Outros systems não podem
- tratar o SaveManager como runtime authority do jogo
- usar o save como substituto do manager dono do estado

## 4.6 Transition Runtime Domain
**Dono único:** `BP_TransitionManager`

### Variáveis deste domínio
- `CurrentTransitionData`
- `bTransitionInProgress`
- `FadeWidgetRef`
- estado local de fade/travel

### Só este BP pode escrever
- `BP_TransitionManager`

### Outros systems podem
- pedir uma transição
- reagir a início/fim de transição

### Outros systems não podem
- controlar diretamente o fade interno
- alterar pending transition local do TransitionManager sem API formal

## 4.7 Scare Runtime Domain
**Dono único:** `BP_ScareManager`

### Variáveis deste domínio
- `CurrentScareActor`
- `SelectedScareId`
- `ActiveScareIds`
- timers de loop
- cooldowns de scare
- caches de seleção
- flags de scare loop runtime

### Só este BP pode escrever
- `BP_ScareManager`

### Outros systems podem
- pedir scare explícito através de API pública
- reagir a `OnScareStarted` e `OnScareFinished`

### Outros systems não podem
- manter estado concorrente oficial de scare loop
- escrever cooldowns ou active scare ids

## 4.8 Audio Runtime Domain
**Dono único:** `BP_AudioManager`

### Variáveis deste domínio
- `CurrentAudioStateId`
- `CurrentAmbienceSound`
- `CurrentMusicSound`
- `CurrentStingerSound`
- volumes runtime
- active looping cues
- flags internas de música/ambience

### Só este BP pode escrever
- `BP_AudioManager`

### Outros systems podem
- anunciar mudanças que impliquem novo áudio
- pedir stingers/eventos sonoros

### Outros systems não podem
- escrever state interno de áudio
- usar o AudioManager para guardar ghost/world mode state

## 4.9 Debug Domain
**Dono único:** `BP_DebugManager` para runtime de feeds e histórico  
**Interface partilhada:** `BFL_Debug`

### Regra
- `BFL_Debug` é serviço de entrada
- `BP_DebugManager` é o executor/runtime holder

## 4.10 Session / Registry Domain
**Dono único:** `GI_Echoes`

### Responsabilidades autorizadas
- guardar referências registradas para os managers
- bootstrap da sessão
- pending startup context leve
- configs/debug profiles globais
- helpers leves de arranque

### Não autorizado
- ser dono de ghost runtime
- ser dono de world mode runtime
- ser dono de objectives
- ser dono de puzzle progress

---

## 5. Tabela final de ownership

| Domínio | Dono Único | Leitura permitida por outros? | Escrita permitida por outros? |
|---|---:|---:|---:|
| Ghost | BP_GhostManager | Sim | Não |
| World Mode | BP_WorldModeSystem | Sim | Não |
| Objectives / Quest | BP_ObjectiveManager | Sim | Não |
| Puzzle Progress | BP_PuzzleController | Sim | Não |
| Save / Persistence | BP_SaveManager | Parcial / via API | Não |
| Transition Runtime | BP_TransitionManager | Sim | Não |
| Scare Runtime | BP_ScareManager | Sim | Não |
| Audio Runtime | BP_AudioManager | Sim | Não |
| Debug Runtime | BP_DebugManager | Sim | Não |
| Session / Registry | GI_Echoes | Sim | Não, fora da sua área |

---

## 6. Regras finais de comunicação entre systems

### 6.1 Regra base
**Read freely, write only your own domain.**

### 6.2 Regra de consequência
Se um system precisa de provocar algo noutro domínio, deve:
1. emitir um evento
2. ou chamar uma API pública explícita do dono
3. nunca escrever diretamente a variável do outro domínio

### 6.3 Regra de referência
Ter uma referência para outro manager não dá direito a escrever o estado dele.

### 6.4 Regra de save
SaveManager não é consultado para saber “o estado atual real” do jogo durante runtime normal. O estado atual real vive nos donos.

### 6.5 Regra de GI
GI_Echoes pode saber quem são os managers, mas não deve duplicar o que eles já sabem.

---

## 7. Relações permitidas e proibidas

## 7.1 GI_Echoes
### Pode fazer
- registrar e devolver refs de managers
- bootstrap do jogo
- guardar configs de sessão
- encaminhar travel helper leve se necessário

### Não deve fazer
- guardar `CurrentGhostId`
- guardar `CurrentWorldMode`
- guardar `CurrentObjectiveId`
- guardar `CurrentPuzzleId`
- tornar-se dono de runtime gameplay

## 7.2 BP_SaveManager
### Pode fazer
- carregar save
- criar save
- construir snapshot
- aplicar snapshot
- escrever em disco
- agendar autosave

### Não deve fazer
- ser consultado como autoridade primária do ghost atual
- ser consultado como autoridade primária do world mode atual
- ser consultado como autoridade primária do puzzle atual
- ser usado como cópia viva dos systems

## 7.3 BP_GhostManager
### Pode fazer
- resolver fantasma atual
- mudar ghost
- mudar ghost state
- expor ghost profile
- emitir `OnGhostChanged`
- emitir `OnGhostStateChanged`

### Não deve fazer
- inicializar objetivos diretamente
- inicializar scares diretamente
- escrever no save como espelho contínuo do runtime

## 7.4 BP_WorldModeSystem
### Pode fazer
- mudar world mode
- expor current world mode
- emitir `OnWorldModeChanged`

### Não deve fazer
- depender do SaveManager para saber o modo atual durante runtime
- escrever estado de outro domínio

## 7.5 BP_ObjectiveManager
### Pode fazer
- iniciar objective
- concluir objective
- falhar objective
- iniciar quest
- gerir objective states
- emitir `OnObjectiveStarted`
- emitir `OnObjectiveCompleted`

### Não deve fazer
- escrever ghost state
- escrever world mode
- escrever puzzle progress
- depender do GI para objective ownership

## 7.6 BP_PuzzleController
### Pode fazer
- iniciar puzzle
- restaurar puzzle
- avançar step
- validar sucesso/falha
- concluir puzzle
- emitir eventos de puzzle

### Não deve fazer
- ser o dono dos objetivos
- ser o dono do ghost state
- ser o dono do world mode
- tratar save como o seu storage runtime oficial
- mandar diretamente em todos os managers como regra normal

## 7.7 BP_ScareManager
### Pode fazer
- escolher scare válido
- gerir cooldowns
- instanciar scare actor
- correr scare loop
- emitir eventos de scare

### Não deve fazer
- ser writer de ghost state
- ser writer de world mode
- tratar save/objective como seu domínio

## 7.8 BP_AudioManager
### Pode fazer
- aplicar audio state
- correr ambience
- tocar stingers
- reagir a changes de ghost/world mode/scares/transitions

### Não deve fazer
- ser writer de `CurrentGhostId`
- ser writer de `CurrentGhostState`
- ser writer de `CurrentWorldMode`

## 7.9 BP_TransitionManager
### Pode fazer
- fades
- pending transition local
- travel
- spawn portal visual
- emitir `OnTransitionStarted`
- emitir `OnTransitionFinished`

### Não deve fazer
- decidir sozinho objetivos
- decidir sozinho ghost state
- decidir sozinho puzzle progress
- escrever world mode como se fosse o dono desse domínio

---

## 8. APIs públicas obrigatórias por manager

## 8.1 BP_GhostManager
### Getters
- `GetCurrentGhostId`
- `GetCurrentGhostState`
- `GetCurrentGhostProfile`

### Commands
- `SetActiveGhost`
- `SetGhostState`
- `RestoreGhostFromSave`

### Dispatchers
- `OnGhostChanged(OldGhostId, NewGhostId)`
- `OnGhostStateChanged(OldState, NewState)`

## 8.2 BP_WorldModeSystem
### Getters
- `GetWorldMode`
- `IsInMR`
- `IsInVRMemory`

### Commands
- `SetWorldMode`
- `RestoreWorldModeFromSave`

### Dispatchers
- `OnWorldModeChanged(OldMode, NewMode)`

## 8.3 BP_ObjectiveManager
### Getters
- `GetCurrentQuestId`
- `GetCurrentObjectiveId`
- `GetObjectiveStates`

### Commands
- `StartObjectiveById`
- `CompleteObjectiveById`
- `FailObjectiveById`
- `RestoreObjectivesFromSave`

### Dispatchers
- `OnObjectiveStarted(ObjectiveId)`
- `OnObjectiveCompleted(ObjectiveId)`

## 8.4 BP_PuzzleController
### Getters
- `GetCurrentPuzzleId`
- `GetCurrentPuzzleStepIndex`
- `IsPuzzleActive`
- `GetCompletedPuzzleIds`

### Commands
- `StartPuzzleByRowName`
- `StartPuzzleByRowNameAndStep`
- `ReportStepSuccess`
- `ReportStepFailure`
- `CompleteCurrentPuzzle`
- `RestorePuzzleFromSave`

### Dispatchers
- `OnPuzzleStarted(PuzzleId)`
- `OnPuzzleStepActivated(PuzzleId, StepId, StepIndex)`
- `OnPuzzleStepSuccess(PuzzleId, StepId, StepIndex)`
- `OnPuzzleStepFailure(PuzzleId, StepId, StepIndex)`
- `OnPuzzleCompleted(PuzzleId)`

## 8.5 BP_SaveManager
### Getters
- `IsSaveSystemReady`
- `GetCurrentSaveData`
- `GetSaveSnapshot`

### Commands
- `InitializeSaveSystem`
- `CreateNewSave`
- `LoadSave`
- `BuildSnapshotFromManagers`
- `ApplySnapshotToManagers`
- `WriteSaveToDisk`
- `RequestAutoSave`
- `RestoreFullGameState`

### Dispatchers opcionais
- `OnSaveLoaded`
- `OnSaveWritten`
- `OnRestoreStarted`
- `OnRestoreFinished`

## 8.6 BP_ScareManager
### Getters
- `CanTriggerScare`
- `GetCurrentScareId`
- `GetActiveScareIds`

### Commands
- `InitializeScareRuntime`
- `TryTriggerNextScare`
- `ExecuteScareById`
- `HandleScareFinished`

### Dispatchers
- `OnScareStarted(ScareId)`
- `OnScareFinished(ScareId)`

## 8.7 BP_AudioManager
### Getters
- `GetCurrentAudioStateId`

### Commands
- `ApplyAudioStateById`
- `RefreshAudioFromCurrentState`
- `PlayStinger`
- `HandleGhostChanged`
- `HandleGhostStateChanged`
- `HandleWorldModeChanged`
- `HandleTransitionStarted`
- `HandleTransitionFinished`

## 8.8 BP_TransitionManager
### Getters
- `IsTransitionInProgress`

### Commands
- `StartMemoryTransition`
- `TravelToLevelName`
- `TravelToLevelRef`
- `PlayFadeIn`
- `PlayFadeOut`
- `RestorePendingTransitionIfAny`

### Dispatchers
- `OnTransitionStarted`
- `OnTransitionFinished`

---

## 9. Mapa oficial de eventos

## 9.1 Ghost events
`BP_GhostManager` emite:
- `OnGhostChanged`
- `OnGhostStateChanged`

Devem ouvir:
- `BP_AudioManager`
- `BP_ScareManager`
- `BP_ObjectiveManager` apenas se houver regra legítima de objective setup
- qualquer UI ou system de narrativa que dependa do ghost

## 9.2 World mode events
`BP_WorldModeSystem` emite:
- `OnWorldModeChanged`

Devem ouvir:
- `BP_AudioManager`
- `BP_ScareManager`
- `BP_TransitionManager` se necessário
- qualquer UI / comfort / VFX dependente do modo

## 9.3 Puzzle events
`BP_PuzzleController` emite:
- `OnPuzzleStarted`
- `OnPuzzleStepActivated`
- `OnPuzzleStepSuccess`
- `OnPuzzleStepFailure`
- `OnPuzzleCompleted`

Devem ouvir:
- `BP_ObjectiveManager`
- `BP_ScareManager`
- `BP_TransitionManager`
- `BP_AudioManager`
- `BP_SaveManager` apenas para snapshot/autosave, não para ownership do puzzle

## 9.4 Objective events
`BP_ObjectiveManager` emite:
- `OnObjectiveStarted`
- `OnObjectiveCompleted`

Devem ouvir:
- UI
- Save snapshot pipeline
- qualquer system de progressão legítimo

## 9.5 Transition events
`BP_TransitionManager` emite:
- `OnTransitionStarted`
- `OnTransitionFinished`

Devem ouvir:
- `BP_AudioManager`
- `BP_WorldModeSystem` se a alteração de modo estiver acoplada ao fim/início
- Save snapshot pipeline
- UI / VFX auxiliares

---

## 10. Fluxos oficiais de runtime

## 10.1 Fluxo oficial: mudança de ghost
1. Algum trigger legítimo pede mudança de ghost
2. `BP_GhostManager.SetActiveGhost` valida e muda ghost
3. `BP_GhostManager` escreve o seu próprio estado
4. `BP_GhostManager` dispara `OnGhostChanged`
5. `BP_AudioManager` reage
6. `BP_ScareManager` reage
7. `BP_ObjectiveManager` reage se o design assim o exigir
8. `BP_SaveManager` guarda snapshot se apropriado

### Regra
O `GhostManager` não chama diretamente ObjectiveManager e ScareManager como regra base.

## 10.2 Fluxo oficial: mudança de world mode
1. Um pedido legítimo entra no `BP_WorldModeSystem`
2. `BP_WorldModeSystem.SetWorldMode` valida e atualiza
3. Escreve apenas o seu domínio
4. Dispara `OnWorldModeChanged`
5. Audio, scare, transition e outros systems reagem
6. Save snapshot opcional

### Regra
Ninguém fora do `WorldModeSystem` escreve `CurrentWorldMode`.

## 10.3 Fluxo oficial: step de puzzle com sucesso
1. Um actor de puzzle reporta sucesso ao `BP_PuzzleController`
2. `BP_PuzzleController` atualiza step/puzzle state
3. `BP_PuzzleController` dispara `OnPuzzleStepSuccess`
4. `BP_ObjectiveManager` reage se existir consequência de objective
5. `BP_ScareManager` reage se existir scare associado
6. `BP_AudioManager` reage se existir stinger/estado
7. `BP_SaveManager` agenda autosave se apropriado

### Regra
O `BP_PuzzleController` não deve escrever objectives nem save state de outro domínio diretamente como fluxo normal.

## 10.4 Fluxo oficial: conclusão de puzzle
1. `BP_PuzzleController` conclui puzzle
2. Atualiza `CompletedPuzzleIds`
3. Dispara `OnPuzzleCompleted`
4. Outros systems reagem
5. Save snapshot

### Regra
A conclusão do puzzle é do domínio do PuzzleController. As consequências são reativas.

## 10.5 Fluxo oficial: transição MR -> VR ou VR -> MR
1. Um system legítimo pede `StartMemoryTransition` ou travel equivalente
2. `BP_TransitionManager` inicia fade e estado interno
3. `OnTransitionStarted`
4. Quando apropriado, `BP_WorldModeSystem` muda de modo
5. `BP_TransitionManager` executa travel
6. `OnTransitionFinished`
7. systems reativos alinham estado audiovisual
8. Save snapshot se necessário

### Regra
TransitionManager executa. WorldModeSystem possui o modo. SaveManager persiste. Cada um fica no seu papel.

---

## 11. Fluxo oficial de save e restore

## 11.1 Save
### Regra fundamental
Save nunca é a autoridade primária do runtime.

### Fluxo
1. Evento de autosave ou save manual
2. `BP_SaveManager.BuildSnapshotFromManagers`
3. GhostManager fornece ghost state
4. WorldModeSystem fornece world mode
5. ObjectiveManager fornece objective state
6. PuzzleController fornece puzzle progress
7. TransitionManager fornece apenas o que fizer sentido persistir
8. SaveManager escreve no save object
9. SaveManager escreve em disco

## 11.2 Restore
### Regra fundamental
Restore tem ordem fixa.

### Ordem oficial de restore
1. `GI_Echoes` garante que as refs dos managers existem
2. `BP_SaveManager.LoadSave`
3. `BP_SaveManager` lê o snapshot
4. `BP_SaveManager.ApplySnapshotToManagers`
5. `BP_GhostManager.RestoreGhostFromSave`
6. `BP_WorldModeSystem.RestoreWorldModeFromSave`
7. `BP_ObjectiveManager.RestoreObjectivesFromSave`
8. `BP_PuzzleController.RestorePuzzleFromSave`
9. systems reativos fazem refresh a partir do estado restaurado
10. `BP_AudioManager.RefreshAudioFromCurrentState`
11. `BP_ScareManager` reinicia loop/runtime adequado
12. `BP_TransitionManager` limpa qualquer resto inválido de transição

### Nota importante
Restore não deve depender de “cada system ir buscar o que quer ao save quando lhe apetece”. O SaveManager distribui; os donos aplicam.

---

## 12. O que deve ser removido da arquitetura atual

### Remover como padrão
- `BP_PuzzleController` a escrever save state de puzzle como se o save fosse o dono do puzzle
- `BP_PuzzleController` a mandar diretamente em objective, ghost state, world mode, save e transition em cadeia rígida
- `BP_GhostManager` a inicializar diretamente objectives/scares como regra base
- `BP_AudioManager` a escrever ghost/world mode state
- `GI_Echoes` a escrever objective/world/ghost/puzzle runtime
- `BP_WorldModeSystem` a depender do save como runtime authority do modo
- `BP_TransitionManager` a escrever world mode como se fosse o dono

### Pode continuar a existir temporariamente durante migração
- alguns calls diretos de conveniência
- alguns getters do save para compatibilidade
- alguns bridges em BeginPlay

### Mas o objetivo final
remover todos os múltiplos writers dos domínios core.

---

## 13. Plano de migração para esta arquitetura

## Fase A — Constituição e contratos
- fechar este documento
- aprovar ownership final
- congelar criação de novas violações

## Fase B — Dispatchers oficiais
- criar ou consolidar todos os dispatchers definidos neste documento

## Fase C — SaveManager passivo
- transformar o SaveManager em snapshot/persistência
- retirar-lhe autoridade runtime sobre ghost/world/objective/puzzle

## Fase D — Ghost e World Mode como donos reais
- garantir que ghost só é escrito pelo GhostManager
- garantir que world mode só é escrito pelo WorldModeSystem

## Fase E — PuzzleController redesenhado
- cortar side effects diretos
- fazer com que emita eventos e mantenha só o puzzle domain

## Fase F — ObjectiveManager isolado
- cortar ownership partilhado com GI ou outros systems

## Fase G — GI emagrecido
- reduzir GI a registry/bootstrap/session helper

## Fase H — Audio e Scare totalmente reativos
- remover writes indevidos
- modularizar reação a events

## Fase I — Restore pipeline final
- implementar ordem oficial de restore
- validar regressão

---

## 14. Critérios de aceitação da re-arquitetura

A re-arquitetura só está concluída quando todas estas afirmações forem verdadeiras.

### Ownership
- cada variável core tem um único writer
- save não é runtime authority
- GI não é gameplay authority paralela

### Comunicação
- flows principais dependem de events/dispatchers
- calls diretas entre managers só existem quando fazem sentido estrutural
- consequência não implica ownership

### Restore
- restore funciona com ordem fixa
- world mode, ghost, objectives e puzzle restauram sem divergência
- áudio e scares alinham depois do restore

### Escalabilidade
- adicionar um novo ghost não obriga a mexer em cinco domínios ao mesmo tempo
- adicionar um novo puzzle não implica aumentar o acoplamento do GI
- adicionar novas consequências de puzzle não obriga a transformar o PuzzleController em deus do projeto

### Robustez
- save/load consistente
- transições consistentes
- objective chain consistente
- puzzle restore consistente

---

## 15. Regras finais que nunca devem ser quebradas

1. Nenhum manager escreve variáveis core de outro domínio.
2. SaveManager não é a verdade viva do runtime.
3. GI_Echoes não é uma cópia dos outros managers.
4. PuzzleController é dono de puzzle progress, não do jogo inteiro.
5. GhostManager muda ghosts; não manda no mundo inteiro.
6. WorldModeSystem é o único dono de MR/VR.
7. ObjectiveManager é o único dono de objectives/quests.
8. Audio e Scare reagem; não governam ghost/world mode.
9. Transitions executam; não decidem narrativa.
10. Restore segue ordem oficial, não improviso.

---

## 16. Conclusão operacional

A arquitetura perfeita para Echoes Between Worlds não é a que tem mais Blueprints nem mais managers. É a que deixa cada system saber exatamente:
- o que lhe pertence
- o que não lhe pertence
- quem anuncia mudanças
- quem reage
- quem persiste
- quem restaura

Se esta constituição for seguida, o projeto deixa de crescer em acoplamento e passa a crescer em previsibilidade.

**Regra final:**  
um projeto modular não é aquele que tem muitos managers.  
É aquele em que cada manager tem um dono claro, um contrato claro e um limite claro.
