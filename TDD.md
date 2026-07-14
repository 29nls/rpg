# Technical Design Document (TDD) — Fantasy Idle RPG (Web + PWA)

> **Dokumen:** TDD Arsitektur Client Game
> **Versi:** 1.0 (MVP)
> **Status:** Draft untuk review engineer
> **Bahasa:** Indonesia
> **Canonical Design Parameters (CDP):** Dokumen ini wajib konsisten dengan parameter desain kanonik (lihat Bagian 10.3 + lampiran). Setiap angka probabilitas, rarity, tick, dan batasan diambil langsung dari CDP.

---

## Daftar Isi

1. [Tujuan & Ruang Lingkup](#1-tujuan--ruang-lingkup)
2. [Pemilihan Game Engine](#2-pemilihan-game-engine)
3. [Arsitektur Kode (ECS + Phaser)](#3-arsitektur-kode-ecs--phaser)
4. [Tech Stack & Tooling](#4-tech-stack--tooling)
5. [Struktur Folder Proyek](#5-struktur-folder-proyek)
6. [Module Boundaries & Dependency Direction](#6-module-boundaries--dependency-direction)
7. [Rendering & Asset Pipeline Integration](#7-rendering--asset-pipeline-integration)
8. [Backend Integration](#8-backend-integration)
9. [Performansi & Batasan](#9-performansi--batasan)
10. [Asumsi, Trade-off, & Open Questions](#10-asumsi-trade-off--open-questions)

Lampiran:
- [A. Glosarium](#lampiran-a-glosarium)
- [B. Kanonikal Design Parameters Ringkasan](#lampiran-b-kanonikal-design-parameters-ringkasan)
- [C. Contoh Kontrak Payload REST/WS](#lampiran-c-contoh-kontrak-payload-restws)
- [D. Diagram Urutan (Sequence)](#lampiran-d-diagram-urutan-sequence)

---

## 1. Tujuan & Ruang Lingkup

### 1.1 Mengapa TDD ini ada

TDD ini adalah satu-satunya dokumen rujukan arsitektur **client** untuk game RPG *Fantasy Turn-Based Idle* berbasis web (desktop-first) yang dikemas sebagai PWA. Tujuannya:

- Menetapkan **arsitektur client** yang benar, aman, dan bisa dioperasikan sebelum baris kode produksi ditulis.
- Menyelaraskan keputusan *engine*, *stack*, struktur folder, dan arah dependensi agar tidak terjadi *spaghetti code* saat tim bertambah.
- Menjadi kontrak internal antara *Game System Designer* (desain gameplay) dan *Software Architect* (implementasi), tanpa ambiguitas pada parameter kanonik (CDP).
- Memberikan dasar estimasi, *code review*, dan *onboarding* engineer baru.

TDD ini **bukan** dokumen *Game Design Document* (GDD). GDD memuat keseimbangan nilai (stat, kurva level, drop rate); TDD memuat *bagaimana* client membangun, menjalankan, dan mengamankan sistem tersebut. Referensi silang ke GDD dan dokumen *Asset Pipeline* dilakukan di Bagian 7.

### 1.2 Ruang lingkup (in scope)

| No | Cakupan | Keterangan |
|----|----------|------------|
| 1 | Arsitektur client (layer, modul, arah dependensi) | Fokus utama dokumen |
| 2 | Pemilihan *engine* (Phaser 3) dan kapan Three.js | MVP + post-MVP |
| 3 | Pola ECS-lite untuk simulasi combat | Entity/Component/System |
| 4 | Integrasi Phaser Scene ↔ ECS World | Tanpa *cyclic dependency* |
| 5 | Tech stack & tooling (TS, Vite, PWA, state, net, test) | Termasuk alasan & alternatif |
| 6 | Struktur folder `src/` lengkap | + contoh file per folder |
| 7 | Integrasi backend (REST + WebSocket) | Client = API consumer |
| 8 | Strategi offline (IndexedDB queue) & PWA | App-shell + cache |
| 9 | Performansi (target 60 FPS, bundle, memory) | Angka konkret & budget |
| 10 | Asumsi, trade-off, open questions | Transparansi keputusan |

### 1.3 Batasan (out of scope)

| No | Di luar ruang lingkup | Alasan / Rujukan |
|----|-----------------------|------------------|
| 1 | Implementasi backend (NestJS, PostgreSQL, Redis, BullMQ) | Dokumen terpisah (Backend TDD/ADR) |
| 2 | Desain nilai gameplay (balance, stat curve) | GDD |
| 3 | Pembuatan aset visual & audio (sprite, atlas, BGM) | Asset Pipeline Doc |
| 4 | Three.js hero showcase 3D | Post-MVP (Bagian 2.4) |
| 5 | Infrastruktur DevOps (CI/CD, k8s, CDN setup) | Ops Runbook |
| 6 | Skema database & migrasi | Backend TDD |
| 7 | Keamanan level server (WAF, rate limit gateway) | Backend/Ops |

> **Catatan:** Meskipun backend *out of scope*, Bagian 8 mendefinisikan **kontrak integrasi** (endpoint, skema payload, WS event) yang wajib dipatuhi client agar server-authoritative tetap terjamin.

### 1.4 Audiens

Dokumen ditujukan untuk **engineer** (frontend/game client engineer, tech lead, dan reviewer). Diasumsikan pembaca memahami TypeScript, konsep ECS, siklus hidup PWA, dan REST/WebSocket secara dasar. Tidak ada penjelasan *intro* bahasa pemrograman.

### 1.5 Definisi "MVP selesai" (exit criteria client)

- Game loop idle berjalan stabil 60 FPS (fixed timestep) di desktop Chrome/Edge/Firefox terbaru.
- 1 party (maks 5 hero) dapat memasuki combat turn-based vs 1 wave enemy; status effect & elemen bekerja sesuai CDP.
- Gacha standar (C60/R30/E9/L1) dengan pity counter per banner berfungsi, hasil ditentukan server.
- Idle accrual (tick 5 dtk, offline cap 24 jam) dihitung server-side.
- PWA installable; app-shell offline via Cache API; offline action di-queue ke IndexedDB.
- Auth email/password; semua mutasi currency/drop/gacha/battle outcome server-authoritative.

### 1.6 Architecture Decision Records (ADR) — ringkasan

| ADR | Keputusan | Konsekuensi |
|-----|-----------|---------------|
| ADR-01 | Phaser 3 sebagai primary engine MVP | Three.js ditunda post-MVP |
| ADR-02 | ECS-lite murni TS, terpisah dari Phaser | Testable di Node; butuh RenderBridge |
| ADR-03 | Server-authoritative untuk ekonomi & battle | Perlu WS reconnect & offline queue |
| ADR-04 | Zustand untuk state client | Ringan; kurang devtools vs Redux |
| ADR-05 | Fixed timestep + decoupled render | Stabil 60 FPS; loop lebih kompleks |
| ADR-06 | vite-plugin-pwa (Workbox) | App-shell offline otomatis; SW di-generate |
| ADR-07 | Offline queue hanya idle/claim + read cache | Aksi ekonomi butuh online (aman) |

---

## 2. Pemilihan Game Engine

### 2.1 Kandidat

Empat kandidat dievaluasi: **Phaser 3**, **Three.js**, **PixiJS**, **vanilla Canvas 2D**.

| Engine | Tipe | Render | Cocok untuk | Tidak cocok untuk |
|--------|------|--------|-------------|-------------------|
| Phaser 3 | Framework game 2D | WebGL/Canvas | Game 2D sprite-heavy, scene management, input, audio, tween bawaan | 3D真实感, simulasi fisika kompleks |
| Three.js | Library 3D | WebGL | Scene 3D, model 3D, kamera 3D | Game 2D UI-heavy (terlalu rendah-level) |
| PixiJS | Renderer 2D | WebGL | Rendering 2D berkinerja tinggi, banyak sprite | Tidak ada scene/input/audio bawaan |
| vanilla Canvas 2D | API browser | Canvas 2D | Kontrol penuh, ukuran kecil | Perlu bangun semua (loop, input, scene) sendiri |

### 2.2 Kriteria penilaian

| Kriteria | Bobot | Alasan |
|----------|-------|--------|
| Kecepatan pengembangan MVP | Tinggi | Turn-based + idle tidak butuh mesin fisika berat |
| 2D sprite & atlas | Tinggi | Gameplay sprite-heavy (hero, enemy, status icon) |
| Manajemen Scene/UI | Tinggi | Banyak layar (login, gacha, battle, inventory, ladder) |
| Performansi 60 FPS | Tinggi | CDP: loop stabil 60 FPS |
| Dukungan PWA | Sedang | Phaser kompatibel (tidak menghalang) |
| Ukuran bundle | Sedang | Phaser ~1MB gzip; dapat di-code-split |
| Jalur ke 3D (post-MVP) | Rendah (MVP) | Three.js hanya pasca-MVP |

### 2.3 Perbandingan & Skor (skala 1–5)

| Engine | Dev Speed | 2D Sprite | Scene/UI | Perf | PWA | Bundle | 3D path | Skor |
|--------|-----------|-----------|----------|------|-----|--------|---------|------|
| **Phaser 3** | 5 | 5 | 5 | 4 | 4 | 3 | 1 (butuh integrasi) | **31** |
| Three.js | 2 | 2 | 1 | 4 | 4 | 3 | 5 | 21 |
| PixiJS | 3 | 5 | 2 | 5 | 4 | 4 | 1 | 24 |
| vanilla Canvas | 2 | 3 | 1 | 4 | 5 | 5 | 2 | 22 |

**Keputusan:** Pilih **Phaser 3** sebagai *primary engine* untuk MVP.

### 2.4 Alasan pemilihan Phaser 3 (MVP)

1. **Turn-based, bukan real-time physics.** Combat idle RPG adalah diskrit per giliran; tidak butuh solver fisika. Phaser menyediakan semua yang dibutuhkan tanpa *overhead* 3D.
2. **Sprite-heavy & atlas.** Phaser memiliki `Loader` + `TextureAtlas`/`Spritesheet` bawaan; cocok untuk hero/enemy/status icon (lihat Bagian 7).
3. **Scene management bawaan.** Phaser `Scene` cocok untuk memisahkan layar UI/flow (Login, Home/Idle, Battle, Gacha, Inventory, Ladder). Ini menyelaraskan dengan arsitektur lapisan (Bagian 3).
4. **Input & tween bawaan.** Mengurangi kode *boilerplate*; animasi antarmuka (panel slide, damage number) cepat dibuat.
5. **WebGL/Canvas fallback.** `AUTO` mode memilih WebGL, fallback Canvas; aman untuk kompatibilitas.
6. **Ekosistem & dokumentasi.** Komunitas besar, contoh banyak, mempercepat MVP.

### 2.5 Kapan Three.js layak (post-MVP)

Three.js **hanya** untuk *hero showcase 3D* pasca-MVP (CDP: *"Three.js HANYA untuk showcase 3D hero (post-MVP), tidak di MVP"*). Kasus penggunaan:

- Layar profil hero dengan model 3D yang bisa diputar (drag-rotate).
- Intro/banner animasi 3D.
- Transisi *victory* epik.

Three.js **tidak** digunakan untuk combat MVP karena: (a) menambah bundle ~600KB+ gzip, (b) menambah kompleksitas *asset pipeline* (glTF rigging), (c) tidak menambah nilai gameplay turn-based. Integrasi dilakukan sebagai *isolated scene/module* berdampingan dengan Phaser, bukan mengganti Phaser. Lihat Bagian 6.4 (boundary Three.js).

### 2.6 Trade-off

| Trade-off | Dampak | Mitigasi |
|-----------|--------|----------|
| Phaser bundle ~1MB gzip | Waktu load pertama | Code-splitting per scene; preload kritis; PWA precache app-shell |
| Phaser kurang *idiomatic* untuk ECS murni | Perlu disiplin pemisahan ECS (Bagian 3) | ECS dibangun *di atas* Phaser, bukan pakai GameObject sebagai entity |
| Phaser `GameObject` menggoda OOP coupling | Scene melekat ke logic | Aturan impor ketat (Bagian 6) |
| Post-MVP butuh Three.js terpisah | Dua renderer | Modul terisolasi; hanya load saat dibutuhu |

---

## 3. Arsitektur Kode (ECS + Phaser)

### 3.1 Prinsip arsitektur

1. **Server-authoritative.** Client tidak pernah menjadi sumber kebenaran untuk currency, drop, gacha, dan *battle outcome*. Client hanya *cache* & *prediksi* (lihat Bagian 8).
2. **ECS-lite untuk simulasi, Phaser Scene untuk UI/flow.** ECS menjalankan *combat simulation* (deterministik, testable). Phaser `Scene` mengelola layar, input UI, dan rendering.
3. **Dekoupling via event bus & adapter.** ECS World tidak tahu Phaser; Phaser tidak tahu detail komponen ECS. Komunikasi lewat *event* dan *adapter* (RenderBridge).
4. **Fixed timestep simulation, decoupled render.** Update logika pada langkah waktu tetap (mis. 60 Hz) terpisah dari render (requestAnimationFrame). Idle tick 5 dtk real-time dihitung server-side, client hanya memvisualisasikan.

### 3.2 Apa itu ECS-lite

ECS memisahkan **data** (Component) dari **logika** (System) dengan **Entity** sebagai ID netral.

- **Entity** = `number` (id) + daftar component. Tidak ada metode.
- **Component** = struktur data polos (POJO/interface TS). Contoh: `StatsComponent`, `StatusEffectComponent`, `ElementComponent`, `TurnComponent`, `ManaComponent`.
- **System** = fungsi/ kelas yang iterasi entity dengan komponen tertentu dan memutasi komponen. Contoh: `TurnOrderSystem`, `StatusEffectSystem`, `DamageSystem`, `ElementMatrixSystem`.
- **World** = penyimpanan entity + component + daftar system; menjalankan sistem per tick.

"Lite" artinya: kita **tidak** membangun ECS generik dengan *archetype* allocator tingkat rendah (seperti spec/bevy). Untuk MVP dengan maks 5 hero + beberapa enemy per battle, ECS berbasis `Map<EntityId, ComponentMap>` sudah cukup dan jauh lebih sederhana. Ini mengurangi kompleksitas tanpa mengorbankan testability.

### 3.3 Hubungan ECS ↔ Phaser

| Aspek | ECS World (Simulation) | Phaser Scene (Presentation) |
|-------|------------------------|----------------------------|
| Tanggung jawab | Logika combat, status, turn order, damage | UI layar, input, sprite, tween, audio trigger |
| State | Komponen entity (data murni) | GameObject (sprite, text, container) |
| Determinisme | Ya (input sama → hasil sama) | Tidak perlu deterministik |
| Ketergantungan | Tidak tahu Phaser | Tahu ECS via adapter/event |
| Lifecycle | Mulai saat battle, bersih saat selesai | Siklus Scene Phaser |

**Alur battle:**
1. `BattleScene` menerima `BattleStartPacket` dari server (daftar hero & enemy, seed, elemen).
2. `BattleScene` meminta `BattleFactory` membuat ECS World + entities dari packet.
3. `World.update(dt)` dijalankan pada fixed timestep; sistem memproses turn, status, damage.
4. Setiap perubahan penting di-emit sebagai `SimEvent` (mis. `DamageDealt`, `StatusApplied`, `TurnChanged`).
5. `RenderBridge` (adapter) mendengarkan `SimEvent` dan menerjemahkannya ke tween/sprite/audio di `BattleScene`.
6. Saat battle selesai, `BattleScene` mengirim `BattleResult` ke server; server adalah otoritas final (client prediction hanya untuk responsivitas; lihat 8.4).

### 3.4 Diagram arsitektur lapisan (ASCII)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          PRESENTATION LAYER                           │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ PHASER SCENES (ui/flow)                                         │  │
│  │  BootScene  PreloadScene  LoginScene  HomeScene (Idle)           │  │
│  │  BattleScene  GachaScene  InventoryScene  LadderScene  ...      │  │
│  └───────────────────────────────┬────────────────────────────────┘  │
│                                   │ (RenderBridge adapter)            │
├───────────────────────────────────┼──────────────────────────────────┤
│  RENDER LAYER                     │                                   │
│   - Phaser Game Objects (Sprite,  │                                   │
│     Text, Container, Tween)       │                                   │
│   - Damage numbers, VFX, HP bar   │                                   │
├───────────────────────────────────┼──────────────────────────────────┤
│  ECS / SIMULATION LAYER (pure TS) │                                   │
│  ┌─────────────────────────────────┴─────────────────────────────┐   │
│  │ ECS WORLD                                                      │   │
│  │  Entities (id) ── Components (data) ── Systems (logic)        │   │
│  │  Systems: TurnOrder, ManaRegen, StatusEffect, Damage,         │   │
│  │           ElementMatrix, DeathCheck, VictoryCheck              │   │
│  │  Emits: SimEvent[]  (DamageDealt, StatusApplied, TurnChanged) │   │
│  └────────────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────────┤
│  NET / STATE LAYER                                                   │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────────┐    │
│  │ REST Client  │   │ WS Client    │   │ Client Store (Zustand)│    │
│  │ (Axios)      │   │ (socket.io)  │   │  - auth, player,     │    │
│  │              │   │              │   │    party, currency,   │    │
│  └──────┬───────┘   └──────┬───────┘   │    inventory, meta   │    │
│         │                  │            └───────────┬───────────┘    │
│         │                  │                         │                 │
│  ┌──────▼──────────────────▼─────────────────────────▼───────────┐  │
│  │ OFFLINE QUEUE  (IndexedDB) — mutation buffer saat offline      │  │
│  └─────────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────────┤
│  ASSET LAYER                                                         │
│  - Texture Atlas / Spritesheet (Loader)  - JSON config (hero, etc)  │
│  - Audio sprites  - Manifest (hashing untuk cache busting)          │
├──────────────────────────────────────────────────────────────────────┤
│  AUDIO LAYER (WebAudio via Phaser SoundManager / Howler opsional)    │
│  - BGM, SFX, ducking, mute persist, event-driven (SimEvent→SFX)    │
├──────────────────────────────────────────────────────────────────────┤
│  INPUT LAYER (Pointer/Keyboard via Phaser Input + custom intent)     │
│  - Maps UI gesture → Action/Intent → Net/State atau ECS command     │
└──────────────────────────────────────────────────────────────────────┘

ARAH DEPENDENSI (anak panah = "bergantung pada", tanpa siklus):
  Scenes ──▶ RenderBridge ──▶ ECS World (read SimEvent)
  Scenes ──▶ Client Store (read state, dispatch intent)
  Scenes ──▶ Net (REST/WS) via intent layer
  ECS World ──▶ (tidak ada dependency ke Scenes/Net/Store)
  Net/State ──▶ Offline Queue (IndexedDB) ; ──▶ Client Store
  Asset/Audio/Input ──▶ consumed by Scenes (direction inward)
```

### 3.6 Determinisme & testability ECS

ECS-lite wajib **deterministik** agar hasil simulasi dapat diuji ulang dan (bila perlu) divalidasi server:

- **RNG terisolasi.** Tidak pakai `Math.random()` di dalam sistem. Semua randomness lewat `SeedableRNG` (utils/seedable-rng.ts) yang di-seed dari `BattleStartPacket.seed` (server). Ini menjamin `World.update` dengan input sama → hasil sama.
- **Urutan iterasi stabil.** Entity diiterasi berdasar `EntityId` naik (atau urutan insertion), bukan urutan `Map` yang tidak terjamin. `TurnOrderSystem` menghasilkan queue eksplisit, bukan mengandalkan hash map.
- **Mutasi terbatas.** Sistem hanya memutasi komponen entity yang jadi `query`-nya; tidak ada efek samping ke store/net.
- **No I/O di sistem.** Sistem tidak fetch, tidak tulis localStorage, tidak log ke network. Side-effect hanya lewat `SimEvent` yang di-emit ke `EventBus`.

Karena aturan di atas, `ecs/` dapat dijalankan di **Node/Vitest tanpa browser** (tidak impor `phaser`). Ini kunci: seluruh `systems/*.test.ts` berjalan di CI tanpa butuh headless browser, cepat & stabil.

Contoh struktur `World.ts` (fragment konseptual):
```ts
class World {
  private entities = new Map<EntityId, ComponentMap>();
  private systems: System[] = [];
  readonly events = new EventBus<SimEvent>();
  addEntity(id: EntityId, comps: ComponentMap) { this.entities.set(id, comps); }
  query<T extends keyof ComponentMap>(...keys: T[]) { /* filter entity punya semua key */ }
  register(sys: System) { this.systems.push(sys); }
  update(dt: number) {
    for (const sys of this.systems) sys.run(this, dt); // urutan tetap
  }
}
```

### 3.7 Contoh komponen & sistem (fragment)

**`components/StatusEffectComponent.ts`**
```ts
type StatusKind = 'Burn'|'Freeze'|'Poison'|'Stun'|'AtkDown'|'DefDown'|'Shield'|'Regen';
interface StatusInstance { kind: StatusKind; remaining: number; magnitude: number; }
interface StatusEffectComponent { active: StatusInstance[]; }
```

**`systems/StatusEffectSystem.ts`** (aturan CDP)
```ts
// Tick tiap turn/interval:
//  Burn/Poison: damage per tick (magnitude), turun remaining.
//  Freeze/Stun: skip turn (lihat StunSystem), turun remaining.
//  AtkDown/DefDown: modifier ditarik DamageSystem, turun remaining.
//  Shield: absorbs damage (DamageSystem cek dulu), turun remaining/broken.
//  Regen: heal per tick, turun remaining.
run(world, dt) {
  for (const e of world.query('status')) {
    for (const s of e.status.active) {
      applyTick(world, e.id, s);
      s.remaining -= dt;
    }
    e.status.active = e.status.active.filter(s => s.remaining > 0);
  }
}
```

**`systems/TurnOrderSystem.ts`**
```ts
// Turn order by speed (CDP). Entity dengan Speed tinggi → gauge naik cepat.
// Implementasi: each entity punya TurnComponent.gauge += spd*dt;
// saat gauge>=THRESHOLD → masuk turn queue (urut spd desc, tiebreak EntityId).
```

### 3.8 Lifecycle battle

```
BattleScene.create()
  └─ battle.api.start(stageId) → BattleStartPacket (seed, party, enemies)
  └─ BattleFactory.build(world, packet)
  └─ LoopController.start(world)  // fixed timestep
       ├─ each step: world.update(dt)
       │    └─ systems emit SimEvent → RenderBridge → visual
       └─ VictoryCheckSystem → outcome 'win'|'lose'
  └─ battle.api.result(log) → server verify → rewards
  └─ BattleScene.shutdown() → world.dispose() (free entities, events)
```

---

### 3.5 Direksi dependensi (singkat)

- `scenes/` → boleh pakai `ecs/`, `state/`, `net/`, `assets/`, `audio/`, `input/`, `ui/`, `config/`, `utils/`.
- `ecs/` → **hanya** TS murni; tidak boleh impor `phaser`, `scenes/`, `net/`, `state/`, `audio/`. (Agar testable di Node/Vitest tanpa browser.)
- `state/` → tidak boleh impor `scenes/`, `ecs/`. Boleh impor `net/`, `config/`, `utils/`.
- `net/` → tidak boleh impor `scenes/`, `ecs/`. Boleh impor `state/` (update store), `config/`, `utils/`, `workers/` (queue).
- `workers/` → isolasi IndexedDB; tidak boleh impor `phaser`. Komunikasi via message.
- `assets/`, `audio/`, `input/` → leaf modules; tidak impor `scenes/`/`ecs/`/`net/`/`state/`.
- `config/`, `utils/` → leaf; tidak impor modul di atas.

Lihat Bagian 6 untuk aturan lengkap & anti-pattern.

---

## 4. Tech Stack & Tooling

### 4.1 Ringkasan pilihan

| Kategori | Pilihan | Alternatif | Alasan singkat |
|----------|---------|------------|----------------|
| Bahasa | TypeScript 5.x (strict) | JS, Flow | Keamanan tipe untuk kontrak data server |
| Bundler | Vite 5.x | Webpack, esbuild murni | Dev server cepat, HMR, PWA plugin matang |
| PWA | `vite-plugin-pwa` (Workbox) | manual SW | App-shell precache otomatis, strategi cache |
| Sistem ECS | Custom ECS-lite (TS) | bitecs, miniplex | Kontrol penuh, sesuai "lite", tanpa dep berat |
| State mgmt | Zustand | Redux Toolkit, custom | Minimal boilerplate, selector, middleware queue |
| REST | Axios | fetch native | Interceptor (auth, error, retry, offline) |
| WS | socket.io-client | ws native | Reconnect, fallback, room (ladder/notif) |
| Testing unit | Vitest | Jest | Integral Vite, cepat, ESM |
| Testing E2E | Playwright | Cypress | Multi-browser, CI stabil |
| Lint/Format | ESLint + Prettier | Biome (opsi) | Standar industri |
| Asset build | TexturePacker CLI / scripts | — | Atlas otomatis (rujuk Asset Pipeline) |

### 4.2 TypeScript (strict)

- `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` (jika feasible).
- Tipe kontrak server didefinisikan di `src/net/contracts/` dan diimpor baik client maupun (sebagian) backend sebagai *shared types* bila repo monorepo.
- **Alasan:** Combat & gacha melibatkan banyak struktur data; tipe mencegah bug *runtime* di logika kritis.

### 4.3 Vite

- `vite.config.ts` dengan `build.target: 'es2020'`, `splitChunks` manual per scene.
- Dev: HMR untuk iterasi cepat.
- **Alasan:** CDP menyebut Vite eksplisit; ekosistem PWA plugin terbaik di Vite.

### 4.4 vite-plugin-pwa (Workbox)

- `registerType: 'autoUpdate'`.
- `workbox.globPatterns` untuk app-shell (html, js, css, font esensial).
- Strategi: **precache** app-shell; **runtime caching** untuk CDN aset (StaleWhileRevalidate).
- `manifest` PWA (name, icons 192/512, theme).
- **Alasan:** CDP mensyaratkan service worker + Cache API app-shell. Plugin ini membungkus Workbox sehingga tidak tulis SW manual.
- **Alternatif trade-off:** SW manual memberikan kontrol penuh tapi rawan bug; plugin mengurangi risiko.

### 4.5 State management — Zustand

- Store terbagi: `authStore`, `playerStore` (currency, profile), `partyStore`, `inventoryStore`, `metaStore` (banner, config server), `uiStore` (modal, toast), `battleStore` (predicted snapshot lokal).
- Middleware `offlineQueueMiddleware` mencegat aksi mutasi yang butuh server; bila offline, simpan ke IndexedDB via worker.
- **Alasan:** Zustand lebih ringan & *boilerplate* lebih sedikit dibanding Redux Toolkit; cocok untuk client game dengan banyak store terisolasi.
- **Trade-off vs Redux Toolkit:** kurang *devtools* ecosystem & *time-travel*; namun untuk game client, kita tidak butuh time-travel. Bila tim familiar Redux,gunakan Redux Toolkit—tidak mengubah arsitektur.

### 4.6 Networking — Axios + socket.io-client

- Axios: interceptor request (suntik `Authorization`), response (unwrap `ApiEnvelope`, tangani 401 → refresh/redirect login), retry eksponensial untuk 5xx/network.
- socket.io-client: koneksi WS ke backend untuk leaderboard live, notif, PvP matchmaking/relay. Fallback polling bawaan socket.io.
- **Alasan:** Interceptor & reconnect mengurangi kode repetitif. Bila backend tidak pakai socket.io (pakai `ws` murni), ganti dengan wrapper `ws`/`native WebSocket`—abstraksi di `net/ws/` membuat penggantian transparan (lihat 8.2).

### 4.7 Testing

- **Vitest** untuk unit: ECS systems (damage, status, turn order, elemen matrix), util, store reducer. Target: logika kritis 80%+ coverage.
- **Playwright** untuk E2E *smoke*: login → idle accrual visual → gacha → battle → ladder. Jalankan di CI.
- **Alasan:** Vitest cepat & *headless* untuk CI; Playwright cover alur nyata lintas browser.

### 4.8 Tooling pendukung

| Tool | Fungsi |
|------|--------|
| ESLint + Prettier | Konsistensi kode |
| Husky + lint-staged | Pre-commit hook |
| semantic-release (opsi) | Versioning |
| TexturePacker / atlas script | Build sprite atlas (Asset Pipeline) |
| Playwright CI | E2E di pipeline |

### 4.9 Alur build & CI (gambaran)

```
dev    : vite (HMR) → http://localhost:5173
build  : tsc --noEmit (typecheck) → vite build → dist/ (+ SW generated by vite-plugin-pwa)
test   : vitest run (unit) → playwright test (e2e smoke)
preview: vite preview → uji dist + SW offline
```
- **CI wajib:** typecheck + unit test + build + e2e smoke harus hijau sebelum merge ke `main`.
- **Artifact:** `dist/` (termasuk `sw.js` & `manifest.webmanifest`) di-deploy ke static host/CDN.
- **Trade-off:** `tsc --noEmit` menambah waktu CI tapi mencegah tipe rusak masuk produksi.

### 4.10 Environment & versioning

- Variabel via `import.meta.env` (Vite): `VITE_API_URL`, `VITE_WS_URL`, `VITE_CDN_URL`, `VITE_APP_VERSION`.
- File `.env`, `.env.staging`, `.env.prod` tidak commit secret; hanya URL/publik.
- Client cek `meta/config` version vs `VITE_APP_VERSION`; bila beda mayor → paksa reload/update SW (jawab OQ6).
- **Alasan:** pemisahan env mencegah salah target (staging vs prod) dan memudah scale (OQ4).

---

## 5. Struktur Folder Proyek

### 5.1 Pohon direktori lengkap (`src/`)

```
src/
├── main.ts                      # entry: bootstrap Phaser game + PWA + store + net
├── app/
│   ├── bootstrap.ts            # inisialisasi order (config → store → net → game)
│   ├── game-config.ts         # konfigurasi Phaser.Game (scale, physics off, fps)
│   └── service-worker.ts      # registrasi SW (dari vite-plugin-pwa virtual)
│
├── engine/                     # abstraksi Phaser agar tidak tersebar
│   ├── GameRoot.ts            # singleton Phaser.Game wrapper
│   ├── SceneRegistry.ts       # daftar scene + lazy import
│   ├── LoopController.ts      # fixed timestep accumulator (sim) + rAF render
│   ├── Scaler.ts              # desktop-first responsive scaling
│   └── types.ts               # tipe engine-agnostic (Vec2, Rect)
│
├── ecs/                       # PURE TS — tidak impor phaser/scenes/net/state
│   ├── World.ts               # entity store, component store, system runner
│   ├── Entity.ts              # EntityId type + factory
│   ├── Component.ts           # base/registry komponen
│   ├── System.ts              # System interface + base
│   ├── entities/
│   │   ├── HeroEntity.ts      # factory: hero dari BattleUnitPacket
│   │   ├── EnemyEntity.ts     # factory: enemy
│   │   └── StatusEffectEntity.ts # entity status effect (burn/poison/etc)
│   ├── components/
│   │   ├── StatsComponent.ts      # hp, atk, def, spd, crit
│   │   ├── ManaComponent.ts       # mana/energy per hero
│   │   ├── ElementComponent.ts    # Fire/Water/Earth/Wind/Light/Dark
│   │   ├── RarityComponent.ts     # C/R/E/L/Mythic
│   │   ├── TurnComponent.ts       # turn gauge / order
│   │   ├── StatusEffectComponent.ts
│   │   ├── ShieldComponent.ts
│   │   ├── DeathComponent.ts      # flag mati
│   │   └── AiComponent.ts         # behavior tag enemy
│   ├── systems/
│   │   ├── TurnOrderSystem.ts     # urutkan by speed
│   │   ├── ManaRegenSystem.ts
│   │   ├── StatusEffectSystem.ts  # tick Burn/Poison/Regen/Shield expire
│   │   ├── ElementMatrixSystem.ts # hitung multiplier 1.5x/0.75x
│   │   ├── DamageSystem.ts        # aplikasikan damage (shield dulu)
│   │   ├── StunSystem.ts          # skip turn bila stunned
│   │   ├── DeathCheckSystem.ts
│   │   └── VictoryCheckSystem.ts  # party vs enemy habis
│   ├── events/
│   │   ├── SimEvent.ts            # union type semua event sim
│   │   └── EventBus.ts           # emitter dalam World
│   ├── bridge/
│   │   └── BattleFactory.ts       # packet → World (dipanggil dari scene)
│   └── tests/
│       ├── DamageSystem.test.ts
│       ├── ElementMatrix.test.ts
│       └── TurnOrder.test.ts
│
├── scenes/                     # Phaser Scene: UI & flow
│   ├── BootScene.ts            # init minimal, cek PWA update
│   ├── PreloadScene.ts         # load atlas, audio, json manifest
│   ├── LoginScene.ts
│   ├── HomeScene.ts            # idle view: party, accrue display, nav
│   ├── BattleScene.ts          # jalankan ECS World + RenderBridge
│   ├── GachaScene.ts
│   ├── InventoryScene.ts
│   ├── LadderScene.ts          # PvP ladder/ELO view
│   └── common/
│       ├── BaseScene.ts        # helper: toast, modal, transition
│       └── NavBar.ts           # navigasi antar scene
│
├── render/                     # adapter ECS → Phaser visual
│   ├── RenderBridge.ts        # subscribe SimEvent → tween/sprite/audio
│   ├── UnitView.ts             # sprite+HP bar+status icon per unit
│   ├── DamageNumber.ts         # floating text
│   ├── StatusIconView.ts
│   └── Vfx.ts                 # particle sederhana (slash, hit)
│
├── state/                      # client store (Zustand) + offline middleware
│   ├── stores/
│   │   ├── authStore.ts
│   │   ├── playerStore.ts      # currency (Gold/Gems/EventToken), profile
│   │   ├── partyStore.ts       # 5 hero slots
│   │   ├── inventoryStore.ts   # gear/consumable/material
│   │   ├── metaStore.ts        # banner info, server config, pity counters
│   │   ├── uiStore.ts          # modal, toast, loading
│   │   └── battleStore.ts      # predicted local snapshot (bukan otoritas)
│   ├── middleware/
│   │   └── offlineQueueMiddleware.ts
│   └── selectors.ts
│
├── net/                        # API consumer (REST + WS)
│   ├── rest/
│   │   ├── client.ts           # Axios instance + interceptors
│   │   ├── auth.api.ts         # login/register/refresh
│   │   ├── gacha.api.ts        # pull (server-authoritative)
│   │   ├── battle.api.ts       # submit result, fetch battle packet
│   │   ├── idle.api.ts         # claim accrual, sync last_seen
│   │   ├── inventory.api.ts
│   │   ├── ladder.api.ts       # ELO, matchmaking, ranking
│   │   └── meta.api.ts         # banner, config, version
│   ├── ws/
│   │   ├── socket.ts           # socket.io client wrapper
│   │   ├── events.ts           # WS event type map (server→client)
│   │   ├── leaderboard.ws.ts
│   │   ├── notification.ws.ts
│   │   └── pvp.ws.ts
│   ├── contracts/              # tipe payload server (shared)
│   │   ├── auth.ts
│   │   ├── gacha.ts
│   │   ├── battle.ts
│   │   ├── idle.ts
│   │   ├── inventory.ts
│   │   └── ladder.ts
│   └── errors.ts               # ApiError, mapping kode
│
├── audio/                      # audio layer
│   ├── AudioManager.ts         # init, mute persist, ducking
│   ├── BgmPlayer.ts
│   ├── SfxPlayer.ts            # map SimEvent/SfxId → clip
│   └── manifest.ts             # daftar audio + key
│
├── input/                      # input layer
│   ├── InputManager.ts         # pointer/keyboard central
│   ├── intents.ts              # Intent type (UI action → net/ecs)
│   └── keymap.ts
│
├── assets/                     # (bukan file biner; referensi ke manifest)
│   ├── manifest.ts             # hash atlas/audio untuk cache busting
│   ├── atlas/                  # definisi nama atlas (loader keys)
│   │   └── keys.ts
│   ├── audio/                  # key + path (file di /public atau CDN)
│   │   └── keys.ts
│   └── json/                   # static config client (fallback/offline)
│       ├── elements.json       # matrix afinitas (cdg CDP)
│       ├── status-effects.json # definisi status
│       └── tuning.json         # konstanta idle (5dtk, 24h cap)
│
├── ui/                         # komponen UI reusable (Phaser-based)
│   ├── widgets/
│   │   ├── Button.ts
│   │   ├── Panel.ts
│   │   ├── ProgressBar.ts
│   │   ├── Toast.ts
│   │   └── Modal.ts
│   ├── hud/
│   │   ├── CurrencyBar.ts      # Gold/Gems/EventToken
│   │   └── PartyHud.ts
│   └── theme.ts                # warna, font (desktop-first)
│
├── config/                     # konfigurasi statis & env
│   ├── env.ts                  # import.meta.env (API_URL, WS_URL)
│   ├── game.constants.ts       # FPS target, tick, cap (CDP)
│   ├── balance.constants.ts    # referensi ke json (bukan nilai final)
│   └── rarity.ts               # enum + warna rarity
│
├── utils/                      # leaf helpers
│   ├── math.ts                 # clamp, lerp, rng (seedable)
│   ├── time.ts                 # format durasi, now()
│   ├── seedable-rng.ts         # untuk prediksi lokal (bukan otoritas)
│   ├── logger.ts
│   └── id.ts                   # uuid lokal (queue id)
│
├── workers/                    # web worker terisolasi
│   ├── offline-queue.worker.ts # IndexedDB read/write mutation buffer
│   ├── db.ts                   # IndexedDB wrapper (idb lib atau native)
│   └── protocol.ts             # message contract worker↔main
│
└── types/                      # tipe global bersama
    ├── domain.ts               # Hero, Enemy, Gear, Currency
    ├── enums.ts               # Rarity, Element, StatusEffect, SceneKey
    └── global.d.ts
```

### 5.2 Penjelasan per folder (singkat)

| Folder | Peran | Contoh file esensial |
|--------|-------|----------------------|
| `engine/` | Abstraksi Phaser agar konfigurasi game & loop terpusat | `LoopController.ts` (fixed timestep) |
| `ecs/` | Simulasi combat murni TS, testable tanpa browser | `World.ts`, `systems/DamageSystem.ts` |
| `scenes/` | Layar & alur game (Phaser Scene) | `BattleScene.ts`, `HomeScene.ts` |
| `render/` | Adapter menerjemah SimEvent → visual Phaser | `RenderBridge.ts` |
| `state/` | Store client & middleware offline queue | `stores/playerStore.ts` |
| `net/` | Konsumer API: REST + WS + kontrak | `rest/gacha.api.ts`, `ws/socket.ts` |
| `audio/` | Manajemen BGM/SFX, event-driven | `SfxPlayer.ts` |
| `input/` | Input terpusat → intent | `InputManager.ts` |
| `assets/` | Manifest & referensi kunci aset (bukan biner) | `manifest.ts`, `json/elements.json` |
| `ui/` | Widget & HUD reusable | `widgets/ProgressBar.ts` |
| `config/` | Konstanta & env | `game.constants.ts` |
| `utils/` | Helper leaf | `seedable-rng.ts` |
| `workers/` | Queue offline IndexedDB (terisolasi) | `offline-queue.worker.ts` |
| `types/` | Tipe domain & enum global | `enums.ts` |

### 5.3 Contoh file representatif

**`src/config/game.constants.ts`**
```ts
// Konstanta idle & loop sesuai CDP. Nilai final dari server meta dapat override.
export const GAME_CONSTANTS = {
  TARGET_FPS: 60,
  SIM_HZ: 60,                 // fixed timestep simulation
  IDLE_TICK_MS: 5_000,        // tick 5 detik real-time
  OFFLINE_CAP_HOURS: 24,      // accrual offline maksimal
  PARTY_MAX: 5,
  PITY_HARD_LEGENDARY: 90,
  PITY_SOFT_START: 75,
  PITY_RPLUS_PER_PULLS: 10,
  START_ELO: 1000,
} as const;
```

**`src/ecs/systems/ElementMatrixSystem.ts`** (fragment)
```ts
// 6 elemen: Fire, Water, Earth, Wind, Light, Dark.
// Afinitas: 1.5x (strong), 0.75x (weak). CDP.
export function elementMultiplier(atk: Element, def: Element): number {
  if (STRONG_AGAINST[atk] === def) return 1.5;
  if (STRONG_AGAINST[def] === atk) return 0.75;
  return 1.0;
}
```

**`src/state/stores/playerStore.ts`** (fragment)
```ts
// Currency server-authoritative; client hanya cache.
interface PlayerState {
  gold: number; gems: number; eventTokens: number;
  lastSeen: number; // epoch ms, untuk idle accrual
}
// mutasi via action → net/rest/idle.api → server response overwrite
```

**`src/net/rest/gacha.api.ts`** (fragment)
```ts
// Hasil gacha DITENTUKAN SERVER. Client kirim intent + bannerId + jumlah pull.
export async function pull(bannerId: string, count: 10 | 1): Promise<GachaResult> {
  const res = await client.post('/gacha/pull', { bannerId, count });
  return res.data; // server otoritatif: daftar rarity + heroId
}
```

**`src/ecs/bridge/BattleFactory.ts`** (fragment)
```ts
// Dipanggil DARI scene (bukan sebaliknya). Terjemah packet → World.
export function buildWorld(packet: BattleStartPacket): World {
  const w = new World();
  for (const u of packet.party) w.addEntity(makeHero(u));
  for (const e of packet.enemies) w.addEntity(makeEnemy(e));
  w.register(new TurnOrderSystem());
  w.register(new StatusEffectSystem());
  // ... register semua system
  return w;
}
```

**`src/workers/offline-queue.worker.ts`** (fragment)
```ts
// Hanya tau IndexedDB + message protocol. Tidak impor phaser.
self.onmessage = async (ev: MessageEvent<QueueMsg>) => {
  if (ev.data.type === 'enqueue') await db.put('queue', ev.data.item);
  if (ev.data.type === 'flush') { /* baca & kirim via net (dari main, bukan worker) */ }
};
```

**`src/render/RenderBridge.ts`** (fragment)
```ts
// Subscribe SimEvent → terjemah ke UnitView/Vfx/Audio. TIDAK mutasi World.
export function bind(world: World, scene: BattleScene) {
  world.events.on('DamageDealt', e => scene.spawnDamageNumber(e.targetId, e.amount));
  world.events.on('StatusApplied', e => scene.statusIcons.add(e.targetId, e.kind));
  world.events.on('TurnChanged', e => scene.highlightActor(e.actorId));
}
```

---

## 6. Module Boundaries & Dependency Direction

### 6.1 Aturan impor (matrix)

Legenda: ✅ boleh · ❌ dilarang · ⚠ boleh dengan batasan.

| Dari \ Ke | engine | ecs | scenes | render | state | net | audio | input | ui | config | utils | workers | assets |
|-----------|--------|-----|--------|--------|-------|-----|-------|-------|----|--------|-------|---------|--------|
| engine | — | ❌ | ⚠ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| ecs | ❌ | — | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ⚠ | ✅ | ❌ | ⚠ (baca json statis) |
| scenes | ✅ | ✅ | — | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| render | ⚠ | ✅ (read event) | ✅ | — | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ |
| state | ❌ | ❌ | ❌ | ❌ | — | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
| net | ❌ | ❌ | ❌ | ❌ | ✅ | — | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ |
| audio | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | — | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ |
| input | ❌ | ❌ | ✅ | ❌ | ⚠ | ⚠ | ❌ | — | ❌ | ✅ | ✅ | ❌ | ❌ |
| ui | ⚠ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | — | ✅ | ✅ | ❌ | ✅ |
| config | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — | ✅ | ❌ | ❌ |
| utils | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — | ❌ | ❌ |
| workers | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | — | ❌ |
| assets | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | — |

Penjelasan kunci:
- `ecs` **tidak** boleh impor `phaser`, `scenes`, `net`, `state`, `audio`. Ini jaminan testability di Node.
- `scenes` adalah konsumen terbesar; boleh pakai hampir semua (kecuali worker langsung & engine internal).
- `render` hanya *baca* ECS via `SimEvent` (tidak mutasi World langsung).
- `state` dan `net` tidak tahu `scenes`/`ecs` (hindari coupling UI ke data layer).
- `workers` hanya via message ke `state`/`net`; tidak boleh `phaser`.

### 6.2 Anti-pattern (dilarang)

| No | Anti-pattern | Mengapa salah | Benar |
|----|--------------|--------------|-------|
| 1 | Import `Scene` ke dalam `ecs/` | ECS jadi tidak testable & coupling | ECS emit `SimEvent`; Scene baca via `RenderBridge` |
| 2 | `ecs` mutasi `playerStore` (currency) | Currency server-authoritative | Battle result dikirim ke server; server balas update |
| 3 | Logika damage di `BattleScene` | Tidak deterministik & tidak testable | Di `DamageSystem` dalam `ecs/` |
| 4 | `net/` panggil `scenes/` langsung | Coupling data↔UI | Net update `state/`; Scene subscribe store |
| 5 | Axios dipakai langsung di `scenes` | Interceptor & queue terlewati | Lewat `net/rest/*.api.ts` |
| 6 | Tween untuk logika (bukan visual) | Timing render mengubah gameplay | Logika di ECS fixed timestep |
| 7 | Simpan currency di `localStorage` sebagai otoritas | Cheat & inkonsisten | Hanya cache; server adalah sumber |
| 8 | `render/` memutasi `World` | Pelanggaran arah dependensi | Render hanya translate event |

### 6.3 Konvensi lintasan & bundling

- Semua impor relatif atau via *path alias* `@/` → `src/`.
- Scene di-*lazy-load* (`import()`) agar bundle awal kecil (lihat 9.2).
- `ecs/`, `utils/`, `config/`, `contracts/` masuk *vendor/common chunk* bila dipakai lintas scene.

### 6.4 Boundary Three.js (post-MVP)

- Modul `three-showcase/` (belum ada di MVP) berada **di luar** `ecs/`/`scenes/` inti; dipanggil sebagai *overlay* dari `HeroProfileScene` via `lazy()`.
- Tidak boleh `ecs/` impor `three`; tidak boleh `three` impor `phaser` GameObject secara langsung—gunakan container DOM terpisah (`canvas` sendiri) di atas Phaser, atau `Phaser DOM Element`.
- Hanya dimuat saat user buka profil hero 3D; tidak masuk bundle MVP.

### 6.5 Lint & arch-guard (pencegah pelanggaran)

- **ESLint `no-restricted-imports`:** blokir `ecs/**` mengimpor `phaser`, `scenes/`, `net/`, `state/`, `audio/`. Blokir `net/**` & `state/**` mengimpor `scenes/`, `ecs/`.
- **Import order:** leaf (`config`,`utils`) dulu, lalu module sendiri, terakhir cross-module.
- **Path alias:** `@/` → `src/` di `vite.config` & `tsconfig`; hindari `../../` dalam.
- **ArchUnit test (opsi):** `dependency-cruiser` untuk menegakkan matrix 6.1 di CI. Bila melanggar → build gagal. Ini lebih murah daripada review manual.
- **File naming:** `PascalCase.ts` untuk class/Scene/System; `camelCase.ts` untuk fungsi/store; `kebab-case` untuk folder.

### 6.6 Konsekuensi pelanggaran

| Pelanggaran | Deteksi | Tindakan |
|-------------|----------|----------|
| `ecs` impor `phaser` | ESLint/dep-cruiser di CI | Build merah, PR diblokir |
| Logika gameplay di Scene | Code review + vitest coverage ECS rendah | Refactor ke `systems/` |
| Axios langsung di Scene | ESLint no-restricted | Gunakan `net/rest/*.api.ts` |
| Secret di bundle | Secret scan (gitleaks) di CI | Blokir commit |

---

## 7. Rendering & Asset Pipeline Integration

### 7.1 Hubungan dengan Asset Pipeline

Dokumen *Asset Pipeline* (terpisah) mengatur: penamaan aset, pembuatan texture atlas/spritesheet (TexturePacker), kompresi audio, dan *hashing* untuk cache busting. Client membaca **manifest** (`assets/manifest.ts`) yang dihasilkan pipeline tersebut:

```
PreloadScene
   └─ baca assets/manifest.ts (nama atlas, key, hash, versi)
   └─ Phaser Loader.load.atlas(key, pngUrl, jsonUrl)
   └─ load.audio(key, url) dari CDN (atau precache PWA)
   └─ load.json(elements/status-effects/tuning)
```

- **Cache busting:** URL aset menyertakan hash (`hero-atlas.ab12cd.png`). PWA Workbox precache app-shell; aset besar (atlas) via runtime cache (StaleWhileRevalidate) dari CDN.
- **Lazy asset:** Atlas battle hanya di-load saat masuk `BattleScene`; atlas gacha saat `GachaScene`. Mengurangi memori awal.

### 7.2 Spritesheet / Texture Atlas

- Setiap kategori (hero, enemy, status icon, UI) dalam atlas sendiri.
- Frame diakses via `key#frameName` (Phaser). `UiSprite`/`UnitView` merujuk `assets/atlas/keys.ts`.
- **Trade-off:** Atlas besar mengurangi *draw call* tapi memperbesar memori & waktu load. Untuk MVP, pisahkan per scene agar tidak semua atlas di-memori bersamaan.

### 7.3 Audio pipeline

- `AudioManager` inisialisasi `WebAudio` (Phaser SoundManager atau Howler). 
- BGM di-stream/precached; SFX di-decode *on demand* (lazy) untuk hemat memori.
- `SfxPlayer` memetakan `SfxId` dan `SimEvent` → klip. Contoh: `DamageDealt` → `sfx_hit`; `StatusApplied(Burn)` → `sfx_burn`.
- Mute/volume persist di `uiStore`/`localStorage` (preferensi, bukan currency).
- **Autoplay policy:** inisialisasi audio setelah interaksi user pertama (login/click) untuk menghindari block browser.

### 7.5 CDN & loading strategy

- Atlas/audio besar di-host di CDN (CDP). `manifest.ts` memuat URL `VITE_CDN_URL + hash`.
- **Preload taktis:** `HomeScene` memuat atlas yang paling sering dibuka berikutnya (Battle, Gacha) di *background* saat idle, agar transisi mulus.
- **Failover:** bila CDN gagal, coba ulang 1x; bila tetap gagal, tampilkan layar error dengan tombol "coba lagi" (bukan hang).
- **Trade-off:** preload di Home menambah memori awal sedikit, tapi menghilangkan *loading spinner* saat navigasi — layak untuk desktop-first.

### 7.6 Asset versioning & cache invalidation

- Setiap rilis aset → hash baru di `manifest.ts`. PWA Workbox `globPatterns` + `additionalManifestEntries` menangani app-shell; aset CDN pakai `CacheFirst` dengan `maxAgeSeconds` (mis. 30 hari) + `StaleWhileRevalidate`.
- Bila `manifest` versi naik, `PreloadScene` deteksi perbedaan dan muat ulang atlas yang berubah; cache lama di-purge via Workbox `cleanupOutdatedCaches: true`.
- **Alasan:** mencegah player memakai aset usang pasca-update balance/visual (jawab OQ6).

### 7.4 Render decoupling

- `RenderBridge` berlangganan `World.eventBus`. Tiap `SimEvent` diterjemahkan ke `UnitView` (update HP bar, play tween, spawn `DamageNumber`, update `StatusIconView`).
- Render berjalan di rAF; simulasi di fixed timestep. Bila sim menghasilkan banyak event dalam satu tick, RenderBridge men queue visual secara berurutan (anti *visual overlap*).
- **Trade-off:** decoupling menambah sedikit *latency* visual (1 frame) tapi menyelamatkan determinisme & testability.

---

## 8. Backend Integration

### 8.1 Prinsip: client sebagai API consumer

Client **tidak** menghitung hasil akhir yang berdampak ekonomi/gameplay. Server (NestJS + Postgres + Redis + BullMQ + WS) adalah otoritas. Client:

1. **Cache** state (currency, party, inventory, meta) dari respons server.
2. **Prediksi** untuk responsivitas UI (mis. animasi damage sebelum konfirmasi), lalu **rekonsiliasi** dengan respons server.
3. **Queue** aksi saat offline → kirim saat online.

### 8.2 REST (Axios)

Semua respons dibungkus `ApiEnvelope`:
```ts
interface ApiEnvelope<T> { ok: boolean; code: string; data: T; ts: number; }
```
Interceptor:
- Request: suntik `Authorization: Bearer <token>` dari `authStore`.
- Response: `unwrap` `data`; bila `!ok` → lempar `ApiError`.
- 401: coba refresh token; gagal → logout ke `LoginScene`.
- Network error/offline: alihkan ke offline queue (8.5).

Endpoint utama (detail di Lampiran C):

| Grup | Method + Path | Fungsi | Otoritas |
|------|---------------|--------|----------|
| auth | POST /auth/register, /auth/login, /auth/refresh | auth email/password | server |
| meta | GET /meta/config, /meta/banners | banner, pity info, versi | server |
| idle | POST /idle/claim | accrual (now-last_seen clamp 24h) | server |
| gacha | POST /gacha/pull | hasil pull (pity counter per banner) | server |
| battle | GET /battle/start/:stageId, POST /battle/result | packet & validasi hasil | server |
| inventory | GET /inventory, POST /inventory/equip | gear/consumable | server |
| ladder | GET /ladder/rank, POST /ladder/match | ELO, matchmaking | server |
| pvp | WS /pvp | relay turn (server validasi) | server |

### 8.3 WebSocket (socket.io / ws)

- Koneksi persisten untuk: leaderboard live, notifikasi (reward, event), PvP matchmaking & relay turn.
- WS hanya **relay**; server memvalidasi setiap aksi PvP (tidak trust client).
- Reconnect otomatis dengan backoff; saat terputus, UI tunjukkan indikator "reconnecting" (bukan crash).
- Event map di `net/ws/events.ts` (server→client & client→server) sebagai *typed* kontrak.

### 8.4 Optimistic UI vs Server-Authoritative

| Aksi | Strategi | Alasan |
|------|----------|--------|
| Currency update | Server dulu, cache update dari respons | Anti cheat; konsisten |
| Gacha result | Tunggu server; animasi setelah respons | Hasil ditentukan server |
| Battle (MVP) | **Server-authoritative outcome**. Client jalankan ECS untuk *visual prediksi*, lalu submit action log; server validasi & balas hasil final | Cegah manipulasi damage |
| Idle accrual | Display prediksi lokal (estimasi), tapi `claim` menghitung server-side | CDP: accrual server-side |
| Equip gear | Optimistic UI (langsung tampil), rollback bila 409/error | UX responsif, dampak kecil |
| PvP turn | Submit intent; server validasi & broadcast | Anti cheat PvP |

**Rekonsiliasi:** saat respons server tiba, `battleStore`/`playerStore` di-overwrite dengan nilai server. Prediksi lokal dibuang. Bila beda (selisih), log `warn` (berguna deteksi desync/cheat).

### 8.5 Offline queue (IndexedDB via Worker)

- Saat offline (deteksi `navigator.onLine` + WS down + REST gagal), mutasi yang aman di-queue:
  - `idle/claim` (server hitung ulang saat online).
  - Read-only cache tetap bisa dibuka.
  - **Tidak** di-queue: gacha pull, battle result, pvp (butuh otoritas & real-time).
- `offline-queue.worker.ts` tulis ke IndexedDB (`db.ts`). Saat online kembali (`online` event), `net/` flush queue berurutan dengan retry.
- **Trade-off:** queue menambah kompleksitas namun memenuhi CDP "IndexedDB offline queue". Untuk MVP, queue dibatasi hanya `idle/claim` & cache read; aksi ekonomi kritis tetap butuh online.

### 8.6 Keamanan client (batas tanggung jawab)

- Token disimpan di memori (`authStore`) + refresh via httpOnly cookie bila backend dukung; **tidak** simpan secret di `localStorage`.
- Validasi input di client hanya untuk UX (disable tombol, format); **bukan** pengamanan. Pengamanan di server.
- Jangan *hardcode* API secret/key pihak ketiga di bundle (gunakan env & proxy server).
- GDPR/PDPA: consent saat register; endpoint `/account/export` & `/account/delete` (backend). Client sediakan UI konsen & tombol hapus akun.

### 8.7 Error handling & retry

| Skenario | Penanganan client |
|----------|-------------------|
| 401 (token expired) | Interceptor coba `/auth/refresh`; gagal → logout ke LoginScene, hapus token memori |
| 409 (conflict/state stale) | Rollback optimistic UI; fetch state terbaru dari server; toast "sinkron ulang" |
| 422 (invalid input) | Tampilkan pesan validasi; tidak retry otomatis |
| 429 (rate limit) | Backoff eksponensial; queue (idle/claim) ditunda |
| 5xx / network | Retry 2–3x exp backoff; bila gagal → offline queue (8.5) |
| WS disconnect | Indikator "reconnecting"; buffer input PvP lokal; flush saat reconnect |
| Timeout panjang | AbortController (Axios signal); cegah UI hang |

- Semua error jadi `ApiError` ber-tipe (`code`, `message`, `retryable`). `uiStore.toast()` menampilkan pesan ramah (bukan raw error server).
- **Tidak** menampilkan stack trace ke user; log ke console/dev hanya di mode `DEV`.

### 8.8 Observability (client-side)

- **Error tracking:** Sentry/bugsnag (sourcemap privat, OQ5). Capture unhandled rejection & WS error.
- **Perf metric:** kirim `actualFps`, `memory.usedJSHeapSize` tiap N detik ke endpoint `/metrics/client` (atau hanya di DEV). Bantu deteksi regresi 60 FPS.
- **Funnel:** event analitik (login→gacha→battle→ladder) via wrapper `analytics.track()` — *opt-in* & patuh consent GDPR/PDPA.
- **Reconcile log:** bila prediksi lokal ≠ server (8.4), kirim `warn` ke telemetry (indikasi desync/cheat potensial).
- **Trade-off:** telemetry menambah sedikit bandwidth & kode; dikondisikan `import.meta.env.DEV` atau flag feature agar mati saat tidak perlu.

### 8.9 Mock server untuk dev paralel (jawab OQ1)

- Saat backend belum siap, gunakan **MSW (Mock Service Worker)** di `net/` agar `*.api.ts` dipasang ke handler mock yang meniru kontrak Lampiran C.
- Mock gacha gunakan RNG deterministik dengan prob CDP (C60/R30/E9/L1) & pity counter lokal — hanya untuk dev, **bukan** otoritas produksi.
- E2E Playwright jalankan terhadap mock agar alur bisa diuji tanpa backend hidup.
- Bila backend siap: matikan MSW via env (`VITE_USE_MOCK=false`); client menunjuk API nyata. Tidak ada perubahan di `*.api.ts`.

---

## 9. Performansi & Batasan

### 9.1 Target 60 FPS

- **Fixed timestep simulation:** `LoopController` akumulasi `dt`; jalankan `World.update(1/SIM_HZ)` sebanyak step yang wajib; sisa interpolasi untuk render. Ini mencegah *spiral of death* saat frame drop.
- **Decoupled render:** rAF bebas dari logika; bila sim lambat, render tetap halus (interpolasi).
- Budget frame: target < 16.6 ms. Breakdown kasar:

| Tahap | Budget (ms) |
|-------|-------------|
| Sim ECS (battle, maks ~10 entity) | 1–2 |
| Render (sprite/tween) | 6–8 |
| Audio/Input | 1 |
| Net poll/GC | 2 |
| Headroom | ~3–4 |

- Pengujian FPS via `game.loop.actualFps` (Phaser) + overlay debug (mode dev).

### 9.2 Ukuran bundle & code-splitting

- Bundle awal (app-shell): target **< 1.2 MB gzip** (Phaser + bootstrap + Home).
- Scene di-*lazy import*; atlas per-scene tidak dimuat bersamaan.
- `vite-manual-chunks`: pisahkan `phaser` (vendor), `socket.io-client` (vendor), `zustand`.
- PWA precache app-shell; runtime cache aset CDN.
- Hindari impor *barrel* besar yang menarik seluruh lib ke chunk awal.

### 9.3 Batasan PWA offline

| Aspek | Batasan MVP |
|-------|-------------|
| App-shell | Offline (precache): html, js, css, font esensial |
| Atlas/audio besar | Offline via runtime cache bila pernah di-load; else butuh network |
| Gameplay ekonomi | Butuh online (server-authoritative) |
| Idle accrual | Claim butuh online; prediksi lokal saat offline |
| Queue | Hanya `idle/claim` + read cache (lihat 8.5) |

### 9.4 Memory budget

| Kategori | Budget (MB) |
|----------|------------|
| JS heap (game) | 150 |
| Texture (atlas aktif) | 80 |
| Audio (decoded) | 40 |
| Total target | < 270 MB |

- Lepas atlas saat keluar scene (`scene.events.on('shutdown')` → `texture.remove()`).
- Pantau via DevTools Memory; alat CI smoke untuk deteksi *leak* (Playwright + `performance.memory`).

### 9.5 Trade-off performansi

- Fixed timestep menambah kompleksitas loop tapi menjamin determinisme & stabilitas → wajib untuk combat.
- Lazy scene menunda load tapi pertama kali buka scene ada *hit* kecil (ditutupi preload di `HomeScene` via prediksi navigasi).

### 9.6 Profiling checklist (rutin pra-rilis)

| Cek | Alat | Kriteria lulus |
|-----|------|----------------|
| FPS stabil battle | Phaser `actualFps` overlay (DEV) | ≥58 FPS rata-rata 60 dtk |
| JS heap leak | Chrome DevTools Memory / Playwright `performance.memory` | Tidak naik setelah 5x masuk-keluar BattleScene |
| Texture leak | DevTools → Phaser Texture Manager count | Atlas scene keluar = removed |
| Bundle awal | `vite build` report | < 1.2 MB gzip app-shell |
| Idle CPU saat background | Task Manager / DevTools | Render throttled; tidak 100% CPU (rAF pantau visibility) |
| WS reconnect | Putus WS paksa | UI "reconnecting", auto-connect < 5s |
| Offline queue | Toggle `navigator.onLine=false` | idle/claim ter-queue; flush saat online |

### 9.7 Batasan platform & progressive enhancement

- **Desktop-first:** `Scaler` pakai mode `Phaser.Scale.NONE`/`FIT` dengan min-width; UI `ui/theme.ts` dioptimasi layout lebar. Mobile ditangani minimal (readable) tapi bukan target MVP.
- **Visibility throttle:** saat tab background, `LoopController` pause sim (idle accrual tetap dihitung server via `last_seen` saat kembali). Mencegah CPU burn & battery drain.
- **Reduced motion:** hormati `prefers-reduced-motion` untuk VFX berat (opsi; aksesibilitas dasar).

---

## 10. Asumsi, Trade-off, & Open Questions

### 10.1 Asumsi eksplisit

| No | Asumsi | Dampak bila salah |
|----|--------|-------------------|
| A1 | Backend NestJS+Postgres+Redis+BullMQ+WS tersedia & stabil | Client tidak bisa jalan; butuh mock server (lihat OQ) |
| A2 | Server menghitung semua hasil ekonomi (gacha, drop, idle, battle) | Bila client diizinkan otoritas → risiko cheat |
| A3 | Kontrak payload REST/WS disepakati (Lampiran C awal) | Perubahan butuh sinkronisasi tipe |
| A4 | CDP adalah sumber kebenaran angka (prob, tick, cap) | Desain menyimpang bila CDP berubah |
| A5 | Target platform desktop-first (Chrome/Edge/Firefox terbaru) | Mobile butuh penyesuaian scale/input |
| A6 | Aset (atlas/audio/json) disediakan pipeline eksternal | Preload gagal bila manifest absen |
| A7 | Scale awal 10–50 DAU; arsitektur scalable tanpa rewrite | Lonjakan tiba-tiba butuh tuning server, bukan client |
| A8 | Auth email/password (bukan SSO/OAuth di MVP) | Tambahan bila SSO diminta |
| A9 | GDPR/PDPA: consent + export/delete di backend | Client sediakan UI saja |
| A10 | IndexedDB tersedia (PWA standar) | Fallback: queue di memori (hilang bila refresh) |

### 10.2 Trade-off utama

| Keputusan | Trade-off | Mitigasi |
|-----------|-----------|----------|
| ECS-lite vs OOP Phaser GameObject | ECS lebih testable & deterministik, tapi butuh disiplin pemisahan & adapter | Aturan impor ketat (Bagian 6); `RenderBridge` |
| Phaser vs Three.js untuk MVP | Phaser 2D cepat, tapi tidak 3D | Three.js pasca-MVP terisolasi (2.5) |
| Server-authoritative vs client prediction | Aman & anti-cheat, tapi butuh latensi rendah & handle reconnect | Optimistic UI terbatas + queue (8.4/8.5) |
| Zustand vs Redux Toolkit | Ringan, tapi kurang devtools | Boleh ganti tanpa ubah arsitektur |
| Fixed timestep vs rAF murni | Stabil & deterministik, tapi lebih kompleks | `LoopController` terpusat |
| PWA precache app-shell vs full offline | Installable & cepat, tapi gameplay butuh online | Jelas di 9.3 |

### 10.3 Konsistensi CDP (wajib)

Setiap angka di TDD merujuk CDP:
- Rarity: Common/Rare/Epic/Legendary (+Mythic via banner khusus); MVP pool C/R/E/L.
- Gacha standar: C60/R30/E9/L1; pity R+ tiap 10-pull; hard pity L di 90; soft pity dari 75; counter per banner.
- Idle: tick 5 dtk; offline cap 24 jam; accrual server-side `clamp[0,24h]`.
- Party: maks 5 hero. Items: Gear (weapon/armor/accessory + rarity + stat), Consumable, Material.
- PvP: ELO start 1000, K-factor; ladder tier.
- Currency: Gold (soft), Gems (hard), Event Tokens.
- Combat: turn by speed; mana/energy per hero; status Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen; 6 elemen Fire/Water/Earth/Wind/Light/Dark afinitas 1.5x/0.75x.
- Tech: Phaser 3 primary; Three.js hanya 3D hero post-MVP; ECS-lite; TS; Vite; vite-plugin-pwa; 60 FPS fixed timestep; backend Node/NestJS+Postgres+Redis+BullMQ+WS+CDN; client REST+WS; server-authoritative.

### 10.4 Open Questions — RESOLVED

Semua OQ1–OQ7 dikunci di `DECISION_REGISTER.md`. Ringkasan: mock server MSW Ya (`D-14`); PvP relay WS real-time (`D-15`); battle validate-result + sanity untuk MVP, full-replay bila cheat naik (`D-16`); shard env-based (`D-17`); sourcemap privat Ya (`D-18`); forced update via `/meta/config` (`D-19`); region dual GDPR+PDPA, default PDPA (`D-20`).

---

## Lampiran A. Glosarium

| Istilah | Arti |
|---------|------|
| ECS | Entity-Component-System, pola arsitektur simulasi |
| CDP | Canonical Design Parameters, parameter desain wajib |
| PWA | Progressive Web App |
| Gacha | Sistem *summon* acak berbasis currency |
| Pity | Penjaminan rarity setelah N pull |
| Accrual | Akumulasi reward idle |
| TDD | Technical Design Document |
| Server-authoritative | Server sebagai sumber kebenaran mutlak |
| SimEvent | Event yang di-emit ECS World ke layer render |
| RenderBridge | Adapter menerjemah SimEvent → visual Phaser |
| Fixed timestep | Update logika pada interval waktu tetap |

## Lampiran B. Kanonikal Design Parameters Ringkasan

```
PLATFORM   : Desktop-first web + PWA (SW + Cache API app-shell; IndexedDB offline queue)
PARTY      : max 5 hero. Rarity C/R/E/L (+Mythic via banner khusus). MVP pool C/R/E/L.
GACHA     : std C60/R30/E9/L1. Pity R+ per 10-pull; hard pity L@90; soft pity from 75. Per banner.
IDLE       : tick 5s realtime; offline cap 24h; accrual server-side clamp[0,24h].
ITEMS      : Gear (weapon/armor/accessory + rarity + stat), Consumable, Material.
PVP        : simple ELO (start 1000, K-factor), ladder tier.
CURRENCY  : Gold (soft), Gems (hard), Event Tokens.
COMBAT     : turn by speed; mana/energy per hero; status Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen;
             6 elemen Fire/Water/Earth/Wind/Light/Dark affinity 1.5x/0.75x.
SCALE      : 10–50 DAU awal (MVP), scalable. Auth email/password. GDPR/PDPA. Server-authoritative.
ENGINE     : Phaser 3 primary (2D WebGL/Canvas). Three.js ONLY 3D hero showcase post-MVP (not MVP).
ARCH       : ECS-lite (Entity/Component/System) for combat sim; Phaser Scene for UI/flow. TS. Vite. PWA vite-plugin-pwa(Workbox).
TARGET     : stable 60 FPS (fixed timestep update + decoupled render).
BACKEND    : Node/NestJS + PostgreSQL + Redis + BullMQ + WebSocket (leaderboard/PvP/notif) + CDN. Client = API consumer (REST+WS).
STATE      : server-authoritative for currency/drops/gacha/battle outcome; client cache & predict only.
```

## Lampiran C. Contoh Kontrak Payload REST/WS

**Auth — POST /auth/login**
```json
// req
{ "email": "p@x.com", "password": "***" }
// res ApiEnvelope<LoginData>
{ "ok": true, "code": "OK", "ts": 0,
  "data": { "accessToken": "...", "refreshToken": "...", "playerId": "u_123" } }
```

**Meta — GET /meta/banners**
```json
{ "ok": true, "code": "OK", "ts": 0,
  "data": [ { "bannerId": "std01", "name": "Standard", "pityLegendary": 90,
              "softPityStart": 75, "rPlusGuaranteeEvery": 10, "rates": {"C":0.6,"R":0.3,"E":0.09,"L":0.01} } ] }
```

**Idle — POST /idle/claim**
```json
// req
{ "lastSeen": 1710000000000 }
// res — server hitung (now-lastSeen) clamp[0,24h]
{ "ok": true, "code": "OK", "ts": 0,
  "data": { "goldGained": 1200, "gemsGained": 0, "secondsAccrued": 43200, "newLastSeen": 1710043200000 } }
```

**Gacha — POST /gacha/pull** (server-authoritative)
```json
// req
{ "bannerId": "std01", "count": 10 }
// res — server tentukan rarity per pull + pity update
{ "ok": true, "code": "OK", "ts": 0,
  "data": { "results": [ {"rarity":"R","heroId":"h_07"}, ... ],
            "pity": { "pullsSinceRPlus": 3, "pullsSinceLegendary": 42 } } }
```

**Battle — GET /battle/start/:stageId**
```json
{ "ok": true, "code": "OK", "ts": 0,
  "data": { "seed": 12345, "party": [ {"unitId":"h_01","element":"Fire","stats":{...}} ],
            "enemies": [ {"unitId":"e_09","element":"Earth","stats":{...}} ],
            "statusRules": [ "Burn","Freeze","Poison","Stun","AtkDown","DefDown","Shield","Regen" ] } }
```

**Battle — POST /battle/result**
```json
// req — client kirim action log + hasil prediksi; server validasi
{ "stageId": "s_03", "actionLog": [ {"turn":1,"actor":"h_01","skill":"slash","target":"e_09"} ],
  "predictedOutcome": "win", "seed": 12345 }
// res — server otoritatif
{ "ok": true, "code": "OK", "ts": 0,
  "data": { "outcome":"win", "rewards": {"gold":300,"gems":5,"materials":["m_02"]}, "verified": true } }
```

**WS — event (server→client)**
```ts
type ServerToClient = {
  leaderboard: { top: RankEntry[] };
  notification: { kind: 'reward'|'event'|'system'; body: string };
  pvp_match_found: { matchId: string; opponentElo: number };
  pvp_turn: { matchId: string; actor: string; action: PvpAction };
};
```

## Lampiran D. Diagram Urutan (Sequence)

**Gacha (server-authoritative)**
```
User → GachaScene → gacha.api.pull() → REST POST /gacha/pull
REST → Backend(server rolls, updates pity) → res GachaResult
GachaScene → metaStore/partyStore update (from server) → animasi reveal
```
**Idle accrual (offline→online)**
```
App launch → net/idle.api.claim(lastSeen) → REST POST /idle/claim
Server: acc = clamp(now-lastSeen,0,24h) → res rewards
playerStore update gold → HomeScene tampil accrual
Jika offline: claim di-queue IndexedDB → flush saat online
```
**Battle (prediction + verify)**
```
BattleScene → battle.api.start(stageId) → res BattleStartPacket
BattleFactory → World(entities+components) → LoopController fixed timestep
Systems → SimEvent[] → RenderBridge → visual
User intent → ECS command (prediksi) → battle.api.result(log)
Server validasi → res verified rewards → playerStore/inventory update
```

---

*Dokumen selesai. Semua angka merujuk CDP (Bagian 10.3 / Lampiran B). Arsitektur menjamin server-authoritative, ECS terdecouple & testable, PWA offline app-shell, dan target 60 FPS via fixed timestep.*
