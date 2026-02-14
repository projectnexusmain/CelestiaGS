# Minimal Quest System Implementation Guide

If you want to implement basic Quests (like "Eliminate Players" or "Search Chests") without the complexity of reverse-engineering the entire `SendComplexCustomStatEvent` system and `AthenaObjectiveStatXPTable`, you can use a **Hardcoded Hooking Approach**.

This method involves hooking specific gameplay functions directly and manually awarding XP or Accolades when those events occur.

## 1. Implementing "Eliminate Players" Quest

Instead of relying on the stat event system, you can hook the function that handles player death.

### Target Function: `ClientOnPawnDied`
*   **Purpose:** Called on the server when a player pawn dies.
*   **Location:** In `AFortPlayerControllerAthena`.

### Implementation Logic:
```cpp
// 1. Define the Hook
void ClientOnPawnDied_Hook(AFortPlayerControllerAthena* PC, FFortPlayerDeathReport& DeathReport) {

    // 2. Get Key Actors
    auto VictimPS = (AFortPlayerStateAthena*)PC->PlayerState;
    auto KillerPS = (AFortPlayerStateAthena*)DeathReport.KillerPlayerState;
    auto KillerPawn = (AFortPlayerPawnAthena*)DeathReport.KillerPawn;

    // 3. Verify it's a valid kill (Killer exists, is not the victim, is a player)
    if (KillerPS && KillerPawn && KillerPS != VictimPS) {

        // 4. Update Kill Score
        KillerPS->KillScore++;
        KillerPS->OnRep_Kills();

        // 5. GRANT REWARD (Minimal Quest Logic)
        AFortPlayerControllerAthena* KillerPC = (AFortPlayerControllerAthena*)KillerPS->GetOwner();

        if (KillerPC) {
            // Example: Grant 500 XP for a kill
            KillerPC->XPComponent->MatchXp += 500;
            KillerPC->XPComponent->OnXpUpdated(...);

            // Example: Give specific "Elimination" Accolade
            static auto AccoladeDef = FindObject<UFortAccoladeItemDefinition>("/Game/Athena/Items/Accolades/AccoladeId_012_Elimination.AccoladeId_012_Elimination");
            if (AccoladeDef) {
                XP::GiveAccolade(KillerPC, AccoladeDef);
            }
        }
    }

    // 6. Call Original Function
    return ClientOnPawnDied_Original(PC, DeathReport);
}
```

## 2. Implementing "Search Chests" Quest

Hook the interaction function to detect when a player opens a container.

### Target Function: `ServerAttemptInteract`
*   **Purpose:** Called when a player tries to interact with an object (Door, Chest, Vehicle).
*   **Location:** `UFortControllerComponent_Interaction`

### Implementation Logic:
```cpp
// 1. Define the Hook
void ServerAttemptInteract_Hook(UFortControllerComponent_Interaction* Comp, AActor* ReceivingActor, ...) {

    // 2. Call Original FIRST (to let the interaction happen)
    ServerAttemptInteract_Original(Comp, ReceivingActor, ...);

    // 3. Get the Player Controller
    auto PC = Cast<AFortPlayerControllerAthena>(Comp->GetOwner());
    if (!PC) return;

    // 4. Check what was interacted with
    if (auto BuildingActor = Cast<ABuildingActor>(ReceivingActor)) {

        // 5. Is it a Chest?
        // Note: You might need to check specific classes or tags to distinguish Chests from Doors
        static auto ChestClass = FindObject<UClass>("/Game/Building/ActorBlueprints/Containers/Tiered_Chest_Athena.Tiered_Chest_Athena_C");

        if (ReceivingActor->IsA(ChestClass)) {
             // 6. GRANT REWARD (Minimal Quest Logic)

             // Example: Grant 100 XP
             PC->XPComponent->MatchXp += 100;

             // Example: Give "Search Chest" Accolade
             static auto ChestAccolade = FindObject<UFortAccoladeItemDefinition>("/Game/Athena/Items/Accolades/AccoladeId_008_SearchChests_Bronze.AccoladeId_008_SearchChests_Bronze");
             XP::GiveAccolade(PC, ChestAccolade);
        }
    }
}
```

## Summary: Minimal vs. Full System

| Feature | Minimal Approach (This Guide) | Full System (XP.h) |
| :--- | :--- | :--- |
| **Complexity** | Low. Hooks standard gameplay functions. | High. Hooks internal event system & parses tables. |
| **Flexibility** | Low. You must write C++ code for every new quest type. | High. Quests are defined in Data Tables; code handles them generically. |
| **Maintenance** | Medium. Offsets for `ClientOnPawnDied` are easy to find. | Hard. `SendComplexCustomStatEvent` is harder to find. |
| **POI Support** | Very Hard. You'd have to calculate overlaps manually. | Native. Uses the engine's built-in POI volume events. |

### Conclusion
If you only need "Kills" and "Chests", use this **Minimal Approach**. It is much easier to implement and debug. If you need complex quests like "Visit X", "Dance at Y", or "Deal Damage with Shotgun", you will eventually need the Full System.
