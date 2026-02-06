---

### (v3.1.1) GLaDOS Cortex ###

    The Central Nervous System

The Cortex is a specialized script within Home Assistant that acts as the primary Logic Router for the Intention Engine. While the LLM determines what needs to happen (the Intent), the Cortex determines how to execute it on your specific hardware (the Action).

---

📐 Logic Flow
Code snippet

graph LR
    A[User Prompt] --> B[LLM Function Call]
    B --> C[Cortex Script]
    C --> D[Hardware Action]
    style C fill:#232323,stroke:#fff,stroke-width:2px,color:#fff

🛡 Why This Exists

This separation of concerns protects your facility:

    Deterministic Control: The AI cannot "hallucinate" a device ID. It can only trigger pre-approved intent codes defined in this script.

    Latency Optimization: Complex logic (like checking which room you are in) happens locally in milliseconds, not in the cloud.

    Concurrency: The script runs in parallel mode, allowing the system to handle multi-step instructions (e.g., "Turn on the TV and dim the lights") without one action blocking the other.

🧠 The Core Logic Script

Copy the YAML block below and paste it into a new Script in Home Assistant. Name the script entity script.glados_cortex.
YAML

alias: GLaDOS Cortex (Intention Engine Router)
description: >-
  The central nervous system for the GLaDOS Intention Engine (v3.1.1).
  Routes LLM-determined 'intents' to physical hardware actions using 
  deterministic logic gates.
mode: parallel
max: 10
fields:
  intent:
    description: The abstract protocol code (e.g., LIGHTS, MUSIC, SLEEP).
    example: LIGHTS
  payload:
    description: The target device, room, or search string.
    example: living room
  media_type:
    description: (Optional) Music search scope (artist, album, playlist).
    example: playlist
  brightness_pct:
    description: (Optional) Target brightness for lighting arrays (1-100).
    example: 50
  hs_color:
    description: (Optional) Target color for lighting arrays [Hue, Saturation].
    example: [0, 100]
  shuffle:
    description: (Optional) Boolean to randomize auditory stimuli.
    example: true

sequence:
  # ----------------------------------------------------------------------
  # 1. LOGIC GATE: INTENT ROUTING
  # ----------------------------------------------------------------------
  - choose:
      
      # ------------------------------------------------------------------
      # PROTOCOL: CIRCADIAN REGULATION (Sleep)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'SLEEP' }}"
        sequence:
          # Replace with your actual TV/Remote Device ID
          - action: remote.turn_off
            target:
              device_id: YOUR_TV_DEVICE_ID_HERE 
          - action: light.turn_off
            target:
              entity_id: all

      # ------------------------------------------------------------------
      # PROTOCOL: DISTRACTION UNIT (TV Control)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'TV_ON' or intent == 'TV_MODE' }}"
        sequence:
          - action: script.turn_on_tv # Calls a local sub-script for WOL/IR
      - conditions:
          - condition: template
            value_template: "{{ intent == 'TV_OFF' }}"
        sequence:
          - action: remote.turn_off
            target:
              device_id: YOUR_TV_DEVICE_ID_HERE

      # ------------------------------------------------------------------
      # PROTOCOL: COGNITIVE ANCHORS (PC / ThinkBox / PS5)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'PC' }}"
        sequence:
          - action: script.direct_switch_to_think_box_anchor
      - conditions:
          - condition: template
            value_template: "{{ intent == 'THINKBOX' }}"
        sequence:
          - action: script.switch_to_thinkbox
      - conditions:
          - condition: template
            value_template: "{{ intent == 'PS5' }}"
        sequence:
          - action: script.direct_switch_to_games_anchor

      # ------------------------------------------------------------------
      # PROTOCOL: AUDITORY STIMULI (Music Assistant)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'MUSIC' }}"
        sequence:
          - action: script.music_assistant_voice_control
            data:
              media_id: "{{ payload }}"
              media_type: "{{ media_type | default('track') }}"
              # Enriches the metadata for dashboard display
              media_description: "Auditory experiment: {{ payload }}" 
              shuffle: "{{ shuffle | default(false) }}"
              mass_player: media_player.squeeze_lx # DEFINE YOUR DEFAULT PLAYER

      # ------------------------------------------------------------------
      # PROTOCOL: MUSIC TRANSPORT (Pause/Resume/Skip)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent in ['MUSIC_PAUSE', 'MUSIC_RESUME', 'MUSIC_SKIP'] }}"
        sequence:
          - action: >-
              {% if intent == 'MUSIC_PAUSE' %} media_player.media_pause 
              {% elif intent == 'MUSIC_RESUME' %} media_player.media_play 
              {% else %} media_player.media_next_track 
              {% endif %}
            target:
              entity_id: media_player.squeeze_lx

      # ------------------------------------------------------------------
      # PROTOCOL: LIGHTS (Luminous Emission Arrays)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'LIGHTS' }}"
        sequence:
          - variables:
              safe_payload: "{{ payload | default('') | lower }}"
              # AREA MAPPING: Add your rooms here
              # NOTE: Default Fallback is 'living_room' to prevent logic errors on vague inputs.
              target_area: >-
                {% if 'bed' in safe_payload %} bedroom 
                {% elif 'kitchen' in safe_payload %} kitchen
                {% else %} living_room 
                {% endif %}
              # LOGIC GATES: Check if parameters exist before sending
              has_color: >-
                {{ hs_color is defined and hs_color is not none and not
                (hs_color == [0, 0] and 'white' not in safe_payload) }}
              has_brightness: "{{ brightness_pct is defined and brightness_pct is not none }}"
          
          # EXECUTION: Send only the necessary data to prevent "State Reset" (Flashbangs)
          - choose:
              - conditions: "{{ has_color and has_brightness }}"
                sequence:
                  - action: light.turn_on
                    target:
                      area_id: "{{ target_area }}"
                    data:
                      hs_color: "{{ hs_color }}"
                      brightness_pct: "{{ brightness_pct }}"
              - conditions: "{{ has_color }}"
                sequence:
                  - action: light.turn_on
                    target:
                      area_id: "{{ target_area }}"
                    data:
                      hs_color: "{{ hs_color }}"
              - conditions: "{{ has_brightness }}"
                sequence:
                  - action: light.turn_on
                    target:
                      area_id: "{{ target_area }}"
                    data:
                      brightness_pct: "{{ brightness_pct }}"
            default:
              - action: light.turn_on
                target:
                  area_id: "{{ target_area }}"

      # ------------------------------------------------------------------
      # PROTOCOL: LIGHTS_OFF
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'LIGHTS_OFF' }}"
        sequence:
          - action: light.turn_off
            target:
              area_id: >-
                {% set p = payload | default('') | lower %}  
                {% if 'bed' in p %} bedroom  
                {% elif 'kitchen' in p %} kitchen
                {% else %} living_room 
                {% endif %}

🔧 Protocol Expansion Guide (Adding New Capabilities)

To add a new capability (e.g., "Open the Garage"), you must maintain synchronization across the Logic Triad. Failure to align all three files will result in a "Ghost Signal"—where the AI thinks it acted, but no hardware responds.
The Alignment Checklist:

    System Prompt (system_prompt.md): Add intent="GARAGE" to the ENUM list so the Brain knows the concept exists.

    Function Block (system_prompt.md): Add "GARAGE" to the enum list in the tool definition so the Brain has a button to press.

    Cortex (glados_cortex.yaml): Add the routing logic below so the Nervous System knows which wire to pull.

🛠 How to Edit the Cortex

You can edit the Cortex in two ways: via the Visual Editor (UI) for ease of use, or via YAML for speed and precision.
Option A: The Visual Editor (UI) Method

Best for users who are uncomfortable with code syntax.

    Navigate to Settings > Automations & Scenes > Scripts and open glados_cortex.

    Scroll down to the main Choose block (labeled "Sequence").

    Click the + Add Option button at the bottom of the list.

    Conditions:

        Click Add Condition → select Template.

        Paste the trigger logic: {{ intent == 'GARAGE' }}

    Actions:

        Click Add Action → select Device or Call Service.

        Select your hardware (e.g., cover.garage_door → Open).

    Click Save Script.

Option B: The YAML Method (Elite)

Best for speed and complex logic.

    Open the script in YAML Mode (three dots top-right → Edit in YAML).

    Scroll to the choose: block.

    Paste the following block directly before the default: or last item:

YAML

      # ------------------------------------------------------------------
      # PROTOCOL: GARAGE (Vehicle Bay)
      # ------------------------------------------------------------------
      - conditions:
          - condition: template
            value_template: "{{ intent == 'GARAGE' }}"
        sequence:
          - action: cover.open_cover
            target:
              entity_id: cover.main_garage_door
          - action: notify.mobile_app_phone
            data:
              message: "Vehicle bay transport assembly activated."

    Click Save.

⚠️ Common Deployment Errors

    Renaming the Script: The LLM Function block specifically calls script.glados_cortex. If you rename this script to anything else (e.g., script.my_ai_logic), the system will fail silently.

    Forgotten Placeholders: Ensure you have replaced YOUR_TV_DEVICE_ID_HERE and media_player.squeeze_lx with your actual Entity IDs, or the script will error out.

    Ambiguous Room Names: If a user says "Turn on the lights" without specifying a room, this script defaults to living_room. If you do not have a light.living_room entity, you must update the default fallback in the YAML.
