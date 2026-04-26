# BP_AudioManager — Documentação Técnica e de Uso

Projeto: **Echoes Between Worlds**  
Blueprint: **BP_AudioManager**  
Tipo: **Actor / Manager global de áudio**  
Local esperado: colocado nos níveis principais onde o jogo precisa de áudio global, por exemplo MR Hub e VR Memory levels.

---

## 1. Resumo simples

O `BP_AudioManager` é o manager responsável por controlar o áudio global do jogo.

Ele não decide que puzzle está ativo, nem decide que susto vai acontecer. A função dele é outra:

- tocar ambiências;
- tocar stingers;
- tocar sons 2D;
- tocar sons no mundo;
- tocar sons attached a componentes/actors;
- mudar o estado de áudio com base no contexto atual;
- reagir a alterações de `WorldMode`;
- reagir a alterações de `GhostId`;
- reagir a alterações de `GhostState`;
- manter loops de áudio ativos por ID;
- expor funções utilitárias para outros sistemas tocarem áudio de forma centralizada.

Em termos simples:

> O `BP_AudioManager` é a “mesa de mistura” do jogo.  
> Os outros managers dizem “o mundo mudou”, “o ghost mudou”, “o susto precisa de som”, e o AudioManager decide que ambiente/stinger/mix deve aplicar.

---

## 2. Porque é que ele existe

Sem o `BP_AudioManager`, cada sistema ia tocar sons à sua maneira:

- o `ScareManager` tocava sustos diretamente;
- o `PuzzleController` tocava sons diretamente;
- o `TransitionManager` tocava fades/stingers diretamente;
- o `GhostManager` podia tentar mudar música diretamente;
- cada puzzle actor podia fazer `Play Sound 2D` ou `Play Sound at Location` sem regra comum.

Isso rapidamente criava caos:

- volumes inconsistentes;
- sons duplicados;
- ambiências que não param;
- stingers a tocar por cima uns dos outros;
- dificuldade em adaptar o áudio ao ghost/world/state;
- difícil controlar mix global para Quest 3.

A ideia do `BP_AudioManager` é ser o sítio central para tudo o que é áudio global ou contextual.

---

## 3. O que o plano original diz sobre ele

No plano de execução do jogo, o `BP_AudioManager` é descrito como um dos Core Managers. A função dele é aplicar `DT_AudioStates` e controlar ambience, stingers, reverb e LPF. O plano também coloca o `BP_AudioManager` como manager crítico nas fases de implementação, com dependência da fase de áudio e de `DT_AudioStates`. Fonte: plano de execução do projeto.

No production document, o `BP_AudioManager` também aparece como sistema crítico: controla ambiências, stingers, estados de tensão, reverb, LPF e eventos globais de áudio.

---

## 4. Onde é usado

### 4.1 No próprio nível

O `BP_AudioManager` é um **Actor**. Normalmente é colocado no nível, como os outros managers.

Exemplos de níveis onde faz sentido existir:

- `MR_Hub`
- `VR_MemoryBedroom_Isabel`
- níveis de teste de áudio
- níveis de teste de scares
- níveis de teste de integração

Como actor no mundo, ele entra em `BeginPlay` e chama a sua função principal de arranque:

```text
EventGraph
    -> InitializeAudioManager
```

O export mostra que o `EventGraph` chama `Initialize Audio Manager`.

---

### 4.2 No `GI_Echoes`

O `BP_AudioManager` regista-se no `GI_Echoes`.

Fluxo conceptual:

```text
BP_AudioManager.BeginPlay
    -> InitializeAudioManager
        -> Get Game Instance
        -> Cast To GI_Echoes
        -> GI_Echoes.RegisterAudioManager(self)
```

Depois disso, qualquer sistema pode pedir ao `GI_Echoes`:

```text
GI_Echoes.GetAudioManagerRef()
```

Isto é importante porque evita que outros Blueprints façam `Get Actor Of Class` sempre que querem tocar áudio.

---

### 4.3 No `SaveManager` / bootstrap

Na arquitetura nova, o `BP_SaveManager` também conhece o `AudioManagerRef` através do GI.

O papel esperado do `SaveManager` não é controlar áudio diretamente. O papel dele é:

1. esperar que os managers estejam registados;
2. fazer restore do estado do jogo;
3. depois pedir ao AudioManager para refrescar o contexto final.

Fluxo esperado:

```text
SaveManager.TryProjectBootstrap
    -> confirmar que AudioManager está registado
    -> RestoreFullGameState
    -> AudioManager.RefreshAudioContextFromManagers
```

Isto garante que o áudio final não é aplicado antes de o mundo/ghost/state estarem restaurados.

---

### 4.4 Pelo `WorldModeSystem`

O `AudioManager` deve reagir a mudanças de world mode.

Exemplo:

```text
WorldModeSystem.SetWorldMode(MR)
    -> Dispatcher OnWorldModeChanged
        -> AudioManager.HandleWorldModeAudioContextChanged
            -> RefreshAudioContextFromManagers
            -> ResolveAudioStateFromContext
            -> ApplyAudioStateById
```

Ou seja:

- se o jogador está em `MR`, o áudio deve ser de hub / mixed reality;
- se está em `VRMemory`, o áudio deve ser da memória;
- se mudar de um para outro, o audio state deve mudar também.

---

### 4.5 Pelo `GhostManager`

O `AudioManager` deve reagir a:

- mudança de ghost;
- mudança de ghost state;
- ghost limpo/cleared.

Fluxo esperado:

```text
GhostManager.SetActiveGhost(Isabel)
    -> Dispatcher OnGhostChanged
        -> AudioManager.HandleGhostAudioContextChanged
            -> RefreshAudioContextFromManagers
            -> ResolveAudioStateFromContext
            -> ApplyAudioStateById
```

Para mudança de estado:

```text
GhostManager.SetGhostState(Agitated)
    -> Dispatcher OnGhostStateChanged
        -> AudioManager.HandleGhostStateAudioContextChanged
            -> RefreshAudioContextFromManagers
            -> ResolveAudioStateFromContext
            -> ApplyAudioStateById
```

Para ghost limpo:

```text
GhostManager.ClearActiveGhost()
    -> Dispatcher OnGhostCleared / equivalente
        -> AudioManager.HandleGhostClearedAudioContextChanged
            -> RefreshAudioContextFromManagers
            -> ResolveAudioStateFromContext
            -> ApplyAudioStateById
```

A ideia é simples:

> O `GhostManager` muda a narrativa.  
> O `AudioManager` muda a atmosfera sonora.

---

### 4.6 Pelo `ScareManager`

O `ScareManager` deve usar o AudioManager quando precisa de tocar áudio de sustos.

Exemplos:

```text
ScareManager escolhe scare Whisper
    -> AudioManager.PlayAudioCueAtLocation(...)
```

```text
ScareManager escolhe DoorBang
    -> AudioManager.PlayAudioCue2D(...)
```

```text
Scare actor spawna atrás do jogador
    -> AudioManager.PlayAudioCueAttached(...)
```

Dependendo do tipo de susto, o som pode ser:

- 2D: sem posição no mundo;
- at location: numa posição 3D;
- attached: preso a um actor/componente;
- looping: persistente até ser parado;
- stinger: impacto curto.

O `ScareManager` decide **quando e que scare acontece**.  
O `AudioManager` decide **como tocar o som**.

---

### 4.7 Pelo `TransitionManager`

O `TransitionManager` pode usar o AudioManager para sons de transição:

- fade out sonoro;
- stinger de portal;
- som de memória;
- whoosh;
- ritual transition;
- fade in/fade out de música.

Exemplo conceptual:

```text
TransitionManager.StartMemoryTransition
    -> AudioManager.PlayStingerCue(TransitionSFX)
    -> AudioManager.FadeOutMusicSound(...)
    -> abrir level
    -> AudioManager.RefreshAudioContextFromManagers
```

Nem todo este fluxo tem de estar implementado já, mas é o uso correto.

---

### 4.8 Pelo `PuzzleController` ou puzzle actors

Puzzles podem usar o AudioManager para feedback sonoro:

- sucesso;
- falha;
- candle lit;
- rune activated;
- page pickup;
- mirror pulse;
- ritual loop.

Exemplo:

```text
PuzzleActorBase.ReportSuccessToController
    -> PuzzleController.ReportStepSuccess
        -> AudioManager.PlayStingerCue(...)
```

Ou, em puzzles específicos:

```text
BP_Puzzle_Candle
    -> AudioManager.PlayAudioCueAtLocation(CandleLitCue, CandleLocation)
```

O importante é: quando for som global ou contextual, é melhor passar pelo `AudioManager` do que espalhar `PlaySound...` em todo o projeto.

---

## 5. O que chama o AudioManager atualmente

Pelo export atual do Blueprint:

### Chamadas internas confirmadas

O `EventGraph` chama:

```text
InitializeAudioManager
ApplyResolvedAudioStateFromContext
Clear and Invalidate Timer by Handle
```

O `InitializeAudioManager` chama:

```text
RegisterAudioManager
InitializeWorldModeAudioBinding
InitializeGhostAudioBinding
```

O `InitializeWorldModeAudioBinding` chama:

```text
HandleWorldModeAudioContextChanged
```

O `InitializeGhostAudioBinding` chama:

```text
HandleGhostAudioContextChanged
```

Os handlers chamam:

```text
RefreshAudioContextFromManagers
```

E o refresh chama:

```text
ApplyResolvedAudioStateFromContext
```

Depois:

```text
ApplyResolvedAudioStateFromContext
    -> ResolveAudioStateFromContext
    -> ApplyAudioStateById
```

Finalmente:

```text
ApplyAudioStateById
    -> PlayStinger2D
    -> PlayAmbienceSound
    -> StopAmbienceSound
```

---

## 6. Fluxo principal completo

Este é o fluxo mental mais importante.

```text
1. BP_AudioManager entra em BeginPlay
2. Chama InitializeAudioManager
3. Regista-se no GI_Echoes
4. Faz binding ao WorldModeSystem
5. Faz binding ao GhostManager
6. Quando WorldMode/Ghost/GhostState muda:
       -> handler correspondente
       -> RefreshAudioContextFromManagers
7. Refresh lê:
       -> WorldMode atual
       -> GhostId atual
       -> GhostState atual
8. ResolveAudioStateFromContext decide que AudioState usar
9. ApplyAudioStateById lê DT_AudioStates
10. Aplica:
       -> ambience
       -> stinger
       -> volumes
       -> estado atual
```

Em português simples:

> O AudioManager fica à escuta dos sistemas principais.  
> Quando o contexto muda, ele pergunta: “em que mundo estou, que fantasma está ativo e em que estado ele está?”  
> Depois procura na `DT_AudioStates` o estado sonoro certo e aplica-o.

---

## 7. Variáveis principais

### 7.1 Registry / referências

| Variável | Função |
|---|---|
| `CachedGameInstance` | Guarda a referência ao `GI_Echoes`. |
| `WorldModeSystemRef` | Referência ao sistema que sabe se estamos em MR ou VRMemory. |
| `GhostManagerRef` | Referência ao manager que sabe qual ghost está ativo e o estado dele. |

Estas refs são usadas para o AudioManager saber o contexto atual do jogo.

---

### 7.2 Estado contextual

| Variável | Função |
|---|---|
| `CurrentWorldMode` | Guarda o world mode atual visto pelo AudioManager. |
| `CurrentGhostId` | Guarda o ghost atual visto pelo AudioManager. |
| `CurrentGhostState` | Guarda o estado atual do ghost visto pelo AudioManager. |

Estas variáveis são cópias locais do contexto. O AudioManager lê os managers principais e atualiza estas variáveis.

---

### 7.3 Estado de áudio

| Variável | Função |
|---|---|
| `CurrentAudioStateId` | Nome do estado de áudio atualmente aplicado. |
| `CurrentAmbienceSound` | Ambience atual. |
| `CurrentAmbienceVolume` | Volume da ambience atual. |
| `CurrentStingerSound` | Stinger atual. |
| `CurrentMusicSound` | Música atual. |
| `CurrentMusicVolume` | Volume da música atual. |
| `bMusicPlaying` | Indica se há música em playback. |

---

### 7.4 Configuração

| Variável | Função |
|---|---|
| `DT_AudioStates` | DataTable que define os estados de áudio por contexto. |
| `DebugEnabled` | Controla debug local do AudioManager. |

---

### 7.5 Looping cues

| Variável | Função |
|---|---|
| `ActiveLoopingCues` | Mapa com loops ativos por ID. Permite tocar/parar loops sem duplicar. |

Exemplo:

```text
PlayLoopingCue("RitualLoop")
StopLoopingCueById("RitualLoop")
StopAllLoopingCues()
```

---

## 8. Funções principais

## 8.1 InitializeAudioManager

### Responsabilidade

Inicializa o manager.

Faz:

- verifica se já foi inicializado;
- obtém `GameInstance`;
- faz cast para `GI_Echoes`;
- guarda `CachedGameInstance`;
- regista-se no GI;
- inicializa bindings de world mode;
- inicializa bindings de ghost;
- marca `bInitialized`.

### Quem chama

```text
EventGraph / BeginPlay
```

### Atenção

Na arquitetura nova, é importante garantir que esta função não aplique o estado final de áudio cedo demais, antes do restore do SaveManager.

Idealmente:

- `InitializeAudioManager` regista e prepara;
- o `SaveManager` chama `RefreshAudioContextFromManagers` depois do restore.

---

## 8.2 InitializeWorldModeAudioBinding

### Responsabilidade

Liga o AudioManager ao dispatcher de world mode.

Fluxo esperado:

```text
AudioManager.InitializeWorldModeAudioBinding
    -> valida WorldModeSystemRef
    -> se inválida, tenta resolver via GI_Echoes
    -> se ainda inválida, agenda retry
    -> se válida, bind ao OnWorldModeChanged
    -> marca bWorldModeBindingInitialized = true
```

Quando o world mode muda:

```text
HandleWorldModeAudioContextChanged
    -> RefreshAudioContextFromManagers
```

### Uso

Serve para o áudio reagir automaticamente a transições MR/VR.

---

## 8.3 InitializeGhostAudioBinding

### Responsabilidade

Liga o AudioManager ao `GhostManager`.

Fluxo esperado:

```text
AudioManager.InitializeGhostAudioBinding
    -> valida GhostManagerRef
    -> bind ao OnGhostChanged
    -> bind ao OnGhostStateChanged
    -> bind ao OnGhostCleared, se existir
```

Quando o ghost muda:

```text
HandleGhostAudioContextChanged
    -> RefreshAudioContextFromManagers
```

Quando o state muda:

```text
HandleGhostStateAudioContextChanged
    -> RefreshAudioContextFromManagers
```

Quando o ghost é limpo:

```text
HandleGhostClearedAudioContextChanged
    -> RefreshAudioContextFromManagers
```

### Nota importante

No export atual, esta função lê `GhostManagerRef` e, se estiver inválida, faz warning. Isto pode ser frágil se o GhostManager ainda não se tiver registado quando o AudioManager tenta fazer bind.

Recomendação:

- reforçar como fizemos noutros managers;
- tentar resolver via `GI_Echoes`;
- se continuar inválido, fazer retry;
- não desistir para sempre.

---

## 8.4 RefreshAudioContextFromManagers

### Responsabilidade

Esta é uma das funções mais importantes.

Ela atualiza o contexto local do AudioManager a partir dos managers reais.

Lê:

- `WorldModeSystemRef`;
- `GhostManagerRef`;
- `CurrentGhostId`;
- `CurrentGhostState`.

Escreve:

- `CurrentWorldMode`;
- `CurrentGhostId`;
- `CurrentGhostState`;
- `GhostManagerRef`.

Depois chama:

```text
ApplyResolvedAudioStateFromContext
```

### Em linguagem simples

Esta função pergunta:

> “Qual é o mundo atual? Qual é o ghost atual? Em que estado está?”

Depois atualiza o áudio com base nessa resposta.

---

## 8.5 ResolveAudioStateFromContext

### Responsabilidade

Decide qual row da `DT_AudioStates` deve ser usada.

Ela lê:

- `DT_AudioStates`;
- `CurrentWorldMode`;
- `CurrentGhostId`;
- `CurrentGhostState`.

E devolve:

- `Found`;
- `ResolvedStateId`.

### Como deve funcionar conceptualmente

A função deve tentar encontrar o audio state mais específico possível.

Ordem recomendada:

```text
1. WorldMode + GhostId + GhostState
2. WorldMode + GhostId
3. WorldMode + GhostState
4. WorldMode apenas
5. fallback geral
```

Exemplo:

```text
CurrentWorldMode = VRMemory
CurrentGhostId = Isabel
CurrentGhostState = Agitated

Tenta resolver:
VRMemory_Isabel_Agitated
se não existir:
VRMemory_Isabel
se não existir:
VRMemory_Agitated
se não existir:
VRMemory_Default
```

---

## 8.6 ApplyResolvedAudioStateFromContext

### Responsabilidade

É a ponte entre “decidir o estado certo” e “aplicar o estado”.

Fluxo:

```text
ApplyResolvedAudioStateFromContext
    -> ResolveAudioStateFromContext
    -> se Found = true
        -> ApplyAudioStateById(ResolvedStateId)
    -> se Found = false
        -> debug / não muda áudio
```

---

## 8.7 ApplyAudioStateById

### Responsabilidade

Aplica uma row concreta da `DT_AudioStates`.

Ela lê:

- `DT_AudioStates`.

E escreve:

- `CurrentAudioStateId`;
- `CurrentStingerSound`;
- `CurrentAmbienceSound`;
- `CurrentAmbienceVolume`.

Também chama:

- `PlayStinger2D`;
- `PlayAmbienceSound`;
- `StopAmbienceSound`.

### Em linguagem simples

Esta função é onde o áudio muda mesmo.

Exemplo:

```text
ApplyAudioStateById("MR_Isabel_Dormant")
    -> lê row da DT_AudioStates
    -> toca/atualiza ambience
    -> toca stinger se houver
    -> guarda CurrentAudioStateId
```

---

## 9. Funções de playback

## 9.1 PlayAudioCue2D

Usa `Play Sound 2D`.

Serve para sons sem posição 3D:

- stingers globais;
- susto súbito global;
- UI;
- transições;
- impacto narrativo.

Exemplo:

```text
AudioManager.PlayAudioCue2D(S_AudioCue)
```

---

## 9.2 PlayAudioCueAtLocation

Usa `Play Sound at Location`.

Serve para sons posicionais no mundo:

- whisper atrás do jogador;
- door bang numa porta;
- object drop numa mesa;
- footsteps num canto.

Exemplo:

```text
AudioManager.PlayAudioCueAtLocation(S_AudioCue, Location)
```

---

## 9.3 PlayAudioCueAttached

Usa `Spawn Sound Attached`.

Serve para sons presos a um componente/actor:

- ghost ligado a um mesh;
- objeto que continua a emitir som;
- portal;
- vela;
- puzzle object.

Exemplo:

```text
AudioManager.PlayAudioCueAttached(S_AudioCue, AttachComponent)
```

---

## 9.4 PlayStingerCue / PlayStinger2D

Serve para sons curtos de impacto.

Exemplos:

- scare hit;
- puzzle success;
- puzzle fail;
- transição;
- ritual completed;
- blackout.

---

## 9.5 PlayLoopingCue

Serve para loops que têm ID.

Exemplo:

```text
PlayLoopingCue(
    LoopId = "RitualLoop",
    AudioCue = RitualLoopCue,
    Location = RitualTableLocation,
    ReplaceIfExists = true
)
```

Se o loop já existir e `ReplaceIfExists = true`, para o antigo e toca o novo.

---

## 9.6 StopLoopingCueById

Para um loop específico.

Exemplo:

```text
StopLoopingCueById("RitualLoop")
```

---

## 9.7 StopAllLoopingCues

Para todos os loops ativos.

Útil em:

- mudança de level;
- fim de ritual;
- retorno ao hub;
- cleanup;
- EndPlay;
- ghost banished.

---

## 9.8 Música

Funções existentes:

- `PlayMusicSound`
- `FadeInMusicSound`
- `FadeOutMusicSound`
- `StopMusicSound`
- `SetMusicVolume`

Estas usam o componente `AC_Music`.

Uso conceptual:

```text
FadeOutMusicSound(2.0)
TransitionManager muda de level
FadeInMusicSound(NewMusic, 2.0)
```

---

## 10. Como a DT_AudioStates deve ser pensada

A `DT_AudioStates` deve ser o cérebro data-driven do áudio contextual.

Cada row deve representar um estado sonoro.

Exemplos de rows:

```text
MR_Default
MR_Isabel_Dormant
MR_Isabel_Agitated
MR_Isabel_Manifesting

VRMemory_Default
VRMemory_Isabel_Dormant
VRMemory_Isabel_Agitated
VRMemory_Isabel_Manifesting
VRMemory_Isabel_Banished
```

Cada row pode definir:

- ambience;
- ambience volume;
- stinger;
- music;
- music volume;
- reverb send;
- LPF frequency;
- notas/dev tags.

Mesmo que alguns campos ainda não estejam totalmente implementados no Blueprint, a intenção da arquitetura é esta.

---

## 11. Contextos de uso reais no jogo

## 11.1 Ao iniciar o MR Hub

Estado esperado:

```text
WorldMode = MR
GhostId = Isabel
GhostState = Dormant
```

AudioManager resolve algo tipo:

```text
MR_Isabel_Dormant
```

E toca:

- ambience baixa;
- ruído subtil;
- talvez sem música;
- sem stinger agressivo.

---

## 11.2 Ao passar para VR Memory

Estado esperado:

```text
WorldMode = VRMemory
GhostId = Isabel
GhostState = Dormant ou Agitated
```

AudioManager resolve:

```text
VRMemory_Isabel_Dormant
```

Ou:

```text
VRMemory_Isabel_Agitated
```

E aplica:

- ambience da memória;
- talvez filtro/reverb diferente;
- stinger de entrada, se definido.

---

## 11.3 Quando o ghost fica Agitated

Estado muda:

```text
GhostState = Agitated
```

AudioManager resolve:

```text
MR_Isabel_Agitated
```

ou

```text
VRMemory_Isabel_Agitated
```

E aplica uma ambiência mais tensa.

---

## 11.4 Quando o ghost manifesta

Estado muda:

```text
GhostState = Manifesting
```

AudioManager resolve:

```text
VRMemory_Isabel_Manifesting
```

Isto pode ativar:

- ambience mais intensa;
- stinger;
- reverb mais agressivo;
- LPF;
- drones;
- sub-rumble.

---

## 11.5 Quando o ghost é banido

Estado muda:

```text
GhostState = Banished
```

ou ghost é limpo.

AudioManager resolve:

```text
MR_Isabel_Banished
```

ou fallback:

```text
MR_Default_Cleansed
```

Resultado:

- parar loops assustadores;
- baixar tensão;
- tocar stinger de resolução;
- trocar ambience do hub.

---

## 12. Relação com outros managers

## 12.1 GI_Echoes

O GI guarda a referência do AudioManager.

```text
AudioManager -> RegisterAudioManager(self)
GI_Echoes -> guarda AudioManagerRef
Outros sistemas -> GetAudioManagerRef
```

O GI não deve controlar áudio.  
Ele só guarda a referência.

---

## 12.2 SaveManager

O SaveManager não deve tocar sons.

Papel dele:

```text
Depois do restore:
    AudioManager.RefreshAudioContextFromManagers
```

Isto sincroniza o áudio com o estado restaurado.

---

## 12.3 WorldModeSystem

É fonte de verdade para MR/VRMemory.

O AudioManager lê:

```text
WorldModeSystem.GetWorldMode
```

---

## 12.4 GhostManager

É fonte de verdade para:

- ghost ativo;
- estado do ghost;
- ghost limpo.

O AudioManager lê:

```text
GhostManager.HasActiveGhost
GhostManager.GetCurrentGhostId
GhostManager.GetCurrentGhostState
```

---

## 12.5 ScareManager

O ScareManager deve pedir playback ao AudioManager para scares.

O ScareManager não deve ser dono de ambience global.

---

## 12.6 PuzzleController

O PuzzleController pode pedir stingers ou feedback sonoro.

Mas o puzzle state continua a ser dele, não do AudioManager.

---

## 12.7 TransitionManager

Pode usar o AudioManager para:

- portal SFX;
- fade musical;
- stingers de transição.

---

## 13. O que o AudioManager NÃO deve fazer

O `BP_AudioManager` não deve:

- decidir qual puzzle está ativo;
- decidir qual scare vai acontecer;
- decidir o ghost atual;
- decidir o world mode atual;
- gravar save;
- abrir levels;
- spawnar scares;
- ativar objetivos;
- controlar diretamente a progressão.

Ele só responde ao contexto.

---

## 14. Problema atual provável / coisa a reforçar

Há uma fragilidade importante:

`InitializeGhostAudioBinding` parece depender de `GhostManagerRef` já estar válido. Se não estiver, faz warning e sai.

Isso pode causar isto:

```text
AudioManager inicia primeiro
GhostManager ainda não registou no GI
AudioManager tenta InitializeGhostAudioBinding
GhostManagerRef = None
Binding falha
Depois GhostManager muda, mas AudioManager não ouve
```

### Reforço recomendado

Criar um sistema semelhante ao que fizemos noutros managers:

- `bGhostAudioBindingInitialized`
- `GhostAudioBindingRetryHandle`
- retry de 0.2 segundos
- resolver `GhostManagerRef` via `GI_Echoes`
- bind só quando válido
- não desistir permanentemente se estiver inválido no primeiro frame

---

## 15. Sequência ideal de bootstrap

Fluxo recomendado com a nova arquitetura:

```text
1. AudioManager BeginPlay
2. InitializeAudioManager
3. RegisterAudioManager no GI_Echoes
4. Não aplicar estado final ainda
5. SaveManager espera todos os managers
6. SaveManager faz RestoreFullGameState
7. SaveManager chama:
       AudioManager.InitializeWorldModeAudioBinding
       AudioManager.InitializeGhostAudioBinding
       AudioManager.RefreshAudioContextFromManagers
8. AudioManager aplica estado correto
```

Se mantiveres os bindings no `InitializeAudioManager`, eles têm de ser robustos com retry.

A versão mais limpa é:

```text
InitializeAudioManager:
    - regista
    - prepara refs
    - não força refresh final cedo demais

SaveManager bootstrap:
    - chama bindings
    - chama RefreshAudioContextFromManagers depois do restore
```

---

## 16. Como testar

## Teste 1 — registo

1. Play.
2. Ver log:

```text
[AudioManager] Registered AudioManagerRef
```

3. Numa tecla de debug, chamar:

```text
GI_Echoes.GetAudioManagerRef
```

Resultado esperado:

```text
AudioManagerRef valid
```

---

## Teste 2 — world mode

1. Começar em MR.
2. Carregar tecla que muda para VRMemory.
3. Ver se aparece log do AudioManager:

```text
HandleWorldModeAudioContextChanged
RefreshAudioContextFromManagers
ApplyResolvedAudioStateFromContext
ApplyAudioStateById
```

Resultado esperado:

- `CurrentWorldMode` muda;
- `CurrentAudioStateId` muda;
- ambience muda.

---

## Teste 3 — ghost change

1. Carregar tecla que faz `SetActiveGhost(Isabel)`.
2. Ver se AudioManager reage:

```text
HandleGhostAudioContextChanged
RefreshAudioContextFromManagers
```

Resultado esperado:

- `CurrentGhostId = Isabel`;
- audio state recalculado.

---

## Teste 4 — ghost state change

1. Carregar tecla que muda ghost state para `Agitated`.
2. Ver se AudioManager reage:

```text
HandleGhostStateAudioContextChanged
RefreshAudioContextFromManagers
```

Resultado esperado:

- `CurrentGhostState = Agitated`;
- audio state muda para estado mais tenso.

---

## Teste 5 — restore

1. Guardar jogo com:
   - `WorldMode = VRMemory`
   - `GhostId = Isabel`
   - `GhostState = Agitated`
2. Fechar.
3. Abrir.
4. Depois do restore, ver:

```text
AudioManager.RefreshAudioContextFromManagers
CurrentWorldMode = VRMemory
CurrentGhostId = Isabel
CurrentGhostState = Agitated
CurrentAudioStateId = VRMemory_Isabel_Agitated
```

---

## Teste 6 — loops

1. Chamar:

```text
PlayLoopingCue("TestLoop")
```

2. Chamar outra vez com `ReplaceIfExists = false`.

Resultado esperado:

- não duplica.

3. Chamar:

```text
StopLoopingCueById("TestLoop")
```

Resultado esperado:

- loop para e sai de `ActiveLoopingCues`.

---

## 17. Checklist de saúde do manager

- [ ] `BP_AudioManager` está no nível certo.
- [ ] `DT_AudioStates` está atribuída.
- [ ] `AC_Ambience`, `AC_Stinger`, `AC_Music` existem no BP.
- [ ] `RegisterAudioManager` acontece no BeginPlay.
- [ ] `GI_Echoes.GetAudioManagerRef` devolve válido.
- [ ] `WorldModeSystemRef` resolve via GI.
- [ ] `GhostManagerRef` resolve via GI.
- [ ] `InitializeWorldModeAudioBinding` tem retry.
- [ ] `InitializeGhostAudioBinding` tem retry.
- [ ] `RefreshAudioContextFromManagers` é chamado depois do restore.
- [ ] `ResolveAudioStateFromContext` encontra uma row válida.
- [ ] `ApplyAudioStateById` toca ambience/stinger corretamente.
- [ ] Loops são parados no EndPlay ou transição.
- [ ] Não há warnings de refs inválidas ao fechar o jogo.

---

## 18. Regras práticas para usares daqui para a frente

### Regra 1

Se é ambience global, passa pelo AudioManager.

### Regra 2

Se é stinger global, passa pelo AudioManager.

### Regra 3

Se é som posicional de scare, idealmente passa pelo AudioManager.

### Regra 4

Se é som muito específico de um actor simples, pode ser local, mas tenta usar `S_AudioCue` para manter consistência.

### Regra 5

O AudioManager não decide narrativa; ele reflete a narrativa.

### Regra 6

Depois de qualquer restore, transition forte ou ghost change, chama:

```text
RefreshAudioContextFromManagers
```

---

## 19. Resposta direta à tua dúvida

### “O AudioManager é usado onde?”

É usado:

- no nível, como actor global;
- pelo GI, através de `RegisterAudioManager` e `GetAudioManagerRef`;
- pelo SaveManager, para refresh após restore;
- pelo WorldModeSystem, através de dispatcher de mudança de world mode;
- pelo GhostManager, através de dispatchers de ghost/state;
- pelo ScareManager, para tocar sons de sustos;
- pelo TransitionManager, para transições;
- por puzzles, para feedback sonoro.

### “É chamado por qual?”

Atualmente, pelo export:

- o próprio `EventGraph` chama `InitializeAudioManager`;
- `InitializeAudioManager` chama bindings;
- os dispatchers de world mode/ghost devem chamar handlers;
- handlers chamam `RefreshAudioContextFromManagers`;
- `RefreshAudioContextFromManagers` chama `ApplyResolvedAudioStateFromContext`;
- `ApplyResolvedAudioStateFromContext` chama `ApplyAudioStateById`.

### “Quando é que ele é usado?”

Em três momentos principais:

1. **Arranque**
   - regista-se;
   - prepara bindings.

2. **Mudança de contexto**
   - world mode muda;
   - ghost muda;
   - ghost state muda;
   - ghost cleared.

3. **Eventos específicos**
   - scare precisa de som;
   - puzzle precisa de feedback;
   - transição precisa de stinger/fade;
   - ritual precisa de loop.

---

## 20. Mini-diagrama final

```text
                         ┌──────────────────┐
                         │   GI_Echoes       │
                         │ AudioManagerRef   │
                         └─────────┬────────┘
                                   │
                                   │ GetAudioManagerRef
                                   │
┌──────────────────┐       ┌───────▼──────────┐
│ WorldModeSystem  │──────▶│  BP_AudioManager │
│ OnWorldModeChanged│      │                  │
└──────────────────┘       │ Refresh Context  │
                           │ Resolve Audio    │
┌──────────────────┐       │ Apply AudioState │
│ GhostManager     │──────▶│ Play Ambience    │
│ OnGhostChanged   │       │ Play Stingers    │
│ OnGhostState...  │       │ Play Cues        │
└──────────────────┘       └───────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
      ┌───────▼────────┐   ┌───────▼────────┐   ┌───────▼────────┐
      │ ScareManager   │   │ PuzzleController│   │ TransitionMgr  │
      │ scare sounds   │   │ puzzle feedback │   │ portal/fades   │
      └────────────────┘   └────────────────┘   └────────────────┘
```

---

## 21. Conclusão

O `BP_AudioManager` deve ser tratado como um manager reativo.

Ele não conduz o jogo.  
Ele ouve o contexto do jogo e transforma esse contexto em som.

O contexto vem principalmente de:

- `WorldModeSystem`;
- `GhostManager`;
- `SaveManager` após restore;
- eventos de scares/puzzles/transições.

A cadeia mais importante é:

```text
Contexto mudou
    -> RefreshAudioContextFromManagers
        -> ResolveAudioStateFromContext
            -> ApplyAudioStateById
                -> ambience/stinger/music/loops
```

Se perceberes esta cadeia, percebes praticamente todo o AudioManager.
