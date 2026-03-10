

---

## 1. Systems Overview

The King's Men is a card-based time management strategy game. Players manage a falling kingdom by placing cards representing characters, resources, and abstract concepts into Stories — interactive affairs that resolve over time and produce narrative consequences.

The gameplay architecture is organized around six systems:

- **Card System** — Instances with tag-value pairs, equipment (other attached cards), and lifespan
- **Event System** — Background state machine driving the game narrative
- **Story System** — Player-interactive affairs with slots, timers, and stat checks
- **Stat Check System** — Condition-driven dice pool resolution with per-check result branches
- **Turn Manager Subsystem** — Orchestrates the phase pipeline, coordinating all other subsystems
- **Table Subsystem** — Pure data layer; unified source of truth for all game state

### Data-Oriented Implementation

**Definition vs. Instance split.** Every game object has two representations:

- `UXxxDefinition` — immutable `UDataAsset`, the template to create runtime instances, with a unique `DefinitionID` (`FName` generated from `FGuid`) for unambiguous identification in save/load, logging, and cross-system lookups.
- `UXxxInstance` — mutable `UObject`, created from a definition at runtime with a unique `InstanceID` as well.

This data-oriented approach separates logic and data, allows for clean and modular authoring; it serializes only instance state, reference definitions by their stable `DefinitionID` string. New card types, event chains, Story behaviours, roll modifiers, and stat checks are authored in JSON and imported as DataAssets. No C++ changes are required to add new content.


**Event-driven Game States**: To further decouple the system and keep the responsibility clean, UI widgets bind to delegates on `UTableSubsystem` and `UTurnManagerSubsystem`. The UI and player controller never poll or write game state.


**Polymorphism**: The implementation also take advantage of polymorphic instanced object in Data Assets for a modular blueprint experience. 

`URollModifier`, `UEventCondition`, and `UEventAction` all use the same `Abstract` + `EditInlineNew` + `Instanced` UObject pattern. This means that designers can define new variants of Rolls, Conditions, and Action in Blueprint without new C++ changes, following one known path throughout the codebase.


**Tag over Names**: Where possible, identifiers and tag keys use Unreal's `FGameplayTag` / `FGameplayTagContainer` system. This provides hierarchical tag queries (e.g. `Stat.Diplomacy`, `Type.Character.Main`), native editor validation, and future compatibility with the Gameplay Ability System. `FName` is still used for string identifiers that are not tag-based (e.g. `DefinitionID`, `SlotID`, `InstanceID`).

### Cadence 
Cadence System is the data-driven turn manager for any game object that update through phases. It is the concept at the heart of the Events, Story, and Turn Manager System. 

Cadence is consist of:

`ICadence` -  Interface for stateful phase-driven objects.

`UCadenceSystem` - abstruct base class for subsystems hosting a collection of `ICadence` objects and their gameplay logic

`UCadenceConductor` - concrete `UGameInstanceSubsystem` that control turn and phase flow, ticks all registered UCadenceSystem in order. 

### Responsibilities

| System                  | Owns                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| `UTableSubsystem`       | Data only — Board zones, Hand, Pool, Graveyard, Globals, delegates                               |
| `UEventSubsystem`       | Event Cadence management — activate, trigger, resolve, chain                                     |
| `UStorySubsystem`       | Story Cadence management — placement, commit, tick, resolve, expire                              |
| `UTurnManagerSubsystem` | Phase pipeline (`UCadenceConductor`) — orchestrates all subsystems in order, drains action queue |

---

## 2. Directory Structure

```
Source/
	Cadence/
	    CadenceConductor.h/.cpp  # Concrete turn manager
	    CadenceSystem.h/.cpp  # Reusable abstract base
	    Cadence.h                     # ICadence interface
	KingsMen/
	    Cards/
	      CardDefinition.h/.cpp           # UDataAsset — card template
	      CardInstance.h/.cpp             # UObject — live card state
	      CardTag.h                       # FCardTag struct (FGameplayTag + int32)
	    Events/
	      EventDefinition.h/.cpp          # UDataAsset — event template
	      EventInstance.h/.cpp            # UObject — live event state
	      EventCondition.h/.cpp           # Abstract base for trigger conditions
	      EventAction.h/.cpp              # Abstract base for actions
	    Stories/
	      StoryDefinition.h/.cpp          # UDataAsset — Story template
	      StoryInstance.h/.cpp            # UObject — live Story state
	      StorySlotDefinition.h           # FStorySlotDefinition struct (immutable)
	      StorySlotState.h                # FStorySlotState struct (runtime)
	    StatChecks/
	      StatCheck.h/.cpp                # UStatCheck — one check definition (concrete)
	      StatCheckResult.h/.cpp          # UStatCheckResult — per-check result branch
	      StatCheckSession.h/.cpp         # Runtime session — owns roll state + reroll pool
	      StatCheckOutcome.h              # FStatCheckOutcome struct
	      RollModifier.h/.cpp             # Abstract base URollModifier
	      PendingRollData.h               # FPendingRollData struct (delegate payload)
	      RollModifiers/
	        RollMod_CardInSlot.h/.cpp
	        RollMod_PriorCheckTier.h/.cpp
	        RollMod_GlobalVariable.h/.cpp
	        RollMod_CardHasTag.h/.cpp
	        RollMod_TagValue.h/.cpp
	
		Table/
		    TableSubsystem.h/.cpp           # UGameInstanceSubsystem — pure data layer
		    TurnManagerSubsystem.h/.cpp     # UCadenceConductor — phase pipeline
		    EventSubsystem.h/.cpp           # UCadenceSystem subclass
		    StorySubsystem.h/.cpp           # UCadenceSystem subclass
		    DefinitionRegistry.h/.cpp       # UGameInstanceSubsystem — all definitions
	    Undo/
			UndoSnapshot.h/.cpp             # FUndoSnapshot — serialized board state
		    UndoSubsystem.h/.cpp            # UGameInstanceSubsystem — manages undo stack
	    Serialization/
		    DefinitionBase.h/.cpp           # Shared ToJson/FromJson + source tracking
		    DefinitionTypeRegistry.h/.cpp   # UEngineSubsystem — maps "$type" strings to UClass*
		    ModLoaderSubsystem.h/.cpp       # Runtime JSON mod loading
	    UI/
		    BoardWidget.h/.cpp
		    HandWidget.h/.cpp
		    StoryWidget.h/.cpp
		    CardWidget.h/.cpp
		    StatCheckWidget.h/.cpp          # Dice display, result preview, reroll button
	    Editor/
		    DefinitionSyncChecker.h/.cpp    # UEditorSubsystem — drift detection on startup
		    KingsDefinitionDetails.h     # Read-only detail panel with "Edit Source" button

Content/
	DataAssets/                               # Generated DataAssets — build artifacts
	    Cards/                            # DA_Edwin.uasset ...
	    Stories/                          # DA_PrivateAudience.uasset ...
	    Events/                           # DA_SuccessionCrisis.uasset ...
Data/                             # JSON source files — THE authority
  Cards/                            # card_edwin.json ...
  Stories/                          # story_private_audience.json ...
  Events/                           # event_succession_crisis.json ...
Mods/                                 # Runtime mod directory — JSON only, no engine required
```

---

## 3. Card System

Cards are the core representation of player's resources. Their mechanics and attributes are represented through tag-value pairs, and equipment. 
Cards are created from immutable definition template, and managed at runtime through instances. Serialization will record both the template, and runtime changes in rarity level, tags, and equipments. 



---

### 3a. UCardDefinition (Data Asset)

```cpp
UCLASS(BlueprintType)
class UCardDefinition : public UKingsDefinitionBase
{
    GENERATED_BODY()
public:
	
    UPROPERTY(EditAnywhere) FText                  CardName;
    UPROPERTY(EditAnywhere) FText                  CardText;
	// Categorical — hierarchical queries, slot validation 
	UPROPERTY(EditAnywhere) FGameplayTagContainer Tags; 
	
	// Numeric — rarity level, stat values, other number mechanics, anything that gets summed
	UPROPERTY(EditAnywhere) TMap<FGameplayTag, int32> Stats;
	
    // Equipment tags are seperated out for easier authoring, but are merged together in instance. 
    UPROPERTY(EditAnywhere) FGameplayTagContainer   EquipmentSlots;    

	// JSON Serailization
    TSharedPtr<FJsonObject> ToJson() const override; // for json export 
    bool FromJson(const TSharedPtr<FJsonObject>& JsonObject) override; // for json export
};
```

All card's logic are defined through tags, and how other system react / use the tags. 
Not only rairty, type, and attributes are expressed through tags, behaviours such as expiration and equipment. Only equipment slots are maintained separately from tags. 

---

### 3c. UCardInstance (Runtime Object)

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
    UPROPERTY(BlueprintReadWrite) TArray<FCardTag> Tags;
    UPROPERTY(BlueprintReadWrite) TMap<FGameplayTag, TObjectPtr<UCardInstance>> EquippedCards;
    UPROPERTY(BlueprintReadWrite) int32 RemainingLifespan;
    UPROPERTY(BlueprintReadOnly)  bool  bIsDestroyed = false;

    // --- Cached Tag Values ---
    // Rebuilt when tags change or equipment changes. Includes own tags + all equipment bonuses.
    // Keyed by FGameplayTag for O(1) stat lookups. Call InvalidateTagCache() after mutations.
    UPROPERTY(Transient) TMap<FGameplayTag, int32> CachedTagValues;

    static UCardInstance* CreateFromDefinition(UObject* Outer, UCardDefinition* Def);

    bool  HasTag(FGameplayTag Tag) const;
    int32 GetTagValue(FGameplayTag Tag) const;     // returns 0 if absent; reads from cache
    int32 SumTagValues(FGameplayTag Tag) const;    // returns cached value (own + equipment)

    void  InvalidateTagCache();                    // rebuilds CachedTagValues from Tags + EquippedCards
    void  RebuildTagCache();

    bool Equip(UCardInstance* Card, FGameplayTag SlotTag);
    void Unequip(FGameplayTag SlotTag);
    bool CanEquip(UCardInstance* Card, FGameplayTag SlotTag) const;

    // Called by UTurnManagerSubsystem each StartOfTurn.
    void TickLifespan();
};
```

**Notes**
**`TWeakObjectPtr` for `Definition`:**
`TWeakObjectPtr` holds a reference that does not prevent garbage collection. If the underlying `UCardDefinition` DataAsset is unloaded — for example during a hot-reload of mod definitions, or if the definition registry replaces a shipped definition with a mod override — the weak pointer gracefully becomes null rather than keeping a stale definition alive in memory. This avoids two problems: (1) memory leaks where replaced definitions are never GC'd because instances still hold strong references, and (2) silent bugs where instances reference an outdated definition after a mod override. Code that reads `Definition` should null-check it (in practice, definitions are always valid during normal gameplay, but the weak pointer makes the contract explicit and safe against edge cases).

**Cached Stat values.** `CachedStatValues` is a `TMap<FGameplayTag, int32>` that aggregates the card's own Stats plus all equipped card bonuses. It is rebuilt by `RebuildStatCache()` whenever stats or equipment change. All stat queries — `GetTagValue()`, `SumTagValues()` — read from the cache for O(1) lookups, eliminating repeated linear scans through tag arrays and equipment trees during stat checks.

---

### 3d. Example Cards

```
Edwin
  Tags: 
  Character.Main, Character.Gender.Male, Character.Human.Commoner
  Stats: 
  Card.Rarity: 2, Stat.Sociability:5, Stat.Prowess:3, Stat.Poise:5, Stat.scholarship:2, Stat.Subterfuge:3, Mechanic.Reroll: 2
  Equipment slots: Slot.Weapon, Slot.Accessory, Slot.Accessory, Slot.Attire, Slot.Animal

Inside Information
  Tags: Type.Consumable.Advantage 
  Stats:
  Card.Rarity: 2, Stat.Diplomacy:4, Mechanic.Reroll:2

Diamond Necklace
  Tags: Type.Equipment.Accessory,
  Stats:
  Card.Rarity: 2, Stat.Poise:2
  Equipment slots: Slot.Adornment

Hyacinth Flower
  Tags: Type.Equipment.Adornment, Trait.FreshFlower, 
  Stats:
  Mechanic.Reroll:1, Card.Rarity: 1, Expire: 2 (destroyed after 2 turns)

Gold Coin
  Tags: Type.Resource.Gold
  Card.Rarity: 3

Cracked Diamond
  Tags: Type.Resource.Gem, Slot.Adornment
  Stats:
  Card.Rarity: 4
```

---

## 4. Cadence System

Events and Stories share the same structural skeleton: a collection of stateful objects with discrete states, a per-phase tick that advances state, and actions that fire on transitions. This is extracted into a reusable **Cadence** pattern that both `UEventSubsystem` and `UStorySubsystem` extend.

This pattern is reusable across future projects. Any system with stateful objects that tick through a phase pipeline — buffs, quests, faction agendas, contracts — can extend `UCadenceSystem` with minimal modification.

---

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
    virtual bool Tick(FGameplayTag Phase) = 0;

    // Unique instance ID for logging, save/load, and delegate payloads.
    virtual FName GetInstanceID() const = 0;

    // Definition ID for cross-referencing.
    virtual FName GetDefinitionID() const = 0;
};
```

**Flexible Component.** 

**Phase-filtered ticking.** Rather than iterating all units every phase, `GetRelevantPhases()` lets each unit declare which phases it responds to. The cadence system partitions units by phase at registration time and only ticks the relevant subset. This is O(relevant units per phase) instead of O(all units), which matters as content scales to hundreds of active events.

**Configurable phases.** The phase pipeline is defined as `FGameplayTag` values rather than a hard-coded enum, allowing designers to add custom phases in the editor without C++ changes. See Section 9 for phase configuration.

---

### 4b. UCadenceSystem (Abstract Base)

```cpp
UCLASS(Abstract)
class UCadenceSystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    // Called by UTurnManagerSubsystem each phase.
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
    virtual void PrePhase(FGameplayTag Phase)  {}
    virtual void PostPhase(FGameplayTag Phase) {}
};
```

`TickPhase` captures each unit's state before calling `Tick()`. If the state changes, it broadcasts `OnUnitStateChanged`. Units returning `true` are removed and `OnUnitRemoved` is broadcast. Subclasses add typed, domain-specific delegates on top, wrapping the base delegates with fully-typed payloads.

---

## 5. Event System (Cadence)

Events are the **invisible engine** of the board. They never accept direct player input. Their role is to activate Stories, create cards, modify globals, and chain into further events — driving the game's narrative state machine in the background.

### 5a. Why a State Machine?

The three states give precise control over _when_ an event acts, not just _whether_ it acts:

- **Activated** — listening for trigger conditions. The processor only evaluates conditions for Activated events, keeping the check set small.
- **Triggered** — conditions met; the resolution pipeline processes only Triggered events this phase.
- **Resolved** — prevents double-firing within a phase. Repeatable events reset to Activated after resolving; one-shot events are deactivated.

**Event ID uniqueness on the Board.** No two event instances with the same `DefinitionID` may exist on the Board simultaneously. When `ActivateEvent()` is called, the subsystem first checks whether an instance of that definition is already active. If so, the activation is silently ignored (no-op). This eliminates cascading duplicate events and makes it structurally impossible for the same event to fire more than once at a time.

**Chaining rules.** When Event A's action activates Event B, Event B enters `Activated` this phase but won't `Trigger` until the next eligible phase check. `EEventTriggerPhase::Immediate` means "evaluate at the next phase boundary regardless of which phase that is" — it does **not** mean "evaluate right now within the current action drain." This ensures deterministic ordering and prevents unbounded recursive chains.

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

---

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
// UAction_ModifyGlobal     — changes a global variable by delta or set
// UAction_DestroyCard      — sends a card to the Graveyard
```

---

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

---

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

    // ICadence
    FGameplayTag GetState() const override;
    FGameplayTagContainer GetRelevantPhases() const override;
    FName GetInstanceID() const override { return InstanceID; }
    FName GetDefinitionID() const override { return Definition->DefinitionID; }
    bool  Tick(FGameplayTag Phase, UTableSubsystem* Table) override;

    bool CheckTrigger(const UTableSubsystem* Table) const;
    void ExecuteActions(UTableSubsystem* Table);

    // Repeatable events reset to Activated; one-shot events return true (remove).
    void Resolve(UTableSubsystem* Table);
};
```

---

### 5e. UEventSubsystem


```cpp
UCLASS()
class UEventSubsystem : public UCadenceSystem
{
    GENERATED_BODY()
public:
    // Activates an event. Returns nullptr (no-op) if an instance with the same
    // DefinitionID is already active on the Board. This enforces event uniqueness.
    UEventInstance* ActivateEvent(UEventDefinition* Def);

    void            DeactivateEvent(UEventInstance* Event);

    // Returns true if an event with this DefinitionID is currently active.
    bool IsEventActive(FName DefinitionID) const;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEventActivated, UEventInstance*, Event);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnEventTriggered, UEventInstance*, Event);
    UPROPERTY(BlueprintAssignable) FOnEventActivated OnEventActivated;
    UPROPERTY(BlueprintAssignable) FOnEventTriggered OnEventTriggered;

protected:
    // Tracks active DefinitionIDs for O(1) uniqueness checks.
    TSet<FName> ActiveDefinitionIDs;

    // PostPhase collects actions from newly Triggered events and
    // routes them to UTurnManagerSubsystem's action queue.
    void PostPhase(FGameplayTag Phase, UTableSubsystem* Table) override;
};
```

---

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

---

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
    UPROPERTY(EditAnywhere) EStorySlotType     SlotType;

    // All listed tags must be present on every card placed in this slot.
    UPROPERTY(EditAnywhere) FGameplayTagContainer RequiredTags;

    // For named-card slots (e.g. "Annual Report of the Great Counties").
    UPROPERTY(EditAnywhere) FName              RequiredCardID;   // DefinitionID, empty = any matching card

    // --- Stacking ---
    // If true, multiple cards can be placed in this slot (e.g. 10 Gold Coins).
    UPROPERTY(EditAnywhere) bool               bAllowStack = false;

    // Maximum number of cards allowed in this slot. 0 = unlimited.
    // Only meaningful when bAllowStack is true; ignored otherwise.
    UPROPERTY(EditAnywhere) int32              MaxStackSize = 0;

    // Minimum number of cards required in this slot for it to count as "filled."
    // For non-stacking slots this is implicitly 1 (any card present = filled).
    // For stacking slots, e.g. MinStackSize = 10 means "at least 10 gold coins."
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

**Stacking resolves the resource ambiguity.** Rather than encoding "10 gold" as a tag value on a single card, the player places 10 individual Gold Coin cards into a stacking slot with `MinStackSize = 10`. This keeps cards as discrete atomic units, makes resource spending visible and interactive, and works naturally with the existing card system (each Gold Coin has its own lifespan, tags, and can be individually destroyed or recalled).

---

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

    // --- On-Expire Behavior ---
    // Actions executed when the Story expires without being committed.
    UPROPERTY(EditAnywhere, Instanced)
    TArray<TObjectPtr<UEventAction>> OnExpire;

    // Default behavior: all non-consumable cards placed in the Story are returned to Hand.
    // Consumable cards (tag Type.Consumable) are destroyed. Override actions in OnExpire
    // can destroy, move, or transform cards as needed.
    UPROPERTY(EditAnywhere) bool bReturnCardsOnExpire = true;

    // --- On-End Behavior ---
    // Default end-of-Story behavior (after all checks resolve):
    // Non-consumable cards are returned to Hand. Consumable cards are destroyed.
    // Check result actions may override this per-card (e.g. Action_DestroyCard).
    UPROPERTY(EditAnywhere) bool bReturnCardsOnEnd = true;

    TSharedPtr<FJsonObject> ToJson() const override;
    bool FromJson(const TSharedPtr<FJsonObject>& JsonObject) override;
};
```

**No Story-level success/failure actions.** All outcome logic is defined per-check via result branches (see Section 7). This makes each check self-contained: its conditions determine what happens, and the Story is simply the container. This avoids the ambiguity of mapping mixed check outcomes to a single "Story success" — designers define exactly what each check result does.

---

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

    // --- Cached Tag Values ---
    // Aggregated tag values across all cards placed in all slots (including equipment).
    // Rebuilt when cards are placed or removed. Used for stat check queries.
    UPROPERTY(Transient) TMap<FGameplayTag, int32> CachedSlotTagValues;

    // ICadence
    FGameplayTag GetState() const override;
    FGameplayTagContainer GetRelevantPhases() const override;
    FName GetInstanceID() const override { return InstanceID; }
    FName GetDefinitionID() const override { return Definition->DefinitionID; }
    bool  Tick(FGameplayTag Phase, UTableSubsystem* Table) override;

    // Slot management — called by UStorySubsystem on behalf of player input.
    bool CanPlaceCard(UCardInstance* Card, FName SlotID) const;
    bool PlaceCard(UCardInstance* Card, FName SlotID);
    void RemoveCard(UCardInstance* Card, FName SlotID);  // removes specific card from slot (supports stacks)
    void RemoveAllCards(FName SlotID);
    bool AreRequiredSlotsFilled() const;

    // Slot queries.
    const FStorySlotState* GetSlotState(FName SlotID) const;
    const FStorySlotDefinition* GetSlotDefinition(FName SlotID) const;
    int32 GetSlotCardCount(FName SlotID) const;

    // Aggregate stat from cards in specified slots (or all if LimitToSlots is empty).
    // Reads from CachedSlotTagValues for efficiency.
    int32 GetTotalStatValue(FGameplayTag StatTag, const TArray<FName>& LimitToSlots = {}) const;

    void InvalidateSlotTagCache();
    void RebuildSlotTagCache();

    // Player commits the Story. All required slots must be filled.
    // TurnsToResolve == 0: calls Resolve() immediately.
    // TurnsToResolve  > 0: sets State = Resolving, locks all placed cards.
    bool Commit(UTableSubsystem* Table);

    void Resolve(UTableSubsystem* Table);  // creates UStatCheckSession, fires check result actions

    // Returns cards to Hand (non-consumable) or destroys them (consumable),
    // then fires OnExpire actions.
    void Expire(UTableSubsystem* Table);

    // Called after resolution or expiry to return cards per Story definition rules.
    void ReturnOrDestroyCards(UTableSubsystem* Table);
};
```

---

### 6e. UStorySubsystem

```cpp
UCLASS()
class UStorySubsystem : public UCadenceSystem
{
    GENERATED_BODY()
public:
    UStoryInstance* AddStory(UStoryDefinition* Def);
    void            RemoveStory(UStoryInstance* Story);

    // All three validate that the game is in Planning phase before acting.
    bool PlaceCard(UCardInstance* Card, UStoryInstance* Story, FName SlotID);
    bool RecallCard(UCardInstance* Card, UStoryInstance* Story, FName SlotID);
    bool CommitStory(UStoryInstance* Story, UTableSubsystem* Table);

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStoryCommitted, UStoryInstance*, Story);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnStoryResolved,  UStoryInstance*, Story);
    UPROPERTY(BlueprintAssignable) FOnStoryCommitted OnStoryCommitted;
    UPROPERTY(BlueprintAssignable) FOnStoryResolved  OnStoryResolved;

protected:
    // PrePhase(StartOfTurn): checks deadlines, expires overdue Stories,
    // returns/destroys cards from expired Stories.
    void PrePhase(FGameplayTag Phase, UTableSubsystem* Table) override;
};
```

---

## 7. Stat Check System

Stat checks evaluate the outcomes of a Story. Each check is **self-contained**: it defines its own stat sources, dice behavior, and an ordered array of **result branches**, each with a condition, narrative text, and actions. There is no Story-level success/failure — all logic lives in the checks.

### 7a. The Roll Formula

```
final_result = Aggregate(n × d20) + SummedStat
```

Where:

- `n = 1 + Σ(modifier.Evaluate())` — base pool is always 1; modifiers contribute bonus dice when their conditions are met
- `Aggregate = Max` (advantage, default) or `Min` (disadvantage, punishing)
- `SummedStat` = sum of named tags across cards in specified slots, including equipment
- `final_result` is then evaluated against the check's result branches in order

Static checks skip dice entirely: evaluate `SummedStat` against result branch conditions. No roll, no modifiers, no rerolls.

---

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

---

### 7c. ERollAggregate

```cpp
UENUM(BlueprintType)
enum class ERollAggregate : uint8
{
    Max,    // Take highest die — advantage (default)
    Min,    // Take lowest die — disadvantage, punishing
};
```

---

### 7d. URollModifier — Attachable Dice Bonus Conditions

Roll modifiers use the same `Abstract` + `EditInlineNew` + `Instanced` pattern as `UEventCondition` and `UEventAction`. Each modifier evaluates a condition at roll time and contributes bonus dice to the pool if met.

```cpp
UCLASS(Abstract, EditInlineNew, BlueprintType, Blueprintable)
class URollModifier : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) int32 BonusDice = 1;
    UPROPERTY(EditAnywhere) FText DisplayLabel;

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
// Bonus dice if a card matching RequiredTags is placed in SlotID.
UCLASS(EditInlineNew)
class URollMod_CardInSlot : public URollModifier
{
    UPROPERTY(EditAnywhere) FName                  SlotID;
    UPROPERTY(EditAnywhere) FGameplayTagContainer  RequiredTags;
};

// Bonus dice if a previous check in this session achieved a minimum tier.
UCLASS(EditInlineNew)
class URollMod_PriorCheckTier : public URollModifier
{
    UPROPERTY(EditAnywhere) int32              CheckIndex;
    UPROPERTY(EditAnywhere) FName              MinimumResultID;  // result branch ID from earlier check
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

// Bonus dice equal to the value of a named tag on cards in SlotID.
UCLASS(EditInlineNew)
class URollMod_TagValue : public URollModifier
{
    UPROPERTY(EditAnywhere) FName              SlotID;
    UPROPERTY(EditAnywhere) FGameplayTag       TagName;
};
```

---

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
    // Unique ID for this result branch within the check (e.g. "critical_success", "failure").
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

**Branch evaluation is first-match.** Branches are tested in array order; the first one whose conditions all pass is selected. This means designers should order branches from most specific (highest threshold) to least specific (fallback). The last branch can have no conditions to serve as a guaranteed fallback.

---

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

---

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

---

### 7h. FPendingRollData — Delegate Payload Struct

```cpp
// Wraps roll data for Blueprint-safe delegate broadcasting.
// Used instead of passing raw TArray<int32> by const reference in dynamic delegates.
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

---

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
    UPROPERTY() int32 TotalRerolls;        // summed from all placed cards' "Mechanic.Reroll" tags at creation
    UPROPERTY() int32 RemainingRerolls;    // decremented as player spends them
    UPROPERTY() int32 CurrentCheckIndex = 0;
    UPROPERTY() TArray<FStatCheckOutcome> Outcomes;

    // Resolves the current check.
    //   Static:   resolves immediately, auto-advances without UI input.
    //             NOTE: UI should still display a brief animation beat (~200ms)
    //             before advancing to give the player visual feedback that
    //             a check was evaluated. The session fires the delegate and
    //             the UI is responsible for the pause timing.
    //   DiceRoll: rolls dice, stores pending result, broadcasts to UI.
    void ResolveCurrentCheck(UTableSubsystem* Table);

    // Player spends 1 reroll on the current check.
    bool SpendReroll();

    // Player accepts current result. Advances to next check.
    void ConfirmCurrentCheck(UTableSubsystem* Table);

    bool IsComplete() const;

    // Called after all checks are confirmed.
    // Fires each check's selected result branch actions.
    // Queues all resulting actions into UTurnManagerSubsystem's ActionQueue.
    void Finalize(UTableSubsystem* Table);

    bool  CanReroll() const;
    FName GetPendingResultID() const;

    // Broadcast after every roll and reroll — UI updates dice display and result preview.
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPendingRollReady, FPendingRollData, RollData);
    UPROPERTY(BlueprintAssignable) FOnPendingRollReady OnPendingRollReady;

private:
    FPendingRollData PendingRoll;

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

---

### 7j. JSON Schema for a Check with Result Branches

```json
{
  "check_id": "r1",
  "stat_sources": [
    { "Stat.Sociability": ["s1"] },
    { "Stat.Subterfuge": [] }
  ], //by default, stat check source stat bonus from all cards placed in a story, unless s slot id like s1 is specified in the array
  "check_type": "Roll",
  "aggregate": "Max",
  "dice_mod": [
  ], // any bonus dice to roll based on circunstances
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

---

### 7k. Full Story JSON Example

```json
{
  "id": "story.private_audience",
  "type": "StoryDefinition",
  "name": "Private Audience with the King",
  "must_resolve_by_turn": 3,
  "turns_to_resolve": 2,
  "persistent": false,
  "return_cards_on_end": true,
  "return_cards_on_expire": true,
  "slots": [
    { "slot_id": "main_character", "type": "Required",
      "required_tags": ["Type.Character.Main"],
      "allow_stack": false },
    { "slot_id": "companion", "type": "Optional",
      "required_tags": ["Type.Character"],
      "allow_stack": false },
    { "slot_id": "gift", "type": "Required",
      "required_tags": ["Resource.Gold"],
      "allow_stack": true, "min_stack_size": 10, "max_stack_size": 0 },
    { "slot_id": "intel", "type": "Required",
      "required_card": "card.annual_report",
      "allow_stack": false },
    { "slot_id": "consumable", "type": "Optional",
      "required_tags": ["Type.Consumable"],
      "allow_stack": false }
  ],
  "stat_checks": [
    {
      "check_id": "audience_diplomacy",
      "stat_sources": [{ "stat_tag": "Stat.Diplomacy", "limit_to_slots": [] }],
      "check_type": "DiceRoll",
      "aggregate": "Max",
      "dice_modifiers": [
        { "$type": "RollMod_CardInSlot", "slot_id": "companion",
          "required_tags": ["Type.Character"], "bonus_dice": 1,
          "display_label": "Companion present" },
        { "$type": "RollMod_GlobalVariable", "key": "gv.RoyalFavor",
          "operator": ">=", "value": 50, "bonus_dice": 1,
          "display_label": "High royal favor" }
      ],
      "results": [
        {
          "result_id": "critical_success",
          "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 20 }],
          "result_title": "Royal Favor",
          "result_text": "The King is deeply impressed by your counsel and generosity...",
          "actions": [
            { "$type": "Action_ModifyGlobal", "key": "gv.RoyalFavor", "delta": 30 },
            { "$type": "Action_ModifyGlobal", "key": "gv.KingdomStability", "delta": 10 }
          ]
        },
        {
          "result_id": "success",
          "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 15 }],
          "result_title": "A Productive Meeting",
          "result_text": "The King listens attentively and nods along...",
          "actions": [
            { "$type": "Action_ModifyGlobal", "key": "gv.RoyalFavor", "delta": 20 },
            { "$type": "Action_ModifyGlobal", "key": "gv.KingdomStability", "delta": 5 }
          ]
        },
        {
          "result_id": "partial",
          "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 10 }],
          "result_title": "Polite but Distant",
          "result_text": "The King acknowledges your words but seems distracted...",
          "actions": [
            { "$type": "Action_ModifyGlobal", "key": "gv.RoyalFavor", "delta": 10 }
          ]
        },
        {
          "result_id": "failure",
          "conditions": [{ "check_id": "audience_diplomacy", "operator": ">=", "value": 1 }],
          "result_title": "A Cold Reception",
          "result_text": "The King barely acknowledges your presence...",
          "actions": []
        },
        {
          "result_id": "critical_failure",
          "conditions": [],
          "result_title": "The King's Displeasure",
          "result_text": "Your words fall flat. The King's expression darkens...",
          "actions": [
            { "$type": "Action_ActivateEvent", "event_id": "event.king_displeasure" }
          ]
        }
      ]
    }
  ],
  "on_expire": [
    { "$type": "Action_ModifyGlobal", "key": "gv.KingdomStability", "delta": -5 }
  ]
}
```

---

## 8. Table Subsystem

`UTableSubsystem` is the **pure data layer** — it owns all game state collections and exposes delegates for the UI. It contains no logic; all logic lives in `UTurnManagerSubsystem`, `UEventSubsystem`, and `UStorySubsystem`.

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
    // Maintained automatically via delegates. Maps tags to cards that have them.
    // Used for bulk queries like "find all cards with Stat.Diplomacy >= 3".
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

    // --- Queries ---
    TArray<UCardInstance*> GetCardsWithTag(FGameplayTag Tag) const;     // uses TagIndex
    UStoryInstance*        FindStoryByID(FName DefinitionID) const;
    UStoryInstance*        FindStoryByInstanceID(FName InstanceID) const;

    // --- Tag Index Maintenance ---
    void RegisterCardInIndex(UCardInstance* Card);
    void UnregisterCardFromIndex(UCardInstance* Card);
};
```

---

## 9. Turn Manager Subsystem

`UTurnManagerSubsystem` owns the phase pipeline and orchestrates all other subsystems. It is the only place phase transitions occur. All player input methods here are the single path by which UI affects game state.

### 9a. Configurable Phase Pipeline

Phases are defined as `FGameplayTag` values rather than a hard-coded enum. The default phase sequence is configured as a `TArray<FGameplayTag>` on the Turn Manager, editable in Blueprint. This allows designers to insert custom phases (e.g. "Phase.Diplomacy", "Phase.Espionage") without C++ changes.

```cpp
// Default phase tags (registered in Project Settings → Gameplay Tags):
//   Phase.StartOfTurn
//   Phase.Planning
//   Phase.Execution
//   Phase.EventResolution
//   Phase.Cleanup
//   Phase.EndOfTurn
```

---

### 9b. UTurnManagerSubsystem

```cpp
UCLASS()
class UTurnManagerSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadOnly) int32          CurrentTurn = 1;
    UPROPERTY(BlueprintReadOnly) FGameplayTag   CurrentPhase;

    // Ordered phase sequence. Editable in Blueprint to add/reorder phases.
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TArray<FGameplayTag> PhaseSequence;

    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPhaseChanged, FGameplayTag, NewPhase);
    UPROPERTY(BlueprintAssignable) FOnPhaseChanged OnPhaseChanged;

    // Player input — all validated against CurrentPhase before acting.
    UFUNCTION(BlueprintCallable) void AdvancePhase();
    UFUNCTION(BlueprintCallable) bool PlaceCardInStory(UCardInstance* Card, UStoryInstance* Story, FName SlotID);
    UFUNCTION(BlueprintCallable) bool RecallCardFromStory(UCardInstance* Card, UStoryInstance* Story, FName SlotID);
    UFUNCTION(BlueprintCallable) bool CommitStory(UStoryInstance* Story);

    // Called by UEventSubsystem and UStatCheckSession to enqueue
    // actions for processing during EventResolution.
    void QueueActions(const TArray<UEventAction*>& Actions);

    // --- Safety Valve ---
    // Maximum actions drained per EventResolution phase.
    // If exceeded, remaining actions overflow to the next turn's EventResolution.
    // A warning is logged when the limit is hit.
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxActionsPerPhase = 1000;

private:
    TArray<TObjectPtr<UEventAction>> ActionQueue;
    int32 CurrentPhaseIndex = 0;

    void ExecutePhase(FGameplayTag Phase);
    void Phase_StartOfTurn();
    void Phase_Execution();
    void Phase_EventResolution();
    void Phase_Cleanup();

    UTableSubsystem*  GetTable()  const;
    UEventSubsystem*  GetEvents() const;
    UStorySubsystem*  GetStories() const;
    UUndoSubsystem*   GetUndo()   const;
};
```

---

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
  ├─ PlaceCardInStory() / RecallCardFromStory()
  │    → UStorySubsystem validates phase, delegates to UStoryInstance
  │
  ├─ CommitStory()
  │    → UStorySubsystem validates all required slots filled
  │    │
  │    ├─ [TurnsToResolve == 0] → UStoryInstance::Commit() → Resolve() fires immediately
  │    │    ├─ UStatCheckSession created; checks resolved sequentially with player input
  │    │    ├─ Per-check result branch actions → QueueActions()
  │    │    ├─ ReturnOrDestroyCards() per Story definition rules
  │    │    └─ Story removed from Board (bPersistent → reset to Available instead)
  │    │
  │    └─ [TurnsToResolve  > 0] → State = Resolving, placed cards locked
  │         └─ UI shows countdown; Story is no longer interactive this turn
  │
  └─ AdvancePhase() → player ends turn

Execution
  └─ UStorySubsystem::TickPhase(Phase.Execution):
       For each Story in state Resolving:
         ├─ Decrement TurnsRemaining
         └─ TurnsRemaining == 0 → Resolve() → UStatCheckSession → QueueActions()

EventResolution
  ├─ Drain ActionQueue — execute up to MaxActionsPerPhase actions in order
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
- The `ActionQueue` is the single handoff point between resolution and consequence. Nothing executes mid-resolution — all effects are deferred to EventResolution, keeping phase behavior predictable.
- **Card return on Story end.** After resolution, non-consumable cards are returned to Hand by default. Consumable cards (`Type.Consumable` tag) are destroyed. Individual check result actions can override this (e.g. `Action_DestroyCard` to sacrifice a character).

---

## 1 Undo System

The Undo system captures full board snapshots at the start of each turn, allowing the player to revert to the state at the end of the previous turn (before any current-turn mutations). Up to 5 snapshots are stored.

```cpp
USTRUCT()
struct FUndoSnapshot
{
    GENERATED_BODY()

    UPROPERTY() int32 TurnNumber;

    // Serialized state of all subsystems at the start of this turn.
    // Captured before StartOfTurn mutations (lifespan ticks, expirations, event triggers).
    UPROPERTY() TArray<uint8> SerializedTableState;
    UPROPERTY() TArray<uint8> SerializedEventState;
    UPROPERTY() TArray<uint8> SerializedStoryState;
};

UCLASS()
class UUndoSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    // Called at the very beginning of StartOfTurn, before any mutations.
    void CaptureSnapshot(UTableSubsystem* Table, UEventSubsystem* Events, UStorySubsystem* Stories);

    // Reverts to the snapshot of a prior turn. Restores all subsystem state.
    // Returns false if the requested turn is not available.
    bool UndoToTurn(int32 TargetTurn, UTableSubsystem* Table,
                    UEventSubsystem* Events, UStorySubsystem* Stories);

    // Returns the list of available undo turns.
    TArray<int32> GetAvailableUndoTurns() const;

    bool CanUndo() const { return Snapshots.Num() > 0; }

private:
    // Ring buffer of up to MaxSnapshots.
    static constexpr int32 MaxSnapshots = 5;
    TArray<FUndoSnapshot> Snapshots;
};
```

**Design notes:**

- Snapshots are captured at the **start** of each turn, which means undoing returns to the state at the end of the previous turn (before the current turn's StartOfTurn phase ran).
- The undo operation fully replaces all subsystem state — Board, Hand, Pool, Graveyard, Globals, active Events, active Stories.
- Undo is available during Planning phase only. It is not available mid-resolution.
- Save/load design (TBD) will need to serialize the undo stack as well.

---

## 11. UI Architecture

Widgets are **purely presentational.** They hold read-only references to runtime instances and receive updates through delegates. They never directly modify game state — all player actions route through `UTurnManagerSubsystem`.

```
UBoardWidget
  ├─ Binds to: UTableSubsystem::OnStoryAdded, OnStoryRemoved
  ├─ Contains: TArray<UStoryWidget*>
  └─ Does not hold game state

UStoryWidget
  ├─ Holds a read-only ref to UStoryInstance
  ├─ Displays: slots (with stack counts), placed cards, turn countdown, stat preview, committed state
  └─ Dispatches:
       OnCardDropped     → UTurnManagerSubsystem::PlaceCardInStory()
       OnRecallClicked   → UTurnManagerSubsystem::RecallCardFromStory()
       OnCommitClicked   → UTurnManagerSubsystem::CommitStory()

UHandWidget
  ├─ Binds to: UTableSubsystem::OnCardAddedToHand, OnCardRemovedFromHand
  └─ Contains: TArray<UCardWidget*>

UCardWidget
  ├─ Holds a read-only ref to UCardInstance
  ├─ Renders: name, tags, equipment slots, rarity, lifespan
  └─ Is draggable via UDragDropOperation

UStatCheckWidget
  ├─ Created by UStoryWidget when a UStatCheckSession becomes active
  ├─ Binds to: UStatCheckSession::OnPendingRollReady
  ├─ Displays: all individual d20 values, aggregated roll, result branch preview
  │    Result text: shown from the previewed branch's ResultTitle + ResultText
  │    Reroll button: visible only when UStatCheckSession::CanReroll() == true
  │    Reroll button: hidden entirely for static checks (they auto-advance)
  │    Reroll pool: shown as shared count across the full session
  │    NOTE: For static checks, the widget should display a brief animation
  │    pause (~200ms) before auto-advancing to give the player visual
  │    feedback that a check was evaluated. The session fires the delegate;
  │    the widget is responsible for timing the pause before confirming.
  └─ Dispatches:
       OnRerollClicked   → UStatCheckSession::SpendReroll()
       OnConfirmClicked  → UStatCheckSession::ConfirmCurrentCheck()

UGlobalsWidget
  └─ Binds to: UTableSubsystem::OnGlobalChanged → updates HUD stats

UUndoButton
  └─ Dispatches: OnUndoClicked → UUndoSubsystem::UndoToTurn()
       Enabled only during Planning phase when UUndoSubsystem::CanUndo()
```

---

## 12. JSON Serialization & Modding Pipeline

### 12a. Philosophy: JSON is the Authority

DataAssets are **build artifacts** generated from JSON source files — analogous to compiled binaries from source code. Designers author in JSON; DataAssets are an engine-optimized representation of that source.

This approach:

- Preserves the full benefits of DataAssets: type safety, editor validation, polymorphic UObject references, and visual editing
- Enables modding without the engine: mod authors drop JSON files in `/Mods/`, no UE required
- Produces human-readable, diffable source files in version control
- Keeps the base game and mod content on the same pipeline for consistency

**Mods are JSON-only.** They load at runtime via `UModLoaderSubsystem` and register into `UDefinitionRegistry` alongside shipped content. No drift is possible on the mod side because no DataAsset step exists for mods. Mod security validation is planned for a future revision.

---

### 12b. UKingsDefinitionBase

All definition DataAssets inherit from this base, which adds the JSON interface and source tracking.

```cpp
UCLASS(Abstract)
class UKingsDefinitionBase : public UDataAsset
{
    GENERATED_BODY()
public:
    // Stable string key used in JSON cross-references, mod overrides, and runtime lookups.
    UPROPERTY(EditAnywhere, Category="Source")
    FName DefinitionID;

    // SHA1 hash of the JSON file at last import.
    UPROPERTY(VisibleAnywhere, Category="Source")
    FString SourceJsonHash;

    // Relative path to the JSON source file on disk.
    UPROPERTY(VisibleAnywhere, Category="Source")
    FString SourceJsonPath;

    virtual TSharedPtr<FJsonObject> ToJson() const;
    virtual bool FromJson(const TSharedPtr<FJsonObject>& JsonObject);

    void PostSave(const FObjectPostSaveContext& Context) override;

private:
    bool ExportToJson();
};
```

---

### 12c. Bidirectional Sync

**DataAsset → JSON (on save):**

```
Designer saves DataAsset (Ctrl+S)
  → PostSave fires
  → AutoReimport watcher paused (prevents circular reimport loop)
  → ExportToJson() writes to Content/Source/
  → Write confirmed → SourceJsonHash updated
  → AutoReimport watcher resumed
  → Both DA_Edwin.uasset and card_edwin.json committed together
```

**JSON → DataAsset (on startup or manual reimport):**

```
Engine starts / designer edits JSON externally
  → UDefinitionSyncChecker hashes all JSON source files
  → Hash mismatch detected → auto-reimport triggered
  → DataAsset updated → SourceJsonHash updated
```

**Circular trigger prevention.** Without suppression, exporting JSON would trigger UE's `FAutoReimportManager` file watcher, which would reimport the JSON back into the DataAsset, looping infinitely. The `IgnoreFileChanges` scope guard in `ExportToJson()` breaks this loop.

**Atomic hash update.** `SourceJsonHash` is only updated after confirming the file write succeeded. A failed write leaves the hash mismatched, so `UDefinitionSyncChecker` correctly flags it on next launch.

---

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

Reports are loud and specific:

```
[Kings] DRIFT DETECTED — 2 definition(s) out of sync with JSON source:
  DA_PrivateAudience  stored hash: a3f9...  current JSON hash: 7b2c...  → REIMPORT REQUIRED
  DA_Edwin            source JSON not found at: Content/Source/Cards/card_edwin.json
                      → file may have been moved or deleted
```

---

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

---

### 12f. Polymorphic Type Registry

Conditions, Actions, and RollModifiers all serialize with a `$type` discriminator so the deserializer knows which subclass to instantiate.

The registry uses `UEngineSubsystem` — it needs to be available before any `UGameInstance` exists (e.g. during editor startup and asset import), and it has no per-game-instance state. `UEngineSubsystem` is initialized once with the engine and persists for the entire process lifetime.

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

---

### 12g. Definition Registry (Runtime)

All systems look up definitions by `DefinitionID` through `UDefinitionRegistry`, never by direct UObject pointer. This makes mod overrides transparent.

```cpp
UCLASS()
class UDefinitionRegistry : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    void Initialize(FSubsystemCollectionBase& Collection) override;

    void RegisterCardDef(UCardDefinition* Def);
    void RegisterStoryDef(UStoryDefinition* Def);
    void RegisterEventDef(UEventDefinition* Def);

    UCardDefinition*  FindCard(FName ID)  const;
    UStoryDefinition* FindStory(FName ID) const;
    UEventDefinition* FindEvent(FName ID) const;

private:
    TMap<FName, TObjectPtr<UCardDefinition>>  Cards;
    TMap<FName, TObjectPtr<UStoryDefinition>> Stories;
    TMap<FName, TObjectPtr<UEventDefinition>> Events;

    void LoadShippedDefinitions();
    void LoadMods();
};
```

---

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

Mods are JSON files in `<GameDir>/Mods/`. A mod that registers with an existing `DefinitionID` replaces the shipped definition. All other systems are unaware of the distinction.

---

### 12i. Source Control Conventions

Binary `.uasset` merge conflicts are handled at the **workflow level**, not in code. The industry standard is file locking:

- **Git + Git LFS:** `git lfs lock Content/Data/**/*.uasset` before editing a DataAsset.
- **Perforce:** Exclusive checkout is native.

JSON source files remain freely mergeable as text.

---

### 12j. JSON Schemas

```json
// card_edwin.json
{
  "id": "card.edwin",
  "name": "Edwin",
  "text": "You — a mere common human professor, and now, entrusted by the young King himself, the unlikely Seneschal of the royal council. What awaits you in this whirlpool of power?"
  "tags": ["Character.Main", "Character.Human.Commoner", "Character.Gender.Male"
  ],
  "stats": [
    { "Rarity": 2 },
    { "Stat.Prowess": 3 },
    { "Stat.Sociability": 1 },
    { "Stat.Poise": 4 },
    { "Stat.Scholarship": 5 },
    { "Stat.Subterfuge": 2 },
    { "Stat.Influence": 0 }
  ],
  "equipment_slots": ["Slot.Weapon", "Slot.Accessory", "Slot.Accessory", "Slot.Attire", "Slot.Animal"]
}

// card_HyacinthFlower.json
{
  "id": "card.HyacinthFlower",
  "name": "Hyacinth Flower",
  "text": "Beautiful purple flower. "
  "tags": ["Equipment.Adornment", "Trait.FreshFlower"  ],
  "stats": [
    { "Rarity": 1 },
    { "Mechanic.Reroll": 1 },
    { "Mechanic.Expire": 2 }
  ],
  "equipment_slots": []
}

// event_succession_crisis.json
{
  "id": "event.succession_crisis",
  "type": "EventDefinition",
  "repeatable": false,
  "trigger_phase": "StartOfTurn",
  "conditions": [
    { "$type": "Condition_TurnNumber",     "turn": 5 },
    { "$type": "Condition_GlobalVariable", "key": "gv.KingdomStability",
                                           "operator": "LessThan", "value": 30 }
  ],
  "actions": [
    { "$type": "Action_CreateStory",  "story_id": "story.royal_audit" },
    { "$type": "Action_ModifyGlobal", "key": "gv.CourtTension", "delta": 15 }
  ]
}
```

Full Story JSON example is in Section 7k.

---

## 13. Data Flow Summary

```
JSON source file  →  [Import / PostSave export]  →  UDataAsset
                                                          │
                                              UDefinitionRegistry
                                           (lookup by DefinitionID)
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
                         │         │              PlaceCardInStory()
                         │    UStoryInstance               │
                         │    (Board_Stories)       UStoryInstance::PlaceCard()
                         │                          (supports stacking)
                         │                                │
                         │                       CommitStory()
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
                         └───────────────→  UTurnManagerSubsystem::QueueActions()
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

---

## 14. Implementation Order

|Phase|Milestone|Systems|
|---|---|---|
|1|Card foundation|`FCardTag` (FGameplayTag), `UCardDefinition`, `UCardInstance`, `CachedTagValues`, basic `UTableSubsystem` Hand ops|
|2|Table & globals|Full `UTableSubsystem`, `UDefinitionRegistry`, Global Variables (FGameplayTag keys), TagIndex|
|3|Cadence base|`ICadence`, `UCadenceSystem`, phase-filtered ticking|
|4|Story placement|`UStoryDefinition`, `UStoryInstance`, `FStorySlotDefinition`/`FStorySlotState`, slot stacking, `EStoryState`, `UStorySubsystem`|
|5|Turn loop|`UTurnManagerSubsystem`, configurable phase sequence, Planning → Execution pipeline|
|6|Stat checks — static|`UStatCheck` (concrete), `FStatScope`, `UStatCheckResult`, `UStatCheckSession` static path|
|7|Stat checks — dice|`ERollAggregate`, `URollModifier` base + shipped subclasses, d20 pool resolution, `FPendingRollData`|
|8|Reroll system|`RemainingRerolls`, `SpendReroll()`, `OnPendingRollReady`, `UStatCheckWidget`|
|9|Event backbone|`UEventDefinition`, `UEventCondition`/`UEventAction` base classes, `UEventSubsystem` with uniqueness enforcement|
|10|Event content|Concrete Condition/Action subclasses, `UDefinitionTypeRegistry` (UEngineSubsystem)|
|11|Event pipeline|`UEventInstance`, full cadence integration, chaining via `ActionQueue`, `MaxActionsPerPhase` overflow|
|12|Undo system|`FUndoSnapshot`, `UUndoSubsystem`, snapshot capture/restore|
|13|UI|Board, Hand, Card, Story, StatCheck widgets; drag-drop; all delegate bindings; static check animation pause|
|14|Serialization|`UKingsDefinitionBase`, `ToJson`/`FromJson` per type|
|15|Editor tooling|`UDefinitionSyncChecker`, read-only detail panel, `PostSave` auto-export|
|16|Mod loading|`UModLoaderSubsystem`, mod override support in `UDefinitionRegistry`|
|17|Save/Load|TBD — will be designed separately|
|18|First content|Edwin card, Private Audience Story, default event collection, first playable loop|