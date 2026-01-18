## LootCollector 0.7.47.1 - PvP Death Chest False Positives Fix

### Bug Fixes
- **PvP Death Chest Filtering:**
  - Implemented blacklist for "Unclaimed Belongings" and "Human Corpse" loot containers.
  - Prevents PvP death chests from being incorrectly recorded as world discoveries.
  - Fixed issue where opening a player's dropped loot would spawn a permanent map pin.

## LootCollector 0.7.47 - Astrolabe Integration for Smooth Minimap Rotation

### Major Improvements

- **Astrolabe Library Integration:**
  - Integrated Astrolabe-0.4 library (DongleStub, Astrolabe, AstrolabeMapMonitor)
  - Replaced ~80 lines of manual minimap rotation code with Astrolabe API
  - Minimap pins now rotate as smoothly as GatherMate pins
  - Automatic handling of minimap shape, zoom scaling, and edge clamping

### Technical Implementation

- **Library Files Added:**
  - `Libs/Astrolabe/DongleStub.lua` - Library versioning system
  - `Libs/Astrolabe/Astrolabe.lua` - Core minimap icon positioning
  - `Libs/Astrolabe/AstrolabeMapMonitor.lua` - World map state monitoring

- **API Usage:**
  - `Astrolabe:PlaceIconOnMinimap(pin, continent, zone, x, y)` - Automatic rotation/positioning
  - `Astrolabe:RemoveIconFromMinimap(pin)` - Clean removal from Astrolabe tracking
  - `Astrolabe:GetDistanceToIcon(pin)` - Distance check for visibility filtering

- **Distance Filtering:**
  - Pins beyond `minimapRadius * 1.5` are hidden to prevent edge positioning issues
  - Prevents distant pins from clustering at minimap edges

### Ticker Optimization

- Increased ticker rate from 0.1s (10 FPS) to 0.016s (60 FPS) for smoother updates
- Astrolabe handles rotation via its own internal OnUpdate handler

---

## LootCollector 0.7.46 - Performance Optimizations Phase 3

### Performance Improvements

- **String Interning:**
  - Added `InternLink()` function to deduplicate repeated item link strings
  - Reduces memory usage by sharing identical string references
  - Applied to discovery record creation in Core.lua

- **Weak Table Caching:**
  - Converted Viewer.lua caches to use weak tables (`itemInfo`, `characterClass`, `worldforged`, `zoneNames`)
  - Allows garbage collector to automatically clean unused cache entries
  - Added `IsWeakCachingActive()` method for verification

- **Batch Processing Utility:**
  - Added `ProcessInChunks()` coroutine utility for spreading heavy operations across frames
  - Prevents UI freezing during large database operations
  - Processes 50 items per frame by default

### New Commands

- **`/lcstats`** - Shows optimization status including:
  - String interning count
  - Weak table caching status
  - Total discoveries in database

- **`/lcbench`** - Benchmarks minimap pin performance (spatial hashing comparison)

### Technical Notes

- Spatial hashing for minimap pins was implemented but reverted due to pin rotation/visibility bugs
- Event coalescing deferred for future implementation

---



### Bug Fixes

- **Fixed:** Mystic Scrolls (enchants) now correctly remain hidden after being looted, even when other players broadcast them at different locations
  - Extended the dual-key tracking system (previously only for Worldforged items) to also apply to Mystic Scrolls
  - Looted Mystic Scrolls are now tracked by both GUID and `(itemID, zone)` key
  - This ensures the looted state persists when the same scroll appears at a new spawn location
  - Affects `IsLootedByChar`, `ToggleLootedState`, and manual "Set as looted/unlooted" menu options
  - **Migration:** Existing looted Mystic Scrolls are automatically backfilled to the new tracking system
  - **Zone-aware updates:** On zone change, the addon re-checks for any looted scrolls that need to be hidden

### Technical Details

Mystic Scrolls, like Worldforged items, are unique per zone. The previous implementation only tracked Mystic Scrolls by GUID, which includes coordinates. When another player found the same scroll at a different location, a new GUID was generated, causing the scroll to reappear. The fix extends the `lootedByItemZone` tracking mechanism to Mystic Scrolls and includes a migration function that runs on login (full scan) and on zone change (zone-specific scan).

---

## LootCollector 0.7.44 - Worldforged Loot Persistence Fix

### Bug Fixes

- **Fixed:** Worldforged items now correctly remain hidden after being looted, even when other players broadcast them at different locations
  - Looted Worldforged items are now tracked by both GUID and `(itemID, zone)` key
  - This ensures the looted state persists when the same item appears at a new spawn location
  - Affects both automatic loot detection and manual "Set as looted/unlooted" menu options

### Technical Details

Worldforged items are unique per zone (only one of each itemID exists at a time). The previous implementation tracked looted items solely by GUID, which includes coordinates. When another player found the same item at a different location, a new GUID was generated, causing the item to reappear. The fix adds a secondary tracking mechanism by `(itemID, zone)` key for Worldforged items.

---



### Performance Improvements

- **Core.lua - New Performance Utilities:**
  - Added `AcquireTable()`/`ReleaseTable()` table pool to reduce GC pressure by reusing short-lived tables
  - Added `GetAdaptiveTimeBudget()` that adjusts processing time based on current FPS (1-5ms range)
  - Added additional cached globals (`GetTime`, `GetFramerate`, `wipe`, `tremove`, `debugprofilestop`)

- **Toast.lua Optimizations:**
  - Replaced 5 `table.insert()` calls with direct assignment pattern (`t[#t + 1] = v`)
  - Faster queue processing in spam detection and toast management

- **Tooltip.lua Optimizations:**
  - Added localized globals for `pairs`, `ipairs`, `tonumber`, `tostring`, `type`, `select`, `GetItemInfo`, `GetItemStats`
  - Replaced 11 `table.insert()` calls with direct assignment pattern
  - Faster upgrade chain building and tooltip rendering

### Technical Details

Direct table assignment (`t[#t + 1] = v`) is ~10-15% faster than `table.insert(t, v)` in Lua 5.1. The table pool prevents frequent garbage collection by reusing tables instead of creating new ones. The adaptive time budget allows processing to scale with available frame time.

---



### Performance Improvements

- **Toast.lua Optimizations:**
  - Added global function caching for `GetTime`, `math.*`, `table.*`, `string.*` to reduce lookup overhead
  - Replaced per-call table creation in `checkQueueForSpam()` with reusable table using `wipe()`
  - Dispatcher frame now hides when idle (empty queue, no ticker, no anon buffer) and auto-wakes when work is added
  - This reduces CPU usage when no toasts are pending

- **Map.lua Optimizations:**
  - `updateTicker` frame now hides when no data changes are pending and auto-wakes when data changes
  - Added `wakeDataChangeHandler()` helper called from all locations that set `L.DataHasChanged = true`

- **Core.lua & DBSync.lua:**
  - Added wake calls at all `L.DataHasChanged = true` locations to support the new idle optimization

### Technical Details

These changes reduce CPU usage by preventing OnUpdate handlers from running every frame when there's no work to do. The dispatcher and updateTicker frames now start hidden and only become active when work is added.

---

## LootCollector 0.7.41 - Frame Strata Fix

- **Fixed Tooltip Visibility:** Tooltips now display above map pins and overlays instead of behind them
  - Changed map pins, overlays, and hover buttons from `TOOLTIP` strata to `HIGH` strata
  - Changed context menus and autocomplete dropdowns from `TOOLTIP` strata to `DIALOG` strata
  - This prevents addon frames from overlapping game tooltips

## LootCollector 0.7.40 - Fixes + Small Optimizations**

- Fixed few bugs.
- Testing some alternative solutions and optimizations.
- Added feature to disable Mystic Scrolls processing.
- Removed HistoryTab feature.

## LootCollector 0.6.90 - Fixes**

- Fixed a bug where HideDiscoveryTooltip was hiding other map tooltips.  
- Improved minimap icon visibility.  
- Fixed a bug where Mystic Scrolls were not filtered correctly.  
- The default addon channel is now hidden from users.
- You can now reposition the LC map button by holding Shift and dragging it.

## LootCollector 0.6.40 - Fixes + Small Optimizations**

- Fixed few reported issues in Map, Settings and Viewer.
- Map Optimization.

## LootCollector 0.6.10 - Fixes + Small Optimizations**

- Fixed issue where out-of-distance minimap icons appeared on the edges of minimap and "floated".
- Minimum compatible version to accept discoveries is now 0.6.10

## LootCollector 0.6.00 - Fixes**

- Added support when `GetCurrentMapContinent()` returning -1 in some cases.
- Improved detection of Mystic Scroll acquisition from quests.
- Various minor fixes and stability improvements.

## **LootCollector 0.5.97 - Fixes**

Bunch of fixes

## **LootCollector 0.5.94 - Fixes**

Bunch of fixes

## **LootCollector 0.5.90 - Realm Buckets & Stability Overhaul**

### **Highlights**
*   **Realm Buckets (Database Isolation):**
    *   Completely restructured the database to store discoveries and vendors separately per Realm (e.g., *Bronzebeard* / *Malfurion* vs *Elune*). This prevents data corruption and "bleeding" when switching between Seasonal and Main realms.
    *   Includes an automatic **Migration System** that moves your existing data to the correct realm bucket upon login.
*   **Zone ID Standardization:**
    *   The addon now enforces stricter Zone ID validation using modern MapIDs instead of legacy indexes.
    *   Added an automatic repair tool (`/lcczfix`) to detect and correct records with invalid Continent-Zone mismatches.
    *   Introduced **Tombstones with Expiration**: When a bad record is fixed, a "tombstone" is created to prevent older data from syncing back to you for a set duration.
*   **Mystic Scroll discoveries are once again shared! Automatic vendor sharing is still on the to-do list, but you can share them manually with the "Show to" feature.

### **Fixes**
*Too many but still not enough :p*

### **Technical & Optimizations**
*   **Database Accessors:** Refactored every module (`Core`, `Map`, `Viewer`, `DBSync`, etc.) to use secure accessors (`GetDiscoveriesDB`) instead of accessing global tables directly. This improves stability and supports the new Realm Bucket architecture.
*   **Network Security:** Strengthened validation for incoming data packets to reject malformed zone IDs immediately.

### **Feature Refinements**
*   **World Map:** Clarified interaction logic. For best results, use `/script WorldMapFrame:Show()` or a map addon (like Magnify/ElvUI) to enable full interactivity (Context Menus, Proximity Lists) which are restricted in the default protected map mode.
*   **Vendor Tracking:** Improved logic for associating specific inventories with Black Market vendors in the database.

*Database Update: Updated starter database to include verified coordinates for the latest content.*

## **LootCollector 0.5.8/0.5.81 - Fixes **

Some emergency fixes.
Expanded block list functionality.
Added option to delete data from blocked players.

## **LootCollector 0.5.7 - Performance optimizations + Fix **
Several performance optimizations that may improve the addon's performance. 
Fix for disappearing tooltip.

Database Update: Updated starter database to 2k+ discoveries.

## **LootCollector 0.5.6 - Fixes **
Bunch of fixes

Database Update: Updated starter database to 1840+ discoveries.

## **LootCollector 0.5.5 - New command + Fixes **

Highlights
- **`/lctoggle`  New command**  
Toggles visibility of Map and Minimap discoveries (you can keybind it too!)

Fixes
Database cleanup and optimization to ensure stability and performance.
Improved the WF tooltip with richer details.
Fixed tracking and auto-tracking functionality to now work correctly in all starter zones*.

Database Update: Updated starter database to 1760+ discoveries.

## **LootCollector 0.5.4 - Fixes **
Mainly related to the new player experience with the addon.
Changed item and vendor detection.
A bunch of other fixes.

Database Update: Updated starter database to 1640+ discoveries.

## **LootCollector 0.5.3 - Enhanced WF Tooltip + Fixes**
Highlights
Enhanced WF Tooltip - new feature for WF items that displays statistics after certain upgrade levels, providing detailed progression information directly in tooltips.

Fixes
Database Error Fix: Resolved issue where new discovery reports for older records caused errors when attempting to access the non-existent fp_votes table.

Toast Notification Fix: Reduced redundant toast spam that incorrectly displayed notifications for already existing discoveries.

Database Update: Updated starter database to 1590 discoveries.



## **LootCollector 0.5.2 - Viewer + Fixes**
Highlights
Integrated Markosz’s Viewer module and its minimap icon into the addon.

Fixes
Resolved locked movement during display of certain popup dialogs.
Added a short delay to addon minimap operations after changing zones to avoid UI lag.
Corrected fallback channel name resolution in Comm.lua.
Fixed pause state handling.
Possibly fixed another /1 hijack issue and fixed channel joining.
Fixed an issue where looted discoveries were not automatically marked as looted
Corrected square map detection and minimap rotation handling. //Markosz
Fixed the Show() function to target the correct discovery item. //Markosz

## **LootCollector Changelog: Version 0.5.1S (from 0.4.9)**

Changes focusing on database optimization, network protocol enhancements, community moderation, and  UI/UX and QOL improvements.

### **Major New Features & Architectural Changes**

*   **Database Schema v6 Overhaul & Migration:**   
    *   A new automated, one-time **Database Migration** system prompts users with older databases. It safely preserves all character-specific "looted" history while upgrading the main discovery database to the new format and automatically importing the latest starter database.

*   **Map Clustering & Zone Awareness:**
    *   When viewing a continent or parent zone (e.g., Kalimdor), discoveries are now grouped into a single "cluster" pin at the zone's entrance, showing the total count. (implemented just not applied except Mulgore)
    *   Clicking a cluster pin zooms into that zone, decluttering the world view and improving performance.

*   **Advanced Toast Notification System:**
    *   **Anti-Spam Ticker:** When many discoveries are received at once, the addon switches from individual pop-ups to a single, scrolling "ticker" frame to prevent UI spam. (can be disabled)
    *   **Smart Queue & Spam Detection:** The system now detects and suppresses toast spam from individual players to maintain a clean user experience.
    *   **Interactivity:** Toasts are now interactive. Move mouse pointer over an item to see details or Shift+click to opens map and focus on its location.

*   **Community Moderation & Data Integrity:**
    *   **"Report as Gone":** Right-clicking a map pin and reporting it as gone now sends a "deletion vote" to the network.
    *   **Status by Consensus:** If enough players report a discovery as gone, its status is automatically downgraded to `FADING` and then `STALE`, keeping the database clean. `STALE` items are visually distinct on the map.
    *   **Coordinate Consensus:** The system refines a discovery's coordinates over time by averaging reports from multiple players, improving pin accuracy.

*   **"Show to..." a Player Feature:**
    *   You can now right-click a map pin to select "Show to...". This allows you to send a specific discovery location directly to another player and focus his map on it.
    *   The recipient gets a confirmation popup to "Allow," "Deny," or "Block Sender," preventing unsolicited map pings.

### **Communication & Network**

*   **New v5 Network Protocol:** The communication system has been rewritten for efficiency, supporting new features like deletion votes (`ACK`), data corrections, and "Show" requests.
*   **Advanced Rate Limiting:** A "leaky bucket" algorithm has been implemented for channel messages. This smooths out network traffic to prevent disconnects and ensures no messages are dropped during busy periods.
*   **Enhanced Security & Validation:**
    *   Incoming messages are now strictly validated for correct data types and ranges. Invalid messages are tracked, and repeat offenders are automatically session-ignored or permanently blacklisted.
*   **Version Enforcement:** The addon now enforces a minimum compatible version, ignoring messages from outdated clients to prevent data corruption.
*   **Important Change to Sharing:** To ensure data quality, automatic sharing is temporarily limited to **Worldforged** items. Mystic Scrolls can still be shared manually via `/lcshare` and the "Show to..." feature. Default automatic sharing for MS will be restored in future versions.


### **UI & Quality of Life Improvements**

*   **Map Enhancements:**
    *   **Search & Filter Bar:** An optional search bar can now be displayed on the World Map to filter pins by item or zone name in real-time.
    *   **Map Focus Animation:** A new "Focus" feature (accessible via Shift+click) opens the map and plays a pulsing animation at a discovery's location.
    *   **Improved Pin Appearance:** Pin borders are now color-coded by item quality. `STALE` items are clearly marked with a distinct orange color.

*   **Arrow (TomTom) Improvements:**
    *   A new "Auto-track Nearest Unlooted" option automatically activates the arrow.
    *   You can now temporarily "skip" the nearest target for the current session via the map filter menu.
    *   The arrow now automatically finds the next target after you loot the one it was pointing to.

*   **Minimap Overhaul:**
	*   Dynamic minimap shape detection to display markers along the edges correctly has been implemented based on **Markosz** code.
	*   A new option allows limiting minimap pins to a specific yardage from your character.
    *   The minimap pin logic has been completely rewritten for accuracy and performance.    

*   **Advanced Filtering:**
    *   Filters for discoveries now include "Usable by Class" and "Equipment Slot," which work instantly even for uncached items.

*   **Import/Export:**
    *   Added a new **"Import from File"** method for safely importing massive community databases without freezing the client.
    *   Player block/whitelists can now be included in exports.
    *   Added a **"Factory Reset"** button to the Discoveries panel to completely wipe all addon data.

### **Core Logic & Bug Fixes**

*   **Centralized Zone Data:** A new `ZoneList.lua` module provides accurate Zone, Instance, and Sub-zone name resolution, which is critical for map clustering and display accuracy.
*   **Item Metadata Storage:** The addon now stores more metadata for each discovery (item type, subtype, class restrictions) directly in the database.
*   **Delayed Broadcasting:** Newly found discoveries are now briefly buffered. This allows the client to cache item information, ensuring more complete and accurate data is broadcast to other players.
*   **Coordinate Precision:** All coordinates now use 4-decimal precision for improved accuracy.
*   **Fixed `/1` Hijacking:** Resolved an issue where the addon could sometimes interfere with the official chat channels.