# System Risks

## High Coupling / Hubs
- BFL_Debug has high coupling (incoming=99, outgoing=1)
- BP_AudioManager has high coupling (incoming=0, outgoing=41)
- BP_GhostManager has high coupling (incoming=11, outgoing=21)
- BP_ObjectiveManager has high coupling (incoming=16, outgoing=7)
- BP_PuzzleController has high coupling (incoming=0, outgoing=42)
- BP_SaveManager has high coupling (incoming=43, outgoing=10)
- BP_ScareBase has high coupling (incoming=3, outgoing=4)
- BP_ScareManager has high coupling (incoming=6, outgoing=11)
- BP_Scare_Whisper has high coupling (incoming=0, outgoing=6)
- BP_TransitionManager has high coupling (incoming=8, outgoing=31)
- BP_WorldModeSystem has high coupling (incoming=10, outgoing=10)
- GI_Echoes has high coupling (incoming=6, outgoing=18)

## Circular Dependency Detection
- Potential cycle detected:
  - BP_AudioManager
  - BP_WorldModeSystem
  - BP_SaveManager
  - BP_GhostManager
  - BP_SaveManager

