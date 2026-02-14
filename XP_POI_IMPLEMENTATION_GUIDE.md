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

## Detailed IDA Pro Tutorial: Finding the Function

This tutorial assumes you have IDA Pro installed and have loaded your Fortnite game binary (e.g., `FortniteClient-Win64-Shipping.exe`).

### Part A: Locating `SendComplexCustomStatEvent`

1.  **Open Strings View:**
    *   Go to `View` -> `Open subviews` -> `Strings` (or press `Shift + F12`).
2.  **Search for Key Strings:**
    *   Press `Ctrl + F` and search for **"ComplexCustom"**.
    *   *Why?* The enum value `EFortQuestObjectiveStatEvent::ComplexCustom` is often converted to a string or used in logging near where the event is processed.
3.  **Follow the Reference:**
    *   Double-click the "ComplexCustom" string to jump to its location in the `.rdata` section.
    *   Click on the string name (e.g., `aComplexcustom`) to highlight it.
    *   Press `X` (or right-click -> Jump to xref) to see code that references this string.
4.  **Analyze the Code:**
    *   You will likely land in a function that looks like a giant `switch` statement (handling different event types like `Kill`, `Damage`, `ComplexCustom`).
    *   This function *is* (or calls) the event handler.
    *   Look for a function call that takes `UFortQuestManager` (often `this` or `RCX`) as the first argument.
    *   **Verification:** Check if the function signature roughly matches: `(Manager, Object, Tags, Tags, bool, bool, int)`.

### Part B: Making it "Find All" (Signature Scanning)

To avoid doing this manually for every version, creating a **Pattern Signature** is best.

1.  **Select the Start of the Function:** Go to the very beginning of the `SendComplexCustomStatEvent` function in IDA (press `P` to ensure IDA recognizes it as a function).
2.  **Generate Signature:**
    *   If you have a plugin like "SigMaker", use "Create Function Pattern".
    *   If not, look at the hex bytes view. Pick the first 10-20 bytes.
    *   Replace any bytes that change (offsets, addresses) with `?` (wildcards).
    *   *Example:* `48 89 5C 24 ? 48 89 74 24 ? 57 48 83 EC 20`
3.  **Use in Code:** Use this signature in your GS loader to automatically find the function address at runtime.

## Step 2: Finding the POI Return Address

The `SendComplexCustomStatEvent` function is called for *many* things. To specifically detect "POI Discovery", you need to filter these calls. The Celestia codebase does this by checking the **return address** (`_ReturnAddress()`).

### The Logic:
The game engine has a specific function (likely inside `AFortPoiVolume` logic or a related delegate) that calls `SendComplexCustomStatEvent` specifically when a player enters a new POI.

### Finding the Magic Number (Return Address)

**Method 1: Dynamic Analysis (Recommended & Easiest)**
This is the "Find All" method for the return address because it requires zero reversing skill, just hooking.

1.  **Hook the Function:** Apply your hook to `SendComplexCustomStatEvent` (found in Step 1).
2.  **Add Logging:**
    ```cpp
    void Hooked_SendComplexCustomStatEvent(...) {
        // Calculate offset from Base Address
        uintptr_t callerOffset = (uintptr_t)_ReturnAddress() - (uintptr_t)GetModuleHandle(NULL);

        // Log it clearly
        Log("[EVENT] SendComplexCustomStatEvent called from Offset: 0x%llX", callerOffset);

        return Original_SendComplexCustomStatEvent(...);
    }
    ```
3.  **Run the Game:** Launch your server and game client.
4.  **Perform the Action:** Walk into a named POI (e.g., Pleasant Park).
5.  **Check the Log:** You will see a log entry appear instantly. That offset (e.g., `0x30d976c`) is your POI Return Address.

**Method 2: Static Analysis (Advanced)**
If you *must* find it without running the game:
1.  Search for **"PoiVolume"** or **"AFortPoiVolume"** strings in IDA.
2.  Find VTables or functions related to `AFortPoiVolume::OnOverlap` or `Enter`.
3.  Look for a call *inside* those functions that jumps to your `SendComplexCustomStatEvent` address.
4.  The address of the instruction *immediately following* that call is your Return Address.

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
        // Optional: Handle other generic events
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

## Reference Implementation Details (Celestia)

The following offsets are used in the Celestia codebase (likely targeting Fortnite Season 13/v13.40):

*   **OFFSET 1: The Main Event Function (Quests)**
    *   **Offset:** `0x2a286d0`
    *   **What it is:** The address of the `SendComplexCustomStatEvent` function itself.
    *   **Usage:** You hook this function to intercept **ALL** complex events (e.g., Opening Chests, Ammo Boxes, *and* Discovering POIs). This is essential for the entire Quest system.

*   **OFFSET 2: The Caller Check (POIs)**
    *   **Offset:** `0x30d976c`
    *   **What it is:** The return address (the location in code that *called* the function).
    *   **Usage:** This specific address tells you that the event was triggered by the **POI Discovery** logic in the game engine (specifically when overlapping an `AFortPoiVolume`). You check `_ReturnAddress() == ImageBase + 0x30d976c` inside your hook to distinguish a "New Location" event from a generic "Open Chest" event.

**Note:** These offsets are version-specific. You **must** find the correct offsets for your target game version using the methods described above.

## Compatibility Check

**Can I use these specific offsets in a Chapter 2 Season 2 (v12.xx) GameServer?**

**NO.** The memory offsets (addresses) listed above (`0x2a286d0` and `0x30d976c`) are specific to the exact version of the Fortnite executable (binary) that the Celestia codebase targets (likely v13.40).

*   **Different Binaries:** Every time the game is compiled (even for small updates), functions shift around in memory. An offset valid for v13.40 will almost certainly point to garbage or a different function in v12.xx.
*   **The Technique Works:** While the *offsets* are wrong, the **methodology** described in this guide works for almost all Fortnite versions (Chapter 1 and 2). You simply need to repeat "Step 2: Finding the POI Return Address" on your C2S2 binary.
*   **Structure Changes:** Be aware that class structures (like `UFortQuestManager`) might have minor differences between seasons. Always verify your SDK/struct definitions against your specific game version.

## Summary Checklist

1.  [ ] Locate `SendComplexCustomStatEvent` in your GS binary.
2.  [ ] Hook it using a library like MinHook (or `Utils::Hook` if available).
3.  [ ] Log `_ReturnAddress()` and walk into a POI to find the specific caller offset.
4.  [ ] Implement the conditional check inside the hook using the found offset.
5.  [ ] Copy/Adapt the `SendStatEvent` logic to check against `AthenaObjectiveStatXPTable`.
