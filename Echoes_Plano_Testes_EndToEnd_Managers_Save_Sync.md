# Plano de Testes End-to-End — Echoes Between Worlds

**Projeto:** Echoes Between Worlds  
**Engine alvo:** Unreal Engine 5.5 / 5.6 com Meta XR  
**Objetivo deste documento:** testar a arquitetura nova de ponta a ponta, cobrindo sincronismo entre managers, save/load, restore, world mode, ghost, objectives, puzzles, audio, scares e transitions.

---

## 0. Como usar este documento

Este plano não é para testar “tudo ao mesmo tempo”.  
A ideia é testar por camadas, como se estivéssemos a verificar uma casa:

1. Primeiro vemos se os alicerces existem.
2. Depois vemos se a eletricidade chega às divisões.
3. Depois vemos se os interruptores ligam as luzes certas.
4. Só no fim testamos a casa inteira com tudo ligado.

No teu jogo, os “alicerces” são:

- `GI_Echoes`
- `BP_SaveManager`
- `BP_GhostManager`
- `BP_WorldModeSystem`
- `BP_ObjectiveManager`
- `BP_PuzzleController`
- `BP_AudioManager`
- `BP_ScareManager`
- `BP_TransitionManager`
- `BP_DebugManager`

A ordem recomendada é:

```text
Fase 1  — Preparação do ambiente
Fase 2  — Verificar GI_Echoes e registo dos managers
Fase 3  — Verificar SaveManager isolado
Fase 4  — Verificar restore completo
Fase 5  — Verificar GhostManager
Fase 6  — Verificar WorldModeSystem
Fase 7  — Verificar ObjectiveManager
Fase 8  — Verificar PuzzleController
Fase 9  — Verificar PuzzleActors
Fase 10 — Verificar TransitionManager
Fase 11 — Verificar AudioManager
Fase 12 — Verificar ScareManager
Fase 13 — Teste MR completo
Fase 14 — Teste MR → VRMemory
Fase 15 — Teste VRMemory completo
Fase 16 — Teste VRMemory → MR
Fase 17 — Teste final de persistência
Fase 18 — Teste de regressão rápida
```

---

# 1. Regras da arquitetura que o teste tem de validar

Antes dos testes, tens de ter estas regras bem claras.

## 1.1 Donos oficiais dos dados

| Sistema | Dono oficial de quê |
|---|---|
| `GI_Echoes` | Sessão, registry de managers, pending transition, debug global |
| `BP_SaveManager` | Save/load, snapshot, restore, autosave |
| `BP_GhostManager` | Ghost atual, estado do ghost, perfil do ghost, ghosts limpos |
| `BP_WorldModeSystem` | `E_WorldMode`, MR ou VRMemory |
| `BP_ObjectiveManager` | Objetivos, objective states, objective ativo |
| `BP_PuzzleController` | Puzzle atual, step atual, puzzle progress |
| `BP_AudioManager` | Ambience, music, stingers, audio state |
| `BP_ScareManager` | Seleção, validação e execução de scares |
| `BP_TransitionManager` | Fade, transição entre níveis, pending arrival |
| `BP_DebugManager` / `BFL_Debug` | Debug centralizado |

## 1.2 Regra importante

Nenhum manager deve “mandar” no domínio de outro diretamente.

Exemplo errado:

```text
AudioManager decide mudar CurrentGhostId.
```

Exemplo certo:

```text
GhostManager muda CurrentGhostId.
AudioManager recebe evento ou lê contexto.
AudioManager adapta áudio.
```

Exemplo errado:

```text
PuzzleActor decide avançar para o próximo step.
```

Exemplo certo:

```text
PuzzleActor reporta sucesso.
PuzzleController decide avançar para o próximo step.
```

---

# 2. Preparação inicial

## 2.1 Confirmar Game Instance

### Onde ir

```text
Edit
→ Project Settings
→ Project
→ Maps & Modes
→ Game Instance Class
```

### Esperado

```text
Game Instance Class = GI_Echoes
```

### Se estiver errado

Seleciona:

```text
GI_Echoes
```

Guarda o projeto.

---

## 2.2 Confirmar managers no mapa de teste

No teu mapa principal MR, confirma se existem estes actores no World Outliner:

```text
BP_SaveManager
BP_GhostManager
BP_WorldModeSystem
BP_ObjectiveManager
BP_PuzzleController
BP_AudioManager
BP_ScareManager
BP_TransitionManager
BP_DebugManager
```

### Esperado

Todos existem **uma vez**.

### Não deve acontecer

```text
2 BP_SaveManager no mesmo mapa
2 BP_GhostManager no mesmo mapa
2 BP_PuzzleController no mesmo mapa
```

Se houver duplicados, remove os duplicados.  
Ter dois managers iguais é como ter dois maestros a conduzir a mesma orquestra: vai dar confusão.

---

## 2.3 Criar save slot de teste

Para não misturares com saves antigos, usa um save slot temporário.

### Onde

Abre:

```text
BP_SaveManager
```

Procura a variável:

```text
SaveSlotName
```

### Valor recomendado

```text
Echoes_TestSlot_01
```

### Resultado esperado

Todo o teste usa este slot novo e limpo.

---

## 2.4 Ativar debug

No `GI_Echoes` ou no teu sistema de debug, garante que as categorias principais estão ativas:

```text
Save
Ghost
WorldMode
Objective
Puzzle
Audio
Scare
Transition
Core
```

Se estiveres a usar `BFL_Debug`, os logs devem usar mensagens claras com:

```text
Category
SenderName
Message
Duration
```

---

# 3. Criar ferramenta temporária de teste: `BP_TestHarness_E2E`

Esta parte é opcional, mas altamente recomendada.

O objetivo é ter um actor simples no mapa para chamar funções de teste sem andares sempre a criar nodes espalhados nos managers.

## 3.1 Criar o Blueprint

### Passos

```text
Content Browser
→ Add
→ Blueprint Class
→ Actor
→ Nome: BP_TestHarness_E2E
```

Coloca este actor no mapa MR de teste.

---

## 3.2 Variáveis do `BP_TestHarness_E2E`

Cria estas variáveis:

| Nome | Tipo | Instance Editable | Descrição |
|---|---|---|---|
| `GIRef` | `GI_Echoes Object Reference` | Não | Referência ao Game Instance |
| `SaveManagerRef` | `BP_SaveManager Object Reference` | Não | Save manager |
| `GhostManagerRef` | `BP_GhostManager Object Reference` | Não | Ghost manager |
| `WorldModeSystemRef` | `BP_WorldModeSystem Object Reference` | Não | World mode |
| `ObjectiveManagerRef` | `BP_ObjectiveManager Object Reference` | Não | Objective manager |
| `PuzzleControllerRef` | `BP_PuzzleController Object Reference` | Não | Puzzle controller |
| `AudioManagerRef` | `BP_AudioManager Object Reference` | Não | Audio manager |
| `ScareManagerRef` | `BP_ScareManager Object Reference` | Não | Scare manager |
| `TransitionManagerRef` | `BP_TransitionManager Object Reference` | Não | Transition manager |

---

# 4. Função de teste: `ResolveManagerRefs`

Esta função vai buscar as referências ao `GI_Echoes`.

## 4.1 Criar função

No `BP_TestHarness_E2E`:

```text
My Blueprint
→ Functions
→ +
→ Nome: ResolveManagerRefs
```

---

## 4.2 `ResolveManagerRefs` — node por node

### Node 1

```text
Function Entry
```

### Node 2

```text
Get Game Instance
```

Liga:

```text
Function Entry exec → Get Game Instance
```

### Node 3

```text
Cast To GI_Echoes
```

Liga:

```text
Get Game Instance Return Value → Cast To GI_Echoes Object
Get Game Instance exec → Cast To GI_Echoes exec
```

### Node 4

```text
Set GIRef
```

Liga:

```text
Cast To GI_Echoes As GI_Echoes → Set GIRef
Cast Success exec → Set GIRef exec
```

### Node 5

```text
Get SaveManagerRef
```

Este node é uma chamada à função do `GI_Echoes`:

```text
GetSaveManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 6

```text
Set SaveManagerRef
```

Liga:

```text
GetSaveManagerRef Return Value → Set SaveManagerRef
Set GIRef exec → Set SaveManagerRef exec
```

### Node 7

```text
Get GhostManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 8

```text
Set GhostManagerRef
```

Liga:

```text
GetGhostManagerRef Return Value → Set GhostManagerRef
Set SaveManagerRef exec → Set GhostManagerRef exec
```

### Node 9

```text
Get WorldModeSystemRef
```

Liga:

```text
GIRef → Target
```

### Node 10

```text
Set WorldModeSystemRef
```

Liga:

```text
GetWorldModeSystemRef Return Value → Set WorldModeSystemRef
Set GhostManagerRef exec → Set WorldModeSystemRef exec
```

### Node 11

```text
Get ObjectiveManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 12

```text
Set ObjectiveManagerRef
```

Liga:

```text
GetObjectiveManagerRef Return Value → Set ObjectiveManagerRef
Set WorldModeSystemRef exec → Set ObjectiveManagerRef exec
```

### Node 13

```text
Get PuzzleControllerRef
```

Liga:

```text
GIRef → Target
```

### Node 14

```text
Set PuzzleControllerRef
```

Liga:

```text
GetPuzzleControllerRef Return Value → Set PuzzleControllerRef
Set ObjectiveManagerRef exec → Set PuzzleControllerRef exec
```

### Node 15

```text
Get AudioManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 16

```text
Set AudioManagerRef
```

Liga:

```text
GetAudioManagerRef Return Value → Set AudioManagerRef
Set PuzzleControllerRef exec → Set AudioManagerRef exec
```

### Node 17

```text
Get ScareManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 18

```text
Set ScareManagerRef
```

Liga:

```text
GetScareManagerRef Return Value → Set ScareManagerRef
Set AudioManagerRef exec → Set ScareManagerRef exec
```

### Node 19

```text
Get TransitionManagerRef
```

Liga:

```text
GIRef → Target
```

### Node 20

```text
Set TransitionManagerRef
```

Liga:

```text
GetTransitionManagerRef Return Value → Set TransitionManagerRef
Set ScareManagerRef exec → Set TransitionManagerRef exec
```

### Node 21

```text
Print String
```

Texto:

```text
[TestHarness] ResolveManagerRefs completed
```

Liga:

```text
Set TransitionManagerRef exec → Print String exec
```

---

# 5. Função de teste: `DebugRegisteredManagers`

Esta função imprime se cada manager está válido.

## 5.1 Criar função

```text
BP_TestHarness_E2E
→ Functions
→ +
→ DebugRegisteredManagers
```

---

## 5.2 `DebugRegisteredManagers` — node por node

### Node 1

```text
Function Entry
```

### Node 2

```text
ResolveManagerRefs
```

Liga:

```text
Function Entry exec → ResolveManagerRefs exec
```

### Node 3

```text
Is Valid
```

Para `SaveManagerRef`.

### Node 4

```text
Select String
```

Condition:

```text
Is Valid Return Value
```

True:

```text
SaveManagerRef = VALID
```

False:

```text
SaveManagerRef = INVALID
```

### Node 5

```text
Print String
```

Liga:

```text
ResolveManagerRefs exec → Is Valid exec/flow visual se usares sequência
Select String Return Value → Print String In String
```

Como precisas de vários prints, usa:

```text
Sequence
```

---

## 5.3 Versão com `Sequence`

### Node 1

```text
Function Entry
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Sequence
```

Liga:

```text
Function Entry → ResolveManagerRefs → Sequence
```

Cria os pins:

```text
Then 0
Then 1
Then 2
Then 3
Then 4
Then 5
Then 6
Then 7
```

Cada pin testa um manager.

---

## 5.4 Teste do `SaveManagerRef`

### Node A

```text
Is Valid
```

Input:

```text
SaveManagerRef
```

### Node B

```text
Branch
```

Condition:

```text
Is Valid Return Value
```

### Node C — True

```text
Print String
```

Texto:

```text
[TestHarness] SaveManagerRef = VALID
```

### Node D — False

```text
Print String
```

Texto:

```text
[TestHarness] SaveManagerRef = INVALID
```

### Ligações

```text
Sequence Then 0 → Branch exec
SaveManagerRef → IsValid Object
IsValid ReturnValue → Branch Condition
Branch True → Print VALID
Branch False → Print INVALID
```

Repete igual para:

```text
GhostManagerRef
WorldModeSystemRef
ObjectiveManagerRef
PuzzleControllerRef
AudioManagerRef
ScareManagerRef
TransitionManagerRef
```

---

# 6. BeginPlay do `BP_TestHarness_E2E`

No Event Graph:

## Node 1

```text
Event BeginPlay
```

## Node 2

```text
Delay
```

Duration:

```text
1.0
```

Isto dá tempo para os managers se registarem.

## Node 3

```text
DebugRegisteredManagers
```

### Ligações

```text
Event BeginPlay → Delay
Delay Completed → DebugRegisteredManagers
```

---

# 7. Fase 1 — Teste de arranque limpo

## Objetivo

Confirmar que todos os managers arrancam e se registam.

## Preparação

Usa o save slot:

```text
Echoes_TestSlot_01
```

Apaga saves antigos se necessário.

## Como executar

```text
1. Abre o mapa MR principal.
2. Garante que BP_TestHarness_E2E está no mapa.
3. Carrega Play.
4. Espera 2 segundos.
5. Lê os logs.
```

## Esperado

Logs parecidos com:

```text
[GI_Echoes] InitializeSession completed
[GhostManager] InitializeGhostManager completed
[WorldModeSystem] InitializeWorldModeSystem completed
[ObjectiveManager] InitializeObjectiveManager completed
[PuzzleController] InitializePuzzleController completed
[AudioManager] InitializeAudioManager completed
[ScareManager] InitializeScareManager completed
[TransitionManager] InitializeTransitionManager completed
[SaveManager] InitializeSaveSystem completed
[TestHarness] SaveManagerRef = VALID
[TestHarness] GhostManagerRef = VALID
[TestHarness] WorldModeSystemRef = VALID
[TestHarness] ObjectiveManagerRef = VALID
[TestHarness] PuzzleControllerRef = VALID
[TestHarness] AudioManagerRef = VALID
[TestHarness] ScareManagerRef = VALID
[TestHarness] TransitionManagerRef = VALID
```

## Falha

Se algum aparecer `INVALID`, verifica:

```text
1. O actor existe no mapa?
2. O BeginPlay dele corre?
3. Ele chama RegisterXManager no GI?
4. O GI_Echoes está mesmo definido em Maps & Modes?
```

---

# 8. Fase 2 — Teste do SaveManager com save novo

## Objetivo

Confirmar que o SaveManager cria save novo quando não existe.

## Como executar

```text
1. Apaga o save do slot Echoes_TestSlot_01.
2. Play.
3. Espera 2 segundos.
4. Para o PIE.
```

## Esperado

Logs:

```text
[SaveManager] DoesSaveGameExist = false
[SaveManager] CreateNewSave
[SaveManager] WriteSaveToDisk
[SaveManager] SaveSystemReady = true
```

## Verificação manual

Depois de parar o PIE, abre outra vez o jogo.

## Esperado na segunda execução

```text
[SaveManager] DoesSaveGameExist = true
[SaveManager] LoadSave
[SaveManager] RestoreFullGameState
[SaveManager] ApplySnapshotToManagers
```

## Falha

Se cria save novo todas as vezes:

```text
Possíveis causas:
- SaveSlotName muda entre execuções.
- UserIndex muda.
- WriteSaveToDisk não está a executar.
- SaveGameObject não está válido.
```

---

# 9. Fase 3 — Teste de restore completo

## Objetivo

Validar a ordem de restore entre managers.

## Ordem ideal

```text
Managers registam no GI
→ SaveManager InitializeSaveSystem
→ TryInitialRestoreAfterBootstrap
→ ResolveOwnerManagerRefs
→ RestoreFullGameState
→ ApplySnapshotToManagers
→ GhostManager.RestoreGhostFromSave
→ WorldModeSystem.RestoreWorldModeFromSave
→ ObjectiveManager.RestoreObjectivesFromSave
→ PuzzleController.RestorePuzzleFromSave
→ AudioManager.ApplyResolvedAudioStateFromContext
→ ScareManager.RestartScareLoopForCurrentState
→ TransitionManager.HandlePendingArrivalTransition
```

## Como executar

```text
1. Garante que já existe save.
2. Play.
3. Não interajas.
4. Observa logs de ordem.
```

## Esperado

```text
RestoreFullGameState corre apenas uma vez.
ApplySnapshotToManagers corre apenas uma vez.
Ghost, WorldMode, Objectives e Puzzle são restaurados.
Audio aplica estado depois do contexto estar definido.
Scare loop começa depois de ghost/world válidos.
```

## Não deve acontecer

```text
Puzzle arranca automaticamente antes do restore.
Scare loop começa com CurrentGhostId = None.
Audio aplica MR e imediatamente depois VR sem motivo.
WorldMode muda duas vezes no arranque sem transição.
```

---

# 10. Fase 4 — Teste do GhostManager

## Objetivo

Confirmar que o ghost ativo é definido, restaurado e propagado.

## 10.1 Criar input temporário para forçar ghost

No `BP_TestHarness_E2E`, no Event Graph, cria um input temporário.

### Node 1

```text
Keyboard G
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid
```

Object:

```text
GhostManagerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
SetActiveGhost
```

Target:

```text
GhostManagerRef
```

Input:

```text
GhostId = Isabel
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] Forced SetActiveGhost Isabel
```

### Ligações

```text
Keyboard G Pressed → ResolveManagerRefs
ResolveManagerRefs → Branch
GhostManagerRef → IsValid
IsValid ReturnValue → Branch Condition
Branch True → SetActiveGhost
SetActiveGhost → Print String
```

## 10.2 Executar teste

```text
1. Play.
2. Carrega G.
3. Observa logs.
```

## Esperado

```text
[GhostManager] SetActiveGhost Isabel
[GhostManager] OnGhostChanged Old=None New=Isabel
[ObjectiveManager] HandleGhostChanged Isabel
[PuzzleController] HandleGhostChanged Isabel
[AudioManager] HandleGhostAudioContextChanged
[ScareManager] HandleGhostChanged Isabel
[SaveManager] RequestAutoSave Reason=GhostChanged
```

## Falha

Se só o GhostManager muda, mas os outros não reagem:

```text
Problema provável:
- Bind ao OnGhostChanged não foi feito.
- GhostManagerRef estava inválido quando o binding correu.
- Falta retry no manager que não reagiu.
```

---

# 11. Fase 5 — Teste do WorldModeSystem

## Objetivo

Confirmar que `MR` e `VRMemory` são controlados pelo `BP_WorldModeSystem`.

## 11.1 Criar input para MR

No `BP_TestHarness_E2E`.

### Node 1

```text
Keyboard 1
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid
```

Object:

```text
WorldModeSystemRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
SetWorldMode
```

Target:

```text
WorldModeSystemRef
```

Input:

```text
NewWorldMode = MR
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] Forced WorldMode MR
```

---

## 11.2 Criar input para VRMemory

Igual, mas com:

```text
Keyboard 2
SetWorldMode NewWorldMode = VRMemory
Print String = [TestHarness] Forced WorldMode VRMemory
```

---

## 11.3 Executar teste

```text
1. Play.
2. Carrega 1.
3. Confirma MR.
4. Carrega 2.
5. Confirma VRMemory.
6. Carrega 1 outra vez.
7. Confirma MR.
```

## Esperado

Ao mudar para MR:

```text
[WorldModeSystem] SetWorldMode MR
[WorldModeSystem] OnWorldModeChanged Old=... New=MR
[AudioManager] HandleWorldModeAudioContextChanged
[ScareManager] HandleWorldModeChanged
[PuzzleController] HandleWorldModeChanged
[SaveManager] RequestAutoSave Reason=WorldModeChanged
```

Ao mudar para VRMemory:

```text
[WorldModeSystem] SetWorldMode VRMemory
[AudioManager] aplica áudio VRMemory
[ScareManager] passa a aceitar scares VRMemory
[PuzzleController] atualiza contexto
```

## Falha

Se o áudio ou scares não mudam:

```text
Provável:
- Binding ao OnWorldModeChanged não foi feito.
- WorldModeSystemRef inválido no manager.
```

---

# 12. Fase 6 — Teste do ObjectiveManager

## Objetivo

Confirmar que objetivos iniciam, completam, guardam e restauram.

## 12.1 Criar input para começar objetivo

No `BP_TestHarness_E2E`.

### Node 1

```text
Keyboard O
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid ObjectiveManagerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
StartObjective
```

Input exemplo:

```text
ObjectiveId = OBJ_Isabel_FindRunes
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] StartObjective OBJ_Isabel_FindRunes
```

## 12.2 Criar input para completar objetivo

### Node 1

```text
Keyboard P
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
CompleteObjective
```

Input:

```text
ObjectiveId = OBJ_Isabel_FindRunes
```

### Node 4

```text
Print String
```

Texto:

```text
[TestHarness] CompleteObjective OBJ_Isabel_FindRunes
```

---

## 12.3 Executar teste

```text
1. Play.
2. Carrega O.
3. Confirma objetivo ativo.
4. Carrega P.
5. Confirma objetivo completed.
6. Espera autosave.
7. Fecha PIE.
8. Abre outra vez.
```

## Esperado

Depois do Start:

```text
ObjectiveId = OBJ_Isabel_FindRunes
Status = Active
SaveManager RequestAutoSave
```

Depois do Complete:

```text
OBJ_Isabel_FindRunes = Completed
Próximo objective = Active
SaveManager RequestAutoSave
```

Depois de reabrir:

```text
OBJ_Isabel_FindRunes continua Completed
Próximo objective continua Active
```

## Falha

Se ao reabrir volta tudo a Locked:

```text
Provável:
- ObjectiveStates não estão a ser guardados no snapshot.
- RestoreObjectivesFromSave não está a aplicar estados.
- CurrentObjectiveId não está a ser reconstruído.
```

---

# 13. Fase 7 — Teste do PuzzleController

## Objetivo

Confirmar que o puzzle arranca, ativa steps, avança, guarda step e restaura.

---

## 13.1 Criar input para começar puzzle MR

No `BP_TestHarness_E2E`.

### Node 1

```text
Keyboard 3
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid PuzzleControllerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
StartPuzzleByRowName
```

Input:

```text
PuzzleRowName = PZ_Isabel_MR_RuneSequence
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] StartPuzzle PZ_Isabel_MR_RuneSequence
```

---

## 13.2 Criar input para completar step atual

### Node 1

```text
Keyboard 4
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid PuzzleControllerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
ReportStepSuccess
```

Target:

```text
PuzzleControllerRef
```

Se a função exigir StepId, usa:

```text
CurrentStepId
```

através de:

```text
PuzzleControllerRef.GetCurrentStepId
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] Forced ReportStepSuccess
```

---

## 13.3 Executar teste

```text
1. Apaga save.
2. Play.
3. Carrega 3 para iniciar puzzle MR.
4. Confirma StepIndex = 0.
5. Carrega 4 para completar step.
6. Confirma StepIndex = 1.
7. Espera autosave.
8. Fecha PIE.
9. Abre outra vez.
```

## Esperado

Antes de completar:

```text
CurrentPuzzleId = PZ_Isabel_MR_RuneSequence
CurrentStepIndex = 0
bHasActivePuzzle = true
bStepActive = true
```

Depois de completar:

```text
CurrentStepIndex = 1
Step 1 ativo
SaveManager RequestAutoSave Reason=PuzzleProgressChanged
```

Depois de reabrir:

```text
CurrentPuzzleId = PZ_Isabel_MR_RuneSequence
CurrentStepIndex = 1
Step 1 ativo
Step 0 não volta a ativar
```

## Falha

Se volta para Step 0:

```text
Possíveis causas:
- SyncPuzzleProgressToSave não chamou RequestAutoSave.
- BuildSnapshotFromManagers não leu CurrentStepIndex.
- RestorePuzzleFromSave não chamou StartPuzzleByRowNameAndStep.
```

---

# 14. Fase 8 — Teste dos PuzzleActors

## Objetivo

Confirmar que só o actor certo reage ao step certo.

## Preparação

Cada actor de puzzle deve ter:

```text
ExpectedPuzzleId
ExpectedStepId
bIsCurrentStepActive
```

## Teste

```text
1. Play.
2. Inicia PZ_Isabel_MR_RuneSequence.
3. Observa todos os actors de puzzle.
```

## Esperado

No Step 0:

```text
Actor com ExpectedStepId = Step0 → ativo
Actor com ExpectedStepId = Step1 → inativo
Actor com ExpectedStepId = Step2 → inativo
```

No Step 1:

```text
Actor Step0 → inativo
Actor Step1 → ativo
Actor Step2 → inativo
```

## Logs recomendados no `BP_PuzzleActorBase.HandleStepActivated`

```text
[PuzzleActorBase] HandleStepActivated
Received PuzzleId={PuzzleId}
Received StepId={StepId}
ExpectedPuzzleId={ExpectedPuzzleId}
ExpectedStepId={ExpectedStepId}
Match={true/false}
```

## Se precisares de adicionar este debug — node por node

Abre:

```text
BP_PuzzleActorBase
→ Function HandleStepActivated
```

### Node 1

```text
Function Entry
```

Já existe.

### Node 2

```text
Equal Name
```

Comparar:

```text
Input PuzzleId == ExpectedPuzzleId
```

### Node 3

```text
Equal Name
```

Comparar:

```text
Input StepId == ExpectedStepId
```

### Node 4

```text
AND Boolean
```

Liga:

```text
Puzzle match → AND A
Step match → AND B
```

### Node 5

```text
Set bIsCurrentStepActive
```

Valor:

```text
AND Return Value
```

### Node 6

```text
Format Text
```

Texto:

```text
[PuzzleActorBase] Activated? {Match} | Puzzle={PuzzleId} Step={StepId} | ExpectedPuzzle={ExpectedPuzzleId} ExpectedStep={ExpectedStepId}
```

### Node 7

```text
To String (Text)
```

### Node 8

```text
BFL_Debug.DebugInfo
```

Inputs:

```text
WorldContextObject = Self
Category = Puzzle
SenderName = GetSenderName(Self) ou "BP_PuzzleActorBase"
Message = ToString result
Duration = 5
```

### Ligações exec

```text
Function Entry → Set bIsCurrentStepActive → DebugInfo
```

---

# 15. Fase 9 — Teste do TransitionManager

## Objetivo

Confirmar MR → VRMemory e VRMemory → MR.

---

## 15.1 Criar input para testar transição manual

No `BP_TestHarness_E2E`.

### Node 1

```text
Keyboard T
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid TransitionManagerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

Chama:

```text
TravelToLevelName
```

ou:

```text
StartMemoryTransition
```

Depende da tua implementação atual.

### Se usares `TravelToLevelName`

Inputs exemplo:

```text
TargetLevelName = NomeDoMapaVR
TargetWorldMode = VRMemory
```

### Se usares `StartMemoryTransition`

Cria/usa um `S_MemoryTransition` com:

```text
TargetWorldMode = VRMemory
TargetLevelSoftRef = mapa VR
FadeOutSec = 1.0
FadeInSec = 1.0
```

## Esperado

```text
[TransitionManager] StartMemoryTransition
[TransitionManager] EnsureFadeWidget
[TransitionManager] PlayFadeOut
[TransitionManager] ExecuteLevelTravel
OpenLevel
Novo mapa carrega
[TransitionManager] HandlePendingArrivalTransition
FadeIn
WorldMode = VRMemory
```

## Falha

Se o mapa abre mas o mode fica errado:

```text
Verificar:
- PendingTransitionData no GI.
- TargetWorldMode.
- RestoreFullGameState está a sobrescrever WorldMode?
```

---

# 16. Fase 10 — Teste do AudioManager

## Objetivo

Confirmar que o áudio segue o contexto.

## Teste 16.1 — Estado inicial MR

```text
1. Apaga save.
2. Play.
3. Confirma audio state.
```

## Esperado

```text
AudioState = MR_Calm ou equivalente
Ambience MR toca
Sem stinger indevido
```

## Teste 16.2 — Mudar para VRMemory

Usa a tecla `2` criada antes.

## Esperado

```text
WorldMode = VRMemory
AudioManager.HandleWorldModeAudioContextChanged
ResolveAudioStateFromContext
ApplyAudioStateById
Ambience VR toca
Ambience MR pára
```

## Teste 16.3 — Mudar ghost state

Cria input temporário:

```text
Keyboard 5
→ GhostManager.SetGhostState(Agitated)
```

## Node-by-node

### Node 1

```text
Keyboard 5
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid GhostManagerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
SetGhostState
```

Input:

```text
NewState = Agitated
```

### Esperado

```text
GhostState = Agitated
AudioManager.HandleGhostStateAudioContextChanged
AudioState muda para estado de tensão
```

---

# 17. Fase 11 — Teste do ScareManager

## Objetivo

Confirmar que sustos só correm com contexto válido.

## 17.1 Teste de loop

```text
1. Play.
2. Espera restore.
3. Observa logs do ScareManager.
```

## Esperado

```text
ScareManager tem GhostManagerRef válido
ScareManager tem WorldModeSystemRef válido
CurrentGhostId válido
CurrentWorldMode válido
Scare loop só começa depois disto
```

## Não deve acontecer

```text
TryTriggerNextScare aborted -> CurrentGhostId is None
```

Se aparecer uma vez no arranque e depois corrigir, pode ser aceitável.  
Se aparecer repetidamente, é problema de sincronização.

---

## 17.2 Criar input para forçar scare por ID

No `BP_TestHarness_E2E`.

### Node 1

```text
Keyboard 6
```

### Node 2

```text
ResolveManagerRefs
```

### Node 3

```text
Is Valid ScareManagerRef
```

### Node 4

```text
Branch
```

### Node 5 — True

```text
ExecuteScareById
```

Input exemplo:

```text
ScareId = Isabel_MR_WhisperBack_01
```

### Node 6

```text
Print String
```

Texto:

```text
[TestHarness] Forced scare Isabel_MR_WhisperBack_01
```

---

## 17.3 Executar teste

```text
1. Play em MR.
2. Carrega 6.
3. Confirma se o scare executa.
4. Muda para VRMemory.
5. Tenta forçar scare MR.
```

## Esperado em MR

```text
Scare permitido.
Executa.
Regista cooldown.
```

## Esperado em VRMemory com scare MR

```text
Scare bloqueado por WorldMode.
```

## Teste de cooldown

```text
1. Carrega 6.
2. Carrega 6 outra vez logo a seguir.
```

## Esperado

```text
Primeira vez executa.
Segunda vez bloqueia por cooldown.
```

---

# 18. Fase 12 — Teste completo MR

## Objetivo

Testar a cadeia MR com puzzle, objetivo, autosave e scare.

## Como executar

```text
1. Apaga save.
2. Play no mapa MR.
3. Confirma Ghost = Isabel.
4. Confirma WorldMode = MR.
5. Confirma Objective inicial.
6. Confirma Puzzle MR ativo.
7. Completa Step 0.
8. Espera autosave.
9. Fecha PIE.
10. Abre outra vez.
11. Confirma Step 1.
12. Completa todos os steps.
13. Confirma CompleteCurrentPuzzle.
```

## Esperado

```text
CurrentGhostId = Isabel
CurrentWorldMode = MR
CurrentPuzzleId = PZ_Isabel_MR_RuneSequence
CurrentStepIndex avança corretamente
Objective atualiza
CompletedPuzzleIds recebe puzzle MR
Autosave após step
Autosave após puzzle completion
```

## Falha

Se StepIndex não restaura:

```text
Verificar SyncPuzzleProgressToSave.
Verificar BuildSnapshotFromManagers.
Verificar RestorePuzzleFromSave.
```

Se objective não muda:

```text
Verificar HandlePuzzleCompletionConsequences.
Verificar ObjectiveManager.StartObjective/CompleteObjective.
```

---

# 19. Fase 13 — Teste MR → VRMemory

## Objetivo

Testar transição real depois do puzzle MR.

## Como executar

```text
1. Completa puzzle MR.
2. Observa consequência.
3. Deixa TransitionManager fazer a transição.
4. Entra no mapa VRMemory.
```

## Esperado

```text
TransitionManager.StartMemoryTransition
FadeOut
OpenLevel para mapa VR
Novo mapa carrega
Managers registam novamente
SaveManager RestoreFullGameState
WorldMode = VRMemory
Ghost = Isabel
Puzzle VR começa/restaura
FadeIn
Audio VRMemory ativo
ScareManager em contexto VRMemory
```

## Não deve acontecer

```text
WorldMode fica MR no mapa VR.
Puzzle MR volta a arrancar dentro do mapa VR.
Objective volta para o anterior.
Fade fica preso preto.
```

---

# 20. Fase 14 — Teste completo VRMemory

## Objetivo

Testar puzzle VR, save a meio e restore.

## Como executar

```text
1. Já dentro do mapa VRMemory.
2. Confirma CurrentWorldMode = VRMemory.
3. Confirma puzzle VR ativo.
4. Completa Step 0.
5. Espera autosave.
6. Fecha PIE.
7. Abre novamente.
```

## Esperado

```text
Volta ao mapa/contexto esperado.
CurrentWorldMode = VRMemory.
CurrentPuzzleId = puzzle VR.
CurrentStepIndex = 1.
Step 1 ativo.
```

## Completar puzzle VR

```text
1. Completa todos os steps VR.
2. Observa CompleteCurrentPuzzle.
```

## Esperado

```text
Puzzle VR entra em CompletedPuzzleIds.
GhostState muda, se configurado.
Objective atualiza.
Transition para MR é disparada, se configurado.
```

---

# 21. Fase 15 — Teste VRMemory → MR

## Objetivo

Garantir que o retorno ao MR mantém progresso.

## Como executar

```text
1. Completa puzzle/ritual VR.
2. Deixa TransitionManager voltar ao mapa MR.
3. Observa restore no mapa MR.
```

## Esperado

```text
Mapa MR carrega.
WorldMode = MR.
Puzzle VR continua completo.
Puzzle MR continua completo.
Objective final ou próximo objetivo está correto.
GhostState = Banished ou estado final esperado.
Audio MR pós-memória ativo.
ScareManager não usa scares VR antigos.
```

## Falha

Se ao voltar ao MR o puzzle MR recomeça:

```text
CompletedPuzzleIds não foi guardado/restaurado.
```

Se ghost volta a Dormant:

```text
GhostState não foi guardado/restaurado.
```

---

# 22. Fase 16 — Teste final de persistência

## Objetivo

Confirmar que o jogo inteiro sobrevive a fechar/reabrir.

## Como executar

```text
1. Completa MR → VRMemory → MR.
2. Espera autosave final.
3. Fecha PIE.
4. Abre outra vez.
```

## Esperado

```text
Não recomeça puzzle MR.
Não recomeça puzzle VR.
Objectives antigos continuam Completed.
WorldMode final correto.
GhostState final correto.
CompletedPuzzleIds corretos.
ClearedGhostIds correto, se usado.
```

## Valores esperados no save

```text
CurrentWorldMode = MR
CurrentGhostId = Isabel ou None, conforme design final
CurrentGhostState = Banished ou estado final equivalente
CompletedPuzzleIds contém puzzle MR e puzzle VR
ObjectiveStates contém os estados finais
ClearedGhostIds contém Isabel, se a Isabel foi limpa
```

---

# 23. Fase 17 — Teste de autosave

## Objetivo

Confirmar que o autosave é chamado nos momentos certos.

## Eventos que devem pedir autosave

```text
Ghost changed
Ghost state changed
World mode changed
Objective started
Objective completed
Puzzle step changed
Puzzle completed
Transition relevante
```

## Como executar

Durante os testes anteriores, observa logs.

## Esperado

```text
[SaveManager] RequestAutoSave Reason=...
[SaveManager] AutoSave already pending. Updated reason=...
[SaveManager] ExecuteAutoSave
[SaveManager] WriteSaveToDisk
```

## Não deve acontecer

```text
AutoSave request ignored because auto save is disabled
```

durante gameplay normal, a menos que tenhas desligado de propósito.

---

# 24. Fase 18 — Teste de regressão rápida

Este é o teste que deves repetir sempre que mexeres em managers.

## Passos

```text
1. Apagar save.
2. Play no MR.
3. Confirmar managers válidos.
4. Confirmar Ghost = Isabel.
5. Confirmar WorldMode = MR.
6. Confirmar Puzzle MR Step 0.
7. Completar Step 0.
8. Fechar PIE.
9. Abrir novamente.
10. Confirmar Step 1.
11. Completar puzzle MR.
12. Confirmar transição para VRMemory.
13. Completar Step 0 do puzzle VR.
14. Fechar PIE.
15. Abrir novamente.
16. Confirmar Step 1 do puzzle VR.
17. Completar puzzle VR.
18. Confirmar retorno MR.
19. Fechar PIE.
20. Abrir novamente.
21. Confirmar estado final.
```

## Esperado

Tudo persiste.

---

# 25. Checklist final de aprovação

Marca cada ponto quando passar.

```text
[ ] GI_Echoes é o Game Instance ativo.
[ ] Todos os managers existem uma vez no mapa.
[ ] Todos os managers registam no GI.
[ ] TestHarness mostra todas as refs VALID.
[ ] SaveManager cria save novo quando não existe.
[ ] SaveManager carrega save existente.
[ ] RestoreFullGameState corre uma vez.
[ ] ApplySnapshotToManagers chama restores certos.
[ ] GhostManager restaura CurrentGhostId.
[ ] GhostManager restaura CurrentGhostState.
[ ] WorldModeSystem restaura MR/VRMemory.
[ ] ObjectiveManager restaura ObjectiveStates.
[ ] PuzzleController restaura CurrentPuzzleId.
[ ] PuzzleController restaura CurrentStepIndex.
[ ] PuzzleActor certo ativa no step certo.
[ ] ReportStepSuccess avança step.
[ ] Autosave grava após mudança de step.
[ ] Fechar e abrir mantém step atual.
[ ] CompleteCurrentPuzzle atualiza CompletedPuzzleIds.
[ ] Puzzle completion atualiza objectives.
[ ] Puzzle completion dispara transition, se configurado.
[ ] MR → VRMemory funciona.
[ ] VRMemory → MR funciona.
[ ] AudioManager aplica estado MR.
[ ] AudioManager aplica estado VRMemory.
[ ] AudioManager reage a GhostState.
[ ] ScareManager só corre com GhostId válido.
[ ] ScareManager respeita WorldMode.
[ ] ScareManager respeita cooldown.
[ ] Estado final persiste após fechar/reabrir.
```

---

# 26. Problemas comuns e como diagnosticar

## 26.1 ManagerRef inválido

### Sintoma

```text
SaveManagerRef = INVALID
GhostManagerRef = INVALID
```

### Causas prováveis

```text
Actor não existe no mapa.
BeginPlay não correu.
Register no GI não foi chamado.
GI_Echoes não é o Game Instance ativo.
```

---

## 26.2 Puzzle volta ao Step 0 depois de abrir

### Causas prováveis

```text
CurrentStepIndex não está no snapshot.
SyncPuzzleProgressToSave não chama RequestAutoSave.
Autosave está desligado.
RestorePuzzleFromSave não usa StartPuzzleByRowNameAndStep.
```

---

## 26.3 WorldMode errado depois de transição

### Causas prováveis

```text
TargetWorldMode errado no transition data.
Save restore sobrescreveu WorldMode.
WorldModeSystem não recebeu RestoreWorldModeFromSave.
```

---

## 26.4 Audio não muda

### Causas prováveis

```text
AudioManager não fez bind ao WorldModeSystem.
AudioManager não fez bind ao GhostManager.
ResolveAudioStateFromContext não encontra linha na DT_AudioStates.
ApplyAudioStateById não toca o som esperado.
```

---

## 26.5 Scares não acontecem

### Causas prováveis

```text
CurrentGhostId None.
WorldMode não corresponde.
Scare bloqueado por cooldown.
Scare fora da distância.
ScareDataTable sem linha para o ScareId.
SpawnActorClass vazio.
```

---

## 26.6 Transição fica em preto

### Causas prováveis

```text
FadeWidgetClass não definido.
WBP_Fade não chama evento de fim.
FinishTransition não é chamado.
HandlePendingArrivalTransition não corre no novo mapa.
```

---

# 27. Recomendação final de execução

A ordem prática que recomendo é:

```text
1. Fazer Fase 1 a 4 sem tocar em puzzles.
2. Só depois testar Ghost e WorldMode.
3. Só depois testar Objectives.
4. Só depois testar Puzzle MR.
5. Só depois testar Transition.
6. Só depois testar VRMemory.
7. Só no fim testar Scare e Audio em stress.
```

Se tentares testar logo MR → VR → MR, qualquer falha vai ser difícil de localizar.

---

# 28. Resultado final esperado

Quando tudo estiver saudável, o teu fluxo completo deve ser:

```text
Abrir jogo
→ GI_Echoes inicializa sessão
→ Managers registam no GI
→ SaveManager carrega/cria save
→ SaveManager restaura estado completo
→ GhostManager define Isabel
→ WorldModeSystem define MR
→ ObjectiveManager ativa objetivo inicial
→ PuzzleController ativa puzzle MR step 0
→ AudioManager aplica áudio MR
→ ScareManager começa loop MR

Jogador completa puzzle MR
→ PuzzleController completa puzzle
→ ObjectiveManager atualiza objetivo
→ SaveManager grava
→ TransitionManager faz fade out
→ Abre mapa VRMemory
→ Managers registam novamente
→ SaveManager restaura
→ WorldModeSystem = VRMemory
→ PuzzleController ativa puzzle VR
→ AudioManager aplica áudio VR
→ ScareManager usa scares VR

Jogador completa memória
→ GhostState muda
→ Objective final atualiza
→ SaveManager grava
→ TransitionManager volta ao MR
→ Estado final persiste
```

Se este fluxo acontecer sem resets indevidos, sem managers inválidos e sem perda de save, a arquitetura nova está aprovada.
