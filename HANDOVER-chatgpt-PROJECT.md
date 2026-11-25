# Project handover: Metrolist customisation (for ChatGPT)

Prepared by Claude. Read this whole document first. The patch is in section 12.

## 1. The project

- **Upstream:** `MetrolistGroup/Metrolist` on GitHub, licensed GPL-3.0. It is a YouTube Music client for Android, desktop, and iOS.
- **Version we worked from:** 14.0.1 (zip snapshot, not a git clone). The owner has forked it to their own GitHub account.
- **Tech:** Kotlin Multiplatform with Compose. Modules: `shared` (most logic and UI), `androidApp`, `desktopApp`, `iosApp`, `listenTogetherProto`, `metroproto` (submodule, do not modify), `opusDecoder` (submodule, do not modify).
- **External dependency:** InnerTubeX, a separate YouTube API library. Changes to YouTube access cannot be made inside this repo.
- **Build requirements:** Temurin JDK 25 (25.0.5.1 or newer, because of a security fix), the Android SDK, `local.properties` with `sdk.dir`, and submodules initialised. The project's own guide is in `docs/development.md`.

## 2. The owner

- Uses an Android phone. Cannot run builds or tests on the phone, so send short, phone-friendly replies and avoid asking for long manual steps.
- Owns the GitHub fork and has approved the following: commit on a branch, push the branch to the fork, open a draft pull request into the fork's `main`, and run CI in the fork. Nothing goes to the upstream repository.
- Has authorised full responsibility for the work on this project. Still ask before anything that changes playback, the queue, or stored preferences.
- Prefers being told only when work is complete, unless there's a decision to make.

## 3. Rules from the project's own `AGENTS.md`

- One task per pull request. Keep features separate.
- Do not commit, push, merge, or rewrite history without explicit human approval.
- Never force-push `main`. Prefer pull requests over direct pushes.
- Do not bump `appVersion` unless release work is authorised.
- No hardcoded user-facing strings. Put them in `shared/src/commonMain/composeResources/values/strings.xml`. Only English goes in `values/`.
- No hardcoded language or country fallbacks such as `"en"` or `"US"`.
- Define preference keys in `PreferenceKeys.kt` and access them through `Settings.shared`.
- Keep pull request bodies to a few plain sentences, with no lists and no unverified claims.
- Rejected features (do not propose or implement): Liquid Glass UI, other streaming services as playback backends, Google Play Store publication, comment sections, YouTube Shorts, music alarms, new-music notifications, multiple queues, Listen Together profile pictures uploaded from local storage without uploading, Tizen TV support, a web client, audio visualizers, custom app icons.
- Do not delete existing functionality.

## 4. What already exists in the app (verified from the code)

The owner should not be offered these as new features:
- Sleep timer with fade-out and finish-current-song option.
- Equalizer with saved profiles and AutoEq catalogue.
- Playback speed and audio boost.
- Crossfade (setting `crossfadeEnabled`) and skip silence.
- SponsorBlock.
- Synced lyrics with several display styles, lyrics offset, and an AI-assisted lyrics option.
- Last.fm integration and Discord integration.
- Casting, Listen Together, Android Auto with pinned playlists.
- Custom fonts, animated covers, and an AI-slop blocklist.
- Backup and auto-backup.
- Music recognition with a local recognition history table (`recognition_history`).
- Playlist import from Spotify (public URL and Liked Songs via a session cookie), Tidal, Apple Music, Amazon Music, Deezer, CSV, M3U, JSON, and plain text.

## 5. Owner's requests and where each one stands

**A. Import Spotify Liked Songs**
- Already supported. The Spotify importer includes Liked Songs as a source and logs in with the owner's Spotify session cookie.
- Nothing to code. The owner runs the import in the app themselves.
- Fallback: export likes to CSV with a third-party tool and use the CSV importer (needs title, artist, album columns).
- Note: the importer uses Spotify's internal web API, so it can break if Spotify changes it.

**B. Sort and categorise downloads by artist and album**
- Status: implemented in the patch (see section 6) and pushed to the owner's fork as branch `feature/downloads-grouping`. Not compiled or tested. CI result unknown at handover.
- Scope: Android Downloaded view only. Group by None (default), Artist, or Album. Artist and Album groups show sticky headers.
- Existing sort options (by artist, title, size, and so on) were already in the app. The missing part was grouping with headers.

**C. Import Shazam library**
- Not implemented. Nothing in the code imports a Shazam library. The app only saves recognitions to a local history.
- Apple has no public API for Shazam data. The owner should request their data from Apple's privacy site, which should include Shazam history. The export format is unverified.
- Next step: the owner sends a sample file, with personal data redacted. Then an importer can be written in the style of the existing CSV and Amazon importers.
- Treat as a separate task and a separate pull request.

**D. Look for gaps in the code**
Confirmed gaps:
- Downloads grouped by artist or album (addressed by item B).
- Shazam library import (item C).

Unverified candidates (check before proposing):
- Adding recognised songs to a playlist in bulk.
- Listening statistics or richer queue editing. These were not checked in the code.
- Clearer labelling on the Downloads screen, which shows the download queue and not the downloaded library.

## 6. The current patch (item B)

- Commit: `Add artist and album grouping for downloaded songs`, author "Claude (AI assistant)".
- Base: zip snapshot of Metrolist 14.0.1. If the owner's `main` has moved on, `git am` may fail. If so, stop and report the error rather than forcing anything.
- Files (7 changed, +212 / -23):
  - `shared/.../model/SongGrouping.kt` (new): `SongGroupBy` enum (NONE, ARTIST, ALBUM), `SongGroup`, and `groupedBy()`.
  - `shared/src/commonTest/.../model/SongGroupingTest.kt` (new): four unit tests.
  - `shared/.../constants/PreferenceKeys.kt`: `DownloadedSongGroupByKey`.
  - `shared/.../settings/Settings.kt`: `downloadedSongGroupBy`, default `NONE`.
  - `shared/src/commonMain/composeResources/values/strings.xml`: five English strings.
  - `shared/.../ui/screens/PlaylistContent.kt`: optional `songGroups`, `unknownGroupTitle`, and `groupByControl` parameters. Sticky headers appear only when grouping is on, not reordering, and not searching. The song row body moved into a local `songRow` composable with no logic change.
  - `androidApp/.../ui/screens/AutoPlaylistScreen.kt`: builds groups, reorders the Downloaded list to match, and shows the control.

## 7. Design rules for that patch

- Playback indexes must stay consistent. Grouping reorders the list before the UI receives it, so index-based callbacks match what the user sees. Do not add a second ordering path.
- Headers appear only when there is no search filter, not in reorder mode, and when group sizes match the displayed list. Keep these guards.
- Only the Downloaded view sets `songGroups`. Other playlists pass `null` and behave as before.

## 8. Known limits (ask the owner before changing these)

- With grouping on, "play from this song" uses the normal queue order. Play All and Shuffle use the grouped order.
- Search results show a flat list with no headers.
- Reorder mode shows no headers.
- Only the Android Downloaded view has the control.

## 9. Compile risks (not yet verified)

- `PlaylistContent.kt`: the local `songRow` composable is called from inside `item {}` and `itemsIndexed` content.
- `PlaylistContent.kt`: `stickyHeader` needs `ExperimentalFoundationApi`. The file-level opt-in was added. If the compiler still complains, move it to the function.
- `PlaylistContent.kt`: `songGroups.sumOf { ... }` relies on smart-casting a parameter.
- `AutoPlaylistScreen.kt`: `Settings.shared.downloadedSongGroupBy` is an enum preference. Compare with `downloadedCollectionSort`.
- `AutoPlaylistScreen.kt`: imports were appended and may fail ktlint ordering. Reorder only if lint fails.
- `SongGroupingTest.kt`: expected values were checked by a Python simulation, not by running Kotlin.

## 10. Environment and tools

- The Claude sandbox has no Android SDK, no Kotlin compiler, no network access, and empty submodules. It cannot build or test this project.
- CircleCI is available as a connector but was not connected at the time of writing.
- A GitHub connector was not found in the partial connector search.
- Whoever continues should use the owner's CI (GitHub Actions in the fork, or CircleCI once connected) for all builds.

## 11. Steps to continue

1. Read the CI result on the owner's fork (branch `feature/downloads-grouping`, draft pull request). Ask the owner for the branch name, commit SHA, whether `git am` applied, and the CI status.
2. If the build or tests failed, fix only what the errors point to. Commit the fix on the same branch and explain it in the pull request.
3. For Shazam import, ask the owner for a sample export with personal data redacted, then write a separate importer on a new branch.
4. For Spotify, nothing to build. Help the owner run the import and troubleshoot if it fails.
5. Keep each change to one feature per branch and pull request.

## 12. The patch

===== BEGIN PATCH =====
From 563608508614b3f986470907bf12f5dec02b1289 Mon Sep 17 00:00:00 2001
From: "Claude (AI assistant)" <noreply@anthropic.com>
Date: Wed, 25 Nov 2026 23:04:54 +0000
Subject: [PATCH] Add artist and album grouping for downloaded songs

Adds a Group by control (None, Artist, Album) to the Android Downloaded
view. Artist and Album groups show sticky headers. Songs are reordered
before display so playback indexes stay consistent. Unnamed items fall
into an "Unknown" group. Headers are hidden during search and reordering.

Includes SongGrouping model, unit tests, a downloadedSongGroupBy
preference, and English strings.

Not yet compiled or tested on device.
---
 .../android/ui/screens/AutoPlaylistScreen.kt  | 41 +++++++++-
 .../composeResources/values/strings.xml       |  5 ++
 .../shared/constants/PreferenceKeys.kt        |  1 +
 .../metrolist/shared/model/SongGrouping.kt    | 49 ++++++++++++
 .../com/metrolist/shared/settings/Settings.kt |  3 +
 .../shared/ui/screens/PlaylistContent.kt      | 76 +++++++++++++------
 .../shared/model/SongGroupingTest.kt          | 60 +++++++++++++++
 7 files changed, 212 insertions(+), 23 deletions(-)
 create mode 100644 shared/src/commonMain/kotlin/com/metrolist/shared/model/SongGrouping.kt
 create mode 100644 shared/src/commonTest/kotlin/com/metrolist/shared/model/SongGroupingTest.kt

diff --git a/androidApp/src/main/kotlin/com/metrolist/android/ui/screens/AutoPlaylistScreen.kt b/androidApp/src/main/kotlin/com/metrolist/android/ui/screens/AutoPlaylistScreen.kt
index ccc40db..5127ad3 100644
--- a/androidApp/src/main/kotlin/com/metrolist/android/ui/screens/AutoPlaylistScreen.kt
+++ b/androidApp/src/main/kotlin/com/metrolist/android/ui/screens/AutoPlaylistScreen.kt
@@ -206,6 +206,11 @@ import org.jetbrains.compose.resources.pluralStringResource
 import org.jetbrains.compose.resources.stringResource
 import org.koin.compose.koinInject
 import kotlin.time.Clock
+import com.metrolist.shared.model.SongGroupBy
+import com.metrolist.shared.model.groupedBy
+import androidx.compose.foundation.horizontalScroll
+import androidx.compose.foundation.rememberScrollState
+import com.metrolist.shared.ui.components.ExpressiveFilterChip
 
 @OptIn(ExperimentalMaterial3Api::class)
 @Composable
@@ -456,7 +461,7 @@ fun AutoPlaylistScreen(
                 emptyList()
             }
         }
-    val displayedSongs =
+    val ungroupedDisplayedSongs =
         if (currentType == "downloaded" &&
             (downloadFilterState.loading || downloadFilterState.failed || preparedDownloadFilter != downloadFilterState)
         ) {
@@ -469,6 +474,37 @@ fun AutoPlaylistScreen(
             preparedSongs
         }
 
+    val downloadSongGroupBy = Settings.shared.downloadedSongGroupBy
+    val songGroups =
+        remember(ungroupedDisplayedSongs, downloadSongGroupBy, currentType) {
+            if (currentType == "downloaded" && downloadSongGroupBy != SongGroupBy.NONE) {
+                ungroupedDisplayedSongs.groupedBy(downloadSongGroupBy)
+            } else {
+                null
+            }
+        }
+    val displayedSongs = songGroups?.flatMap { it.songs } ?: ungroupedDisplayedSongs
+    val downloadGroupByControl: @Composable () -> Unit = {
+        Row(
+            modifier = Modifier.horizontalScroll(rememberScrollState()).padding(horizontal = 16.dp, vertical = 8.dp),
+            verticalAlignment = Alignment.CenterVertically,
+            horizontalArrangement = Arrangement.spacedBy(8.dp),
+        ) {
+            Text(text = stringResource(Res.string.group_by), style = MaterialTheme.typography.labelLarge)
+            listOf(
+                SongGroupBy.NONE to Res.string.group_by_none,
+                SongGroupBy.ARTIST to Res.string.group_by_artist,
+                SongGroupBy.ALBUM to Res.string.group_by_album,
+            ).forEach { (option, label) ->
+                ExpressiveFilterChip(
+                    selected = downloadSongGroupBy == option,
+                    onClick = { Settings.shared.downloadedSongGroupBy = option },
+                    label = { Text(stringResource(label)) },
+                )
+            }
+        }
+    }
+
     suspend fun orderedCachedSongs(): List<SongItem> {
         val committed = commitPendingCachedOrder()
         val unfilteredInput =
@@ -857,6 +893,9 @@ fun AutoPlaylistScreen(
                             null
                         },
                     songs = displayedSongs,
+                    songGroups = songGroups,
+                    unknownGroupTitle = stringResource(if (downloadSongGroupBy == SongGroupBy.ALBUM) Res.string.unknown_album else Res.string.unknown_artist),
+                    groupByControl = if (currentType == "downloaded") downloadGroupByControl else null,
                     artworkSongs = songs,
                     downloadedSongIds = downloadedSongIds,
                     text = playlistContentText(useTrackCount = currentType in setOf("podcasts", "SE")),
diff --git a/shared/src/commonMain/composeResources/values/strings.xml b/shared/src/commonMain/composeResources/values/strings.xml
index 9878823..4f6dfe6 100644
--- a/shared/src/commonMain/composeResources/values/strings.xml
+++ b/shared/src/commonMain/composeResources/values/strings.xml
@@ -5693,4 +5693,9 @@
     <string name="wrapped_home_title">Your %1$s Wrapped is here</string>
     <string name="wrapped_home_body">See your year in music and share it.</string>
     <string name="wrapped_home_card_dismissed">You can find your %1$s Wrapped in Statistics until December 31.</string>
+    <string name="group_by">Group by</string>
+    <string name="group_by_none">None</string>
+    <string name="group_by_artist">Artist</string>
+    <string name="group_by_album">Album</string>
+    <string name="unknown_album">Unknown album</string>
 </resources>
diff --git a/shared/src/commonMain/kotlin/com/metrolist/shared/constants/PreferenceKeys.kt b/shared/src/commonMain/kotlin/com/metrolist/shared/constants/PreferenceKeys.kt
index 6a1f820..5f3a18c 100644
--- a/shared/src/commonMain/kotlin/com/metrolist/shared/constants/PreferenceKeys.kt
+++ b/shared/src/commonMain/kotlin/com/metrolist/shared/constants/PreferenceKeys.kt
@@ -229,6 +229,7 @@ val CachedCollectionSortDirectionKey = "cachedCollectionSortDirection"
 val CachedSongsCustomOrderKey = "cachedSongsCustomOrder"
 val DownloadedCollectionSortKey = "downloadedCollectionSort"
 val DownloadedCollectionSortDirectionKey = "downloadedCollectionSortDirection"
+val DownloadedSongGroupByKey = "downloadedSongGroupBy"
 val PlaylistPickerSortKey = "playlistPickerSort"
 val PlaylistPickerSortDirectionKey = "playlistPickerSortDirection"
 val CollectionCoversKey = "collectionCoversJson"
diff --git a/shared/src/commonMain/kotlin/com/metrolist/shared/model/SongGrouping.kt b/shared/src/commonMain/kotlin/com/metrolist/shared/model/SongGrouping.kt
new file mode 100644
index 0000000..1368e7d
--- /dev/null
+++ b/shared/src/commonMain/kotlin/com/metrolist/shared/model/SongGrouping.kt
@@ -0,0 +1,49 @@
+/*
+ * Metrolist Project (C) 2026
+ * Licensed under GPL-3.0 | See git history for contributors
+ */
+
+package com.metrolist.shared.model
+
+enum class SongGroupBy {
+    NONE,
+    ARTIST,
+    ALBUM,
+}
+
+/**
+ * [key] is the display name shared by the group's songs. It is null when no artist or album is
+ * known; the UI should label those groups with a localized string. For [SongGroupBy.NONE] there
+ * is a single group with a null key, and the UI should ignore the key.
+ */
+data class SongGroup(
+    val key: String?,
+    val songs: List<SongItem>,
+)
+
+/**
+ * Groups songs by primary artist or album. Matching ignores case, so "Queen" and "queen" share a
+ * group. Named groups are sorted alphabetically and the unknown group goes last. Songs keep their
+ * original order inside each group.
+ */
+fun List<SongItem>.groupedBy(by: SongGroupBy): List<SongGroup> {
+    if (by == SongGroupBy.NONE) return listOf(SongGroup(key = null, songs = this))
+
+    fun nameFor(song: SongItem): String? =
+        when (by) {
+            SongGroupBy.ARTIST -> song.artists.firstOrNull()?.name
+            SongGroupBy.ALBUM -> song.album?.name
+            SongGroupBy.NONE -> null
+        }?.trim()?.takeIf { it.isNotEmpty() }
+
+    return groupBy { nameFor(it)?.lowercase() }
+        .map { (_, songs) -> SongGroup(key = nameFor(songs.first()), songs = songs) }
+        .sortedWith { a, b ->
+            when {
+                a.key == null && b.key == null -> 0
+                a.key == null -> 1
+                b.key == null -> -1
+                else -> a.key.compareTo(b.key, ignoreCase = true)
+            }
+        }
+}
diff --git a/shared/src/commonMain/kotlin/com/metrolist/shared/settings/Settings.kt b/shared/src/commonMain/kotlin/com/metrolist/shared/settings/Settings.kt
index fcdd5af..870e89a 100644
--- a/shared/src/commonMain/kotlin/com/metrolist/shared/settings/Settings.kt
+++ b/shared/src/commonMain/kotlin/com/metrolist/shared/settings/Settings.kt
@@ -75,6 +75,7 @@ import com.metrolist.shared.constants.DiscordAlbumTemplateKey
 import com.metrolist.shared.constants.DiscordPendingAuthSessionKey
 import com.metrolist.shared.constants.DownloadQueueJsonKey
 import com.metrolist.shared.constants.DownloadedCollectionSortDirectionKey
+import com.metrolist.shared.constants.DownloadedSongGroupByKey
 import com.metrolist.shared.constants.DownloadedCollectionSortKey
 import com.metrolist.shared.constants.EnableHapticFeedbackKey
 import com.metrolist.shared.constants.EnableLandscapeScalingKey
@@ -214,6 +215,7 @@ import com.metrolist.shared.model.CollectionSort
 import com.metrolist.shared.model.LyricsProvider
 import com.metrolist.shared.model.PlaylistPickerSort
 import com.metrolist.shared.model.PlaylistSongSort
+import com.metrolist.shared.model.SongGroupBy
 import com.metrolist.shared.model.SortDirection
 import com.metrolist.shared.playback.SleepTimerDefaults
 import com.metrolist.shared.youtube.playback.PlaybackNetworkState
@@ -803,6 +805,7 @@ class Settings(
 
     var downloadedCollectionSort by preference(DownloadedCollectionSortKey, CollectionSort.NEWEST)
     var downloadedCollectionSortDirection by preference(DownloadedCollectionSortDirectionKey, SortDirection.DESCENDING)
+    var downloadedSongGroupBy by preference(DownloadedSongGroupByKey, SongGroupBy.NONE)
     private var collectionCoversJson by preference(CollectionCoversKey, "{}")
     val collectionCovers: Map<String, String> by derivedStateOf {
         runCatching { Json.decodeFromString<Map<String, String>>(collectionCoversJson) }.getOrDefault(emptyMap())
diff --git a/shared/src/commonMain/kotlin/com/metrolist/shared/ui/screens/PlaylistContent.kt b/shared/src/commonMain/kotlin/com/metrolist/shared/ui/screens/PlaylistContent.kt
index d3f9796..65009d7 100644
--- a/shared/src/commonMain/kotlin/com/metrolist/shared/ui/screens/PlaylistContent.kt
+++ b/shared/src/commonMain/kotlin/com/metrolist/shared/ui/screens/PlaylistContent.kt
@@ -1,4 +1,4 @@
-@file:OptIn(ExperimentalMaterial3ExpressiveApi::class)
+@file:OptIn(ExperimentalMaterial3ExpressiveApi::class, ExperimentalFoundationApi::class)
 
 package com.metrolist.shared.ui.screens
 
@@ -99,6 +99,9 @@ import org.jetbrains.compose.resources.stringResource
 import sh.calvin.reorderable.ReorderableItem
 import sh.calvin.reorderable.ReorderableLazyListState
 import sh.calvin.reorderable.rememberReorderableLazyListState
+import com.metrolist.shared.model.SongGroup
+import androidx.compose.foundation.ExperimentalFoundationApi
+import androidx.compose.foundation.background
 
 data class PlaylistContentText(
     val play: String,
@@ -158,6 +161,9 @@ fun PlaylistContent(
     isAddingSuggestion: Boolean = false,
     suggestionFooter: (@Composable () -> Unit)? = null,
     downloadedSongIds: Set<String> = emptySet(),
+    songGroups: List<SongGroup>? = null,
+    unknownGroupTitle: String = "",
+    groupByControl: (@Composable () -> Unit)? = null,
     onArtistClick: ((String) -> Unit)? = null,
     header: @Composable (() -> Unit)? = null,
     orderControls: @Composable (() -> Unit)? = null,
@@ -477,27 +483,7 @@ fun PlaylistContent(
                                         emptyContent()
                                     }
                                 } else {
-                                    Box(
-                                        modifier = Modifier.weight(1f).fillMaxWidth(),
-                                    ) {
-                                        DragScrollableColumn(
-                                            state = activeListState,
-                                            // Keep scroll input nodes attached during reorder; the drag handle consumes movement.
-                                            modifier =
-                                                Modifier
-                                                    .fillMaxSize()
-                                                    .padding(end = 16.dp)
-                                                    .desktopArrowScroll(
-                                                        activeListState,
-                                                        enabled = reorderableState?.isAnyItemDragging != true,
-                                                    ),
-                                        ) {
-                                            itemsIndexed(displayedSongs, key = {
-                                                index,
-                                                _,
-                                                ->
-                                                songKeys[index]
-                                            }, contentType = { _, _ -> "song" }) { index, song ->
+                                            val songRow: @Composable (Int, SongItem) -> Unit = { index, song ->
                                                 PlaylistSongItemRow(
                                                     song = song,
                                                     index = index,
@@ -522,6 +508,52 @@ fun PlaylistContent(
                                                     isSelected = isTrackSelected(song),
                                                 )
                                             }
+                                    Box(
+                                        modifier = Modifier.weight(1f).fillMaxWidth(),
+                                    ) {
+                                        DragScrollableColumn(
+                                            state = activeListState,
+                                            // Keep scroll input nodes attached during reorder; the drag handle consumes movement.
+                                            modifier =
+                                                Modifier
+                                                    .fillMaxSize()
+                                                    .padding(end = 16.dp)
+                                                    .desktopArrowScroll(
+                                                        activeListState,
+                                                        enabled = reorderableState?.isAnyItemDragging != true,
+                                                    ),
+                                        ) {
+                                            groupByControl?.let { control ->
+                                                item(key = "group_by_control") { control() }
+                                            }
+                                            if (songGroups == null || reorderMode || displayedIndices != null || songGroups.sumOf { it.songs.size } != displayedSongs.size) {
+                                                itemsIndexed(displayedSongs, key = {
+                                                    index,
+                                                    _,
+                                                    ->
+                                                    songKeys[index]
+                                                }, contentType = { _, _ -> "song" }) { index, song ->
+                                                    songRow(index, song)
+                                                }
+                                            } else {
+                                                var groupStart = 0
+                                                songGroups.forEach { group ->
+                                                    val start = groupStart
+                                                    stickyHeader(key = "group_header_$start") {
+                                                        Text(
+                                                            text = group.key ?: unknownGroupTitle,
+                                                            style = MaterialTheme.typography.titleSmall,
+                                                            color = MaterialTheme.colorScheme.primary,
+                                                            modifier = Modifier.fillMaxWidth().background(MaterialTheme.colorScheme.surface).padding(horizontal = 16.dp, vertical = 8.dp),
+                                                        )
+                                                    }
+                                                    group.songs.indices.forEach { offset ->
+                                                        val index = start + offset
+                                                        item(key = songKeys[index], contentType = "song") { songRow(index, displayedSongs[index]) }
+                                                    }
+                                                    groupStart += group.songs.size
+                                                }
+                                            }
 
                                             if (paginationContent != null) {
                                                 item(key = "pagination") { paginationContent() }
diff --git a/shared/src/commonTest/kotlin/com/metrolist/shared/model/SongGroupingTest.kt b/shared/src/commonTest/kotlin/com/metrolist/shared/model/SongGroupingTest.kt
new file mode 100644
index 0000000..f8e0db5
--- /dev/null
+++ b/shared/src/commonTest/kotlin/com/metrolist/shared/model/SongGroupingTest.kt
@@ -0,0 +1,60 @@
+package com.metrolist.shared.model
+
+import kotlin.test.Test
+import kotlin.test.assertEquals
+import kotlin.test.assertNull
+
+class SongGroupingTest {
+    private fun song(
+        id: String,
+        artist: String?,
+        album: String?,
+    ) = SongItem(
+        id = id,
+        title = "Song $id",
+        thumbnail = null,
+        artists = listOfNotNull(artist?.let { ArtistTiny(id = null, name = it) }),
+        album = album?.let { AlbumTiny(id = null, name = it) },
+    )
+
+    private val songs =
+        listOf(
+            song("1", "queen", "A Night at the Opera"),
+            song("2", null, null),
+            song("3", "Abba", "Gold"),
+            song("4", "Queen", "Greatest Hits"),
+            song("5", "Abba", ""),
+        )
+
+    @Test
+    fun noneReturnsOneUngroupedListInOrder() {
+        val groups = songs.groupedBy(SongGroupBy.NONE)
+
+        assertEquals(1, groups.size)
+        assertNull(groups[0].key)
+        assertEquals(listOf("1", "2", "3", "4", "5"), groups[0].songs.map { it.id })
+    }
+
+    @Test
+    fun artistGroupsIgnoreCaseSortAndPutUnknownLast() {
+        val groups = songs.groupedBy(SongGroupBy.ARTIST)
+
+        assertEquals(listOf("Abba", "queen", null), groups.map { it.key })
+        assertEquals(listOf("3", "5"), groups[0].songs.map { it.id })
+        assertEquals(listOf("1", "4"), groups[1].songs.map { it.id })
+        assertEquals(listOf("2"), groups[2].songs.map { it.id })
+    }
+
+    @Test
+    fun albumGroupsTreatBlankNamesAsUnknown() {
+        val groups = songs.groupedBy(SongGroupBy.ALBUM)
+
+        assertEquals(listOf("A Night at the Opera", "Gold", "Greatest Hits", null), groups.map { it.key })
+        assertEquals(listOf("2", "5"), groups.last().songs.map { it.id })
+    }
+
+    @Test
+    fun emptyListProducesNoGroups() {
+        assertEquals(emptyList(), emptyList<SongItem>().groupedBy(SongGroupBy.ARTIST))
+    }
+}
-- 
2.43.0

===== END PATCH =====
