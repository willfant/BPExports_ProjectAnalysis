# System Risks

## High Coupling / Hubs
- BFL_Debug has high coupling (incoming=333, outgoing=1)
- BP_AudioManager has high coupling (incoming=2, outgoing=47)
- BP_GhostManager has high coupling (incoming=12, outgoing=24)
- BP_ObjectiveManager has high coupling (incoming=4, outgoing=73)
- BP_PuzzleController has high coupling (incoming=5, outgoing=81)
- BP_SaveManager has high coupling (incoming=20, outgoing=44)
- BP_ScareBase has high coupling (incoming=3, outgoing=4)
- BP_ScareManager has high coupling (incoming=5, outgoing=91)
- BP_Scare_Whisper has high coupling (incoming=0, outgoing=6)
- BP_TransitionManager has high coupling (incoming=4, outgoing=9)
- BP_WorldModeSystem has high coupling (incoming=11, outgoing=5)
- GI_Echoes has high coupling (incoming=15, outgoing=20)

## Circular Dependency Detection
- Potential cycle detected:
  - BFL_Debug
  - GI_Echoes
  - BP_SaveManager
  - BP_GhostManager
  - BP_SaveManager

