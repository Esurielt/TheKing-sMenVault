# The King's Men —  Architecture 

**Engine:** Unreal Engine 5  
**Genre:** Card-Based Time Management Strategy    
**Scope:** Single-player 

-----

## Table of Contents

1. [Systems Overview](#1-systems-overview)
2. [Directory Structure](#2-directory-structure)
3. [Card System](#3-card-system)
4. [Cadence System](#4-cadence-system)
5. [Event System](#5-event-system-cadence)
6. [Story System](#6-story-system-cadence)
7. [Stat Check System](#7-stat-check-system)
8. [Table Subsystem](#8-table-subsystem)
9. [Turn Manager Subsystem](#9-turn-manager-subsystem)
10. [Undo System](#10-undo-system)
11. [UI Architecture](#11-ui-architecture)
12. [JSON Serialization & Modding Pipeline](#12-json-serialization--modding-pipeline)
13. [Data Flow Summary](#13-data-flow-summary)
14. [Implementation Order](#14-implementation-order)

-----

## Systems Overview

The King's Men is a card-based time management strategy game. Players manage a falling kingdom by placing cards representing characters, resources, and abstract concepts into Stories — interactive affairs that resolve over time and produce narrative consequences.

The gameplay architecture is organized around five systems:

- **Card** — Instances with rarity, tags, tag-value pairs, equipment (other attached cards)
- **Event System** — Background mini state machine chaining together, engine of the game narrative
- **Story System** — Player-interactive text with slots, timers, stat check, and branching resolution
- **Turn Manager Subsystem** — Orchestrates the phase pipeline, coordinating all other subsystems
- **Table Subsystem** — Pure data layer; unified source of truth for all game state including Event, Story, and card ownership. 

### Data-Oriented Implementation

Every game object has two representations:

- `UXxxDefinition` — immutable `UDataAsset`, the template to create runtime instances, with a unique `DefinitionID` (`FName` generated from `FGuid`) for unambiguous identification in save/load, logging, and cross-system lookups.
- `UXxxInstance` — mutable `UObject`, created from a definition at runtime with a unique `InstanceID` as well.

This data-oriented approach separates logic and data, allows for clean and modular authoring; it serializes only instance state, reference definitions by their stable `DefinitionID` string. New card types, event chains, Story behaviours, roll modifiers, and stat checks can be authored in-engine as DataAssets or in JSON and imported on startup. No C++ changes are required to add new content.

**Polymorphism.** The implementation also takes advantage of polymorphic instanced objects in Data Assets for a modular blueprint experience.

`URollModifier`, `UEventCondition`, and `UEventAction` all use the same `Abstract` + `EditInlineNew` + `Instanced` UObject pattern. This means that designers can define new variants of Rolls, Conditions, and Actions in Blueprint without new C++ changes, following one known path throughout the codebase.

**Tag over Names.** Where possible, identifiers and tag keys use Unreal's `FGameplayTag` / `FGameplayTagContainer` system. This provides hierarchical tag queries (e.g. `Stat.Diplomacy`, `Type.Character.Main`), native editor validation, and future compatibility with the Gameplay Ability System. `FName` is still used for string identifiers that are not tag-based (e.g. `DefinitionID`, `SlotID`, `InstanceID`).

**Tags vs. Stats split.** Card properties are divided into two structures: `FGameplayTagContainer` for categorical/presence queries (“is this a character?”, “does this have any Equipment.* tag?”) and `TMap<FGameplayTag, int32>` for numeric values that get summed, compared, and aggregated (“what is this card's Stat.Diplomacy?”). This avoids polluting the tag container with fake boolean values and gives each query type its optimal data structure.

**Event-driven Game States.** To  keep the responsibility clean, UI widgets bind to delegates on `UTableSubsystem` and each of all subsystems. The UI and player controller never poll or write game state on the Table — all player actions route through the `UStorySubsystem` (for card placement and story interaction) or `UTurnManagerSubsystem` (for phase advancement).

### Cadence

Cadence System is the data-driven turn manager for any game object that updates through phases. It is the concept at the heart of the Events, Story, and Turn Manager systems. Cadence has zero project-specific dependencies — it is a standalone module for any stateful content.

Cadence consists of:

- `ICadence` — Interface for stateful phase-driven objects. Does not depend on any project-specific types.
- `UCadenceSystem` — Abstract base class for subsystems hosting a collection of `ICadence` objects and their gameplay logic.
- `UCadenceConductor` — Abstract `UGameInstanceSubsystem` that controls turn and phase flow, ticks all registered `UCadenceSystem` instances in order. Project-specific conductors (e.g. `UTurnManagerSubsystem`) extend this to add game-specific phase hooks.

### Responsibilities

|System                 |Owns                                                                                                 |
|-----------------------|-----------------------------------------------------------------------------------------------------|
|`UTableSubsystem`      |Data only — Board zones, Hand, Pool, Graveyard, Globals, delegates                                   |
|`UEventSubsystem`      |Event Cadence management — activate, trigger, resolve, chain; owns ActionQueue                       |
|`UStorySubsystem`      |Story Cadence management — placement, commit, tick, resolve, expire; handles player input            |
|`UTurnManagerSubsystem`|Phase pipeline (extends `UCadenceConductor`) — orchestrates all subsystems, game-specific phase hooks|

-----

## 2. Directory Structure

```
Source/
  Cadence/                               # Standalone module — no project dependencies
    CadenceConductor.h/.cpp              # Abstract base turn/phase controller
    CadenceSystem.h/.cpp                 # Abstract base for subsystems hosting ICadence objects
    Cadence.h                            # ICadence interface
  KingsMen/
    Cards/
      CardDefinition.h/.cpp              # UDataAsset — card template
      CardInstance.h/.cpp                # UObject — live card state
    Events/
      EventDefinition.h/.cpp             # UDataAsset — event template
      EventInstance.h/.cpp               # UObject — live event state
      EventCondition.h/.cpp              # Abstract base for trigger conditions
      EventAction.h/.cpp                 # Abstract base for actions
    Stories/
      StoryDefinition.h/.cpp             # UDataAsset — Story template
      StoryInstance.h/.cpp               # UObject — live Story state
      StorySlotDefinition.h              # FStorySlotDefinition struct (immutable)
      StorySlotState.h                   # FStorySlotState struct (runtime)
    StatChecks/
      StatCheck.h/.cpp                   # UStatCheck — one check definition (concrete)
      StatCheckResult.h/.cpp             # UStatCheckResult — per-check result branch
      StatCheckSession.h/.cpp            # Runtime session — owns roll state + reroll pool
      StatCheckOutcome.h                 # FStatCheckOutcome struct
      RollModifier.h/.cpp                # Abstract base URollModifier
      PendingRollData.h                  # FPendingRollData struct (delegate payload)
      RollModifiers/
        RollMod_CardInSlot.h/.cpp
        RollMod_PriorCheckTier.h/.cpp
        RollMod_GlobalVariable.h/.cpp
        RollMod_CardHasTag.h/.cpp
        RollMod_TagValue.h/.cpp
    Table/
      TableSubsystem.h/.cpp              # UGameInstanceSubsystem — pure data layer
      TurnManagerSubsystem.h/.cpp        # UCadenceConductor subclass — game-specific phase hooks
      EventSubsystem.h/.cpp              # UCadenceSystem subclass
      StorySubsystem.h/.cpp              # UCadenceSystem subclass
      DefinitionRegistry.h/.cpp          # UGameInstanceSubsystem — all definitions
    Undo/
      UndoSnapshot.h/.cpp                # FUndoSnapshot — serialized board state
      UndoSubsystem.h/.cpp               # UGameInstanceSubsystem — manages undo stack
    Serialization/
      DefinitionBase.h/.cpp              # Shared ToJson/FromJson + source tracking
      DefinitionTypeRegistry.h/.cpp      # UEngineSubsystem — maps "$type" strings to UClass*
      ModLoaderSubsystem.h/.cpp          # Runtime JSON mod loading
    UI/
      BoardWidget.h/.cpp
      HandWidget.h/.cpp
      StoryWidget.h/.cpp
      CardWidget.h/.cpp
      StatCheckWidget.h/.cpp             # Dice display, result preview, reroll button
    Editor/
      DefinitionSyncChecker.h/.cpp       # UEditorSubsystem — drift detection on startup
      KingsDefinitionDetails.h           # Read-only detail panel with "Edit Source" button

Content/
  DataAssets/                            # Generated DataAssets — build artifacts
    Cards/                               # DA_Edwin.uasset ...
    Stories/                             # DA_PrivateAudience.uasset ...
    Events/                              # DA_SuccessionCrisis.uasset ...
Data/                                    # JSON source files — THE authority
  Cards/                                 # card_edwin.json ...
  Stories/                               # story_private_audience.json ...
  Events/                                # event_succession_crisis.json ...
Mods/                                    # Runtime mod directory — JSON only, no engine required
```

-----

## 3. Card System

Cards are the core representation of player's resources. Their mechanics and attributes are represented through two complementary structures — a `FGameplayTagContainer` for categorical presence queries and a `TMap<FGameplayTag, int32>` for numeric stats — plus equipment (attached cards).

Cards are created from immutable definition templates, and managed at runtime through instances. Serialization will record both the template and runtime changes in stats, tags, and equipment.

-----

### 3a. UCardDefinition (Data Asset)

```cpp
UCLASS(BlueprintType)
class UCardDefinition : public UKingsDefinitionBase
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) FText CardName;
    UPROPERTY(EditAnywhere) FText CardText;

    // Categorical — presence queries, hierarchical matching, slot validation.
    // e.g. Character.Main, Type.Consumable, Trait.FreshFlower
    UPROPERTY(EditAnywhere) FGameplayTagContainer Tags;

    // Numeric — stat values, rarity, mechanic values, anything that gets summed or compared.
    // e.g. Stat.Diplomacy: 5, Card.Rarity: 2, Mechanic.Reroll: 2, Mechanic.Expire: 2
    UPROPERTY(EditAnywhere) TMap<FGameplayTag, int32> Stats;

    // Equipment slots are separated out for easier authoring.
    UPROPERTY(EditAnywhere) FGameplayTagContainer EquipmentSlots;

    // JSON Serialization
    TSharedPtr<FJsonObject> ToJson() const override;
    bool FromJson(const TSharedPtr<FJsonObject>& JsonObject) override;
};
```

All card logic is defined through tags and stats, and how other systems react to / use them. Not only rarity, type, and attributes are expressed this way, but also behaviours such as expiration (`Mechanic.Expire`) and rerolls (`Mechanic.Reroll`). Only equipment slots are maintained separately from tags.

-----

### 3b. UCardInstance (Runtime Object)

```cpp
UCLASS(BlueprintType, Blueprintable)
class UCardInstance : public UObject
{
    GENERATED_BODY()
public:
    // --- Identity ---
    UPROPERTY(BlueprintReadOnly)
    FName InstanceID;  // Unique per instance, generated from FGuid::NewGuid()

    UPROPERTY(BlueprintReadOnly)
    TWeakObjectPtr<UCardDefinition> Definition;

    // --- Live State ---
    UPROPERTY(BlueprintReadWrite) FGameplayTagContainer Tags;                       // mutable copy from definition
    UPROPERTY(BlueprintReadWrite) TMap<FGameplayTag, int32> Stats;                  // mutable copy from definition
    UPROPERTY(BlueprintReadWrite) TMap<FGameplayTag, TObjectPtr<UCardInstance>> EquippedCards;
    UPROPERTY(BlueprintReadOnly)  bool bIsDestroyed = false;

    // --- Cached (includes equipment contributions) ---
    UPROPERTY(Transient, BlueprintReadOnly) FGameplayTagContainer MergedTags;       // own + all equipped card tags
    UPROPERTY(Transient)                    TMap<FGameplayTag, int32> MergedStats;   // own + all equipped card stats

    static UCardInstance* CreateFromDefinition(UObject* Outer, UCardDefinition* Def);

    // --- Presence Queries (reads MergedTags) ---
    bool HasTag(FGameplayTag Tag) const;               // hierarchical match
    bool HasTagExact(FGameplayTag Tag) const;           // exact match only
    bool HasAllTags(const FGameplayTagContainer& Required) const;
    bool HasAnyTag(const FGameplayTagContainer& Query) const;

    // --- Value Queries (reads MergedStats) ---
    int32 GetStat(FGameplayTag StatTag) const;          // 0 if absent

    // --- Cache Management ---
    void RebuildCache();  // rebuilds both MergedTags and MergedStats from own + equipment

    // --- Equipment ---
    bool Equip(UCardInstance* Card, FGameplayTag SlotTag);
    void Unequip(FGameplayTag SlotTag);
    bool CanEquip(UCardInstance* Card, FGameplayTag SlotTag) const;

    // --- Lifespan ---
    // Reads Mechanic.Expire from Stats. Called by UTurnManagerSubsystem each StartOfTurn.
    // If Mechanic.Expire > 0, decrements it. When it reaches 0, card is destroyed.
    void TickLifespan();
    int32 GetRemainingLifespan() const;  // returns Stats value for Mechanic.Expire; 0 = permanent
};
```

**Notes:**

**`TWeakObjectPtr` for `Definition`.** `TWeakObjectPtr` holds a reference that does not prevent garbage collection. If the underlying `UCardDefinition` DataAsset is unloaded — for example during a hot-reload of mod definitions, or if the definition registry replaces a shipped definition with a mod override — the weak pointer gracefully becomes null rather than keeping a stale definition alive in memory. This avoids two problems: (1) memory leaks where replaced definitions are never GC'd because instances still hold strong references, and (2) silent bugs where instances reference an outdated definition after a mod override. Code that reads `Definition` should null-check it (in practice, definitions are always valid during normal gameplay, but the weak pointer makes the contract explicit and safe against edge cases).

**Cached merged values.** `MergedTags` and `MergedStats` aggregate the card's own data plus all equipped card contributions. `MergedTags` is a `FGameplayTagContainer` built by merging the card's `Tags` with every equipped card's `Tags` — enabling hierarchical presence queries like `HasTag("Equipment")` or `HasAnyTag(SlotRequirements)`. `MergedStats` is a `TMap<FGameplayTag, int32>` built by summing the card's `Stats` with every equipped card's `Stats`. Both are rebuilt by `RebuildCache()` whenever tags, stats, or equipment change. All queries read from the cache for O(1) lookups, eliminating repeated linear scans during stat checks.

**Lifespan via Stats.** Expiration is stored as `Mechanic.Expire` in the Stats map rather than as a separate field. `TickLifespan()` reads and decrements this value. A card with no `Mechanic.Expire` key (or value 0) is permanent.

-----

### 3c. Example Cards

```
Edwin
  Tags: Character.Main, Character.Human.Commoner, Character.Gender.Male
  Stats: Card.Rarity:2, Stat.Sociability:5, Stat.Prowess:3, Stat.Poise:5,
         Stat.Scholarship:2, Stat.Subterfuge:3, Mechanic.Reroll:2
  Equipment slots: Slot.Weapon, Slot.Accessory, Slot.Accessory, Slot.Attire, Slot.Animal

Inside Information
  Tags: Type.Consumable.Advantage
  Stats: Card.Rarity:2, Stat.Diplomacy:4, Mechanic.Reroll:2

Diamond Necklace
  Tags: Type.Equipment.Accessory
  Stats: Card.Rarity:2, Stat.Poise:2
  Equipment slots: Slot.Adornment

Hyacinth Flower
  Tags: Equipment.Adornment, Trait.FreshFlower
  Stats: Card.Rarity:1, Mechanic.Reroll:1, Mechanic.Expire:2

Gold Coin
  Tags: Asset.Gold
  Stats: Card.Rarity:3

Cracked Diamond
  Tags: Asset.Gem, Equipment.Adornment, Trait.Broken
  Stats: Card.Rarity:4
```

-----

## 4. Cadence System

Events and Stories share the same structural skeleton: a collection of stateful objects with discrete states, a per-phase tick that advances state, and actions that fire on transitions. This is extracted into a reusable **Cadence** pattern that both `UEventSubsystem` and `UStorySubsystem` extend.

**Cadence has zero project-specific dependencies.** The `ICadence` interface, `UCadenceSystem`, and `UCadenceConductor` form a standalone module. They do not reference `UTableSubsystem`, card types, or any game-specific code. Concrete game objects (Events, Stories) implement `ICadence` and cache their own project-specific dependencies at creation time.

This pattern is reusable across future projects. Any system with stateful objects that tick through a phase pipeline — buffs, quests, faction agendas, contracts — can extend `UCadenceSystem` with minimal modification.

-----

### 4a. ICadence

```cpp
UINTERFACE(BlueprintType)
class UCadence : public UInterface { GENERATED_BODY() };

class ICadence
{
    GENERATED_BODY()
public:
    // Current state as a gameplay tag — each subclass defines its own state enum
    // and exposes it here. e.g. "State.Activated", "State.Resolving", "State.Expired".
    virtual FGameplayTag GetState() const = 0;

    // Returns the set of phases this unit responds to. The cadence system only calls
    // Tick() during these phases, skipping units that are irrelevant to the current phase.
    virtual FGameplayTagContainer GetRelevantPhases() const = 0;

    // Called by the cadence system each relevant phase.
    // Returns true if this unit is finished and should be removed.
    // NOTE: No project-specific parameters. Concrete implementations cache their
    // own dependencies (e.g. UTableSubsystem) at creation time.
    virtual bool Tick(FGameplayTag Phase) = 0;

    // Unique instance ID for logging, save/load, and delegate payloads.
    virtual FName GetInstanceID() const = 0;

    // Definition ID for cross-referencing.
    virtual FName GetDefinitionID() const = 0;
};
```

**Phase-filtered ticking.** Rather than iterating all units every phase, `GetRelevantPhases()` lets each unit declare which phases it responds to. The cadence system partitions units by phase at registration time and only ticks the relevant subset. This is O(relevant units per phase) instead of O(all units), which matters as content scales to hundreds of active events.

**Configurable phases.** The phase pipeline is defined as `FGameplayTag` values rather than a hard-coded enum, allowing designers to add custom phases in the editor without C++ changes. See Section 9 for phase configuration.

-----

### 4b. UCadenceSystem (Abstract Base)

```cpp
UCLASS(Abstract)
class UCadenceSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    // Called by UCadenceConductor each phase.
    void TickPhase(FGameplayTag Phase);

    void AddUnit(TScriptInterface<ICadence> Unit);
    void RemoveUnit(TScriptInterface<ICadence> Unit);

    TArray<TScriptInterface<ICadence>> GetUnitsInState(FGameplayTag State) const;
    TScriptInterface<ICadence>         FindUnitByInstanceID(FName InstanceID) const;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUnitAdded,   FName, InstanceID);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnUnitRemoved, FName, InstanceID);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnUnitStateChanged, FName, InstanceID, FGameplayTag, NewState);

    UPROPERTY(BlueprintAssignable) FOnUnitAdded          OnUnitAdded;
    UPROPERTY(BlueprintAssignable) FOnUnitRemoved        OnUnitRemoved;
    UPROPERTY(BlueprintAssignable) FOnUnitStateChanged   OnUnitStateChanged;

protected:
    // Units partitioned by relevant phase for efficient ticking.
    TMap<FGameplayTag, TArray<TScriptInterface<ICadence>>> UnitsByPhase;

    // Master list for lookup and iteration.
    TArray<TScriptInterface<ICadence>> AllUnits;

    // Subclasses override for pre/post-tick logic specific to their domain.
    // No project-specific parameters — subclasses cache their own dependencies.
    virtual void PrePhase(FGameplayTag Phase)  {}
    virtual void PostPhase(FGameplayTag Phase) {}
};
```

`TickPhase` captures each unit's state before calling `Tick()`. If the state changes, it broadcasts `OnUnitStateChanged`. Units returning `true` are removed and `OnUnitRemoved` is broadcast. Subclasses add typed, domain-specific delegates on top, wrapping the base delegates with fully-typed payloads.

-----

### 4c. UCadenceConductor (Abstract Base)

The conductor controls the rhythm of the game — it owns the phase sequence, tracks the current turn and phase, and ticks all registered cadence systems in order. It is abstract so that each project can extend it with game-specific phase hooks.

```cpp
UCLASS(Abstract)
class UCadenceConductor : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadOnly) int32        CurrentTurn = 1;
    UPROPERTY(BlueprintReadOnly) FGameplayTag CurrentPhase;

    // Ordered phase sequence. Editable in Blueprint to add/reorder phases.
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FGameplayTag> PhaseSequence;

    // Register cadence systems to be ticked. Order of registration determines tick order.
    void RegisterSystem(UCadenceSystem* System);
    void UnregisterSystem(UCadenceSystem* System);

    // Advance to the next phase. Subclasses may override to add pre/post hooks.
    UFUNCTION(BlueprintCallable)
    virtual void AdvancePhase();

    // Phase query — any system can check this.
    bool IsInPhase(FGameplayTag Phase) const;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPhaseChanged, FGameplayTag, NewPhase);
    UPROPERTY(BlueprintAssignable) FOnPhaseChanged OnPhaseChanged;

protected:
    int32 CurrentPhaseIndex = 0;
    TArray<TObjectPtr<UCadenceSystem>> RegisteredSystems;

    // Called by AdvancePhase(). Subclasses override for game-specific logic per phase.
    virtual void OnPhaseExecute(FGameplayTag Phase) {}
};
```

-----

## 5. Event System (Cadence)

Events are the **invisible engine** of the board. They never accept direct player input. Their role is to activate Stories, create cards, modify globals, and chain into further events — driving the game's narrative state machine in the background.

### 5a. Why a State Machine?

The three states give precise control over *when* an event acts, not just *whether* it acts:

- **Activated** — listening for trigger conditions. The processor only evaluates conditions for Activated events, keeping the check set small.
- **Triggered** — conditions met; the resolution pipeline processes only Triggered events this phase.
- **Resolved** — prevents double-firing within a phase. Repeatable events reset to Activated after resolving; one-shot events are deactivated.

**Event ID uniqueness on the Board.** No two event instances with the same `DefinitionID` may exist on the Board simultaneously. When `ActivateEvent()` is called, the subsystem first checks whether an instance of that definition is already active. If so, the activation is silently ignored (no-op). This eliminates cascading duplicate events and makes it structurally impossible for the same event to fire more than once at a time.

**Chaining rules.** When Event A's action activates Event B, Event B enters `Activated` this phase but won't `Trigger` until the next eligible phase check. `EEventTriggerPhase::Immediate` means “evaluate at the next phase boundary regardless of which phase that is” — it does **not** mean “evaluate right now within the current action drain.” This ensures deterministic ordering and prevents unbounded recursive chains.

```cpp
UENUM(BlueprintType)
enum class EEventState : uint8
{
    Activated,
    Triggered,
    Resolved
};

UENUM(BlueprintType)
enum class EEventTriggerPhase : uint8
{
    StartOfTurn,
    AfterStoryResolved,
    AfterEvent,
    Immediate          // Evaluate at the very next phase boundary, whichever it is
};
```

-----

### 5b. UEventCondition and UEventAction

Both use `Abstract` + `EditInlineNew` + `Instanced`, allowing designers to configure complex logic entirely in the DataAsset editor without writing new C++ for each variant.

```cpp
UCLASS(Abstract, EditInlineNew, BlueprintType, Blueprintable)
class UEventCondition : public UObject
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent)
    bool IsMet(const UTableSubsystem* Table) const;

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& Obj);
};

// Shipped subclasses:
// UCondition_TurnNumber      — fires on a specific turn
// UCondition_GlobalVariable  — checks a global threshold (>=, <=, ==, >, <)
// UCondition_StoryResolved   — checks if a Story with given ID resolved this turn
// UCondition_CardInHand      — checks if a card with a given tag exists in Hand
// UCondition_CardInSlot      — checks if a card with given tags/stats is in a specific slot

UCLASS(Abstract, EditInlineNew, BlueprintType, Blueprintable)
class UEventAction : public UObject
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent)
    void Execute(UTableSubsystem* Table);

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& Obj);
};

// Shipped subclasses:
// UAction_ActivateEvent    — adds an EventInstance to UEventSubsystem (no-op if already active)
// UAction_CreateStory      — instantiates a Story on the Board via UStorySubsystem
// UAction_CreateCard       — adds a card instance to Hand or CardPool
// UAction_ModifyGlobal     — changes a global variable by delta
// UAction_SetGlobal        — sets a global variable to an absolute value
// UAction_GlobalTag        — pushes a tag to global variable container (value = -1, presence only)
// UAction_DestroyCard      — sends a card to the Graveyard
// UAction_BoxOption        — presents the player with a dialogue box and branching options
// UAction_Speech           — displays a speech bubble on a target card
```

-----

### 5c. UEventDefinition (Data Asset)

```cpp
UCLASS(BlueprintType)
class UEventDefinition : public UKingsDefinitionBase
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) bool               bRepeatable = false;
    UPROPERTY(EditAnywhere) EEventTriggerPhase TriggerPhase;

    // All conditions must be true to transition to Triggered.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventCondition>> TriggerConditions;

    // Actions executed when Triggered.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventAction>> Actions;

    TSharedPtr<FJsonObject> ToJson() const override;
    bool FromJson(const TSharedPtr<FJsonObject>& JsonObject) override;
};
```

-----

### 5d. UEventInstance (Runtime)

```cpp
UCLASS()
class UEventInstance : public UObject, public ICadence
{
    GENERATED_BODY()
public:
    UPROPERTY() FName InstanceID;  // Unique, generated from FGuid::NewGuid()
    UPROPERTY() TWeakObjectPtr<UEventDefinition> Definition;
    UPROPERTY() EEventState State = EEventState::Activated;

    // Cached at creation by UEventSubsystem::ActivateEvent().
    UPROPERTY() TWeakObjectPtr<UTableSubsystem> Table;

    // ICadence — no project-specific parameters
    FGameplayTag GetState() const override;
    FGameplayTagContainer GetRelevantPhases() const override;
    FName GetInstanceID() const override { return InstanceID; }
    FName GetDefinitionID() const override { return Definition->DefinitionID; }
    bool  Tick(FGameplayTag Phase) override;  // uses cached Table reference internally

    bool CheckTrigger() const;
    void ExecuteActions();

    // Repeatable events reset to Activated; one-shot events return true (remove).
    void Resolve();
};
```

-----

### 5e. UEventSubsystem

```cpp
UCLASS()
class UEventSubsystem : public UCadenceSystem
{
    GENERATED_BODY()
public:
    // Activates an event. Returns nullptr (no-op) if an instance with the same
    // DefinitionID is already active on the Board. This enforces event uniqueness.
    // Sets Table reference on the new instance.
    UEventInstance* ActivateEvent(UEventDefinition* Def);

    void            DeactivateEvent(UEventInstance* Event);

    // Returns true if an event with this DefinitionID is currently active.
    bool IsEventActive(FName DefinitionID) const;

    // --- Action Queue ---
    // Actions from event triggers and stat check results are queued here.
    // Drained during PostPhase on Phase.EventResolution.
    void QueueActions(const TArray<UEventAction*>& Actions);

    // Maximum actions drained per EventResolution phase.
    // If exceeded, remaining actions overflow to the next turn's EventResolution.
    // A warning is logged when the limit is hit.
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxActionsPerPhase = 1000;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEventActivated, UEventInstance*, Event);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEventTriggered, UEventInstance*, Event);
    UPROPERTY(BlueprintAssignable) FOnEventActivated OnEventActivated;
    UPROPERTY(BlueprintAssignable) FOnEventTriggered OnEventTriggered;

protected:
    // Tracks active DefinitionIDs for O(1) uniqueness checks.
    TSet<FName> ActiveDefinitionIDs;

    // Action queue — drained during EventResolution.
    TArray<TObjectPtr<UEventAction>> ActionQueue;

    // Cached at Initialize().
    TWeakObjectPtr<UTableSubsystem> Table;

    // PostPhase: on EventResolution, drains ActionQueue (up to MaxActionsPerPhase).
    // On AfterStoryResolved/AfterEvent, evaluates newly Activated events.
    void PostPhase(FGameplayTag Phase) override;
};
```

-----

## 6. Story System (Cadence)

Stories are the **player-facing affairs** on the Board. They accept card placements, hold slot requirements, count down over turns, and produce outcomes through stat checks.

### 6a. EStoryState

Stories track their own lifecycle independently of the turn phase, enabling immediate resolution during Planning and multi-turn resolution during Execution.

```cpp
UENUM(BlueprintType)
enum class EStoryState : uint8
{
    Available,      // On the board; player can freely place and remove cards
    Committed,      // Player confirmed; cards locked, countdown started
    Resolving,      // Counting down turns (TurnsToResolve > 0)
    Resolved,       // Successfully completed
    Failed,         // Stat check failed
    Expired         // Deadline passed without being committed
};
```

-----

### 6b. FStorySlotDefinition and FStorySlotState

Definition data and runtime state are cleanly separated. `FStorySlotDefinition` lives on the DataAsset and is never mutated. `FStorySlotState` lives on the instance and carries all runtime data.

```cpp
// --- IMMUTABLE: Lives on UStoryDefinition ---

UENUM(BlueprintType)
enum class EStorySlotType : uint8 { Required, Optional };

USTRUCT(BlueprintType)
struct FStorySlotDefinition
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere) FName              SlotID;
    UPROPERTY(EditAnywhere) EStorySlotType     SlotType = EStorySlotType::Required;

    // Designer-facing label and description.
    UPROPERTY(EditAnywhere) FText              Label;     // e.g. "Main Character", "Gift"
    UPROPERTY(EditAnywhere) FText              Text;      // flavor text shown on hover

    // --- Tag Requirements ---
    // All listed tags must be present on every card placed in this slot.
    UPROPERTY(EditAnywhere) FGameplayTagContainer RequiredTags;

    // At least one of these tags must be present. Empty = no any-tag requirement.
    UPROPERTY(EditAnywhere) FGameplayTagContainer AnyTags;

    // --- Stat Requirements ---
    // Optional stat conditions on the card itself (e.g. "Card.Rarity >= 3").
    // All must pass for the card to be valid in this slot.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventCondition>> RequiredStatConditions;

    // For named-card slots (e.g. "Annual Report of the Great Counties").
    UPROPERTY(EditAnywhere) FName              RequiredCardID;   // DefinitionID, empty = any matching card

    // --- Stacking ---
    // If true, multiple cards can be placed in this slot (e.g. 10 Gold Coins).
    UPROPERTY(EditAnywhere) bool               bAllowStack = false;

    // Maximum number of cards allowed in this slot. 0 = unlimited.
    UPROPERTY(EditAnywhere) int32              MaxStackSize = 0;

    // Minimum number of cards required in this slot for it to count as "filled."
    UPROPERTY(EditAnywhere) int32              MinStackSize = 1;
};


// --- MUTABLE: Lives on UStoryInstance ---

USTRUCT(BlueprintType)
struct FStorySlotState
{
    GENERATED_BODY()

    // References the corresponding definition by SlotID.
    UPROPERTY() FName SlotID;

    // For non-stacking slots: single card (PlacedCards[0] or empty).
    // For stacking slots: all cards currently in the stack.
    UPROPERTY() TArray<TObjectPtr<UCardInstance>> PlacedCards;

    // Convenience accessors.
    int32 GetCardCount() const { return PlacedCards.Num(); }
    UCardInstance* GetFirstCard() const { return PlacedCards.Num() > 0 ? PlacedCards[0] : nullptr; }
    bool IsEmpty() const { return PlacedCards.Num() == 0; }
};
```

**Stacking resolves the resource ambiguity.** Rather than encoding “10 gold” as a tag value on a single card, the player places 10 individual Gold Coin cards into a stacking slot with `MinStackSize = 10`. This keeps cards as discrete atomic units, makes resource spending visible and interactive, and works naturally with the existing card system.

-----

### 6c. UStoryDefinition (Data Asset)

```cpp
UCLASS(BlueprintType)
class UStoryDefinition : public UKingsDefinitionBase
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) FText  StoryName;
    UPROPERTY(EditAnywhere) int32  MustResolveByTurn = 0;     // 0 = no deadline
    UPROPERTY(EditAnywhere) int32  TurnsToResolve    = 1;     // 0 = immediate on Commit
    UPROPERTY(EditAnywhere) bool   bPersistent       = false; // resets to Available after resolving

    UPROPERTY(EditAnywhere) TArray<FStorySlotDefinition> Slots;

    // Stat checks evaluated on resolution. See Section 7.
    // Each check owns its own result branches — there is no Story-level success/failure.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UStatCheck>> StatChecks;

    // --- Resolution Prior ---
    // Prioritized result branches evaluated immediately after commit (regardless of duration),
    // before the story fully resolves. If any branch here fires, the main resolution
    // results for that result_id are skipped. Useful for critical interrupts, branching
    // paths, or early-exit conditions (e.g. presenting a blackmail item).
    // Only static checks are recommended here.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UStatCheckResult>> ResolutionPrior;

    // --- On-Expire Behavior ---
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventAction>> OnExpire;

    // Default: non-consumable cards returned to Hand, consumable cards destroyed.
    UPROPERTY(EditAnywhere) bool bReturnCardsOnExpire = true;

    // --- On-End Behavior ---
    UPROPERTY(EditAnywhere) bool bReturnCardsOnEnd = true;

    TSharedPtr<FJsonObject> ToJson() const override;
    bool FromJson(const TSharedPtr<FJsonObject>& JsonObject) override;
};
```

**No Story-level success/failure actions.** All outcome logic is defined per-check via result branches (see Section 7). This makes each check self-contained. `ResolutionPrior` allows early-exit branches that preempt main resolution — such as presenting a blackmail item that completely changes the story's direction.

-----

### 6d. UStoryInstance (Runtime)

```cpp
UCLASS()
class UStoryInstance : public UObject, public ICadence
{
    GENERATED_BODY()
public:
    UPROPERTY() FName InstanceID;  // Unique, from FGuid::NewGuid()
    UPROPERTY() TWeakObjectPtr<UStoryDefinition> Definition;
    UPROPERTY() TArray<FStorySlotState> SlotStates;     // runtime state, parallel to Definition->Slots
    UPROPERTY() int32              TurnsRemaining;
    UPROPERTY() int32              TurnPlacedOn;
    UPROPERTY() EStoryState        State = EStoryState::Available;

    // --- Cached Slot Aggregates ---
    UPROPERTY(Transient) FGameplayTagContainer SlotMergedTags;       // all placed cards' tags merged
    UPROPERTY(Transient) TMap<FGameplayTag, int32> SlotMergedStats;  // all placed cards' stats summed

    // Cached at creation by UStorySubsystem::AddStory().
    UPROPERTY() TWeakObjectPtr<UTableSubsystem> Table;

    // ICadence — no project-specific parameters
    FGameplayTag GetState() const override;
    FGameplayTagContainer GetRelevantPhases() const override;
    FName GetInstanceID() const override { return InstanceID; }
    FName GetDefinitionID() const override { return Definition->DefinitionID; }
    bool  Tick(FGameplayTag Phase) override;  // uses cached Table reference internally

    // Slot management — called by UStorySubsystem on behalf of player input.
    bool CanPlaceCard(UCardInstance* Card, FName SlotID) const;
    bool PlaceCard(UCardInstance* Card, FName SlotID);
    void RemoveCard(UCardInstance* Card, FName SlotID);
    void RemoveAllCards(FName SlotID);
    bool AreRequiredSlotsFilled() const;

    // Slot queries.
    const FStorySlotState* GetSlotState(FName SlotID) const;
    const FStorySlotDefinition* GetSlotDefinition(FName SlotID) const;
    int32 GetSlotCardCount(FName SlotID) const;

    // Aggregate stat from cards in specified slots (or all if LimitToSlots is empty).
    int32 GetTotalStatValue(FGameplayTag StatTag, const TArray<FName>& LimitToSlots = {}) const;

    void RebuildSlotCache();

    bool Commit();    // uses cached Table
    void Resolve();   // creates UStatCheckSession, fires check result actions
    void Expire();    // returns cards, fires OnExpire actions
    void ReturnOrDestroyCards();
};
```

-----

### 6e. UStorySubsystem

```cpp
UCLASS()
class UStorySubsystem : public UCadenceSystem
{
    GENERATED_BODY()
public:
    UStoryInstance* AddStory(UStoryDefinition* Def);
    void            RemoveStory(UStoryInstance* Story);

    // --- Player Input ---
    // These are the single path by which the UI affects story state.
    // All validate that the conductor is in Planning phase before acting.
    UFUNCTION(BlueprintCallable)
    bool PlaceCard(UCardInstance* Card, UStoryInstance* Story, FName SlotID);

    UFUNCTION(BlueprintCallable)
    bool RecallCard(UCardInstance* Card, UStoryInstance* Story, FName SlotID);

    UFUNCTION(BlueprintCallable)
    bool CommitStory(UStoryInstance* Story);

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStoryCommitted, UStoryInstance*, Story);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStoryResolved,  UStoryInstance*, Story);
    UPROPERTY(BlueprintAssignable) FOnStoryCommitted OnStoryCommitted;
    UPROPERTY(BlueprintAssignable) FOnStoryResolved  OnStoryResolved;

protected:
    // Cached at Initialize().
    TWeakObjectPtr<UTableSubsystem> Table;
    TWeakObjectPtr<UCadenceConductor> Conductor;  // for phase validation

    // PrePhase(StartOfTurn): checks deadlines, expires overdue Stories,
    // returns/destroys cards from expired Stories.
    void PrePhase(FGameplayTag Phase) override;
};
```

-----

## 7. Stat Check System

Stat checks evaluate the outcomes of a Story. Each check is **self-contained**: it defines its own stat sources, dice behavior, and an ordered array of **result branches**, each with a condition, narrative text, and actions. There is no Story-level success/failure — all logic lives in the checks.

### 7a. The Roll Formula

```
final_result = Aggregate(n × d20) + SummedStat
```

Where:

- `n = 1 + Σ(modifier.Evaluate())` — base pool is always 1; modifiers contribute bonus dice when their conditions are met
- `Aggregate = Max` (advantage, default) or `Min` (disadvantage, punishing)
- `SummedStat` = sum of named stats across cards in specified slots, including equipment
- `final_result` is then evaluated against the check's result branches in order

Static checks skip dice entirely: evaluate `SummedStat` against result branch conditions. No roll, no modifiers, no rerolls.

-----

### 7b. FStatScope — Where Stats Come From

```cpp
USTRUCT(BlueprintType)
struct FStatScope
{
    GENERATED_BODY()

    // If empty: sum stat from ALL placed cards (including their equipment).
    // If populated: only sum from cards in these specific slot IDs.
    UPROPERTY(EditAnywhere) TArray<FName> LimitToSlots;

    UPROPERTY(EditAnywhere) FGameplayTag StatTag;  // e.g. "Stat.Diplomacy", "Stat.Poise"
};
```

Multiple `FStatScope` entries on one check are summed together before being added to the roll, enabling expressions like `Slot2.Stat.Support + Slot3.Stat.Support`.

-----

### 7c. ERollAggregate

```cpp
UENUM(BlueprintType)
enum class ERollAggregate : uint8
{
    Max,    // Take highest die — advantage (default)
    Min,    // Take lowest die — disadvantage, punishing
};
```

-----

### 7d. URollModifier — Attachable Dice Bonus Conditions

Roll modifiers use the same `Abstract` + `EditInlineNew` + `Instanced` pattern as `UEventCondition` and `UEventAction`. Each modifier evaluates a condition at roll time and contributes bonus dice to the pool if met.

Modifiers can optionally carry their own narrative text (`ResultTitle`, `ResultText`) displayed when their condition is met, adding context to why the player got bonus dice.

```cpp
UCLASS(Abstract, EditInlineNew, BlueprintType, Blueprintable)
class URollModifier : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) int32 BonusDice = 1;
    UPROPERTY(EditAnywhere) FText Hint;          // shown on Story intro (empty = hidden)
    UPROPERTY(EditAnywhere) FText ResultTitle;    // shown when condition met during resolution
    UPROPERTY(EditAnywhere) FText ResultText;     // narrative text for this modifier's activation

    // Conditions — all must be met for this modifier to grant bonus dice.
    // Reuses UEventCondition for consistency with the polymorphic pattern.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventCondition>> Conditions;

    UFUNCTION(BlueprintNativeEvent)
    int32 Evaluate(const UStoryInstance* Story,
                   const UStatCheckSession* Session,
                   const UTableSubsystem* Table) const;

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& Obj);
};
```

**Shipped subclasses:**

```cpp
// Bonus dice if a card matching conditions is placed in SlotID.
UCLASS(EditInlineNew)
class URollMod_CardInSlot : public URollModifier
{
    UPROPERTY(EditAnywhere) FName                  SlotID;
    UPROPERTY(EditAnywhere) FGameplayTagContainer  RequiredTags;
};

// Bonus dice if a previous check in this session achieved a minimum result.
UCLASS(EditInlineNew)
class URollMod_PriorCheckTier : public URollModifier
{
    UPROPERTY(EditAnywhere) int32              CheckIndex;
    UPROPERTY(EditAnywhere) FName              MinimumResultID;
};

// Bonus dice if a global variable meets a condition.
UCLASS(EditInlineNew)
class URollMod_GlobalVariable : public URollModifier
{
    UPROPERTY(EditAnywhere) FGameplayTag       Key;
    UPROPERTY(EditAnywhere) EComparisonOp      Operator;
    UPROPERTY(EditAnywhere) int32              Value;
};

// Bonus dice if any card in SlotID has a specific tag.
UCLASS(EditInlineNew)
class URollMod_CardHasTag : public URollModifier
{
    UPROPERTY(EditAnywhere) FName              SlotID;
    UPROPERTY(EditAnywhere) FGameplayTag       RequiredTag;
};

// Bonus dice based on the numeric value of a stat across cards in SlotID.
UCLASS(EditInlineNew)
class URollMod_StatMet : public URollModifier
{
    UPROPERTY(EditAnywhere) FName              SlotID;       // empty = all slots
    UPROPERTY(EditAnywhere) FGameplayTag       StatTag;
    UPROPERTY(EditAnywhere) EComparisonOp      Operator;
    UPROPERTY(EditAnywhere) int32              Value;
};
```

-----

### 7e. UStatCheckResult — Per-Check Result Branch

Each check has an ordered array of result branches. After the final roll (or static value) is determined, the branches are evaluated **in order** — the first branch whose condition is met fires its actions. This replaces the old tier system with a fully data-driven conditional structure.

```cpp
USTRUCT(BlueprintType)
struct FCheckResultCondition
{
    GENERATED_BODY()

    // Which check's result to compare. Typically the current check's own ID.
    // Can reference a prior check's result for cross-check conditionals.
    UPROPERTY(EditAnywhere) FName CheckID;

    UPROPERTY(EditAnywhere) EComparisonOp Operator;  // >, >=, <, <=, ==, !=
    UPROPERTY(EditAnywhere) int32 Value;
};

UCLASS(EditInlineNew, BlueprintType)
class UStatCheckResult : public UObject
{
    GENERATED_BODY()
public:
    // Unique ID for this result branch within the check.
    UPROPERTY(EditAnywhere) FName ResultID;

    // Conditions — all must be true for this branch to activate.
    UPROPERTY(EditAnywhere) TArray<FCheckResultCondition> Conditions;

    // Narrative output.
    UPROPERTY(EditAnywhere) FText ResultTitle;
    UPROPERTY(EditAnywhere) FText ResultText;

    // Actions executed when this branch is selected.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventAction>> Actions;

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& Obj);
};
```

**Branch evaluation is first-match.** Branches are tested in array order; the first one whose conditions all pass is selected. Designers should order branches from most specific (highest threshold) to least specific (fallback). The last branch can have empty conditions to serve as a guaranteed fallback.

**Result grouping.** In JSON, results under the same `result_id` form a group — only one result from each group fires. Multiple groups with different IDs are evaluated independently, allowing parallel narrative tracks (e.g. a diplomacy outcome and a separate narration block both fire from the same story).

-----

### 7f. UStatCheck — One Check Definition (Concrete)

```cpp
UCLASS(EditInlineNew, BlueprintType, Blueprintable)
class UStatCheck : public UObject
{
    GENERATED_BODY()
public:
    // Unique ID within the Story, used for cross-check references in result conditions.
    UPROPERTY(EditAnywhere) FName CheckID;

    // Which stats to collect and from which slots. Multiple scopes are summed.
    UPROPERTY(EditAnywhere) TArray<FStatScope> StatSources;

    UPROPERTY(EditAnywhere) EStatCheckType  CheckType  = EStatCheckType::DiceRoll;
    UPROPERTY(EditAnywhere) ERollAggregate  Aggregate  = ERollAggregate::Max;

    // Base pool is always 1 die. Each modifier that evaluates true adds to it.
    // Ignored entirely when CheckType == Static.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<URollModifier>> DiceModifiers;

    // Ordered result branches. First branch whose conditions are met is selected.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UStatCheckResult>> Results;

    TSharedPtr<FJsonObject> ToJson() const;
    bool FromJson(const TSharedPtr<FJsonObject>& Obj);
};
```

Note: `UStatCheck` is **concrete** (not `Abstract`). It is instantiated directly on `UStoryDefinition` via the `Instanced` property. There are no subclasses — all variation is expressed through `CheckType`, `DiceModifiers`, and `Results`.

-----

### 7g. FStatCheckOutcome — Per-Check Record

```cpp
USTRUCT(BlueprintType)
struct FStatCheckOutcome
{
    GENERATED_BODY()

    UPROPERTY() FName              CheckID;
    UPROPERTY() EStatCheckType     CheckType;
    UPROPERTY() int32              StatValue;        // summed stat before adding to roll
    UPROPERTY() TArray<int32>      AllRolls;         // all individual d20 values (empty for static)
    UPROPERTY() int32              RollUsed;         // the aggregated die value (0 for static)
    UPROPERTY() int32              FinalResult;      // RollUsed + StatValue (or StatValue for static)
    UPROPERTY() FName              SelectedResultID; // which result branch was selected
    UPROPERTY() int32              RerollsSpentHere;
};
```

-----

### 7h. FPendingRollData — Delegate Payload Struct

```cpp
USTRUCT(BlueprintType)
struct FPendingRollData
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly) TArray<int32> AllRolls;
    UPROPERTY(BlueprintReadOnly) int32         RollUsed = 0;
    UPROPERTY(BlueprintReadOnly) int32         FinalResult = 0;
    UPROPERTY(BlueprintReadOnly) FName         PreviewResultID;
};
```

-----

### 7i. UStatCheckSession — Runtime Resolution

The session owns the shared reroll pool and manages sequential resolution of all checks in a Story. It is created when a Story is committed and lives until `Finalize()` completes.

```cpp
UCLASS()
class UStatCheckSession : public UObject
{
    GENERATED_BODY()
public:
    static UStatCheckSession* CreateForStory(UObject* Outer, UStoryInstance* Story);

    UPROPERTY() TObjectPtr<UStoryInstance>  OwningStory;
    UPROPERTY() int32 TotalRerolls;        // summed from all placed cards' "Mechanic.Reroll" stats at creation
    UPROPERTY() int32 RemainingRerolls;
    UPROPERTY() int32 CurrentCheckIndex = 0;
    UPROPERTY() TArray<FStatCheckOutcome> Outcomes;

    // Resolves the current check.
    //   Static:   resolves immediately, auto-advances without UI input.
    //             NOTE: UI should still display a brief animation beat (~200ms)
    //             before advancing to give the player visual feedback that
    //             a check was evaluated. The session fires the delegate and
    //             the UI is responsible for the pause timing.
    //   DiceRoll: rolls dice, stores pending result, broadcasts to UI.
    void ResolveCurrentCheck();

    bool SpendReroll();
    void ConfirmCurrentCheck();

    bool IsComplete() const;

    // Called after all checks are confirmed.
    // Fires each check's selected result branch actions.
    // Queues all resulting actions into UEventSubsystem's ActionQueue.
    void Finalize();

    bool  CanReroll() const;
    FName GetPendingResultID() const;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPendingRollReady, FPendingRollData, RollData);
    UPROPERTY(BlueprintAssignable) FOnPendingRollReady OnPendingRollReady;

private:
    FPendingRollData PendingRoll;

    // Cached references — set at creation.
    TWeakObjectPtr<UTableSubsystem> Table;
    TWeakObjectPtr<UEventSubsystem> Events;  // for QueueActions()

    int32         SumStatForCheck(const UStatCheck* Check) const;
    TArray<int32> RollDice(int32 NumDice);
    FName         EvaluateResultBranches(const UStatCheck* Check, int32 FinalResult) const;
};
```

**Resolution flow for one DiceRoll check:**

```
ResolveCurrentCheck():
  1. SumStatForCheck() — aggregate FStatScope sources via UStoryInstance::GetTotalStatValue()
  2. Evaluate all DiceModifiers → n = 1 + Σ(modifier.Evaluate())
  3. Roll n × d20 — store all values in PendingRoll.AllRolls
  4. Aggregate(Max or Min) → PendingRoll.RollUsed
  5. PendingRoll.FinalResult = RollUsed + SummedStat
  6. EvaluateResultBranches() → first-match result ID → PendingRoll.PreviewResultID
  7. Broadcast OnPendingRollReady → UI shows dice, result preview, reroll button if CanReroll()

Player may SpendReroll() any number of times while RemainingRerolls > 0:
  → Re-rolls n × d20 with same pool size, decrements RemainingRerolls
  → Re-evaluates result branches with new FinalResult
  → Broadcasts OnPendingRollReady again with new values

Player calls ConfirmCurrentCheck():
  → Records FStatCheckOutcome, increments CurrentCheckIndex
  → If more checks remain, ResolveCurrentCheck() is called for the next
  → Static checks auto-advance without broadcasting or waiting for input
```

-----

### 7j. JSON Schema for a Check with Result Branches

```json
{
  "check_id": "r1",
  "stat_sources": {
    "Stat.Sociability": [1],
    "Stat.Subterfuge": []
  },
  "check_type": {
    "$type": "Roll",
    "aggregate": "Max",
    "dice_modifiers": [
      {
        "$type": "StatMet",
        "slot_id": [],
        "stat": "Stat.Poise",
        "bonus_dice": 1,
        "hint": "",
        "result_title": "",
        "result_text": "You hold your liquor incredibly well at the party."
      }
    ]
  },
  "results": [
    {
      "result_id": "chase_success",
      "conditions": [{ "check_id": "r1", "operator": ">", "value": 10 }],
      "result_title": "A Familiar Shape",
      "result_text": "You chase after the man, shadow flickers on the alley wall in the weak lamp light, almost losing the sense of direction. Fortunately, you see a familiar shape afar — the white dome of Count Cornelius' manor that you've seen on the night of banquet. You slow down, marking the way where the mysterious man have slipped away.",
      "actions": [
        { "$type": "EventOn", "id": "event.HiddenDagger2" },
        { "$type": "GlobalAdd", "key": "gv.MaskedManTraced", "delta": 1 }
      ]
    },
    {
      "result_id": "chase_failure",
      "conditions": [{ "check_id": "r1", "operator": "<=", "value": 10 }],
      "result_title": "Lost in the Dark",
      "result_text": "You chase after the man, as shadow flickers on the alley wall in the weak lamp light, you start to lose the sense of direction. Slowing down, you realized you have no idea where the mysterious man have slipped away.",
      "actions": [
        { "$type": "EventOn", "id": "event.HiddenDaggerLost" },
        { "$type": "SetGlobal", "key": "gv.MaskedManTraced", "value": 0 }
      ]
    }
  ]
}
```

-----

### 7k. Full Story JSON Example

```json
{
  "id": "story.private_audience",
  "name": "Private Audience with the King",
  "deadline": 3,
  "duration": 2,
  "persistent": false,
  "expire": {
    "actions": [
      { "$type": "GlobalAdd", "key": "gv.RoyalFavor", "value": -20 },
      { "$type": "GlobalAdd", "key": "gv.KingdomStability", "value": -10 },
      { "$type": "BoxOption",
        "text": "You missed your chance to meet with the King. The court is abuzz with gossip about your absence...",
        "option": [
          { "text": "Write an apology letter to the King",
            "actions": [{ "$type": "EventOn", "key": "event.audience_missed" }] },
          { "text": "Request Another Audience",
            "actions": [{ "$type": "EventOn", "key": "event.audience_secondchance" }] }
        ]
      }
    ]
  },
  "slots": [
    { "slot_id": 1, "label": "Main Character",
      "text": "You have to be there.",
      "type": "Required",
      "required_tags": ["Character.Main"] },
    { "slot_id": 2, "label": "Companion",
      "text": "A noble companion might be helpful.",
      "type": "Optional",
      "required_tags": ["Character.Companion", "Character.Noble"] },
    { "slot_id": 3, "label": "Gift",
      "text": "A gift is expected, must not be too shabby.",
      "type": "Required",
      "required_stats": [{ "tag": "Rarity", "operator": ">=", "value": 3 }],
      "any_tags": ["Asset", "Item", "Equipment"],
      "stack": true, "min_stack": 10, "max_stack": 0 },
    { "slot_id": 4, "label": "Annual Report",
      "text": "An annual report is required.",
      "type": "Required",
      "required_card": "card.annual_report" },
    { "slot_id": 5, "label": "Consumable",
      "text": "",
      "type": "Optional",
      "required_tags": ["Consumable"],
      "stack": true, "min_stack": 1, "max_stack": 5 }
  ],
  "stat_checks": [
    {
      "check_id": "audience_diplomacy",
      "stat_sources": { "Stat.Diplomacy": [] },
      "check_type": {
        "$type": "Roll",
        "aggregate": "Max",
        "dice_modifiers": [
          { "condition": [{ "$type": "CardInSlot", "slot_id": 2, "required_tags": ["Character.Council"] }],
            "bonus_dice": 1,
            "hint": "Another Council member might be good company.",
            "result_title": "Council Support",
            "result_text": "Your companion's presence reassures the King, giving you an edge in your audience." },
          { "$type": "GlobalCheck", "key": "gv.RoyalFavor",
            "operator": ">=", "value": 50, "bonus_dice": 1,
            "hint": "Royal court's trust aids your goals.",
            "result_title": "Royal Favor",
            "result_text": "The King trusts you, your friendship is still ever dear to him." }
        ]
      }
    }
  ],
  "resolution_prior": [
    { "result_id": "pre_result",
      "conditions": [{ "$type": "CardInSlot", "slot_id": 3, "required_tags": ["special.KingShame"] }],
      "result_title": "A Bitter Reminder",
      "result_text": "The King notices your gift and his eyes grew cold. He takes the report from you and dismiss you without a word. Is your mockery effective in forcing his move?",
      "actions": [
        { "$type": "GlobalTag", "key": "gv.event.audience_blackmailed" },
        { "$type": "GlobalSet", "key": "gv.event.audience_attended", "value": 1 },
        { "$type": "Speech", "target": "Card.Character.Main", "text": "He will surely take the hint." },
        { "$type": "Speech", "target": "Card.Character.Main", "text": "Let's hope Cezar's plan works." }
      ]
    }
  ],
  "resolution": [
    { "result_id": "r1",
      "results": [
        { "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 20 }],
          "result_title": "Royal Favor",
          "result_text": "The King is deeply impressed by your counsel and generosity...",
          "actions": [
            { "$type": "GlobalAdd", "key": "gv.RoyalFavor", "value": 30 },
            { "$type": "GlobalAdd", "key": "gv.KingdomStability", "value": 10 }
          ] },
        { "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 15 }],
          "result_title": "A Productive Meeting",
          "result_text": "The King listens attentively and nods along...",
          "actions": [
            { "$type": "GlobalAdd", "key": "gv.RoyalFavor", "value": 20 },
            { "$type": "GlobalAdd", "key": "gv.KingdomStability", "value": 5 }
          ] },
        { "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 10 }],
          "result_title": "Polite but Distant",
          "result_text": "The King acknowledges your words but seems distracted...",
          "actions": [
            { "$type": "GlobalAdd", "key": "gv.RoyalFavor", "value": 10 }
          ] },
        { "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 1 }],
          "result_title": "A Cold Reception",
          "result_text": "The King barely acknowledges your presence...",
          "actions": [] },
        { "conditions": [],
          "result_title": "The King's Displeasure",
          "result_text": "Your words fall flat. The King's expression darkens...",
          "actions": [
            { "$type": "EventOn", "event_id": "event.king_displeasure" }
          ] }
      ]
    },
    { "result_id": "narration2",
      "results": [
        { "conditions": [],
          "result_title": "",
          "result_text": "Regardless of the outcome, you have made your move in the court, and the consequences will unfold in the days to come.",
          "actions": [
            { "$type": "GlobalSet", "key": "gv.event.audience_attended", "value": 1 }
          ] }
      ]
    }
  ],
  "on_expire": [
    { "$type": "GlobalAdd", "key": "gv.KingdomStability", "value": -5 }
  ]
}
```

-----

## 8. Table Subsystem

`UTableSubsystem` is the **pure data layer** — it owns all game state collections and exposes delegates for the UI. It contains no logic; all logic lives in `UEventSubsystem` and `UStorySubsystem`.



Keeping state in a single subsystem makes cross-system queries trivial (`GetCardsWithTag`, `FindStoryByID`), centralizes the delegate surface for UI bindings, and keeps save/load simple. It extends `UGameInstanceSubsystem` so it persists across level loads and is accessible anywhere via `GetGameInstance()->GetSubsystem<UTableSubsystem>()`.

```cpp
UCLASS()
class UTableSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    // --- Game State ---
    UPROPERTY() TArray<TObjectPtr<UStoryInstance>>  Board_Stories;
    UPROPERTY() TArray<TObjectPtr<UEventInstance>>  Board_Events;
    UPROPERTY() TArray<TObjectPtr<UCardInstance>>   Hand;
    UPROPERTY() TArray<TObjectPtr<UCardInstance>>   CardPool;    // all unlocked card instances
    UPROPERTY() TArray<TObjectPtr<UCardInstance>>   Graveyard;   // uninteractable; audit only
    UPROPERTY() TMap<FGameplayTag, int32>           Globals;     // "gv.KingdomStability", "gv.Treasury" ...

    // --- Tag Index ---
    UPROPERTY(Transient) TMap<FGameplayTag, TArray<TObjectPtr<UCardInstance>>> TagIndex;

    // --- Delegates ---
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStoryChanged, UStoryInstance*, Story);
    UPROPERTY(BlueprintAssignable) FOnStoryChanged OnStoryAdded;
    UPROPERTY(BlueprintAssignable) FOnStoryChanged OnStoryRemoved;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHandChanged, UCardInstance*, Card);
    UPROPERTY(BlueprintAssignable) FOnHandChanged OnCardAddedToHand;
    UPROPERTY(BlueprintAssignable) FOnHandChanged OnCardRemovedFromHand;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnGlobalChanged, FGameplayTag, Key, int32, NewValue);
    UPROPERTY(BlueprintAssignable) FOnGlobalChanged OnGlobalChanged;

    // --- Card Operations ---
    UCardInstance* DrawCard(UCardDefinition* Def);
    void           SendToGraveyard(UCardInstance* Card);
    void           UnlockCard(UCardDefinition* Def);

    // --- Global Variables ---
    int32 GetGlobal(FGameplayTag Key, int32 Default = 0) const;
    void  SetGlobal(FGameplayTag Key, int32 Value);
    void  ModifyGlobal(FGameplayTag Key, int32 Delta);
    void  PushGlobalTag(FGameplayTag Key);  // presence-only tag, value = -1

    // --- Queries ---
    TArray<UCardInstance*> GetCardsWithTag(FGameplayTag Tag) const;
    UStoryInstance*        FindStoryByID(FName DefinitionID) const;
    UStoryInstance*        FindStoryByInstanceID(FName InstanceID) const;

    // --- Tag Index Maintenance ---
    void RegisterCardInIndex(UCardInstance* Card);
    void UnregisterCardFromIndex(UCardInstance* Card);
};
```

-----

## 9. Turn Manager Subsystem

`UTurnManagerSubsystem` extends `UCadenceConductor` to add game-specific phase hooks — undo snapshot capture, card lifespan ticking, and any future project-specific behavior. It is the only place phase transitions occur.

### 9a. Configurable Phase Pipeline

Phases are defined as `FGameplayTag` values. The default phase sequence is configured as a `TArray<FGameplayTag>` on `UCadenceConductor`, editable in Blueprint.

```cpp
// Default phase tags (registered in Project Settings → Gameplay Tags):
//   Phase.StartOfTurn
//   Phase.Planning
//   Phase.Execution
//   Phase.EventResolution
//   Phase.Cleanup
//   Phase.EndOfTurn
```

-----

### 9b. UTurnManagerSubsystem

```cpp
UCLASS()
class UTurnManagerSubsystem : public UCadenceConductor
{
    GENERATED_BODY()
public:
    void Initialize(FSubsystemCollectionBase& Collection) override;

    // Override to add game-specific phase logic.
    void AdvancePhase() override;

protected:
    // Game-specific phase hooks.
    void OnPhaseExecute(FGameplayTag Phase) override;

private:
    // Phase implementations.
    void Phase_StartOfTurn();
    void Phase_Cleanup();

    // Cached references.
    TWeakObjectPtr<UTableSubsystem>  Table;
    TWeakObjectPtr<UEventSubsystem>  Events;
    TWeakObjectPtr<UStorySubsystem>  Stories;
    TWeakObjectPtr<UUndoSubsystem>   Undo;
};
```

**What it does vs. what it doesn't:**

`UTurnManagerSubsystem` adds project-specific hooks to the Cadence phase pipeline:

- `Phase_StartOfTurn()`: captures undo snapshot, ticks card lifespans, sends expired cards to Graveyard.
- `Phase_Cleanup()`: finalizes Graveyard, resets repeatable events, increments turn counter.

It does **not** handle player input (that's `UStorySubsystem`) or the action queue (that's `UEventSubsystem`). Its only job is orchestrating the phase clock and adding game-specific hooks to specific phases.

-----

### 9c. Turn Pipeline

```
StartOfTurn
  ├─ UUndoSubsystem::CaptureSnapshot() — save state before any mutations
  ├─ Tick all card lifespans → expired cards sent to Graveyard
  ├─ UStorySubsystem::PrePhase() → expire Stories past MustResolveByTurn
  │    └─ Expired Stories: ReturnOrDestroyCards(), fire OnExpire actions
  ├─ UEventSubsystem::TickPhase(Phase.StartOfTurn) → evaluate Activated events
  └─ Broadcast OnPhaseChanged → UI refreshes

Planning  [PLAYER INPUT — no automatic advance]
  │
  ├─ UStorySubsystem::PlaceCard() / RecallCard()
  │    → validates Conductor.IsInPhase(Phase.Planning), delegates to UStoryInstance
  │
  ├─ UStorySubsystem::CommitStory()
  │    → validates all required slots filled
  │    │
  │    ├─ [TurnsToResolve == 0] → UStoryInstance::Commit() → Resolve() fires immediately
  │    │    ├─ ResolutionPrior evaluated first — may preempt main resolution
  │    │    ├─ UStatCheckSession created; checks resolved sequentially with player input
  │    │    ├─ Per-check result branch actions → UEventSubsystem::QueueActions()
  │    │    ├─ ReturnOrDestroyCards() per Story definition rules
  │    │    └─ Story removed from Board (bPersistent → reset to Available instead)
  │    │
  │    └─ [TurnsToResolve  > 0] → State = Resolving, placed cards locked
  │         └─ UI shows countdown; Story is no longer interactive this turn
  │
  └─ UCadenceConductor::AdvancePhase() → player ends turn

Execution
  └─ UStorySubsystem::TickPhase(Phase.Execution):
       For each Story in state Resolving:
         ├─ Decrement TurnsRemaining
         └─ TurnsRemaining == 0 → Resolve() → UStatCheckSession → QueueActions()

EventResolution
  ├─ UEventSubsystem::PostPhase() drains ActionQueue (up to MaxActionsPerPhase)
  │    (may create new Stories, Cards, activate Events, modify Globals)
  │    If MaxActionsPerPhase exceeded:
  │      → Log warning: "[KingsMen] ActionQueue overflow: %d actions deferred to next turn"
  │      → Remaining actions stay in ActionQueue for next turn's EventResolution
  └─ UEventSubsystem::TickPhase(Phase.AfterStoryResolved | Phase.AfterEvent)
       → evaluate newly Activated events triggered by this turn's actions

Cleanup
  ├─ Graveyard: destroyed and expired cards finalized
  ├─ UEventSubsystem: repeatable Resolved events reset to Activated
  └─ CurrentTurn++ → return to StartOfTurn
```

**Key design consequences:**

- Cards are **locked on Commit, not on End Turn.** Committing Edwin to a 2-turn Story on Turn 3 means he cannot be placed elsewhere until it resolves on Turn 5.
- Immediate Stories resolve during Planning, but their actions execute in EventResolution. This prevents mid-Planning chain reactions where newly spawned Stories could be immediately committed in the same phase.
- **No rollback for immediate (0-turn) Stories.** This is intentional — once committed, the result stands. However, the Undo system (Section 10) can revert to the end of the previous turn.
- Persistent Stories reset to `Available` after resolving, reappearing for the next turn without being removed from the Board.
- The `ActionQueue` on `UEventSubsystem` is the single handoff point between resolution and consequence. Nothing executes mid-resolution — all effects are deferred to EventResolution, keeping phase behavior predictable.
- **Card return on Story end.** After resolution, non-consumable cards are returned to Hand by default. Consumable cards (`Type.Consumable` tag) are destroyed. Individual check result actions can override this (e.g. `Action_DestroyCard` to sacrifice a character).

-----

## 10. Undo System

The Undo system captures full board snapshots at the start of each turn, allowing the player to revert to the state at the end of the previous turn (before any current-turn mutations). Up to 5 snapshots are stored.

```cpp
USTRUCT()
struct FUndoSnapshot
{
    GENERATED_BODY()

    UPROPERTY() int32 TurnNumber;

    UPROPERTY() TArray<uint8> SerializedTableState;
    UPROPERTY() TArray<uint8> SerializedEventState;
    UPROPERTY() TArray<uint8> SerializedStoryState;
};

UCLASS()
class UUndoSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    void CaptureSnapshot(UTableSubsystem* Table, UEventSubsystem* Events, UStorySubsystem* Stories);

    bool UndoToTurn(int32 TargetTurn, UTableSubsystem* Table,
                    UEventSubsystem* Events, UStorySubsystem* Stories);

    TArray<int32> GetAvailableUndoTurns() const;

    bool CanUndo() const { return Snapshots.Num() > 0; }

private:
    static constexpr int32 MaxSnapshots = 5;
    TArray<FUndoSnapshot> Snapshots;
};
```

**Design notes:**

- Snapshots are captured at the **start** of each turn, before any StartOfTurn mutations.
- The undo operation fully replaces all subsystem state.
- Undo is available during Planning phase only. Not available mid-resolution.
- Save/load design (TBD) will need to serialize the undo stack as well.

-----

## 11. UI Architecture

Widgets are **purely presentational.** They hold read-only references to runtime instances and receive updates through delegates. They never directly modify game state — player card/story actions route through `UStorySubsystem`, and phase advancement routes through `UCadenceConductor`.

```
UBoardWidget
  ├─ Binds to: UTableSubsystem::OnStoryAdded, OnStoryRemoved
  ├─ Contains: TArray<UStoryWidget*>
  └─ Does not hold game state

UStoryWidget
  ├─ Holds a read-only ref to UStoryInstance
  ├─ Displays: slots (with stack counts), placed cards, turn countdown, stat preview, committed state
  └─ Dispatches:
       OnCardDropped     → UStorySubsystem::PlaceCard()
       OnRecallClicked   → UStorySubsystem::RecallCard()
       OnCommitClicked   → UStorySubsystem::CommitStory()

UHandWidget
  ├─ Binds to: UTableSubsystem::OnCardAddedToHand, OnCardRemovedFromHand
  └─ Contains: TArray<UCardWidget*>

UCardWidget
  ├─ Holds a read-only ref to UCardInstance
  ├─ Renders: name, tags, stats, equipment slots, rarity, lifespan
  └─ Is draggable via UDragDropOperation

UStatCheckWidget
  ├─ Created by UStoryWidget when a UStatCheckSession becomes active
  ├─ Binds to: UStatCheckSession::OnPendingRollReady
  ├─ Displays: all individual d20 values, aggregated roll, result branch preview
  │    Result text: shown from the previewed branch's ResultTitle + ResultText
  │    Modifier text: shown from any activated URollModifier's ResultTitle + ResultText
  │    Reroll button: visible only when UStatCheckSession::CanReroll() == true
  │    Reroll button: hidden entirely for static checks (they auto-advance)
  │    Reroll pool: shown as shared count across the full session
  │    NOTE: For static checks, the widget should display a brief animation
  │    pause (~200ms) before auto-advancing to give the player visual
  │    feedback that a check was evaluated.
  └─ Dispatches:
       OnRerollClicked   → UStatCheckSession::SpendReroll()
       OnConfirmClicked  → UStatCheckSession::ConfirmCurrentCheck()

UGlobalsWidget
  └─ Binds to: UTableSubsystem::OnGlobalChanged → updates HUD stats

UUndoButton
  └─ Dispatches: OnUndoClicked → UUndoSubsystem::UndoToTurn()
       Enabled only during Planning phase when UUndoSubsystem::CanUndo()

UAdvancePhaseButton
  └─ Dispatches: OnClicked → UCadenceConductor::AdvancePhase()
       Enabled only during Planning phase
```

-----

## 12. JSON Serialization & Modding Pipeline

### 12a. Philosophy: JSON is the Authority

DataAssets are **build artifacts** generated from JSON source files — analogous to compiled binaries from source code. Designers author in JSON; DataAssets are an engine-optimized representation of that source.

This approach:

- Preserves the full benefits of DataAssets: type safety, editor validation, polymorphic UObject references, and visual editing
- Enables modding without the engine: mod authors drop JSON files in `/Mods/`, no UE required
- Produces human-readable, diffable source files in version control
- Keeps the base game and mod content on the same pipeline for consistency

**Mods are JSON-only.** They load at runtime via `UModLoaderSubsystem` and register into `UDefinitionRegistry` alongside shipped content. Mod security validation is planned for a future revision.

-----

### 12b. UKingsDefinitionBase

```cpp
UCLASS(Abstract)
class UKingsDefinitionBase : public UDataAsset
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, Category="Source")
    FName DefinitionID;

    UPROPERTY(VisibleAnywhere, Category="Source")
    FString SourceJsonHash;

    UPROPERTY(VisibleAnywhere, Category="Source")
    FString SourceJsonPath;

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& JsonObject);

    void PostSave(const FObjectPostSaveContext& Context) override;

private:
    bool ExportToJson();
};
```

-----

### 12c. Bidirectional Sync

**DataAsset → JSON (on save):**

```
Designer saves DataAsset (Ctrl+S)
  → PostSave fires
  → AutoReimport watcher paused (prevents circular reimport loop)
  → ExportToJson() writes to Data/
  → Write confirmed → SourceJsonHash updated
  → AutoReimport watcher resumed
```

**JSON → DataAsset (on startup or manual reimport):**

```
Engine starts / designer edits JSON externally
  → UDefinitionSyncChecker hashes all JSON source files
  → Hash mismatch detected → auto-reimport triggered
  → DataAsset updated → SourceJsonHash updated
```

**Circular trigger prevention.** The `IgnoreFileChanges` scope guard in `ExportToJson()` suppresses UE's `FAutoReimportManager` file watcher during export, breaking the loop.

**Atomic hash update.** `SourceJsonHash` is only updated after confirming the file write succeeded.

-----

### 12d. Drift Detection (UDefinitionSyncChecker)

```cpp
UCLASS()
class UDefinitionSyncChecker : public UEditorSubsystem
{
    GENERATED_BODY()
public:
    void Initialize(FSubsystemCollectionBase& Collection) override { ScanForDrift(); }

private:
    void    ScanForDrift();
    FString HashFile(const FString& FilePath);
};
```

```
[KingsMen] DRIFT DETECTED — 2 definition(s) out of sync with JSON source:
  DA_PrivateAudience  stored hash: a3f9...  current JSON hash: 7b2c...  → REIMPORT REQUIRED
  DA_Edwin            source JSON not found at: Data/Cards/card_edwin.json
                      → file may have been moved or deleted
```

-----

### 12e. DataAsset Editor — Read-Only Panel

```cpp
class FKingsDefinitionDetails : public IDetailCustomization
{
public:
    void CustomizeDetails(IDetailLayoutBuilder& DetailBuilder) override;
    // Marks all properties as read-only.
    // Adds banner: "This asset is generated from JSON. Edit the source file."
    // Adds buttons: [Open Source JSON]  [Reimport Now]
};
```

-----

### 12f. Polymorphic Type Registry

The registry uses `UEngineSubsystem` — it needs to be available before any `UGameInstance` exists and has no per-game-instance state.

```cpp
UCLASS()
class UDefinitionTypeRegistry : public UEngineSubsystem
{
    GENERATED_BODY()
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;

    void RegisterActionType(FName TypeName, UClass* Class);
    void RegisterConditionType(FName TypeName, UClass* Class);
    void RegisterModifierType(FName TypeName, UClass* Class);

    UEventAction*    DeserializeAction(const TSharedPtr<FJsonObject>& Obj, UObject* Outer);
    UEventCondition* DeserializeCondition(const TSharedPtr<FJsonObject>& Obj, UObject* Outer);
    URollModifier*   DeserializeModifier(const TSharedPtr<FJsonObject>& Obj, UObject* Outer);

private:
    TMap<FName, UClass*> ActionTypes;
    TMap<FName, UClass*> ConditionTypes;
    TMap<FName, UClass*> ModifierTypes;
};

// Self-registration macro — place in each subclass .cpp file
REGISTER_ACTION_TYPE(Action_CreateStory,       UAction_CreateStory);
REGISTER_CONDITION_TYPE(Condition_TurnNumber,   UCondition_TurnNumber);
REGISTER_MODIFIER_TYPE(RollMod_CardInSlot,      URollMod_CardInSlot);
```

-----

### 12g. Definition Registry (Runtime)

All systems look up definitions by `DefinitionID` through `UDefinitionRegistry`, never by direct UObject pointer. This makes mod overrides transparent. Typed maps internally avoid downcast costs on every lookup.

```cpp
UCLASS()
class UDefinitionRegistry : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    void Initialize(FSubsystemCollectionBase& Collection) override;

    // Single entry point — routes by UClass internally. Cast once at registration.
    void Register(UKingsDefinitionBase* Def);

    // Typed lookups — no cast, direct map access.
    UCardDefinition*  FindCard(FName ID) const;
    UStoryDefinition* FindStory(FName ID) const;
    UEventDefinition* FindEvent(FName ID) const;

    // Generic fallback for tooling, debug commands, mod validation.
    UKingsDefinitionBase* FindAny(FName ID) const;

private:
    TMap<FName, TObjectPtr<UCardDefinition>>  CardDefs;
    TMap<FName, TObjectPtr<UStoryDefinition>> StoryDefs;
    TMap<FName, TObjectPtr<UEventDefinition>> EventDefs;

    void LoadShippedDefinitions();
    void LoadMods();
};
```

-----

### 12h. Mod Loader

```cpp
UCLASS()
class UModLoaderSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    void LoadAllMods(UDefinitionRegistry* Registry);

private:
    void ScanModDirectory();
    bool LoadDefinitionFile(const FString& FilePath, UDefinitionRegistry* Registry);
};
```

Mods are JSON files in `<GameDir>/Mods/`. A mod that registers with an existing `DefinitionID` replaces the shipped definition.

-----

### 12i. Source Control Conventions

Binary `.uasset` merge conflicts are handled at the **workflow level**:

- **Git + Git LFS:** `git lfs lock Content/DataAssets/**/*.uasset` before editing.
- **Perforce:** Exclusive checkout is native.

JSON source files remain freely mergeable as text.

-----

### 12j. JSON Schemas

```json
// card_edwin.json
{
  "id": "card.edwin",
  "name": "Edwin",
  "text": "You — a mere common human professor, and now, entrusted by the young King himself, the unlikely Seneschal of the royal council. What awaits you in this whirlpool of power?",
  "tags": ["Character.Main", "Character.Human.Commoner", "Character.Gender.Male"],
  "stats": {
    "Card.Rarity": 2,
    "Stat.Prowess": 3,
    "Stat.Sociability": 1,
    "Stat.Poise": 4,
    "Stat.Scholarship": 5,
    "Stat.Subterfuge": 2,
    "Stat.Influence": 0
  },
  "equipment_slots": ["Slot.Weapon", "Slot.Accessory", "Slot.Accessory", "Slot.Attire", "Slot.Animal"]
}

// card_HyacinthFlower.json
{
  "id": "card.HyacinthFlower",
  "name": "Hyacinth Flower",
  "text": "Beautiful purple flower.",
  "tags": ["Equipment.Adornment", "Trait.FreshFlower"],
  "stats": {
    "Card.Rarity": 1,
    "Mechanic.Reroll": 1,
    "Mechanic.Expire": 2
  },
  "equipment_slots": []
}

// event_succession_crisis.json
{
  "id": "event.succession_crisis",
  "repeatable": false,
  "trigger_phase": "StartOfTurn",
  "conditions": [
    { "$type": "Condition_TurnNumber", "turn": 5 },
    { "$type": "Condition_GlobalVariable", "key": "gv.KingdomStability",
      "operator": "LessThan", "value": 30 }
  ],
  "actions": [
    { "$type": "Action_CreateStory", "story_id": "story.royal_audit" },
    { "$type": "Action_ModifyGlobal", "key": "gv.CourtTension", "delta": 15 }
  ]
}
```

Full Story JSON example is in Section 7k.

-----

## 13. Data Flow Summary

```
JSON source file  →  [Import / PostSave export]  →  UDataAsset
                                                          │
                                              UDefinitionRegistry
                                          (Register() routes by type;
                                           typed FindXxx() lookups)
                                                          │
                         ┌────────────────────────────────┤
                         │                                │
               UEventDefinition                   UCardDefinition
                         │                                │
               UEventInstance                     UCardInstance
             (Board_Events)                           (Hand)
               [unique DefinitionID                      │
                enforced on Board]                       │
                         │                                │
            [Trigger] UEventAction::Execute()    [Player drags]
                         │                                │
                         ├──→ UStorySubsystem::AddStory() │
                         │         │       UStorySubsystem::PlaceCard()
                         │    UStoryInstance               │
                         │    (Board_Stories)       UStoryInstance::PlaceCard()
                         │                          (supports stacking)
                         │                                │
                         │                UStorySubsystem::CommitStory()
                         │                               │
                         │               ResolutionPrior evaluated (may preempt)
                         │                               │
                         │                    UStatCheckSession created
                         │                               │
                         │           For each check:
                         │             Evaluate DiceModifiers → n dice
                         │             Roll n × d20 → Aggregate(Max/Min)
                         │             + SummedStat → FinalResult
                         │             Evaluate result branches (first-match)
                         │             Player may SpendReroll (shared pool)
                         │             ConfirmCurrentCheck → next check
                         │                               │
                         │                    Finalize():
                         │             Per-check result actions queued
                         │             Story returns/destroys cards per rules
                         │                               │
                         └───────────────→  UEventSubsystem::QueueActions()
                                                         │
                                              EventResolution phase:
                                              Actions drain (up to MaxActionsPerPhase)
                                              Overflow → next turn's EventResolution
                                              → new Stories, Cards, Globals, Events
                                                         │
                                             UTableSubsystem delegates
                                                         │
                                                    UI Widgets
```

-----

## 14. Implementation Order

|Phase|Milestone           |Systems                                                                                                                                                               |
|-----|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|1    |Card foundation     |`FGameplayTagContainer` Tags, `TMap<FGameplayTag,int32>` Stats, `UCardDefinition`, `UCardInstance`, `MergedTags`/`MergedStats` cache, basic `UTableSubsystem` Hand ops|
|2    |Table & globals     |Full `UTableSubsystem`, `UDefinitionRegistry` (unified `Register()`, typed maps), Global Variables, TagIndex                                                          |
|3    |Cadence module      |`ICadence`, `UCadenceSystem`, `UCadenceConductor` (abstract), phase-filtered ticking                                                                                  |
|4    |Story placement     |`UStoryDefinition`, `UStoryInstance`, `FStorySlotDefinition`/`FStorySlotState`, slot stacking, `EStoryState`, `UStorySubsystem` (with player input methods)           |
|5    |Turn loop           |`UTurnManagerSubsystem` (extends `UCadenceConductor`), game-specific phase hooks, Planning → Execution pipeline                                                       |
|6    |Stat checks — static|`UStatCheck` (concrete), `FStatScope`, `UStatCheckResult`, `UStatCheckSession` static path                                                                            |
|7    |Stat checks — dice  |`ERollAggregate`, `URollModifier` base + shipped subclasses, d20 pool resolution, `FPendingRollData`                                                                  |
|8    |Reroll system       |`RemainingRerolls`, `SpendReroll()`, `OnPendingRollReady`, `UStatCheckWidget`                                                                                         |
|9    |Event backbone      |`UEventDefinition`, `UEventCondition`/`UEventAction` base classes, `UEventSubsystem` with uniqueness enforcement + ActionQueue                                        |
|10   |Event content       |Concrete Condition/Action subclasses (`BoxOption`, `Speech`, `GlobalTag`, etc.), `UDefinitionTypeRegistry` (UEngineSubsystem)                                         |
|11   |Event pipeline      |`UEventInstance`, full cadence integration, `MaxActionsPerPhase` overflow                                                                                             |
|12   |Undo system         |`FUndoSnapshot`, `UUndoSubsystem`, snapshot capture/restore                                                                                                           |
|13   |UI                  |Board, Hand, Card, Story, StatCheck widgets; drag-drop; all delegate bindings; static check animation pause                                                           |
|14   |Serialization       |`UKingsDefinitionBase`, `ToJson`/`FromJson` per type                                                                                                                  |
|15   |Editor tooling      |`UDefinitionSyncChecker`, read-only detail panel, `PostSave` auto-export                                                                                              |
|16   |Mod loading         |`UModLoaderSubsystem`, mod override support in `UDefinitionRegistry`                                                                                                  |
|17   |Save/Load           |TBD — will be designed separately                                                                                                                                     |
|18   |First content       |Edwin card, Private Audience Story, default event collection, first playable loop                                                                                     |