# System Risks

## High Coupling / Hubs
- BFL_Debug has high coupling (incoming=367, outgoing=1)
- BP_AudioManager has high coupling (incoming=2, outgoing=51)
- BP_GhostManager has high coupling (incoming=14, outgoing=25)
- BP_ObjectiveManager has high coupling (incoming=7, outgoing=71)
- BP_PuzzleController has high coupling (incoming=5, outgoing=87)
- BP_SaveManager has high coupling (incoming=17, outgoing=76)
- BP_ScareBase has high coupling (incoming=3, outgoing=4)
- BP_ScareManager has high coupling (incoming=5, outgoing=97)
- BP_Scare_Whisper has high coupling (incoming=0, outgoing=6)
- BP_TransitionManager has high coupling (incoming=3, outgoing=8)
- BP_WorldModeSystem has high coupling (incoming=11, outgoing=4)
- GI_Echoes has high coupling (incoming=22, outgoing=20)

## Circular Dependency Detection
- Potential cycle detected:
  - BFL_Debug
  - GI_Echoes
  - BP_SaveManager
  - BP_GhostManager
  - BP_SaveManager

