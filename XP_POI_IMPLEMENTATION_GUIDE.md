# XP Quest and POI Collision Implementation Guide

This guide explains how to port the Quest and POI (Point of Interest) discovery logic from the Celestia codebase to another Fortnite GameServer (GS).

## Overview

The system works by intercepting internal game events. Specifically, when a player's pawn overlaps with a POI Volume (`AFortPoiVolume`), the game engine triggers a generic stat event. By hooking this event function and verifying where it was called from, we can detect POI discoveries and trigger the corresponding Quests and XP rewards.

## Step 1: Finding `SendComplexCustomStatEvent`

The core function to hook is `SendComplexCustomStatEvent` (or a similarly named function depending on your SDK generation, often found in `UFortQuestManager`).

### How to find it:
1.  **String Search:** Search your game binary or IDA database for strings like `"ComplexCustom"` or `"SendComplexCustomStatEvent"`.
2.  **VTable:** Look at `UFortQuestManager`'s virtual table. It is often a virtual function.
3.  **Signature Scanning:** If you have a PDB or a known version, generate a signature.

**Prototype (Example):**
```cpp
void SendComplexCustomStatEvent(
    UFortQuestManager* QuestManager,
    UObject* TargetObject,
    FGameplayTagContainer& AdditionalSourceTags,
    FGameplayTagContainer& TargetTags,
    bool* QuestActive,
    bool* QuestCompleted,
    int32 Count
);
```

## Step 2: Finding the POI Return Address

The `SendComplexCustomStatEvent` function is called for *many* things. To specifically detect "POI Discovery", you need to filter these calls. The Celestia codebase does this by checking the **return address** (`_ReturnAddress()`).

### The Logic:
The game engine has a specific function (likely inside `AFortPoiVolume` logic or a related delegate) that calls `SendComplexCustomStatEvent` specifically when a player enters a new POI.

### How to find the specific address:
1.  **Hook the function:** Implement a hook for `SendComplexCustomStatEvent`.
2.  **Log Return Addresses:** Inside your hook, log the return address for every call.
    ```cpp
    // Inside your hook
    void Hooked_SendComplexCustomStatEvent(...) {
        uintptr_t caller = (uintptr_t)_ReturnAddress() - (uintptr_t)GetModuleHandle(NULL);
        Log("SendComplexCustomStatEvent called from Offset: 0x%llX", caller);

        return Original_SendComplexCustomStatEvent(...);
    }
    ```
3.  **Trigger the Event:** Go in-game and walk into a named POI (e.g., "Pleasant Park") that you haven't discovered yet (or reset your profile so it's undiscovered).
4.  **Identify the Offset:** The offset that appears in your log at the exact moment the "New Location Discovered" UI would normally appear is your magic number.

**In Celestia (`XP.h`), this check looks like:**
```cpp
// ImageBase + 0x30d976c is the specific call site for POI discovery in this version
if (__int64(_ReturnAddress()) == ImageBase + 0x30d976c)
{
    // It's a POI event! Process it as "ComplexCustom"
    SendStatEvent(..., EFortQuestObjectiveStatEvent::ComplexCustom);
}
```

## Step 3: Implementing the Hook

Once you have the address, structure your hook like this:

```cpp
void SendComplexCustomStatEvent_Hook(
    UFortQuestManager* QuestManager,
    UObject* TargetObject,
    FGameplayTagContainer& AdditionalSourceTags,
    FGameplayTagContainer& TargetTags,
    bool* QuestActive,
    bool* QuestCompleted,
    int32 Count
) {
    // 1. Check if this call comes from the POI Discovery logic
    if ((uintptr_t)_ReturnAddress() == Your_Found_POI_Address)
    {
        // 2. Trigger your internal Quest Logic
        // The 'TargetTags' usually contain the POI's specific GameplayTags.
        MyQuestSystem::ProcessEvent(
            QuestManager,
            TargetObject,
            AdditionalSourceTags,
            TargetTags,
            EFortQuestObjectiveStatEvent::ComplexCustom
        );
    }
    else
    {
        // Optional: Handle other generic events or force a default behavior
        // Celestia forces a generic event here for non-POI calls:
        // MyQuestSystem::ProcessEvent(..., EFortQuestObjectiveStatEvent::ComplexCustom);
    }

    // 3. Call the original function to let the game do its standard processing
    return SendComplexCustomStatEvent_Original(QuestManager, TargetObject, AdditionalSourceTags, TargetTags, QuestActive, QuestCompleted, Count);
}
```

## Step 4: The Quest Logic (`SendStatEvent`)

You need a function that iterates through the game's quest data and checks if the event matches any active objectives.

1.  **Get the Table:** Load `AthenaObjectiveStatXPTable` (usually located at `/Game/Athena/Items/Quests/AthenaObjectiveStatXPTable.AthenaObjectiveStatXPTable`).
2.  **Iterate Rows:** Loop through the table rows.
3.  **Check Conditions:**
    *   Does `Row->TargetTags` match the event's `TargetTags`? (The POI tags)
    *   Does `Row->SourceTags` match the player's tags?
    *   Is the `Row->Type` equal to `ComplexCustom` (for POIs)?
4.  **Award Logic:**
    *   If matched, verify if it's a "Once Only" reward (like discovering a named location for the first time).
    *   Call `GiveAccolade` (or your equivalent) to grant XP.

## Summary Checklist

1.  [ ] Locate `SendComplexCustomStatEvent` in your GS binary.
2.  [ ] Hook it.
3.  [ ] Log `_ReturnAddress()` and walk into a POI to find the specific caller offset.
4.  [ ] Implement the conditional check inside the hook.
5.  [ ] Copy/Adapt the `SendStatEvent` logic to check against `AthenaObjectiveStatXPTable`.
