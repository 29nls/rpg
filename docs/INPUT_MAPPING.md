# INPUT MAPPING — RPG Fantasy Turn-Based Idle (Web + PWA)

> **Dokumen:** INPUT_MAPPING.md
> **Engine:** Phaser 3 + TypeScript + Vite + PWA
> **Target platform:** Desktop-first (keyboard + mouse), dengan dukungan touch penuh untuk PWA/mobile
> **Genre:** RPG Fantasi, Turn-Based, Idle
> **Status:** Canonical Design Parameter (CDP) — wajib dipakai sebagai rujukan binding input
> **Versi dokumen:** 1.0.0
> **Bahasa:** Indonesia

---

## Daftar Isi

1. [Tujuan & Input Sources](#1-tujuan--input-sources)
2. [Action Map (Aksi Semantik)](#2-action-map-aksi-semantik)
3. [Default Bindings Table](#3-default-bindings-table)
4. [Context Layers](#4-context-layers)
5. [Rebinding](#5-rebinding)
6. [Event Flow](#6-event-flow)
7. [preventDefault & Focus](#7-preventdefault--focus)
8. [Accessibility](#8-accessibility)
9. [Edge Cases](#9-edge-cases)
10. [Hubungan dengan Dokumen Lain](#10-hubungan-dengan-dokumen-lain)
11. [Asumsi Eksplisit](#11-asumsi-eksplisit)
12. [Glosarium](#12-glosarium)

---

## 1. Tujuan & Input Sources

### 1.1 Tujuan Dokumen

Dokumen ini mendefinisikan **pemetaan input secara menyeluruh** untuk game RPG Fantasi *Turn-Based Idle* berbasis web dan PWA. Tujuannya:

- Menstandarkan bagaimana pemain berinteraksi dengan game di berbagai perangkat (keyboard, mouse, touch).
- Memisahkan **makna aksi** (semantik) dari **tombol fisik** (binding), sehingga satu aksi dapat dipicu dari banyak sumber input.
- Memberikan dasar implementasi yang konsisten bagi `InputManager`, *state machine*, dan layer UI.
- Menjamin input bisa di-*rebind* (dipetakan ulang) tanpa mengubah logika game.
- Menjamin aksesibilitas dan pengalaman *cross-device* yang seragam antara versi desktop dan PWA mobile.

### 1.2 Mengapa Action Map (Pemetaan Semantik) Penting — Bukan Hardcode Key

Pendekatan naive adalah menghardcode `if (key === 'Enter') confirm()`. Ini bermasalah karena:

| Masalah Hardcode Key | Dampak |
|---|---|
| Binding tersebar di banyak file | Sulit diubah, rawan bug duplikat |
| Tidak mendukung rebinding | Pemain tidak bisa menyesuaikan kontrol |
| Tidak cross-device | Touch/mouse butuh kode terpisah |
| Tidak ada konteks | Aksi sama berbeda arti di Menu vs Battle |
| Sulit diuji | Tidak ada titik sentral input |

Solusinya: **Action Map**. Setiap interaksi didefinisikan sebagai *action* semantik (mis. `Confirm`, `Skill1`), lalu action tersebut diikat ke satu atau lebih *physical binding* (keyboard key, mouse button, touch zone). Logika game hanya mendengarkan *action*, tidak mendengarkan tombol.

```
Tombol Fisik  ──►  Action Map  ──►  Semantic Action  ──►  State Machine / Handler
(Keyboard)        (rebindable)     (Confirm)             (Menu.confirm())
(Mouse)
(Touch)
```

Keuntungan:
- **Satu sumber kebenaran** untuk semua input.
- **Rebinding** cukup ubah tabel pemetaan, logika game tetap.
- **Multi-source**: keyboard, mouse, dan touch dapat memicu action yang sama.
- **Context layer**: action yang sama bisa diaktifkan/nonaktifkan per state.

### 1.3 Input Sources

Game mendukung tiga sumber input utama:

#### 1.3.1 Keyboard
- **Desktop utama.** Digunakan untuk navigasi, konfirmasi, skill battle, shortcut.
- Rentang: alphanumeric, arrows, WASD, function keys, modifier (Shift/Ctrl/Alt).
- Catatan: kombinasi modifier (Ctrl/Alt/Shift) didukung untuk shortcut lanjutan namun tidak dipakai sebagai default utama agar ramah pemula.

#### 1.3.2 Mouse
- **Desktop sekunder.** Klik UI, hover tooltip, drag pada market/guild, scroll pada list.
- Left click = `Confirm`/`Primary`, right click = `Cancel`/`Secondary` (pada konteks tertentu), wheel = scroll/navigasi list.
- Mouse tidak menggantikan keyboard sepenuhnya; semua aksi harus tetap reachable via keyboard (aksesibilitas).

#### 1.3.3 Touch (PWA / Mobile)
- **Target PWA di mobile/tablet.** Tap = `Confirm`, tap luar/back gesture = `Cancel`, swipe = navigasi, on-screen button untuk skill.
- Virtual gamepad/onscreen button dirender khusus di layer `Battle` untuk skill 1–5 dan `Summon`.
- Asumsi: layar sentuh minimal 360px lebar (mobile) hingga 1280px (tablet). Layout responsif.

#### 1.3.4 Tabel Ringkas Sumber Input

| Source | Prioritas | Primary Use | Catatan |
|---|---|---|---|
| Keyboard | Tinggi (desktop) | Navigasi, skill, shortcut | Wajib untuk aksesibilitas |
| Mouse | Menengah (desktop) | Klik UI, tooltip, drag | Tidak boleh jadi satu-satunya jalur |
| Touch | Tinggi (PWA/mobile) | Tap, swipe, onscreen button | Onscreen button wajib di Battle |

---

## 2. Action Map (Aksi Semantik)

Daftar aksi semantik canonical. Logika game **hanya** merujuk nama action ini (bukan tombol). Berikut makna masing-masing:

### 2.1 Daftar Action

| Action | Kategori | Makna | Sumber Umum |
|---|---|---|---|
| `Confirm` | Universal | Eksekusi pilihan / lanjut dialog / konfirmasi | Enter, Space, Klik Kiri, Tap |
| `Cancel` / `Back` | Universal | Batal / kembali ke menu sebelumnya / tutup panel | Esc, Klik Kanan, Back gesture |
| `NavigateUp` | Navigasi | Pindah fokus ke atas | W / ArrowUp |
| `NavigateDown` | Navigasi | Pindah fokus ke bawah | S / ArrowDown |
| `NavigateLeft` | Navigasi | Pindah fokus ke kiri | A / ArrowLeft |
| `NavigateRight` | Navigasi | Pindah fokus ke kanan | D / ArrowRight |
| `Skill1` | Battle | Gunakan skill slot 1 (biasanya basic attack / skill utama) | Q atau 1 |
| `Skill2` | Battle | Gunakan skill slot 2 | W atau 2 |
| `Skill3` | Battle | Gunakan skill slot 3 | E atau 3 |
| `Skill4` | Battle | Gunakan skill slot 4 | R atau 4 |
| `Skill5` | Battle | Gunakan skill slot 5 (ultimate / summon-ready) | T atau 5 |
| `Summon` | Battle/Gacha | Memanggil hero/unit via gacha atau battle summon | F |
| `OpenMarket` | Navigasi | Buka panel Market | M |
| `OpenGuild` | Navigasi | Buka panel Guild | G |
| `Pause` | System | Jeda game / buka pause menu | Esc (context) / P |
| `ToggleIdle` | Idle | Nyalakan/matikan mode idle (auto-battle/auto-progress) | I |
| `QuickBattle` | Battle/Idle | Mulai battle cepat (skip ke battle berikutnya) | B |
| `Settings` | System | Buka pengaturan (termasuk rebinding) | O / gear button |

### 2.2 Penjelasan Detail per Action

#### `Confirm`
Aksi paling fundamental. Digunakan di semua context: pilih menu, lanjut dialog, konfirmasi gacha pull, eksekusi di battle. Sumber: `Enter`, `Space` (khusus non-scroll context), klik kiri, tap.

#### `Cancel` / `Back`
Kebalikan `Confirm`. Menutup panel terbuka, membatalkan pilihan, kembali ke state sebelumnya. Sumber: `Esc`, klik kanan (pada UI tertentu), swipe-down / back gesture di PWA.

> **Catatan CDP:** `Esc` dipakai ganda sebagai `Cancel` dan `Pause` tergantung context layer (lihat §4). Di Battle, `Esc` = `Pause`; di Menu, `Esc` = `Cancel`/`Back`. Ini diselesaikan oleh context layer aktif.

#### `NavigateUp/Down/Left/Right`
Navigasi fokus antar elemen UI (menu, list hero, slot inventory). Di Battle, navigasi mungkin dipetakan ke pemilihan target musuh. Sumber utama: `WASD` dan `ArrowKeys`. Keduanya aktif bersamaan agar pemain bebas pilih.

#### `Skill1`..`Skill5`
Aksi battle. Memanggil skill pada slot 1–5. Default keyboard: `Q W E R T` (utama) **dan** `1 2 3 4 5` (alternatif). Touch: onscreen button skill. Mouse: klik ikon skill di HUD.

#### `Summon`
Memanggil hero/unit. Di context Gacha = tarik gacha; di context Battle = aktifkan summon Hero cadangan. Default `F`.

#### `OpenMarket`
Shortcut buka Market (beli/jual item, equipment). Default `M`. Di PWA: tombol di bottom nav.

#### `OpenGuild`
Shortcut buka Guild (raid guild, donasi, chat guild). Default `G`. Di PWA: tombol di bottom nav.

#### `Pause`
Jeda game dan buka pause menu (resume, settings, quit ke menu). Default `P`; di Battle `Esc` juga memetakan ke Pause.

#### `ToggleIdle`
Nyalakan/matikan mode idle/auto. Saat idle aktif, game otomatis battle/progress tanpa input. Default `I`. Status ditampilkan di HUD.

#### `QuickBattle`
Mulai battle cepat / skip animasi ke battle berikutnya. Default `B`. Mempercepat progression idle.

#### `Settings`
Buka panel Settings (termasuk Rebinding, audio, accessibility). Default `O`; juga reachable via gear icon (mouse/touch).

---

## 3. Default Bindings Table

Tabel pemetaan default. **Action** di sebelah kiri, **binding fisik** di kolom sumber. Binding alternatif ditulis dalam kurung.

### 3.1 Tabel Utama

| Action | Keyboard | Mouse | Touch (PWA) |
|---|---|---|---|
| `Confirm` | `Enter`, `Space` | Left Click | Tap |
| `Cancel` / `Back` | `Esc` | Right Click | Back gesture / Tap luar |
| `NavigateUp` | `W` / `ArrowUp` | — | Swipe Up |
| `NavigateDown` | `S` / `ArrowDown` | — | Swipe Down |
| `NavigateLeft` | `A` / `ArrowLeft` | — | Swipe Left |
| `NavigateRight` | `D` / `ArrowRight` | — | Swipe Right |
| `Skill1` | `Q` / `1` | Klik ikon skill 1 | Onscreen btn Skill1 |
| `Skill2` | `W` / `2` | Klik ikon skill 2 | Onscreen btn Skill2 |
| `Skill3` | `E` / `3` | Klik ikon skill 3 | Onscreen btn Skill3 |
| `Skill4` | `R` / `4` | Klik ikon skill 4 | Onscreen btn Skill4 |
| `Skill5` | `T` / `5` | Klik ikon skill 5 | Onscreen btn Skill5 |
| `Summon` | `F` | Klik btn Summon | Onscreen btn Summon |
| `OpenMarket` | `M` | Klik tab Market | Bottom nav Market |
| `OpenGuild` | `G` | Klik tab Guild | Bottom nav Guild |
| `Pause` | `P` (Battle: `Esc`) | Klik btn Pause | Onscreen btn Pause |
| `ToggleIdle` | `I` | Klik toggle Idle | Toggle Idle di HUD |
| `QuickBattle` | `B` | Klik btn Quick | Onscreen btn Quick |
| `Settings` | `O` | Klik gear icon | Gear icon di top bar |

### 3.2 Catatan Konflik Default (Disengaja)

- `W` dipakai ganda: sebagai `NavigateUp` (navigasi) **dan** `Skill2` (battle). Ini aman karena keduanya berada di **context layer berbeda** (Menu vs Battle), tidak aktif bersamaan (lihat §4). Pemain tidak akan bingung karena saat Battle, navigasi WASD nonaktif.
- `Esc` ganda `Cancel`/`Pause` diselesaikan oleh context layer.
- `Space` sebagai `Confirm` hanya di context non-scroll; di list panjang `Space` diubah jadi scroll (lihat §7).

### 3.3 Tabel Alternatif (Power User)

Untuk pemain keyboard enthusiast, disediakan preset alternatif "Classic RPG" (diaktifkan via Settings):

| Action | Classic RPG Preset |
|---|---|
| `Skill1..5` | `1 2 3 4 5` (tanpa QWERTY) |
| `Confirm` | `Enter` only |
| `Cancel` | `Esc` / `Backspace` |
| `Navigate` | `ArrowKeys` only |

> Preset ini disimpan sebagai profil terpisah di save data, tidak mengubah default.

---

## 4. Context Layers

Game menggunakan **state machine** dengan context berbeda. Setiap context memiliki *layer* input yang mengaktifkan subset action tertentu. Action yang tidak ada di layer aktif diabaikan (tidak di-dispatch).

### 4.1 Daftar Context / State

Berdasarkan CDP, state machine punya context: **Menu**, **Battle**, **Map/Idle**. Plus sub-context: **Modal** (overlay), **Auth** (login).

| Context | Deskripsi | Layer Input |
|---|---|---|
| `Menu` | Main menu, inventory, hero list, settings | Menu Layer |
| `Battle` | Turn-based combat aktif | Battle Layer |
| `Map` / `Idle` | World map, idle progress, market/guild view | Map Layer |
| `Modal` | Dialog, confirm box, gacha result overlay | Modal Layer (intercept) |
| `Auth` | Login/register email-password | Auth Layer (text input focus) |

### 4.2 Tabel Layer → Action Aktif

#### Menu Layer (context: Menu, Map/Idle dasar navigasi)
| Action | Aktif? | Keterangan |
|---|---|---|
| `Confirm` | ✅ | Pilih menu |
| `Cancel`/`Back` | ✅ | Kembali |
| `NavigateUp/Down/Left/Right` | ✅ | Navigasi menu/list |
| `Skill1..5` | ❌ | Nonaktif di menu |
| `Summon` | ✅ (Gacha only) | Jika di panel gacha |
| `OpenMarket` | ✅ | |
| `OpenGuild` | ✅ | |
| `Pause` | ❌ | Tidak ada pause di menu |
| `ToggleIdle` | ✅ | Toggle idle dari map |
| `QuickBattle` | ✅ | Mulai battle dari map |
| `Settings` | ✅ | |

#### Battle Layer (context: Battle)
| Action | Aktif? | Keterangan |
|---|---|---|
| `Confirm` | ✅ | Konfirmasi target/aksi |
| `Cancel`/`Back` | ⚠️ | Hanya batalkan pemilihan target; tidak keluar battle |
| `NavigateUp/Down/Left/Right` | ✅ (target select) | Pilih musuh/ally, bukan menu |
| `Skill1..5` | ✅ | **Aktif penuh** — inti battle |
| `Summon` | ✅ | Summon hero cadangan |
| `OpenMarket` | ❌ | Nonaktif saat battle |
| `OpenGuild` | ❌ | Nonaktif saat battle |
| `Pause` | ✅ | `Esc`/`P` → pause menu |
| `ToggleIdle` | ✅ | Auto-battle on/off di battle |
| `QuickBattle` | ✅ | Skip animasi |
| `Settings` | ✅ | |

#### Map / Idle Layer (context: Map, idle progress)
| Action | Aktif? | Keterangan |
|---|---|---|
| `Confirm` | ✅ | Interaksi node |
| `Cancel`/`Back` | ✅ | Tutup panel |
| `Navigate` | ✅ | Navigasi peta/world |
| `Skill1..5` | ❌ | Nonaktif (bukan battle) |
| `Summon` | ❌ | (kecuali gacha panel) |
| `OpenMarket` | ✅ | |
| `OpenGuild` | ✅ | |
| `Pause` | ❌ | |
| `ToggleIdle` | ✅ | **Penting di idle** |
| `QuickBattle` | ✅ | |
| `Settings` | ✅ | |

#### Modal Layer (context: Modal overlay terbuka)
Modal **intercept** semua input dan hanya melewatkan `Confirm`, `Cancel`, `Navigate`. Action lain di-blok sampai modal ditutup.

| Action | Aktif? |
|---|---|
| `Confirm` | ✅ |
| `Cancel`/`Back` | ✅ |
| `NavigateUp/Down` | ✅ (pilih opsi dialog) |
| Lainnya | ❌ (di-blok) |

#### Auth Layer (context: login/register)
Input difokuskan ke field teks email/password. Shortcut game dinonaktifkan agar pengetikan bebas. Hanya `Confirm` (submit) dan `Cancel` (batal) aktif.

| Action | Aktif? |
|---|---|
| `Confirm` | ✅ (submit form) |
| `Cancel`/`Back` | ✅ (kembali/close) |
| Lainnya | ❌ |

### 4.3 Prioritas Layer

Jika beberapa layer overlap (mis. Battle + Modal), **Modal layer paling tinggi prioritas** (intercept). Urutan prioritas:

```
Modal > Auth > Battle > Map/Idle > Menu
```

Artinya: saat modal terbuka di atas Battle, hanya action Modal yang dilewatkan.

---

## 5. Rebinding

Pemain dapat mengubah binding action. Binding disimpan ke save data (`localStorage` via PWA storage).

### 5.1 Aturan Rebinding

- Setiap action dapat diikat ke **satu atau lebih** binding fisik.
- Rebinding dilakukan per action, tidak per tombol (satu tombol bisa dipakai banyak action asal di layer berbeda — lihat `W` ganda §3.2).
- Konflik **dalam layer yang sama** dilarang dan divalidasi.
- Reset-to-default tersedia.

### 5.2 Penyimpanan

Binding disimpan dalam save data dengan struktur:

```json
{
  "version": 1,
  "profile": "default",
  "bindings": {
    "Confirm":    ["Enter", "Space"],
    "Cancel":     ["Escape"],
    "NavigateUp": ["KeyW", "ArrowUp"],
    "Skill1":     ["KeyQ", "Digit1"],
    "Skill2":     ["KeyW", "Digit2"],
    "Summon":     ["KeyF"],
    "OpenMarket": ["KeyM"],
    "OpenGuild":  ["KeyG"],
    "Pause":      ["KeyP", "Escape"],
    "ToggleIdle": ["KeyI"],
    "QuickBattle":["KeyB"],
    "Settings":   ["KeyO"]
  }
}
```

> Kode tombol menggunakan `KeyboardEvent.code` (bukan `key`) agar layout keyboard internasional konsisten (mis. `KeyQ` bukan `'q'`).

### 5.3 Validasi Konflik

Saat rebind, sistem:
1. Cek apakah tombol sudah dipakai action lain **dalam layer yang sama**.
2. Jika ya → tolak dengan pesan "Tombol sudah dipakai oleh <Action>".
3. Jika hanya bentrok di layer berbeda → izinkan (seperti `W` default).
4. Cegah binding `Confirm` dikosongkan (wajib ada minimal satu).

### 5.4 Reset to Default

Tombol "Reset to Default" mengembalikan `bindings` ke tabel §3 tanpa menghapus progres game. Tersimpan sebagai profil `default`.

### 5.5 UI Rebinding (Flow)

1. Buka `Settings` → tab `Controls`.
2. Pilih action dari list.
3. Klik "Rebind" → sistem masuk mode listen.
4. Pemain tekan tombol / klik / tap.
5. Sistem validasi → simpan atau tolak.
6. Perubahan langsung berlaku & persist ke save data.

---

## 6. Event Flow

Alur: input fisik → listener → `InputManager` → resolve action (via Action Map + context layer) → dispatch ke handler / state machine.

### 6.1 Diagram Alur

```
[Keyboard/Mouse/Touch Event]
        │
        ▼
   Event Listener (Phaser Input / DOM)
        │  (capture, preventDefault bila perlu)
        ▼
   InputManager.queue(event)
        │
        ▼
   InputManager.resolve()  ──► Action Map lookup (code → action)
        │                      + active Context Layer filter
        ▼
   Action (semantic) + payload
        │
        ▼
   StateMachine.dispatch(action)  ──► Handler / UI / Battle logic
```

### 6.2 Komponen InputManager (TypeScript)

```typescript
// ===== types.ts =====
export type ActionName =
  | 'Confirm' | 'Cancel' | 'NavigateUp' | 'NavigateDown'
  | 'NavigateLeft' | 'NavigateRight'
  | 'Skill1' | 'Skill2' | 'Skill3' | 'Skill4' | 'Skill5'
  | 'Summon' | 'OpenMarket' | 'OpenGuild' | 'Pause'
  | 'ToggleIdle' | 'QuickBattle' | 'Settings';

export type InputSource = 'keyboard' | 'mouse' | 'touch';

export interface InputEvent {
  source: InputSource;
  code: string;          // KeyboardEvent.code, atau 'mouse-left', 'touch-tap', dll
  repeat: boolean;       // true bila key held (repeat event)
  timestamp: number;
}

export interface Binding {
  action: ActionName;
  codes: string[];       // physical codes, mis. ['KeyQ','Digit1']
}

export type ContextLayer =
  | 'Menu' | 'Battle' | 'Map' | 'Modal' | 'Auth';
```

```typescript
// ===== InputManager.ts =====
import Phaser from 'phaser';
import {
  ActionName, Binding, ContextLayer, InputEvent, InputSource,
} from './types';

const DEFAULT_BINDINGS: Binding[] = [
  { action: 'Confirm',    codes: ['Enter', 'Space', 'mouse-left', 'touch-tap'] },
  { action: 'Cancel',     codes: ['Escape', 'mouse-right', 'touch-back'] },
  { action: 'NavigateUp',    codes: ['KeyW', 'ArrowUp', 'touch-swipe-up'] },
  { action: 'NavigateDown',  codes: ['KeyS', 'ArrowDown', 'touch-swipe-down'] },
  { action: 'NavigateLeft',  codes: ['KeyA', 'ArrowLeft', 'touch-swipe-left'] },
  { action: 'NavigateRight', codes: ['KeyD', 'ArrowRight', 'touch-swipe-right'] },
  { action: 'Skill1', codes: ['KeyQ', 'Digit1', 'touch-skill1'] },
  { action: 'Skill2', codes: ['KeyW', 'Digit2', 'touch-skill2'] },
  { action: 'Skill3', codes: ['KeyE', 'Digit3', 'touch-skill3'] },
  { action: 'Skill4', codes: ['KeyR', 'Digit4', 'touch-skill4'] },
  { action: 'Skill5', codes: ['KeyT', 'Digit5', 'touch-skill5'] },
  { action: 'Summon',     codes: ['KeyF', 'touch-summon'] },
  { action: 'OpenMarket', codes: ['KeyM', 'touch-market'] },
  { action: 'OpenGuild',  codes: ['KeyG', 'touch-guild'] },
  { action: 'Pause',      codes: ['KeyP', 'Escape', 'touch-pause'] },
  { action: 'ToggleIdle', codes: ['KeyI', 'touch-idle'] },
  { action: 'QuickBattle',codes: ['KeyB', 'touch-quick'] },
  { action: 'Settings',   codes: ['KeyO', 'touch-settings'] },
];

// Action aktif per layer
const LAYER_ACTIONS: Record<ContextLayer, ActionName[]> = {
  Menu: ['Confirm','Cancel','NavigateUp','NavigateDown','NavigateLeft','NavigateRight',
         'Summon','OpenMarket','OpenGuild','ToggleIdle','QuickBattle','Settings'],
  Battle: ['Confirm','Cancel','NavigateUp','NavigateDown','NavigateLeft','NavigateRight',
           'Skill1','Skill2','Skill3','Skill4','Skill5','Summon','Pause','ToggleIdle',
           'QuickBattle','Settings'],
  Map: ['Confirm','Cancel','NavigateUp','NavigateDown','NavigateLeft','NavigateRight',
        'OpenMarket','OpenGuild','ToggleIdle','QuickBattle','Settings'],
  Modal: ['Confirm','Cancel','NavigateUp','NavigateDown'],
  Auth: ['Confirm','Cancel'],
};

export class InputManager {
  private bindings: Binding[];
  private activeLayer: ContextLayer = 'Menu';
  private queue: InputEvent[] = [];
  private listeners: ((a: ActionName, e: InputEvent) => void)[] = [];
  private lastDispatch = new Map<string, number>();
  private readonly REPEAT_MS = 180; // debounce repeat

  constructor(private scene: Phaser.Scene) {
    this.bindings = this.loadBindings() ?? DEFAULT_BINDINGS;
    this.attachDomListeners();
  }

  // ---- Persistensi ----
  private loadBindings(): Binding[] | null {
    try {
      const raw = localStorage.getItem('rpg.input.bindings.v1');
      return raw ? JSON.parse(raw) as Binding[] : null;
    } catch {
      return null;
    }
  }

  saveBindings(): void {
    localStorage.setItem('rpg.input.bindings.v1', JSON.stringify(this.bindings));
  }

  resetToDefault(): void {
    this.bindings = DEFAULT_BINDINGS.map(b => ({ ...b, codes: [...b.codes] }));
    this.saveBindings();
  }

  // ---- Rebind ----
  rebind(action: ActionName, codes: string[], layer: ContextLayer): boolean {
    // validasi konflik dalam layer yang sama
    const allowed = new Set(LAYER_ACTIONS[layer]);
    for (const other of this.bindings) {
      if (other.action === action) continue;
      if (!allowed.has(other.action)) continue; // beda layer, boleh bentrok
      const clash = other.codes.find(c => codes.includes(c));
      if (clash) return false; // konflik
    }
    const entry = this.bindings.find(b => b.action === action);
    if (entry) entry.codes = [...codes];
    else this.bindings.push({ action, codes });
    this.saveBindings();
    return true;
  }

  // ---- Context layer ----
  setLayer(layer: ContextLayer): void {
    this.activeLayer = layer;
    this.queue.length = 0; // buang input tersisa saat ganti context
  }

  getLayer(): ContextLayer { return this.activeLayer; }

  // ---- Listener DOM ----
  private attachDomListeners(): void {
    window.addEventListener('keydown', (e) => {
      // abaikan bila fokus di input teks (Auth layer menangani sendiri)
      const tag = (e.target as HTMLElement)?.tagName;
      if (tag === 'INPUT' || tag === 'TEXTAREA') return;
      this.enqueue({
        source: 'keyboard',
        code: e.code,
        repeat: e.repeat,
        timestamp: performance.now(),
      });
    });
    // mouse & touch di-handle via Phaser scene input (lihat bawah)
  }

  enqueue(e: InputEvent): void {
    this.queue.push(e);
  }

  // Dipanggil dari Phaser pointer events
  enqueuePointer(source: InputSource, code: string): void {
    this.enqueue({ source, code, repeat: false, timestamp: performance.now() });
  }

  // ---- Resolve ----
  update(): void {
    while (this.queue.length) {
      const ev = this.queue.shift()!;
      const action = this.resolve(ev);
      if (!action) continue;
      // debounce repeat
      if (ev.repeat) {
        const last = this.lastDispatch.get(action) ?? 0;
        if (ev.timestamp - last < this.REPEAT_MS) continue;
        this.lastDispatch.set(action, ev.timestamp);
      }
      this.dispatch(action, ev);
    }
  }

  private resolve(ev: InputEvent): ActionName | null {
    const allowed = LAYER_ACTIONS[this.activeLayer];
    for (const b of this.bindings) {
      if (!allowed.includes(b.action)) continue;
      if (b.codes.includes(ev.code)) return b.action;
    }
    return null;
  }

  private dispatch(action: ActionName, ev: InputEvent): void {
    for (const l of this.listeners) l(action, ev);
  }

  onAction(cb: (a: ActionName, e: InputEvent) => void): void {
    this.listeners.push(cb);
  }
}
```

### 6.3 Integrasi dengan Phaser Scene

```typescript
// ===== BattleScene.ts (cuplikan) =====
export class BattleScene extends Phaser.Scene {
  private input!: InputManager;

  create() {
    this.input = new InputManager(this);
    this.input.setLayer('Battle');

    this.input.onAction((action) => {
      switch (action) {
        case 'Skill1': this.useSkill(0); break;
        case 'Skill2': this.useSkill(1); break;
        case 'Skill3': this.useSkill(2); break;
        case 'Skill4': this.useSkill(3); break;
        case 'Skill5': this.useSkill(4); break;
        case 'Summon': this.summonHero(); break;
        case 'Pause':  this.openPause(); break;
        case 'ToggleIdle': this.toggleIdle(); break;
        case 'Confirm': this.confirmTarget(); break;
        case 'Cancel':  this.cancelTarget(); break;
        case 'NavigateUp': this.moveTarget(-1); break;
        case 'NavigateDown': this.moveTarget(1); break;
      }
    });

    // pointer (mouse/touch) -> enqueue
    this.input.on('pointerdown', (p: Phaser.Input.Pointer) => {
      if (p.rightButtonDown())
        this.input.enqueuePointer('mouse', 'mouse-right');
      else
        this.input.enqueuePointer('mouse', 'mouse-left');
    });
  }

  update() {
    this.input.update(); // proses queue setiap frame
  }
}
```

### 6.4 Debounce & Repeat Handling

- **Key repeat:** saat tombol ditahan, browser memancarkan event `repeat: true` berkala. `InputManager` mendebounce dengan `REPEAT_MS = 180ms` agar navigasi tidak lompat terlalu cepat.
- **Tap vs hold:** touch tap dianggap single event (`repeat: false`). Hold untuk swipe dideteksi terpisah.
- **Double-trigger cegah:** tiap action dicek `lastDispatch` timestamp untuk hindari duplikat dalam window pendek (kecuali `Confirm` yang boleh cepat di menu).

---

## 7. preventDefault & Focus

### 7.1 Mencegah Scroll & Gesture Browser

Beberapa tombol default memicu perilaku browser yang mengganggu:

| Tombol | Perilaku Browser Default | Tindakan |
|---|---|---|
| `Space` | Scroll halaman ke bawah | `preventDefault()` bila context non-input teks |
| `ArrowUp/Down` | Scroll halaman | `preventDefault()` saat navigasi aktif |
| `ArrowLeft/Right` | Scroll horizontal / back nav | `preventDefault()` |
| `Backspace` | Navigasi mundur (lama) | `preventDefault()` di game |
| `F5 / Ctrl+R` | Reload | **Biarkan** (bukan game binding) |
| `Ctrl/Cmd+W` | Tutup tab | **Biarkan** (jangan cegat) |
| `F11` | Fullscreen | Biarkan (PWA punya tombol sendiri) |
| `F12` | DevTools | Biarkan |

Implementasi:

```typescript
const BLOCKED = new Set([
  'Space','ArrowUp','ArrowDown','ArrowLeft','ArrowRight','Backspace',
]);

window.addEventListener('keydown', (e) => {
  if (BLOCKED.has(e.code)) {
    const tag = (e.target as HTMLElement)?.tagName;
    if (tag !== 'INPUT' && tag !== 'TEXTAREA') e.preventDefault();
  }
}, { passive: false });
```

### 7.2 Menjaga Fokus Canvas

- Canvas Phaser harus tetap fokus agar keyboard bekerja. Saat klik elemen luar (mis. tombol HTML di PWA shell), kembalikan fokus ke canvas game setelah interaksi.
- Gunakan `tabindex="0"` pada container game agar bisa menerima fokus keyboard.
- Saat PWA shell (top/bottom nav) diklik, panggil `gameCanvas.focus()` setelahnya.

```typescript
function refocusGame() {
  document.getElementById('game-canvas')?.focus();
}
// panggil setelah interaksi UI shell
```

### 7.3 PWA & Mobile Focus

- Di PWA, `touch-action: none` pada canvas mencegah gesture zoom/scroll mengganggu input game.
- `user-scalable=no` di viewport meta (PWA game) agar pinch-zoom tidak mengacaukan swipe navigasi.
- Soft keyboard (saat Auth) tidak boleh menutupi canvas; gunakan layout yang shift ke atas.

### 7.4 Aksesibilitas Fokus

- Indikator fokus visual (outline) harus terlihat jelas pada elemen UI yang sedang difokus (lihat §8).

---

## 8. Accessibility

### 8.1 Prinsip

Game harus bisa dimainkan tanpa melihat layar sepenuhnya (keyboard navigable) dan ramah pemain dengan keterbatasan motorik/visual.

### 8.2 Tombol Besar (Touch)

- Onscreen button minimal **44x44 px** (rekomendasi WCAG/Apple HIG) — ideal 56x56 px di mobile.
- Jarak antar tombol minimal 8px untuk hindari *mis-tap*.
- Skill button di Battle diperbesar saat idle mode untuk kemudahan.

### 8.3 Remappable & Custom Layout

- Semua action remappable (§5).
- Layout PWA dapat diatur: pemain bisa pindah posisi onscreen button (kiri/kanan) untuk *left/right hand*.
- Ukuran tombol bisa diperbesar via Settings → Accessibility.

### 8.4 Hint Visual

- Setiap tombol menampilkan **label binding** (mis. "Q", "1", ikon). Bila direbind, label ikut berubah.
- Tooltip muncul saat hover/focus menjelaskan fungsi action.
- Highlight action yang sedang tersedia (skill cooldown → abu-abu + progress).

### 8.5 Screen Reader / Label

- Elemen UI menggunakan `aria-label` yang deskriptif (mis. `aria-label="Buka Market"`).
- Onscreen button PWA: `<button aria-label="Skill 1: Fireball">`.
- State perubahan (idle on/off, cooldown) diumumkan via `aria-live="polite"`.
- Mode "High Contrast" di Settings untuk pemain low-vision.

```html
<button aria-label="Skill 1: Fireball" class="onscreen-btn">
  <span class="key-hint">Q</span>
  <span class="skill-name">Fireball</span>
</button>
```

### 8.6 Navigasi Keyboard Penuh

- Semua menu dapat dinavigasi via `Tab`/`Arrow` + `Enter`.
- Urutan fokus logis (tidak lompat acak).
- `Esc` selalu kembali/offboard dengan aman.

### 8.7 Reduced Motion

- Bila `prefers-reduced-motion: reduce`, animasi transisi input (shake, flash) dikurangi.
- Tetap berikan feedback non-motion (warna/teks).

### 8.8 Tabel Aksesibilitas Ceklis

| Fitur | Status Wajib | Catatan |
|---|---|---|
| Semua aksi reachable via keyboard | ✅ | Tanpa mouse |
| Label aria pada UI | ✅ | |
| Tombol min 44px | ✅ | Touch |
| Hint binding visual | ✅ | |
| Rebinding tersimpan | ✅ | |
| High contrast mode | ✅ | |
| Reduced motion | ✅ | |
| Screen reader announce state | ⚠️ | Recommended |

---

## 9. Edge Cases

### 9.1 Input Saat Modal Terbuka

- Modal layer (§4) **intercept** semua input. Action di luar `Confirm/Cancel/Navigate` diabaikan.
- Jika pemain menekan `Skill1` saat modal gacha result terbuka → diabaikan (tidak leak ke Battle di bawahnya).
- Pastikan `setLayer('Modal')` dipanggil saat modal `open()` dan `setLayer(prev)` saat `close()`.

### 9.2 Input Saat Tab Hidden (`document.hidden`)

- Saat tab tidak terlihat (pindah tab / minimize), **abaikan semua input**.
- Game berjalan terus (idle mode) tanpa butuh input.
- `visibilitychange` listener: saat hidden, pause input queue processing (jangan proses event yang sempat masuk).
- Hindari *auto-repeat* menumpuk saat kembali (clear queue saat `visible` kembali, kecuali idle tetap jalan).

```typescript
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    inputManager.clearQueue(); // buang input tersisa
  }
});
```

### 9.3 Ghost Input

- **Ghost input** = event input yang tidak berasal dari interaksi nyata (mis. event keyboard tiruan, event pointer sisa saat ganti scene).
- Cegah dengan: `clearQueue()` saat `setLayer()` / transisi scene.
- Phaser: pastikan listener di-`off` saat scene `shutdown` agar tidak dobel.
- Jangan biarkan dua scene aktif mendengar input bersamaan tanpa filter layer.

```typescript
this.events.once('shutdown', () => {
  this.input.off('pointerdown'); // bersihkan
});
```

### 9.4 Input Saat Animasi/Cutscene

- Saat cutscene/animasi wajib (intro), semua input di-queue tapi tidak diproses hingga animasi selesai, kecuali `Cancel`/`Settings` (skip? tergantung desain — default: tidak bisa skip cutscene wajib).
- Input tersimpan di queue; bila terlalu tua (>500ms) dibuang saat animasi selesai.

### 9.5 Konflik Mouse Klik Ganda & Touch Double-Tap

- Double-click mouse = dua `Confirm` berturut. Di menu ini aman; di battle bisa picu skill 2x. Cegah dengan debounce `Confirm` 120ms di Battle.
- Double-tap touch = zoom di browser. `touch-action: none` + `user-scalable=no` cegah.

### 9.6 Input Saat Soft Keyboard (Auth)

- Saat field email/password fokus, **nonaktifkan** shortcut game (Auth layer §4).
- Tombol fisik harus mengetik ke field, bukan memicu `OpenMarket` dll.
- `InputManager` skip event bila `e.target` adalah INPUT/TEXTAREA (sudah di §6.2).

### 9.7 Resize / Orientation Change

- Saat rotate device (portrait↔landscape) atau resize window, layout PWA berubah. Re-layout onscreen button; binding tetap sama.
- Hindari input terlewat saat transisi (brief lock 100ms).

### 9.8 Focus Hilang ke DevTools / Address Bar

- Jika fokus pindah ke address bar (klik URL), keyboard game mati. Saat kembali, `refocusGame()` (§7.2) dipanggil otomatis via `window.focus` listener.

### 9.9 Tabel Ringkas Edge Case

| Skenario | Penanganan |
|---|---|
| Modal terbuka | Intercept, blok action non-modal |
| Tab hidden | Abaikan + clear queue |
| Ghost input | Clear queue saat transisi scene |
| Cutscene | Queue tapi jangan proses |
| Double input | Debounce 120ms di Battle |
| Auth typing | Skip shortcut game |
| Rotate/resize | Re-layout, bind tetap |
| Focus hilang | Refocus canvas otomatis |

---

## 10. Hubungan dengan Dokumen Lain

Dokumen ini terhubung erat dengan dokumen arsitektur game lainnya:

### 10.1 State Machine (Context Layers)

- **Dokumen rujukan:** `STATE_MACHINE.md` (asumsi ada).
- Context layer di §4 **harus sinkron** dengan state machine. Setiap transisi state memanggil `inputManager.setLayer(context)`.
- Action di-dispatch ke handler state terkait. Bila state machine tidak punya handler untuk action, action diabaikan (fail-safe).

```
StateMachine.changeState('Battle')
   └─► InputManager.setLayer('Battle')
   └─► BattleHandlers terdaftar di InputManager.onAction
```

### 10.2 Save Data (Persist Binding)

- **Dokumen rujukan:** `SAVE_DATA.md` (asumsi ada).
- Binding disimpan di save data key `rpg.input.bindings.v1` (localStorage / IndexedDB PWA).
- Saat load game, `InputManager` membaca binding tersimpan; bila kosong → default.
- Versioning: `version` field agar migrasi binding aman bila struktur berubah.

### 10.3 Game Loop

- **Dokumen rujukan:** `GAME_LOOP.md` (asumsi ada).
- `InputManager.update()` dipanggil di `scene.update()` setiap frame (bukan event langsung) agar pemrosesan input sinkron dengan game loop dan hindari race condition.
- Action hasil resolve masuk antrian → diproses di update → dikirim ke state machine.

### 10.4 PWA Manifest & Service Worker

- **Dokumen rujukan:** `PWA_SETUP.md` (asumsi ada).
- `display: standalone` agar tidak ada address bar (fokus penuh).
- Touch binding aktif saat PWA diinstal di mobile.
- Offline: input tetap bekerja (logika lokal), server hanya untuk sync market/guild.

### 10.5 Audio / Settings

- **Dokumen rujukan:** `SETTINGS.md`.
- `Settings` action membuka panel yang juga mengatur audio, accessibility, dan rebinding.
- Perubahan accessibility (high contrast, reduced motion) memengaruhi render onscreen button.

### 10.6 Diagram Keterhubungan

```
INPUT_MAPPING.md
   │
   ├─► STATE_MACHINE.md  (context layer sinkron)
   ├─► SAVE_DATA.md      (persist bindings)
   ├─► GAME_LOOP.md      (update() tiap frame)
   ├─► PWA_SETUP.md      (touch, standalone)
   └─► SETTINGS.md       (rebind UI, accessibility)
```

---

## 11. Asumsi Eksplisit

Berikut asumsi yang diambil karena tidak semua detail dispesifikkan. Asumsi ini **wajib** diikuti kecuali ada revisi CDP:

1. **Layout keyboard internasional:** Menggunakan `KeyboardEvent.code` (physical key) bukan `key`, agar konsisten antar-layout (QWERTY, AZERTY, dll).
2. **`W` ganda** (NavigateUp + Skill2) dianggap aman karena beda context layer (Menu vs Battle).
3. **`Esc` ganda** (Cancel + Pause) diselesaikan oleh context layer aktif.
4. **Mouse right-click** = `Cancel` hanya di UI tertentu; di canvas battle tidak memicu cancel langsung (kecuali target select).
5. **Touch gesture** menggunakan kode semantik (`touch-tap`, `touch-swipe-up`, dll) yang dipetakan dari Phaser pointer + deteksi swipe kustom.
6. **PWA minimal lebar 360px**; layout responsif hingga 1280px (tablet).
7. **Save data** menggunakan `localStorage` untuk simplicity; bila butuh quota besar bisa migrasi ke IndexedDB tanpa ubah interface `InputManager`.
8. **Onscreen button** dirender hanya di context Battle/Map saat detect touch device (`'ontouchstart' in window` atau PWA standalone + coarse pointer).
9. **Auth layer** menonaktifkan semua shortcut game agar pengetikan email/password bebas.
10. **Idle mode** berjalan tanpa input (auto-battle/progress); `ToggleIdle` hanya switch on/off, bukan menghentikan game loop.
11. **Modal selalu di atas** semua layer (prioritas tertinggi intercept).
12. **Cutscene wajib** tidak bisa diskip (default); bila desain berubah, `Cancel` bisa diizinkan skip.
13. **Repeat debounce 180ms** untuk navigasi; **Confirm debounce 120ms** di Battle untuk cegah double-skill.
14. **Reset to default** tidak menghapus progres game, hanya mengembalikan bindings.
15. **High contrast & reduced motion** disediakan sebagai mode di Settings (aksesibilitas).

---

## 12. Glosarium

| Istilah | Arti |
|---|---|
| **Action Map** | Tabel pemetaan aksi semantik ke binding fisik |
| **Action (Semantik)** | Nama aksi bermakna (Confirm, Skill1, dll), bukan tombol |
| **Binding** | Pemetaan action ke tombol fisik tertentu |
| **Context Layer** | Kumpulan action aktif per state game |
| **Rebinding** | Mengubah binding action oleh pemain |
| **Ghost Input** | Event input tiruan/sisa yang tidak diinginkan |
| **Modal** | Overlay dialog yang intercept input |
| **Idle Mode** | Mode auto-progress tanpa input aktif |
| **Onscreen Button** | Tombol virtual di layar untuk touch |
| **Debounce** | Penundaan/penyaringan event berulang |
| **CDP** | Canonical Design Parameter (parameter wajib) |
| **PWA** | Progressive Web App |
| **Phaser 3** | Game engine HTML5 |

---

## Lampiran A: Quick Reference (Cheat Sheet)

```
┌─────────────────────────────────────────────────────┐
│ DEFAULT BINDINGS — RPG Fantasy Idle                  │
├──────────────────┬──────────────────────────────────┤
│ Confirm          │ Enter / Space / Klik / Tap        │
│ Cancel / Back    │ Esc / Klik Kanan / Back           │
│ Navigate         │ WASD / Arrows / Swipe             │
│ Skill 1-5        │ Q W E R T  (atau 1 2 3 4 5)        │
│ Summon           │ F                                 │
│ Market           │ M                                 │
│ Guild            │ G                                 │
│ Pause            │ P (Battle: Esc)                   │
│ Toggle Idle      │ I                                 │
│ Quick Battle     │ B                                 │
│ Settings         │ O                                 │
└──────────────────┴──────────────────────────────────┘
```

## Lampiran B: Contoh Alur Penggunaan (User Journey)

1. Pemain buka game → `Menu` layer aktif.
2. Tekan `M` → `OpenMarket` → Market panel (Map layer).
3. Tekan `B` → `QuickBattle` → transisi ke `Battle` layer.
4. Di Battle: `Skill1` (Q) serang, `Skill5` (T) ultimate, `Summon` (F) panggil hero.
5. `Esc`/`P` → `Pause` → pause menu.
6. `I` → `ToggleIdle` → auto-battle nyala; pemain bisa tinggalkan game.
7. Buka `Settings` (O) → `Controls` → rebind `Skill1` ke `Digit1` saja.
8. Perubahan persist ke save data; saat reload, binding tetap.

## Lampiran C: Validasi Rebinding — Pseudocode

```
function tryRebind(action, newCodes, layer):
    allowed = LAYER_ACTIONS[layer]
    for each otherAction in bindings:
        if otherAction == action: continue
        if otherAction not in allowed: continue
        for code in newCodes:
            if code in bindings[otherAction].codes:
                return ERROR("Bentrok dengan " + otherAction)
    if action == 'Confirm' and newCodes empty:
        return ERROR("Confirm wajib ada")
    bindings[action].codes = newCodes
    save()
    return OK
```

## Lampiran D: Template Save Data Bindings

```json
{
  "version": 1,
  "profile": "default",
  "bindings": {
    "Confirm":    ["Enter","Space","mouse-left","touch-tap"],
    "Cancel":     ["Escape","mouse-right","touch-back"],
    "NavigateUp":    ["KeyW","ArrowUp","touch-swipe-up"],
    "NavigateDown":  ["KeyS","ArrowDown","touch-swipe-down"],
    "NavigateLeft":  ["KeyA","ArrowLeft","touch-swipe-left"],
    "NavigateRight": ["KeyD","ArrowRight","touch-swipe-right"],
    "Skill1": ["KeyQ","Digit1","touch-skill1"],
    "Skill2": ["KeyW","Digit2","touch-skill2"],
    "Skill3": ["KeyE","Digit3","touch-skill3"],
    "Skill4": ["KeyR","Digit4","touch-skill4"],
    "Skill5": ["KeyT","Digit5","touch-skill5"],
    "Summon":     ["KeyF","touch-summon"],
    "OpenMarket": ["KeyM","touch-market"],
    "OpenGuild":  ["KeyG","touch-guild"],
    "Pause":      ["KeyP","Escape","touch-pause"],
    "ToggleIdle": ["KeyI","touch-idle"],
    "QuickBattle":["KeyB","touch-quick"],
    "Settings":   ["KeyO","touch-settings"]
  }
}
```

## Lampiran E: Checklist Implementasi (Developer)

- [ ] `InputManager` terpusat, satu instance per game.
- [ ] Action map pakai `KeyboardEvent.code`, bukan `key`.
- [ ] Context layer sinkron dengan state machine (`setLayer` di tiap transisi).
- [ ] `preventDefault` untuk Space/Arrows/Backspace saat bukan input teks.
- [ ] Canvas fokus terjaga; refocus setelah interaksi shell PWA.
- [ ] Touch: `touch-action: none`, onscreen button min 44px.
- [ ] Rebinding UI + validasi konflik + reset default.
- [ ] Binding persist ke save data (`localStorage` key `rpg.input.bindings.v1`).
- [ ] Debounce repeat 180ms; Confirm debounce 120ms di Battle.
- [ ] Queue dibersihkan saat transisi scene / tab hidden (anti ghost input).
- [ ] Modal layer intercept; Auth layer skip shortcut.
- [ ] Aria-label + hint visual + high contrast + reduced motion.
- [ ] `update()` dipanggil tiap frame di game loop.

---

> **Akhir dokumen.** Dokumen ini adalah CDP untuk input mapping. Setiap perubahan binding default atau penambahan action wajib diperbarui di sini dan di `InputManager` secara bersamaan agar tidak divergen.
