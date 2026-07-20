# GAME STATE MACHINE — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Dokumen Desain Sistem**: Mesin Status Aplikasi & Mesin Sub-Status Pertarungan
> **Versi**: 1.0 (Canonical)
> **Penulis Peran**: Game System Designer
> **Platform**: Web Desktop-First + Progressive Web App (PWA)
> **Engine**: Phaser 3 + TypeScript + Vite
> **Bahasa Dokumen**: Indonesia (semua isi)

---

## 0. Ringkasan Eksekutif & Asumsi Eksplisit

Dokumen ini mendefinisikan **Finite State Machine (FSM)** untuk aplikasi game RPG *Fantasy Turn-Based Idle*. FSM dibagi menjadi dua lapisan:

1. **App-Level FSM** — lapisan tertinggi yang mengatur layar/state aplikasi (Boot, MainMenu, Battle, Gacha, dst).
2. **Battle Sub-State Machine** — lapisan lokal yang hanya aktif saat berada di state `Battle`, mengatur alur satu pertarungan (turn order, aksi, resolve, status tick, win check).

Serta lapisan ketiga transversal: **Global/System States** (Online / Offline / Maintenance / Paused) yang memodifikasi perilaku seluruh app state.

### Asumsi Eksplisit (wajib konsisten dengan CDP)

| # | Asumsi | Nilai / Keterangan |
|---|---------|--------------------|
| A1 | Party maksimum | 5 karakter aktif per pertarungan |
| A2 | Rarity | Common (C), Rare (R), Epic (E), Legendary (L); Mythic (M) sebagai tier rahasia/limited |
| A3 | Gacha pity | Soft pity pada pull ke-75, hard pity pada pull ke-90; guarantee rarity L pada 90; pity C breakthrough di 10 |
| A4 | Idle tick | Setiap 5 detik (5dtk) server menghitung akrual reward |
| A5 | Idle cap | Maksimal 24 jam akumulasi idle per sesi offline/away |
| A6 | Item categories | Gear, Consumable, Material |
| A7 | PvP ELO awal | 1000 (ELO standar) |
| A8 | Currencies | Gold, Gems, EventToken |
| A9 | Combat authority | **Server-authoritative**; client hanya mengirim intent, server menghitung hasil |
| A10 | Turn order | Berdasarkan stat `speed` (dekselarasi: speed tinggi duluan) |
| A11 | Resource sistem | `mana` (skill) & `energy` (aktivitas idle/auto) |
| A12 | Status effects | Tepat 8 jenis (lihat §5.6) |
| A13 | Elemen | 6 elemen; multiplier 1.5x (strong) / 0.75x (weak) |
| A14 | Auth | Email + password (session token, refresh token) |
| A15 | Engine | Phaser 3 render, TypeScript logic, Vite bundler |
| A16 | PWA | Installable, service worker, offline claim, background sync |
| A17 | Tick loop | Game loop Phaser (`scene.update`) dijeda saat `Paused`/`Maintenance`/`Offline-sync` |
| A18 | Persistensi | `last_seen` disimpan setiap transisi state untuk keperluan idle accrual |

---

## 1. Konvensi & Notasi

Bagian ini adalah **standar penulisan wajib** agar seluruh diagram dan tabel dapat dibaca konsisten oleh engineer maupun desainer.

### 1.1 Komponen Diagram

Setiap diagram menggunakan 4 primitif:

| Simbol | Nama | Arti |
|--------|------|------|
| `[ STATE ]` | Node State | Kondisi stabil aplikasi/sub-sistem |
| `-->` | Transition | Transisi antar state |
| `( event )` | Trigger / Event | Peristiwa yang memicu transisi (kejadian eksternal/interval) |
| `[ guard ]` | Guard / Kondisi | Predikat boolean; transisi hanya terjadi bila `true` |
| `{ action }` | On-Enter / On-Exit Action | Efek samping saat masuk/keluar state |

### 1.2 Format Penulisan Transition

Notasi kanonik dalam teks:

```
[ FROM ] --( event )[ guard ]--> [ TO ]   { onEnter: ..., onExit: ... }
```

Contoh:

```
[ MainMenu ] --( pilihStage )--> [ StageSelect ]   { onEnter: loadStageList(); onExit: null; }
[ Battle ] --( win )[ partyAlive ]--> [ Reward ]   { onEnter: computeReward(); }
```

### 1.3 Hierarki State

- **L0 — Global/System**: `Online`, `Offline`, `Maintenance`, `Paused`. Bersifat *orthogonal* (dapat menempel pada state L1 mana pun).
- **L1 — App-Level**: 19 state utama (Boot, Preload, AuthGate, MainMenu, dst).
- **L2 — Battle Sub-FSM**: 12 sub-state, hanya valid saat L1 = `Battle`.

### 1.4 Notasi Status Efek & Elemen

- Status effect ditulis `SE#` (SE1..SE8).
- Elemen ditulis singkatan: `FI`(Fire), `WA`(Water), `WO`(Wood), `EA`(Earth), `LI`(Light), `DA`(Dark).
- Multiplier elemen: `1.5x` = elemen menyerang *strong* terhadap target; `0.75x` = *weak*.
- Rarity shorthand: `C R E L M`.

### 1.5 Konvensi Penamaan

| Kelas | Konvensi | Contoh |
|-------|----------|--------|
| State | `PascalCase` dalam kurung siku | `[ Battle ]` |
| Event/Trigger | `camelCase` dalam kurung | `( tapStart )` |
| Guard | `camelCase` dalam kurung siku | `[ hasTicket ]` |
| Action | `camelCase()` dalam kurung kurawal | `{ saveState() }` |
| Currency | Kapital | `Gold`, `Gems`, `EventToken` |

### 1.6 Simbol Khusus di Diagram ASCII

```
( )  = decision / guard branch
[ ]  = state node
-->  = normal transition
==>  = transition wajib / forced (tidak bisa dibatalkan)
<< >> = system state overlay (global)
```

---

## 2. App-Level Finite State Machine

Daftar 19 state aplikasi dengan deskripsi, aset dimuat, dan aksi masuk/keluar.

### 2.1 `Boot`

- **Tujuan**: Inisialisasi runtime PWA, deteksi capability (service worker, indexedDB, network).
- **Aset dimuat**: `index.html`, `manifest.json`, Phaser loader, util intrinsik.
- **On-Enter**: `initRuntime()`, `registerSW()`, `detectNetwork()`.
- **On-Exit**: `bootDone = true`.
- **Transisi**: → `Preload` (selalu).

### 2.2 `Preload`

- **Tujuan**: Memuat aset global (sprite, audio atlas, font, config JSON dari server/CDN).
- **Aset dimuat**: Texture atlas, BGM, SFX, `gameConfig.json`, schema item.
- **On-Enter**: `loadGlobalAssets()`, tampilkan progress bar.
- **On-Exit**: `cacheWarm()`, simpan `last_seen`.
- **Transisi**: → `AuthGate` (aset siap).

### 2.3 `AuthGate`

- **Tujuan**: Menentukan apakah sesi valid (token) → `MainMenu`, atau perlu login → `Login` (sub-flow dalam AuthGate).
- **Aset dimuat**: Form login, logo.
- **On-Enter**: `checkSessionToken()`.
- **On-Exit**: `setAuthContext()`.
- **Guard utama**: `[ sessionValid ]` → MainMenu; `[ !sessionValid ]` → Login form.
- **Transisi**: → `MainMenu` / `Maintenance`.

### 2.4 `MainMenu`

- **Tujuan**: Hub navigasi utama (lobby). Pintu ke semua fitur.
- **Aset dimuat**: Lobby bg, tombol navigasi, widget currency, widget event aktif.
- **On-Enter**: `loadPlayerSnapshot()`, `startIdleAccrualCheck()` (klaim otomatis bila ada), `renderCurrencyHUD()`.
- **On-Exit**: `saveState()`, `last_seen = now()`.
- **Transisi**: → `Home/IdleMap`, `StageSelect`, `Gacha`, `Inventory`, `Marketplace`, `Guild`, `Raid`, `PvP`, `Leaderboard`, `Events`, `Settings`, `OfflineClaim`.

### 2.5 `Home/IdleMap`

- **Tujuan**: Peta idle; karakter party "berjalan" otomatis, menghasilkan resource via tick 5dtk.
- **Aset dimuat**: World map tile, party sprites (max 5), node enemy/idle.
- **On-Enter**: `startIdleLoop(5s)`, `syncIdleProgress()`.
- **On-Exit**: `stopIdleLoop()`, `persistIdleState()`.
- **Transisi**: → `StageSelect` (pilih node), `Battle` (encounter), `MainMenu`.

### 2.6 `StageSelect`

- **Tujuan**: Memilih stage/chapter sebelum masuk pertarungan.
- **Aset dimuat**: Peta chapter, daftar stage, lock state, rekomendasi party.
- **On-Enter**: `loadStageList()`, `loadRecommendedParty()`.
- **On-Exit**: `clearStagePreview()`.
- **Guard**: `[ stageUnlocked ]` → Battle; `[ !stageUnlocked ]` → tetap / notif.
- **Transisi**: → `Battle`, `MainMenu`.

### 2.7 `Battle`

- **Tujuan**: Pertarungan turn-based. Menjalankan **Battle Sub-FSM** (lihat §5).
- **Aset dimuat**: Battle scene, skill VFX, party & enemy sprites, HUD combat.
- **On-Enter**: `enterBattle()`, `initCombatState()`, handoff ke server-authoritative.
- **On-Exit**: `exitBattle()`, `flushCombatLog()`.
- **Transisi**: → `Reward` (win), `MainMenu` (defeat/forfeit), `Error/Maintenance`.

### 2.8 `Gacha`

- **Tujuan**: Tarik karakter/gear via sistem pity.
- **Aset dimuat**: Banner asset, animation pull, rarity table.
- **On-Enter**: `loadBanner()`, `loadPityCounter()`.
- **On-Exit**: `savePity()`, `persistInventory()`.
- **Guard**: `[ gems >= cost ]` untuk single/pull.
- **Transisi**: → `Reward` (hasil pull), `Inventory`, `MainMenu`.

### 2.9 `Inventory`

- **Tujuan**: Kelola Gear/Consumable/Material, equip/unequip, consume.
- **Aset dimuat**: Grid item, tooltip, preview stat.
- **On-Enter**: `loadInventory()`, `loadEquipState()`.
- **On-Exit**: `persistEquip()`, `recalcStats()`.
- **Transisi**: → `MainMenu`, `Marketplace` (jual), `Battle` (equip lalu balik).

### 2.10 `Marketplace`

- **Tujuan**: Jual/beli item dengan Gold/Gems; listing player & NPC vendor.
- **Aset dimuat**: Katalog listing, harga realtime.
- **On-Enter**: `fetchListings()`, `loadWallet()`.
- **On-Exit**: `persistWallet()`, `persistInventory()`.
- **Guard**: `[ balanceOk ]` saat beli.
- **Transisi**: → `Inventory`, `MainMenu`.

### 2.11 `Guild`

- **Tujuan**: Manajemen guild, donation, guild raid queue, chat.
- **Aset dimuat**: Guild board, member list, donation UI.
- **On-Enter**: `loadGuild()`, `loadGuildBuffs()`.
- **On-Exit**: `persistGuildContrib()`.
- **Transisi**: → `Raid` (guild raid), `MainMenu`.

### 2.12 `Raid`

- **Tujuan**: Pertarungan kooperatif / boss besar (bisa sekuel ke Battle sub-FSM dengan mode raid).
- **Aset dimuat**: Raid boss sprite, multi-party layout.
- **On-Enter**: `enterRaid()`, `syncRaidState()`.
- **On-Exit**: `exitRaid()`.
- **Transisi**: → `Reward`, `Guild`, `MainmainMenu`(error).

### 2.13 `PvP`

- **Tujuan**: Pertarungan vs pemain lain; ELO 1000 awal, matchmaking by ELO.
- **Aset dimuat**: Arena scene, opponent snapshot, ELO board mini.
- **On-Enter**: `findMatch(ELO)`, `enterBattle(mode=pvp)`.
- **On-Exit**: `submitResult(ELO_delta)`.
- **Transisi**: → `Battle` (mode pvp), `Leaderboard`, `MainMenu`.

### 2.14 `Leaderboard`

- **Tujuan**: Peringkat global/seasonal (PvP ELO, idle power, guild).
- **Aset dimuat**: Rank list, avatar top, filter season.
- **On-Enter**: `fetchLeaderboard()`.
- **On-Exit**: `cacheLeaderboard()`.
- **Transisi**: → `PvP`, `MainMenu`.

### 2.15 `Events`

- **Tujuan**: Event terbatas waktu; menghasilkan `EventToken`.
- **Aset dimuat**: Event banner, quest tracker, reward track.
- **On-Enter**: `loadEventState()`, `loadEventToken()`.
- **On-Exit**: `persistEventProgress()`.
- **Transisi**: → `Battle` (event stage), `MainMenu`.

### 2.16 `Settings`

- **Tujuan**: Preferensi audio, grafis, akun, logout.
- **Aset dimuat**: Panel setting, toggle.
- **On-Enter**: `loadSettings()`.
- **On-Exit**: `persistSettings()`, `applySettings()`.
- **Transisi**: → `AuthGate` (logout), `MainMenu`.

### 2.17 `OfflineClaim`

- **Tujuan**: Layar klaim reward idle saat kembali online/dari background (cap 24j).
- **Aset dimuat**: Reward summary, animasi koin.
- **On-Enter**: `computeOfflineReward(last_seen, now, 24h_cap)`.
- **On-Exit**: `grantReward()`, `clearOfflineFlag()`.
- **Guard**: `[ elapsed > 0 ]`.
- **Transisi**: → `MainMenu`, `Home/IdleMap`.

### 2.18 `Error/Maintenance`

- **Tujuan**: Tangkap error fatal atau mode maintenance server.
- **Aset dimuat**: Error screen / maintenance notice.
- **On-Enter**: `logError()`, `pauseGameLoop()`.
- **On-Exit**: `resumeOrRestart()`.
- **Transisi**: → `Boot` (restart), `AuthGate` (re-auth), `MainMenu` (recover).

### 2.19 `Loading`

- **Tujuan**: Overlay transisi umum antar state berat (bukan state mandiri persisten).
- **Aset dimuat**: Spinner, tip.
- **On-Enter**: `showLoading()`.
- **On-Exit**: `hideLoading()`.
- **Transisi**: overlay → state tujuan.

> **Catatan YAGNI**: `Loading` diperlakukan sebagai *overlay* bukan state L1 penuh bila transisi singkat; menjadi state eksplisit bila load > 800ms.

---

## 3. Transition Table (App-Level)

Format kolom:
`From → To | Trigger/Event | Guard | On-Enter Action | On-Exit Action`

| # | From | To | Trigger / Event | Guard | On-Enter Action | On-Exit Action |
|---|------|----|-----------------|-------|-----------------|----------------|
| T01 | Boot | Preload | bootComplete | always | `loadGlobalAssets()` | `bootDone=true` |
| T02 | Preload | AuthGate | assetsReady | `cacheWarm` | `checkSessionToken()` | `saveState()` |
| T03 | AuthGate | MainMenu | sessionValid | `[ sessionValid ]` | `loadPlayerSnapshot()`, `startIdleAccrualCheck()` | `setAuthContext()` |
| T04 | AuthGate | AuthGate(Login) | sessionInvalid | `[ !sessionValid ]` | `showLoginForm()` | — |
| T05 | AuthGate | Maintenance | maintenanceFlag | `[ serverMaintenance ]` | `showMaintScreen()` | — |
| T06 | MainMenu | Home/IdleMap | openMap | always | `startIdleLoop(5s)` | `saveState()`, `last_seen=now()` |
| T07 | MainMenu | StageSelect | openStages | always | `loadStageList()` | `saveState()` |
| T08 | MainMenu | Gacha | openGacha | always | `loadBanner()`, `loadPityCounter()` | `saveState()` |
| T09 | MainMenu | Inventory | openInventory | always | `loadInventory()` | `saveState()` |
| T10 | MainMenu | Marketplace | openMarket | always | `fetchListings()` | `saveState()` |
| T11 | MainMenu | Guild | openGuild | always | `loadGuild()` | `saveState()` |
| T12 | MainMenu | Raid | openRaid | `[ guildJoined ]` | `enterRaid()` | `saveState()` |
| T13 | MainMenu | PvP | openPvP | always | `findMatch(ELO)` | `saveState()` |
| T14 | MainMenu | Leaderboard | openBoard | always | `fetchLeaderboard()` | `saveState()` |
| T15 | MainMenu | Events | openEvents | always | `loadEventState()` | `saveState()` |
| T16 | MainMenu | Settings | openSettings | always | `loadSettings()` | `saveState()` |
| T17 | MainMenu | OfflineClaim | appResume/return | `[ elapsed>0 ]` | `computeOfflineReward()` | `saveState()` |
| T18 | Home/IdleMap | StageSelect | pickNode | always | `loadStageList()` | `stopIdleLoop()` |
| T19 | Home/IdleMap | Battle | encounter | always | `enterBattle()` | `stopIdleLoop()`, `persistIdleState()` |
| T20 | Home/IdleMap | MainMenu | back | always | `loadPlayerSnapshot()` | `persistIdleState()` |
| T21 | StageSelect | Battle | startBattle | `[ stageUnlocked ]` | `enterBattle()` | `clearStagePreview()` |
| T22 | StageSelect | MainMenu | back | always | — | `saveState()` |
| T23 | Battle | Reward | win | `[ partyAlive ]` | `computeReward()` | `exitBattle()` |
| T24 | Battle | MainMenu | defeat | `[ !partyAlive ]` | `loadPlayerSnapshot()` | `exitBattle()` |
| T25 | Battle | Error/Maintenance | serverError | `[ combatFault ]` | `logError()` | `exitBattle()` |
| T26 | Battle | MainMenu | forfeit | always | — | `exitBattle()` |
| T27 | Gacha | Reward | pull | `[ gems>=cost ]` | `rollBanner()`, `computeReward()` | `savePity()` |
| T28 | Gacha | Inventory | viewResult | always | `loadInventory()` | `savePity()` |
| T29 | Gacha | MainMenu | back | always | — | `savePity()` |
| T30 | Inventory | Marketplace | sellItem | `[ itemSelected ]` | `fetchListings()` | `persistEquip()` |
| T31 | Inventory | MainMenu | back | always | — | `persistEquip()`, `recalcStats()` |
| T32 | Marketplace | Inventory | buyConfirm | `[ balanceOk ]` | `loadInventory()` | `persistWallet()` |
| T33 | Marketplace | MainMenu | back | always | — | `persistWallet()` |
| T34 | Guild | Raid | startGuildRaid | `[ membersReady ]` | `enterRaid()` | `persistGuildContrib()` |
| T35 | Guild | MainMenu | back | always | — | `persistGuildContrib()` |
| T36 | Raid | Reward | raidWin | `[ bossDefeated ]` | `computeReward()` | `exitRaid()` |
| T37 | Raid | Guild | raidFail | `[ !bossDefeated ]` | — | `exitRaid()` |
| T38 | PvP | Battle | matchFound | `[ opponentAssigned ]` | `enterBattle(mode=pvp)` | `saveState()` |
| T39 | PvP | Leaderboard | viewRank | always | `fetchLeaderboard()` | `saveState()` |
| T40 | PvP | MainMenu | back | always | — | `saveState()` |
| T41 | Leaderboard | PvP | challenge | always | `findMatch(ELO)` | `cacheLeaderboard()` |
| T42 | Leaderboard | MainMenu | back | always | — | `cacheLeaderboard()` |
| T43 | Events | Battle | eventStage | `[ eventActive ]` | `enterBattle(mode=event)` | `persistEventProgress()` |
| T44 | Events | MainMenu | back | always | — | `persistEventProgress()` |
| T45 | Settings | AuthGate | logout | always | `clearSession()` | `persistSettings()` |
| T46 | Settings | MainMenu | back | always | `applySettings()` | `persistSettings()` |
| T47 | OfflineClaim | MainMenu | claim | `[ elapsed>0 ]` | `grantReward()` | `clearOfflineFlag()` |
| T48 | OfflineClaim | Home/IdleMap | claimAndMap | `[ elapsed>0 ]` | `grantReward()`, `startIdleLoop()` | `clearOfflineFlag()` |
| T49 | Reward | MainMenu | continue | always | `loadPlayerSnapshot()` | `saveState()` |
| T50 | Reward | Inventory | viewLoot | always | `loadInventory()` | `saveState()` |
| T51 | Error/Maintenance | Boot | restart | always | `initRuntime()` | — |
| T52 | Error/Maintenance | AuthGate | reauth | `[ tokenExpired ]` | `clearSession()` | — |
| T53 | Error/Maintenance | MainMenu | recover | `[ recoverable ]` | `loadPlayerSnapshot()` | `resumeGameLoop()` |
| T54 | * (any) | Loading | heavyTransit | always | `showLoading()` | `hideLoading()` |
| T55 | * (any L1) | Error/Maintenance | fatalError | `[ !recoverable ]` | `logError()`, `pauseGameLoop()` | — |

> **Keterangan Guard**: `[ partyAlive ]` = setidaknya 1 dari 5 karakter HP>0. `[ bossDefeated ]` = HP boss = 0. `[ combatFault ]` = server mengembalikan error kritis. `[ serverMaintenance ]` = flag maintenance aktif dari endpoint `/status`.

---

## 4. App FSM Diagram (ASCII)

Berikut visualisasi alur transisi utama (L1). Overlay sistem (L0) ditandai `<< >>`.

```
                          << Online / Offline / Maintenance / Paused >>

 +--------------------------------------------------------------+
 |                          [ Boot ]                            |
 +--------------------------------------------------------------+
                         | bootComplete
                         v
 +--------------------------------------------------------------+
 |                        [ Preload ]                           |
 +--------------------------------------------------------------+
                         | assetsReady
                         v
 +--------------------------------------------------------------+
 |                      [ AuthGate ]                            |
 +--------------------------------------------------------------+
            | sessionValid               | !sessionValid     | serverMaintenance
            v                            v                   v
      [ MainMenu ] <--------------+  (Login form)        [ Error/Maintenance ]
            |                                     ^                |
            |                                     | recover/      | restart
            |                                     | reauth        v
            |                          +--------+  +----------+  [ Boot ]
            |                          |        |             |
   +--------+--------+--------+-------+--------+-------------+
   |        |        |        |       |        |             |
   v        v        v        v       v        v             v
[Home/   [Stage-  [Gacha] [Inven- [Market- [Guild]    [Raid]
 IdleMap] Select]         tory]   place]
   |  |     |          |       |        |         |           |
   |  |     |start     |pull   |sell    |buy      |start     |raidWin
   |  |     v          v       v        v         v           v
   |  |  [Battle]<--[Reward]<[Inven- [Inven-  [Reward]<---+
   |  |     |  win   |        tory]   tory]
   |  |     |defeat  |continue
   |  |     v         v
   |  |  [MainMenu] [MainMenu]
   |  |
   |  |  +-----[ PvP ]----> [Battle mode=pvp] ---> [Reward]
   |  |  |     |                                        |
   |  |  +---->[Leaderboard]<----+                       |
   |  |        |  challenge      | viewRank              |
   |  |        +-----------------+                       |
   |  |  +-----[ Events ]----> [Battle mode=event]       |
   |  |  |     |                                        |
   |  +--+     +----------------------------------------+
   |  |
   +--+--back---> [ MainMenu ]
   |
 +--------------------------------------------------------------+
 |  [ Settings ] --> logout --> [ AuthGate ]                   |
 |  [ Settings ] --> back   --> [ MainMenu ]                   |
 +--------------------------------------------------------------+
 +--------------------------------------------------------------+
 |  [ OfflineClaim ] --> claim --> [ MainMenu ] / [ Home ]      |
 +--------------------------------------------------------------+
 +--------------------------------------------------------------+
 |  [ Loading ] = overlay transisi berat (T54)                 |
 +--------------------------------------------------------------+

 Legend:
   --> normal transition
   (X) trigger/event label
   [ State ] node
   << >> global system overlay
```

### 4.1 Diagram Kompak (Satu Baris per Jalur Utama)

```
Boot → Preload → AuthGate →(valid) MainMenu
MainMenu → Home/IdleMap →(encounter) Battle →(win) Reward → MainMenu
MainMenu → StageSelect →(start) Battle →(defeat) MainMenu
MainMenu → Gacha →(pull) Reward → MainMenu
MainMenu → Inventory ⇄ Marketplace
MainMenu → Guild →(raid) Raid →(win) Reward
MainMenu → PvP →(match) Battle(pvp) → Reward
MainMenu → Leaderboard ⇄ PvP
MainMenu → Events →(stage) Battle(event) → Reward
MainMenu → Settings →(logout) AuthGate
MainMenu →(resume) OfflineClaim → MainMenu
Any →(fatal) Error/Maintenance → Boot/AuthGate/MainMenu
```

---

## 5. Battle Sub-State Machine

Hanya aktif saat L1 = `Battle`. Menjalankan satu pertarungan secara server-authoritative.

### 5.1 Daftar Sub-State

| Kode | Sub-State | Tujuan |
|------|-----------|--------|
| B0 | `Init` | Setup combat: snapshot party, enemy, elemen, status awal |
| B1 | `Intro` | Animasi masuk, teks "Battle Start", deploy VFX |
| B2 | `TurnOrder` | Hitung urutan turn by speed (dekselarasi) |
| B3 | `PlayerAction` | Tunggu intent pemain (atau auto dari energy) |
| B4 | `EnemyAction` | Server kirim aksi musuh |
| B5 | `Resolve` | Server hitung damage (elemen 1.5/0.75, status), apply |
| B6 | `StatusTick` | Tick 8 status effect (duration--, dot, dll) |
| B7 | `WinCheck` | Evaluasi kondisi menang/kalah |
| B8 | `NextTurn` | Lanjut ke unit berikutnya di turn order |
| B9 | `Victory` | Pihak pemain menang |
| B10 | `Defeat` | Pihak pemain kalah |
| B11 | `Reward` | (lift ke L1 Reward) grant loot |
| B12 | `Exit` | Kembali ke L1 caller (MainMenu/Home/Stage) |

### 5.2 Alur Utama

```
[ Init ] →(setupDone)→ [ Intro ] →(animDone)→ [ TurnOrder ]
                                                   |
                                                   v
                                            [ PlayerAction ] ←───┐
                                                   | intentSent
                                                   v
                                            [ EnemyAction ]  (jika musuh di urutan)
                                                   | serverAction
                                                   v
                                              [ Resolve ]
                                                   | applied
                                                   v
                                            [ StatusTick ]
                                                   | ticked
                                                   v
                                              [ WinCheck ]
                                            /     |      \
                                  partyAlive   !party   bossBoth?
                                     |        |          |
                                     v        v          v
                                [NextTurn] [Defeat]  [Victory]
                                     |                   |
                                     v                   v
                              (kembali ke            [Exit] → (lift)
                               PlayerAction           Reward(L1)
                               untuk unit berikut)
```

### 5.3 Battle Transition Table

| # | From | To | Trigger / Event | Guard | On-Enter Action | On-Exit Action |
|---|------|----|-----------------|-------|-----------------|----------------|
| BT01 | Init | Intro | setupDone | `[ combatReady ]` | `deployVFX()` | `snapshotCombat()` |
| BT02 | Intro | TurnOrder | animDone | always | `computeTurnOrder()` | — |
| BT03 | TurnOrder | PlayerAction | isPlayerTurn | `[ nextUnit.isPlayer ]` | `awaitIntent()` | — |
| BT04 | TurnOrder | EnemyAction | isEnemyTurn | `[ nextUnit.isEnemy ]` | `requestServerAction()` | — |
| BT05 | PlayerAction | EnemyAction | intentSent | `[ turnHasEnemy ]` | `requestServerAction()` | `sendIntent()` |
| BT06 | PlayerAction | Resolve | intentSent | `[ noEnemyLeft ]` | `sendIntent()` | `lockInput()` |
| BT07 | EnemyAction | Resolve | serverAction | always | `applyEnemyAction()` | — |
| BT08 | Resolve | StatusTick | applied | always | `applyDamage()` | `unlockInput()` |
| BT09 | StatusTick | WinCheck | ticked | always | `decrementDurations()` | — |
| BT10 | WinCheck | NextTurn | partyAlive | `[ partyAlive && enemyAlive ]` | `advancePointer()` | — |
| BT11 | WinCheck | Defeat | !partyAlive | `[ !partyAlive ]` | `playDefeat()` | — |
| BT12 | WinCheck | Victory | !enemyAlive | `[ !enemyAlive ]` | `playVictory()` | — |
| BT13 | NextTurn | TurnOrder | loop | always | — | — |
| BT14 | Victory | Exit | animDone | always | `flagVictory()` | — |
| BT15 | Defeat | Exit | animDone | always | `flagDefeat()` | — |
| BT16 | Exit | Reward (L1) | lift | `[ victory ]` | `computeReward()` | `flushCombatLog()` |
| BT17 | Exit | MainMenu (L1) | lift | `[ defeat ]` | `loadPlayerSnapshot()` | `flushCombatLog()` |

> **Catatan**: `NextTurn` tidak selalu kembali ke `TurnOrder` penuh; jika urutan sudah dihitung di B2, `NextTurn` cukup `advancePointer()` lalu cabang ke `PlayerAction`/`EnemyAction` berdasar owner unit. `TurnOrder` di-recompute hanya saat ada perubahan speed (buff/debuff SE yang memengaruhi speed).

### 5.4 Turn Order by Speed — Algoritma

1. Kumpulkan semua unit (party max 5 + enemy N).
2. Urutkan **dekselarasi** berdasar stat `speed` (tinggi dulu). Tie-break: player dulu, lalu ID terendah.
3. Simpan sebagai `turnQueue[]`.
4. Pointer `i` berjalan siklikal. Setiap unit mendapat 1 aksi per siklus.
5. Jika unit mati (HP=0) saat `StatusTick`, lewati (skip) di `NextTurn`.
6. Recompute `turnQueue` bila ada SE yang mengubah `speed` (mis. SE slow/defense).

```
speed: A=120 B=120 C=90 D=80 E=70 F(enemy)=110
queue: A, B, F, C, D, E   (tie A,B -> player dulu)
```

### 5.5 Status Tick — 8 Status Effect (SE1..SE8)

| # | Kode | Nama | Tipe | Efek per Tick | Durasi |
|---|------|------|------|---------------|--------|
| SE1 | `BURN` | Terbakar | DoT | -X% maxHP/tick (elemen FI) | 3 turn |
| SE2 | `POISON` | Racun | DoT | -Y flat/tick (ignores def) | 4 turn |
| SE3 | `STUN` | Pusing | Control | skip turn | 1 turn |
| SE4 | `SLOW` | Lambat | Debuff | speed -30% (recompute queue) | 2 turn |
| SE5 | `SHIELD` | Perisai | Buff | absorb Z dmg | 2 turn |
| SE6 | `ATKUP` | Str Up | Buff | atk +25% | 3 turn |
| SE7 | `REGEN` | Regenerasi | HoT | +W% maxHP/tick | 3 turn |
| SE8 | `SILENCE` | Bisu | Control | tidak bisa skill (mana locked) | 2 turn |

Aturan `StatusTick` (B6):
- Untuk tiap unit: kurangi durasi tiap SE aktif.
- Terapkan DoT/HoT (SE1, SE2, SE7) ke HP.
- Bila `SHIELD` (SE5) habis, hapus absorb.
- Bila `SLOW` (SE4) habis, **recompute turnQueue**.
- Hapus SE dengan durasi 0.
- `STUN`/`SILENCE` dievaluasi saat giliran unit (lihat B3/B4), bukan di tick.

### 5.6 Elemen & Multiplier

Enam elemen: `FI WA WO EA LI DA`. Matriks relasi (baris = penyerang, kolom = target):

| Atk \ Def | FI | WA | WO | EA | LI | DA |
|-----------|----|----|----|----|----|----|
| **FI** | 1.0 | 1.5 | 0.75 | 1.0 | 1.0 | 1.0 |
| **WA** | 0.75 | 1.0 | 1.5 | 1.0 | 1.0 | 1.0 |
| **WO** | 1.5 | 0.75 | 1.0 | 1.0 | 1.0 | 1.0 |
| **EA** | 1.0 | 1.0 | 1.0 | 1.0 | 0.75 | 1.5 |
| **LI** | 1.0 | 1.0 | 1.0 | 1.5 | 1.0 | 0.75 |
| **DA** | 1.0 | 1.0 | 1.0 | 0.75 | 1.5 | 1.0 |

Rumus damage (server-authoritative):
```
base = atk * skillPower
elem = ELEM_MATRIX[atkElem][defElem]   // 1.5 / 1.0 / 0.75
mitig = def / (def + K)
dmg  = floor(base * elem * (1 - mitig) * rand(0.95,1.05))
if SHIELD active: dmg = max(0, dmg - absorb); absorb -= dmg
```

### 5.7 Resource: Mana & Energy

- **Mana**: dipakai skill aktif. Regenerasi +M per turn. `SILENCE` (SE8) mengunci mana (tidak bisa skill).
- **Energy**: sistem idle/auto. Saat `Home/IdleMap`, energy terisi via tick 5dtk. Di Battle, energy tinggi → mode **auto-battle** (client kirim intent otomatis, server tetap hitung).
- Guard `PlayerAction`: bila `energy >= autoThreshold` → langsung `intentSent` (auto); else tunggu input pemain.

### 5.8 Diagram ASCII Battle (Detail)

```
                         +-------------------+
                         |      [ Init ]     |
                         +-------------------+
                                  | setupDone [combatReady]
                                  v
                         +-------------------+
                         |     [ Intro ]     |
                         +-------------------+
                                  | animDone
                                  v
                    +-------------------------+
                    |     [ TurnOrder ]       |<-------------+
                    +-------------------------+              |
                         |                                   |
            +------------+------------+                      |
            | isPlayerTurn| isEnemyTurn|                      |
            v                         v                       |
   +------------------+      +------------------+             |
   | [ PlayerAction ] |      | [ EnemyAction ]  |             |
   +------------------+      +------------------+             |
            | intentSent (auto jika energy)       | serverAction
            +------------------+------------------+           |
                               |                              |
                               v                              |
                         +------------------+                |
                         |    [ Resolve ]   |                |
                         +------------------+                |
                               | applied                      |
                               v                              |
                         +------------------+                |
                         |  [ StatusTick ]  |                |
                         +------------------+                |
                               | ticked                       |
                               v                              |
                         +------------------+                |
                         |   [ WinCheck ]   |                |
                         +------------------+                |
                  /          |            \                   |
        partyAlive      !partyAlive    !enemyAlive           |
             |              |              |                  |
             v              v              v                 |
      +-----------+   +----------+   +-----------+           |
      | NextTurn  |   | Defeat   |   | Victory   |           |
      +-----------+   +----------+   +-----------+           |
             |              |              |                 |
             |              |              +--> [ Exit ] ----+
             |              |                                |
             +--> (advancePointer) --> kembali ke TurnOrder-+
                            |
                      [ Exit ] (defeat path) --> MainMenu(L1)
```

---

## 6. Global / System States

Lapisan L0 yang orthogonal terhadap L1. Dapat menempel pada state L1 mana pun.

### 6.1 `Online`

- **Default**. Koneksi aktif, sinkronisasi server berjalan.
- **Interaksi**: Semua transisi L1 diizinkan; idle accrual dihitung server.
- **Trigger masuk**: `networkOnline` event.

### 6.2 `Offline` (PWA)

- **Tujuan**: Saat PWA tidak ada koneksi. Service worker menyajikan cache; aksi lokal di-queue (background sync).
- **Interaksi**:
  - Game loop tetap jalan untuk UI lokal, tapi transisi yang butuh server (Gacha, Marketplace, PvP) **diblokir** dengan guard `[ online ]`.
  - Idle accrual disimpan lokal; saat `Online` kembali → `OfflineClaim` (T17) menghitung `now - last_seen`, cap 24j.
- **Trigger**: `networkOffline` (`navigator.onLine === false`).

```
[ Online ] --( offline )--> [ Offline ]
[ Offline ] --( online )--> [ Online ] --( elapsed>0 )--> [ OfflineClaim ]
```

### 6.3 `Maintenance`

- **Tujuan**: Server tutup (deploy/patch). Endpoint `/status` mengembalikan `maintenance=true`.
- **Interaksi**: Paksa transisi L1 → `Error/Maintenance` (T55) bila sedang di state berat; bila di MainMenu, tampilkan banner + blokir fitur.
- **Trigger**: `maintenanceFlag` dari polling status (setiap 30dtk) atau push.

### 6.4 `Paused` (Tab Hidden)

- **Tujuan**: `visibilitychange` → `document.hidden === true`. Menjeda game loop Phaser (`scene.pause`) dan memicu **idle accrual handoff ke server**.
- **Interaksi**:
  - `gameLoop.pause()` — tick 5dtk dihentikan di client.
  - `last_seen = now()` dikirim ke server; server mulai akrual idle otoritatif (cap 24j).
  - Saat `visible` kembali → `gameLoop.resume()` + cek `OfflineClaim` (T17).
- **Trigger**: `visibilitychange` (hidden/visible).

```
[ Running(L1) ] --( hidden )--> [ Paused ] --> server idle handoff
[ Paused ] --( visible )--> [ Running(L1) ] --( elapsed>0 )--> [ OfflineClaim ]
```

### 6.5 Matriks Interaksi Sistem × App State

| L0 \ L1 | MainMenu | Battle | Gacha | Marketplace | PvP |
|---------|----------|--------|-------|-------------|-----|
| Online | penuh | penuh | penuh | penuh | penuh |
| Offline | cache UI | ❌ (abort→Error) | ❌ (guard) | ❌ (guard) | ❌ (guard) |
| Maintenance | banner | ❌ (T55) | ❌ | ❌ | ❌ |
| Paused | loop jeda | loop jeda (server handoff) | loop jeda | loop jeda | loop jeda |

> **Prinsip**: `Battle` saat `Offline`/`Maintenance` = abort ke `Error/Maintenance` karena combat server-authoritative tidak bisa jalan tanpa server. `Paused` di `Battle` diizinkan (server handoff), tapi client tidak boleh advance turn tanpa respon server.

---

## 7. Aturan Implementasi

Panduan wajib bagi engineer agar FSM tidak bocor state.

### 7.1 Single Source of Truth (SSOT)

- **Satu store terpusat** (`GameStateMachine`) memegang state saat ini: `{ l1: State, l0: SystemState, battle: BattleSubState|null }`.
- **DILARANG mutasi langsung** field state dari scene/UI. Semua perubahan via `fsm.transition(event, payload)`.
- Implementasi referensi (TypeScript):

```typescript
// src/state/GameStateMachine.ts  (ringkas, ponytail: no framework)
type L1 = 'Boot'|'Preload'|'AuthGate'|'MainMenu'|'Home'|'StageSelect'
        |'Battle'|'Gacha'|'Inventory'|'Marketplace'|'Guild'|'Raid'
        |'PvP'|'Leaderboard'|'Events'|'Settings'|'OfflineClaim'
        |'Error'|'Loading';
type L0 = 'Online'|'Offline'|'Maintenance'|'Paused';

interface FSMState {
  l1: L1; l0: L0; battle: string | null; lastSeen: number;
}

class GameStateMachine {
  private state: FSMState;
  private handlers: Record<string, (p:any)=>void> = {};
  // ponytail: tanpa library, cukup event-map. Upgrade: XState bila kompleksitas naik.
  transition(event: string, payload?: any) {
    const key = `${this.state.l1}:${event}`;
    const fn = this.handlers[key] ?? this.handlers[`*:${event}`];
    if (!fn) throw new Error(`No transition ${key}`); // no orphan
    fn(payload);
    this.persist(); // persist on transition
  }
  private persist() {
    this.state.lastSeen = Date.now();
    localStorage.setItem('fsm', JSON.stringify(this.state));
  }
}
```

### 7.2 Persist on Transition

- Setiap `transition()` menyimpan snapshot ke `localStorage` (PWA) + kirim `last_seen` ke server.
- Field wajib: `l1`, `l0`, `battle`, `lastSeen`.
- Tujuan: saat app ditutup/refresh, `Boot→Preload→AuthGate` bisa restore ke `MainMenu` tanpa kehilangan idle progress.

### 7.3 No Orphan State

- Setiap state harus punya **minimal satu transisi keluar** (kecuali terminal `Exit` yang lift ke L1).
- Guard harus **total**: untuk tiap event, harus ada cabang `true` (jangan biarkan event menggantung tanpa handler → `throw` di `transition`).
- Battle sub-FSM harus selalu berakhir di `Victory`/`Defeat` → `Exit` (tidak boleh stuck di `Resolve`).

### 7.4 Error Recovery

- Bila transisi gagal (exception), FSM paksa `→ Error/Maintenance` (T55) dengan payload error.
- `Error/Maintenance` punya 3 jalur recovery eksplisit: `restart` (Boot), `reauth` (AuthGate), `recover` (MainMenu).
- Combat fault → `Battle → Error/Maintenance` (T25), player tidak kehilangan progress di luar battle.
- Idle accrual selalu dihitung dari `last_seen` (bukan dari timer client yang bisa dimanipulasi) → anti-cheat dasar.

### 7.5 Guard & Action Terpusat

- Guard didefinisikan sebagai fungsi murni di satu modul (`guards.ts`): `hasTicket()`, `stageUnlocked()`, `gemsOk()`, `online()`, `partyAlive()`, dll.
- On-Enter/On-Exit action dijalankan oleh FSM, bukan oleh scene secara mandiri, agar urutan deterministik.

### 7.6 Validasi State di Boot

- Saat `Preload` selesai, validasi `localStorage.fsm`:
  - Bila `l1 === 'Battle'` saat app ditutup → **jangan** restore langsung ke Battle (combat server-authoritative). Restore ke `MainMenu` + flag "battle interrupted" → grant konsolasi kecil, hindari exploit idle.
  - Bila `l1` valid & `last_seen` lama → `OfflineClaim`.

---

## 8. Hubungan dengan Dokumen Lain

FSM ini adalah tulang punggung; berinteraksi dengan 3 dokumen pendukung.

### 8.1 Game Loop Document (pause/resume)

- **Game Loop** (Phaser `scene.update`) mengonsumsi flag `l0`:
  - `Running` → loop jalan (idle tick 5dtk di `Home/IdleMap`, animasi di `Battle`).
  - `Paused`/`Maintenance`/`Offline` → loop dijeda (`scene.pause()`).
- FSM meng-emit event `pause`/`resume` ke Game Loop saat `l0` berubah.
- Tick 5dtk diimplementasikan sebagai accumulator di `update(dt)`; bukan `setInterval` (agar respek pause).

```
GameLoop.update(dt):
  if fsm.l0 !== 'Running': return   // pause/resume dari FSM
  if fsm.l1 === 'Home': idleAccumulator += dt
  if idleAccumulator >= 5000: tickIdle(); idleAccumulator = 0
```

### 8.2 Save Data Document (persist state)

- **Save Data** bertanggung jawab menyimpan payload game (inventory, pity, ELO, stage progress).
- FSM menyuntikkan `last_seen` + `l1/l0` ke dalam save blob pada setiap `transition()`.
- Restore order: `Boot → Preload → AuthGate → (load Save Data) → MainMenu/OfflineClaim`.
- Idle cap 24j ditegakkan di sini: `reward = f(min(elapsed, 24h), ratePerTick)`.

### 8.3 Input Mapping Document (context layer per state)

- **Input Mapping** menggunakan *context layer*: setiap L1 punya keymap sendiri.
- FSM ekspos `currentContext()` → Input Mapping pasang listener sesuai state.
- Contoh: di `Battle`, tombol `Space` = konfirmasi aksi; di `Home/IdleMap`, `Space` = buka map menu. Tidak ada konflik karena konteks berganti saat transisi.
- Saat `Paused`, seluruh input gameplay diblock (kecuali UI pause menu).

| L1 State | Input Context | Aksi Utama |
|----------|--------------|-----------|
| MainMenu | nav | arah = navigasi tombol; Enter = pilih |
| Home/IdleMap | world | klik node = pick; Tab = inventory |
| Battle | combat | 1-5 = pilih target; Q-W-E-R = skill; Space = confirm |
| Gacha | gacha | klik banner = pull |
| Inventory | grid | klik = select; Drag = equip/move |
| Settings | form | Tab = field; Enter = save |
| Paused (L0) | pause-menu | Esc = resume; Q = quit ke MainMenu |

---

## 9. Ringkasan Kepatuhan CDP

| Parameter CDP | Pemetaan di FSM |
|---------------|-----------------|
| Desktop-first web + PWA | Boot/Preload + Offline/Online states |
| Party max 5 | Guard `partyAlive` (≥1 dari 5), TurnOrder queue |
| Rarity C/R/E/L(+M) | Gacha `rollBanner()` + table rarity |
| Gacha pity 10/75/90 | `loadPityCounter()`/`savePity()` di Gacha |
| Idle tick 5dtk, cap 24j | GameLoop accumulator + Save Data cap |
| Items Gear/Consumable/Material | Inventory state + actions |
| PvP ELO 1000 | PvP `findMatch(ELO)`, `submitResult(ELO_delta)` |
| Currencies Gold/Gems/EventToken | Wallet di Marketplace/Gacha/Events |
| Combat server-authoritative | Resolve/EnemyAction butuh server; Offline abort |
| Turn order by speed | TurnOrder (B2) dekselarasi speed |
| Mana/energy | §5.7, auto-battle via energy |
| 8 status effect | §5.5 SE1..SE8 |
| 6 elemen 1.5/0.75 | §5.6 ELEM_MATRIX |
| Auth email/password | AuthGate session/refresh token |
| Phaser3+TS+Vite | Implementasi §7.1 |

---

## 10. Glosarium

| Istilah | Arti |
|---------|------|
| FSM | Finite State Machine |
| L0/L1/L2 | Level hierarki state (sistem/app/battle) |
| Guard | Kondisi boolean pengizinh transisi |
| Tick | Satuan langkah loop (idle = 5 detik) |
| Pity | Penghitung pull menuju guarantee rarity |
| ELO | Sistem rating PvP |
| Handoff | Serah-terima akrual idle ke server saat pause/offline |
| DoT/HoT | Damage over Time / Heal over Time |
| Dekselarasi | Urut menurun (besar ke kecil) |

---

## 11. Contoh Walkthrough Pertarungan (Worked Example)

Ilustrasi nyata satu battle untuk mengunci pemahaman FSM sub-state.

**Setup**:
- Party (max 5): `A`(speed 120, FI, HP 800), `B`(speed 120, WA, HP 700), `C`(speed 90, WO, HP 600).
- Enemy: `X`(speed 110, EA, HP 1500), `Y`(speed 80, DA, HP 900).
- Elemen: `A`(FI) vs `X`(EA) = 1.0; `B`(WA) vs `Y`(DA) = 1.0; `C`(WO) vs `X`(EA) = 1.5 (strong).

**Alur sub-FSM**:

```
B0 Init      : snapshot; queue=[A,B,X,C,Y] (tie A,B player dulu)
B1 Intro     : "Battle Start" VFX
B2 TurnOrder : queue tetap
B3 A(P)      : player pilih skill FI ke X -> intentSent
B4 X(E)      : server action -> serang A
B5 Resolve   : dmg A->X (1.0x), dmg X->A (mitig)
B6 StatusTick: tidak ada SE aktif
B7 WinCheck  : partyAlive && enemyAlive -> NextTurn
B3 B(P)      : skill WA ke Y
B4 Y(E)      : serang B
B5 Resolve   : dmg B->Y
B6 StatusTick: -
B7 WinCheck  : lanjut
B3 X(E)      : (di B4 karena urutan) serang C, terapkan SE1 BURN ke C
B5 Resolve   : C terbakar (SE1)
B6 StatusTick: C kehilangan HP DoT SE1
B3 C(P)      : skill WO ke X (1.5x strong!) -> dmg besar
...
B7 WinCheck  : X HP<=0 && Y HP<=0 -> Victory (B9)
B14 Exit     : flagVictory()
B16 lift     : -> Reward(L1) computeReward()
```

**Hasil**: Reward dihitung server (Gold + item drop + exp). Pity Gacha tidak berubah (bukan gacha). ELO tidak berubah (bukan PvP).

---

## 12. Idle Accrual & Handoff Matematika

Rumus akrual idle (berlaku `Home/IdleMap` aktif DAN saat `Paused`/`Offline` via server):

```
elapsed   = now - last_seen
capped     = min(elapsed, 24h)              // cap A5
ticks      = floor(capped / 5s)            // tick A4
reward     = ticks * ratePerTick(party)    // rate naik dgn power party
```

**Handoff saat Paused** (§6.4):
1. `visibilitychange` hidden → `fsm.transition('hidden')` → `l0=Paused`.
2. Client kirim `last_seen=now` ke server, client loop di-pause.
3. Server mulai akrual otoritatif (anti-cheat: client tidak hitung).
4. `visible` → `l0=Running`, client fetch `pendingIdle = compute(elapsed,24h)`.
5. Bila `pendingIdle>0` → transisi `OfflineClaim` (T17/T48) → `grantReward()`.

**Edge**: bila user tutup tab saat `Paused`, `last_seen` sudah tersimpan → saat buka lagi, `Boot→AuthGate→OfflineClaim` otomatis. Tidak ada duplikasi karena server pakai `max(last_grant, now-cap)`.

---

## 13. Matriks Edge-Case & Failure

| Skenario | State Saat Kejadian | Penanganan FSM | Transisi |
|----------|---------------------|----------------|----------|
| Token expired di tengah Gacha | Gacha | `AuthGate` reauth (T52) | Gacha→(fatal)→Error→reauth→AuthGate |
| Server down saat Battle | Battle | abort, tidak boleh stuck | Battle→(combatFault)→Error/Maintenance (T25) |
| Refresh browser saat Inventory | Inventory | restore MainMenu, tidak restore Inventory mentah | Boot→...→MainMenu (§7.6) |
| Refresh saat Battle | Battle | jangan restore Battle; grant konsolasi | Boot→...→MainMenu + flag |
| Offline saat buka Marketplace | Marketplace | guard `[ online ]` blokir beli | tetap Marketplace (UI disabled) |
| Maintenance flag nyala saat Home | Home/IdleMap | banner + blokir fitur berat | Home→(maint)→Error/Maintenance (T55) |
| Elapsed > 24j saat kembali | OfflineClaim | cap 24j ditegakkan | OfflineClaim→grant(capped) |
| Party mati semua di StatusTick | Battle (B6) | WinCheck `!partyAlive` | →Defeat→Exit→MainMenu |
| Energy penuh di PlayerAction | Battle (B3) | auto intent terkirim | PlayerAction→(auto)→EnemyAction/Resolve |
| Timeout respon server (PvP) | Battle(pvp) | retry 3x lalu abort | Battle→Error (T25) |
| Duplikat event (double tap) | MainMenu | dedupe di transition guard | ignore 2nd event |

**Anti-Orphan Guarantee**: unit test akan menelusuri semua node L1 memiliki ≥1 outgoing; semua event terdaftar di `handlers`; semua guard total (true/false terdefinisi).

---

## 14. Acceptance Checklist (Verifikasi FSM)

Checklist ini menjadi kriteria "done" untuk implementasi:

- [ ] `GameStateMachine` adalah SSOT; tidak ada mutasi `state.l1` di luar `transition()`.
- [ ] 19 state L1 terdaftar; `Boot→Preload→AuthGate→MainMenu` reachable.
- [ ] Setiap state punya ≥1 transisi keluar (kecuali `Exit` lift).
- [ ] Guards total: tidak ada event tanpa handler (lempar error, bukan silent).
- [ ] `last_seen` persist setiap `transition()` (local + server).
- [ ] Battle sub-FSM selalu terminasi di `Victory`/`Defeat`→`Exit`.
- [ ] Turn order dekselarasi by speed; tie-break player dulu.
- [ ] 8 status effect (SE1..SE8) ditick di `StatusTick`; `SLOW` memicu recompute queue.
- [ ] Matriks elemen 1.5/0.75 diterapkan di `Resolve` (server).
- [ ] Pity 10/75/90 tersimpan & naik per pull; guarantee L di 90.
- [ ] Idle tick 5dtk; cap 24j ditegakkan di client & server.
- [ ] `Paused` (visibilitychange) menjeda loop & handoff ke server.
- [ ] `Offline` memblokir fitur butuh-server via guard `[ online ]`.
- [ ] `Maintenance` memaksa `Error/Maintenance` (T55).
- [ ] Restore dari refresh tidak pernah masuk Battle langsung.
- [ ] Input context ganti sesuai L1; tidak ada konflik key.
- [ ] PvP ELO mulai 1000; `submitResult` update ELO.
- [ ] Currencies (Gold/Gems/EventToken) konsisten di wallet persist.

---

## 15. Catatan YAGNI & Skalabilitas

- `Loading` hanya jadi state eksplisit bila load > 800ms; di bawah itu cukup overlay (§2.19).
- FSM direferensikan dengan event-map murni (§7.1). **Jangan** langsung pindah ke library XState kecuali jumlah state melampaui ~40 atau ada parallel composite state — ponytail ceiling.
- Tidak ada abstract factory untuk state; cukup objek/enum sederhana. Penambahan state = tambah entri `handlers` + baris tabel.
- Party dinamis (1..5) tidak perlu state berbeda; cukup guard `partyAlive` dan queue sepanjang jumlah unit hidup.

---

*Dokumen selesai. Semua state, transisi, guard, dan action konsisten dengan Canonical Design Parameters. Asumsi eksplisit tertulis di §0; kepatuhan CDP di §9; acceptance di §14.*
