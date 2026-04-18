# System Risks

## High Coupling / Hubs
- BFL_Debug has high coupling (incoming=76, outgoing=1)
- BP_AudioManager has high coupling (incoming=0, outgoing=39)
- BP_GhostManager has high coupling (incoming=11, outgoing=21)
- BP_ObjectiveManager has high coupling (incoming=16, outgoing=7)
- BP_PuzzleController has high coupling (incoming=0, outgoing=36)
- BP_SaveManager has high coupling (incoming=28, outgoing=9)
- BP_ScareBase has high coupling (incoming=3, outgoing=4)
- BP_ScareManager has high coupling (incoming=6, outgoing=9)
- BP_Scare_Whisper has high coupling (incoming=0, outgoing=6)
- GI_Echoes has high coupling (incoming=7, outgoing=12)
- GrabComponent has high coupling (incoming=8, outgoing=0)
- Radial_Storm has high coupling (incoming=0, outgoing=9)
- Random_Weather_Variation has high coupling (incoming=15, outgoing=6)
- UDS_Client_Controller has high coupling (incoming=0, outgoing=7)
- UDS_Cloud_Paint_Container has high coupling (incoming=0, outgoing=5)
- UDS_Modifier has high coupling (incoming=5, outgoing=0)
- UDS_OcclusionState has high coupling (incoming=5, outgoing=0)
- UDS_PlayerOcclusion has high coupling (incoming=4, outgoing=3)
- UDS_RenderTarget_State has high coupling (incoming=25, outgoing=0)
- UDS_Weather_Settings has high coupling (incoming=7, outgoing=0)
- UDW_Material_State_Manager has high coupling (incoming=6, outgoing=0)
- UDW_Temperature_Manager has high coupling (incoming=9, outgoing=2)
- UltraDynamicSky_Functions has high coupling (incoming=6, outgoing=12)
- UltraDynamicWeather_Functions has high coupling (incoming=2, outgoing=10)
- Ultra_Dynamic_Sky has high coupling (incoming=45, outgoing=9)
- Ultra_Dynamic_Weather has high coupling (incoming=59, outgoing=88)
- VRPawn has high coupling (incoming=0, outgoing=5)
- WeatherMask has high coupling (incoming=6, outgoing=6)
- Weather_Override_Volume has high coupling (incoming=9, outgoing=40)

## Circular Dependency Detection
- Potential cycle detected:
  - BFL_Debug
  - GI_Echoes
  - BP_SaveManager
  - BP_GhostManager
  - BP_SaveManager

