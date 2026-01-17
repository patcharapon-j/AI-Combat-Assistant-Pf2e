# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PF2e AI Combat Assistant is a **Foundry Virtual Tabletop (VTT) Module** for Pathfinder 2e that provides AI-powered combat suggestions. It analyzes combat state and recommends optimal actions for NPCs/creatures using external LLM APIs.

**Key characteristics:**
- Pure vanilla JavaScript (no TypeScript, no build system)
- Single entry point: `scripts/main.js` (~6,200 lines)
- Targets Foundry VTT v12+ with PF2e system v6.0.0+
- Uses OpenAI-compatible LLM APIs for suggestions

## Development Workflow

**No build/compile step required.** Changes to JavaScript take effect on Foundry reload.

**Testing in Foundry:**
1. Place module folder in Foundry modules directory (typically `<FoundryData>/modules/pf2e-ai-combat-assistant/`)
2. Enable module in Foundry: Game Settings > Manage Modules
3. Configure LLM settings: Game Settings > PF2e AI Combat Assistant
4. Test in an active combat encounter

**Debugging:** Use browser developer console (F12) for logs and errors. Module logs prefixed with `PF2e AI Combat Assistant |`.

## Architecture

### Core Data Flow

```
Combat Turn Change (updateCombat hook)
    → Show AI Turn Offer Dialog
    → User accepts → Initialize turnState flag on actor
    → gatherGameState() - Collect full combat context
    → craftSingleActionPrompt() - Build LLM prompt
    → callLLM() - API request to LLM provider
    → parseLLMSuggestion() - Parse response
    → Display suggestion with action buttons
    → User confirms/skips → Loop until turn ends
    → Generate turn summary
```

### Flag-Based State Management

The module stores state using Foundry's flag system on documents:

```javascript
const FLAGS = {
    DESIGNATIONS: 'designations',     // Combat-level: friendly/enemy status
    TURN_STATE: 'turnState',          // Actor-level: actions, MAP, history
    IS_PROCESSING: 'isProcessing',    // Actor-level: turn processing state
    CACHED_STRIKES: 'cachedStrikes',  // Actor-level: strike data cache
    PERMANENT_NOTES: 'permanentNotes' // Actor-level: persistent AI behavior notes
}
```

Access flags via: `document.getFlag(MODULE_ID, FLAGS.FLAG_NAME)`

### Key Functions

**Main workflow (`scripts/main.js`):**
- `requestNextAISuggestion()` (line ~2049) - Orchestrates getting next AI suggestion
- `gatherGameState()` (line ~2807) - Collects full combat context for prompt
- `craftSingleActionPrompt()` (line ~4525) - Builds system prompt for LLM
- `callLLM()` (line ~5100) - Makes API call to LLM provider
- `parseLLMSuggestion()` (line ~5153) - Parses LLM JSON response

**Event handlers (UI button clicks):**
- `_onAcceptControlClick()` - Accept AI turn control
- `_onConfirmActionClick()` - Confirm suggested action
- `_onSkipActionClick()` - Skip/reject suggestion
- `_onEndTurnClick()` - End combatant turn
- `_onCastSpellClick()` / `_onExecuteStrikeClick()` - Execute spell/attack

**PF2e data extraction:**
- `getNumericRange()` - Extract range from abilities/weapons
- `summarizeAbilityDetails()` - Clean HTML descriptions for prompts
- `parseActionCostValue()` - Parse PF2e action costs
- `isSpellAvailable()` / `isAbilityOnCooldown()` - Resource tracking

### Foundry Integration Points

**Hooks used:**
- `init` / `ready` - Module initialization and settings registration
- `updateCombat` - Detect turn changes, trigger AI offers
- `renderCombatTracker` - Add designation UI to combat tracker
- `deleteCombat` / `deleteCombatant` - Cleanup flags
- `renderChatMessage` - Auto-scroll chat, attach button handlers
- `renderActorSheetPF2eCharacter` - Add AI Notes button to character sheets
- `getActorSheetHeaderButtons` - Add header buttons to sheets

**Settings (registered in `registerSettings()` at line ~5739):**
- `apiKey` - LLM API key (secret)
- `llmEndpoint` - API URL (default: OpenAI)
- `aiModel` - Model identifier (default: gpt-4o)
- `showOfferToPlayers` - Visibility toggle
- `includeReactionsInPrompt` - Include reactions in prompts
- `whisperTurnSummary` - GM-only summaries

## File Structure

```
├── module.json          # Foundry module manifest
├── scripts/main.js      # All module logic
├── styles/main.css      # UI styling for chat messages/dialogs
├── lang/en.json         # Localization strings
└── media/               # README screenshots
```

## PF2e-Specific Patterns

- **Action Economy:** Tracks 3 actions/round, handles conditions (stunned, slowed)
- **Multiple Attack Penalty (MAP):** Auto-increments after confirming attacks
- **Spell Slots:** Checks availability before suggesting spells
- **Traits:** Uses PF2e traits for range (`range-increment-X`, `reach-X`), action types
- **Conditions/Effects:** Parses active conditions for context

## Module Constants

```javascript
const MODULE_ID = 'pf2e-ai-combat-assistant';
```

All flags, settings, and stored data use this ID for namespacing.
