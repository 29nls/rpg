# GAME LOOP SPECIFICATION — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Engine:** Phaser 3 + TypeScript + Vite
> **Target:** 60 FPS stabil (16.67 ms/frame)
> **Genre:** Fantasy Turn-Based Idle RPG, Party 5, Wave Combat
> **Combat Model:** Server-authoritative (client predict, server confirm)
> **Distribusi:** PWA (installable, offline-capable, tab dapat `hidden`)
> **Idle Real-Time Tick:** 5 detik (akumulasi ekonomi/idle di luar render loop)
> **Status Dokumen:** Canonical / Wajib Dipatuhi
> **Versi:** 1.0 — 2026-07-14

---

## Daftar Isi

1. [Tujuan & Prinsip](#1-tujuan--prinsip)
2. [Fixed Timestep Accumulator](#2-fixed-timestep-accumulator)
3. [Delta Clamping & Spiral-of-Death Prevention](#3-delta-clamping--spiral-of-death-prevention)
4. [Interpolation / Render Alpha](#4-interpolation--render-alpha)
5. [Integrasi Phaser 3](#5-integrasi-phaser-3)
6. [Budget Frame](#6-budget-frame)
7. [Idle Real-Time Tick (5 detik)](#7-idle-real-time-tick-5-detik)
8. [Pause / Resume](#8-pauserezume)
9. [Strategi Render](#9-strategi-render)
10. [Verifikasi & Testing](#10-verifikasi--testing)
11. [Hubungan Dokumen Lain](#11-hubungan-dokumen-lain)

- [Lampiran A — Kelas `GameLoop` Lengkap (TypeScript)](#lampiran-a--kelas-gameloop-lengkap-typescript)
- [Lampiran B — Glosarium](#lampiran-b--glosarium)
- [Lampiran C — Asumsi Eksplisit](#lampiran-c--asumsi-eksplisit)

---

## 1. Tujuan & Prinsip

### 1.1 Tujuan Dokumen

Dokumen ini adalah **spesifikasi kanonik tunggal** untuk logika *game loop* (siklus utama permainan) pada RPG Fantasy Turn-Based Idle berbasis web/PWA. Tujuannya adalah menjaga agar pembaruan data (simulasi/logika) dan penggambaran visual (render) **terpisah dengan tegas**, sehingga:

- Game berjalan **stabil di 60 FPS** di berbagai perangkat (low-end hingga high-end).
- Perilaku simulasi **deterministik**: hasil `update` sama untuk input yang sama, tidak bergantung pada refresh rate monitor (60 Hz, 120 Hz, 144 Hz) maupun pada naik-turunnya FPS sesaat.
- Saat tab di-hidden (PWA di background) atau device lambat, **tidak terjadi spiral-of-death** dan tidak ada akumulasi waktu liar yang merusak state.
- Komponen idle (ekonomi/akrual) berjalan **terlepas dari render loop**, sehingga tetap akurat meski halaman tidak digambar.

### 1.2 Prinsip Inti

| # | Prinsip | Penjelasan |
|---|---------|------------|
| P1 | **Pisahkan Update dari Render** | `update(dt)` memajukan simulasi; `render(alpha)` hanya menggambar. Kedua dipanggil dengan frekuensi/kontrak berbeda. |
| P2 | **Fixed Timestep untuk Simulasi** | Logika berjalan pada langkah waktu tetap (`STEP = 1/60 s`), bukan pada delta sesaat yang berfluktuasi. |
| P3 | **Determinisme** | `update` hanya bergantung pada `STEP` dan state sebelumnya, bukan pada `delta` mentah. Reproduksibel untuk replay/debug. |
| P4 | **Render Terinterpolasi** | Gambar posisi antara state-N dan state-(N-1) pakai `alpha` agar halus di layar. |
| P5 | **Clamp Delta** | `delta` dibatasi maksimal (mis. 250 ms) untuk cegah *spiral-of-death*. |
| P6 | **Budget Frame Ketat** | Total kerja per frame < 16.6 ms (p99). Sisa waktu diberikan ke browser/compositor. |
| P7 | **Idle Tick Terpisah** | Tick 5 dtk dijalankan oleh scheduler mandiri, bukan di dalam render loop. |
| P8 | **Respect Visibility** | Saat `hidden`, hentikan render, serahkan akrual ke server. Saat `visible`, reset accumulator. |
| P9 | **Zero-Alloc per Frame** | Hindari alokasi objek/array per frame untuk tekan tekanan GC. |
| P10 | **Server-Authoritative Combat** | Simulasi tempur lokal hanya prediksi; otoritas mutlak ada di server. |

### 1.3 Mengapa Pisahkan Update dan Render?

Monitor memiliki refresh rate beragam (60/120/144 Hz). Jika kita mengikat logika ke `requestAnimationFrame` (rAF) mentah, maka:

- Di 60 Hz, `delta ≈ 16.67 ms`.
- Di 144 Hz, `delta ≈ 6.94 ms`.
- Saat lag, `delta` bisa loncat ke 200–1000 ms.

Bila `update(delta)` dipanggil langsung dengan `delta` beragam, maka **fisika dan logika akan berbeda antar perangkat** (mis. peluru melesat lebih jauh di 144 Hz), dan saat lag akan terjadi *teleport* atau tabrakan terlewat. Memisahkan keduanya menyelesaikan ini:

- `update()` selalu dipanggil dengan `STEP` konstan → perilaku identik di semua perangkat.
- `render()` dipanggil sekali per rAF, menggambar state paling baru (terinterpolasi) → visual halus di refresh rate berapapun.

### 1.4 Kontrak Antarmuka (TypeScript)

```typescript
// Kontrak dasar yang dipatuhi seluruh subsistem.
export interface ISimulatable {
  /** Majukan simulasi satu langkah tetap (STEP detik). */
  fixedUpdate(step: number): void;
}

export interface IRenderable {
  /** Gambar state saat ini dengan faktor interpolasi alpha ∈ [0,1]. */
  render(alpha: number): void;
}

// Game loop memanggil keduanya secara terpisah.
//   while (acc >= STEP) { for (s of sims) s.fixedUpdate(STEP); acc -= STEP }
//   const alpha = acc / STEP
//   for (r of renders) r.render(alpha)
```

---

## 2. Fixed Timestep Accumulator

### 2.1 Konsep

Pola *fixed timestep accumulator* memisahkan waktu nyata (wall-clock `delta` dari rAF) dari waktu simulasi (langkah tetap `STEP`). Kita menampung `delta` ke dalam sebuah *accumulator*, lalu menjalankan `update(STEP)` sebanyak mungkin hingga accumulator kosong. Sisa accumulator dipakai sebagai `alpha` untuk interpolasi render.

```
Waktu nyata (rAF)          : |||||||||||||||||||||||||||  (delta fluktuatif)
Waktu simulasi (fixed)     : |--|--|--|--|--|--|--|--|      (STEP konstan 1/60)
Accumulator                : sisa waktu yang belum diproses
```

### 2.2 Parameter Kanonik

```typescript
// CDP: target 60 FPS -> STEP = 1/60 detik.
export const TARGET_FPS   = 60;
export const STEP         = 1 / TARGET_FPS;   // 0.0166667 s (16.667 ms)
export const STEP_MS      = 1000 / TARGET_FPS; // 16.6667 ms
export const MAX_DELTA    = 0.250;            // 250 ms -> lihat §3
```

### 2.3 Pseudocode Utama (Core Loop)

```pseudocode
// ============================================================
// FIXED TIMESTEP ACCUMULATOR — pseudocode kanonik
// ============================================================
VARIABLES:
  accumulator: Float  // detik, sisa waktu yang belum disimulasikan
  previousTime: Float // timestamp rAF sebelumnya (detik)

FUNCTION mainLoop(currentTime):
  // currentTime dalam detik (Phaser beri 'time' ms -> bagi 1000)
  rawDelta = currentTime - previousTime
  previousTime = currentTime

  // (1) Clamp delta agar aman -> lihat §3
  delta = MIN(rawDelta, MAX_DELTA)

  // (2) Tampung ke accumulator
  accumulator += delta

  // (3) Jalankan update tetap sebanyak mungkin
  WHILE accumulator >= STEP:
    FOR EACH system IN simulationSystems:   // ECS systems, combat predict, timers
      system.fixedUpdate(STEP)
    END FOR
    accumulator -= STEP

  // (4) Hitung alpha untuk interpolasi
  alpha = accumulator / STEP   // ∈ [0, 1)

  // (5) Render sekali, dengan alpha
  FOR EACH view IN renderables:
    view.render(alpha)

  // (6) Lanjut ke frame berikutnya (Phaser/ticker memanggil kembali)
  scheduleNextFrame(mainLoop)
```

### 2.4 Keunggulan vs Variable Timestep

| Aspek | Variable Timestep | Fixed Timestep + Accumulator |
|-------|-------------------|------------------------------|
| Determinisme | ❌ Tergantung FPS | ✅ Identik di semua perangkat |
| Stabilitas fisika | ❌ Bisa eksplosif saat `delta` besar | ✅ Langkah kecil konstan, stabil |
| Reproduksibilitas (replay/debug) | ❌ Sulit | ✅ Mudah (seed + STEP) |
| Replay / netcode | ❌ Sulit sinkron | ✅ Mudah (input + STEP) |
| Halusness visual | ✅ Langsung (tapi tick nyut) | ✅ Lewat interpolasi |
| Kompleksitas | Rendah | Sedang (perlu accumulator + alpha) |

**Kesimpulan:** Untuk RPG idle dengan combat server-authoritative dan kebutuhan determinisme (prediksi client harus cocok dengan konfirmasi server), **fixed timestep wajib**. Variable timestep hanya cocok untuk game tanpa fisika/simulasi kritis.

### 2.5 Catatan Determinisme & Floating Point

- Gunakan tipe `number` (float64) untuk `accumulator` dan `STEP`. Jangan pakai integer ms karena pembagian `1/60` tidak presisi dalam integer.
- Untuk sistem yang butuh determinisme kuat (combat RNG), gunakan **seedable PRNG** (mis. `mulberry32`), bukan `Math.random()`. Prediksi client menggunakan seed yang disepakati server.

```typescript
// PRNG deterministik (wajib untuk combat predict)
export function mulberry32(seed: number): () => number {
  let a = seed >>> 0;
  return function () {
    a |= 0; a = (a + 0x6D2B79F5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}
```

---

## 3. Delta Clamping & Spiral-of-Death Prevention

### 3.1 Apa itu Spiral-of-Death?

*Spiral-of-death* terjadi bila:

1. Frame tiba-tiba lambat (GC pause, tab baru visible, device throttle) → `delta` besar (mis. 500 ms).
2. Accumulator menampung 500 ms → loop menjalankan `update` 30 kali berurutan.
3. Ke-30 `update` itu sendiri butuh waktu lama → frame berikutnya makin lambat → `delta` makin besar.
4. Game "mati" karena terjebak menjalankan `update` tanpa sempat `render`.

### 3.2 Solusi: Clamp Delta

Batasi `delta` maksimal. Bila `rawDelta > MAX_DELTA`, kita anggap itu jeda (bukan 30 update yang harus dikejar). Sisa waktu **dibuang** (bukan ditampung). Ini menyelamatkan loop dari spiral.

```pseudocode
// ============================================================
// DELTA CLAMPING — cegah spiral-of-death
// ============================================================
CONSTANT MAX_DELTA = 0.250   // 250 ms

FUNCTION clampDelta(rawDelta):
  IF rawDelta > MAX_DELTA:
    // Jeda besar (lag / tab baru visible / GC). Abaikan kelebihannya.
    // Opsional: catat event "long frame" untuk telemetri.
    LOG_WARNING("delta clamped:", rawDelta, "->", MAX_DELTA)
    RETURN MAX_DELTA
  RETURN rawDelta
```

### 3.3 Catatan Penting: Mengapa Sisa Dibuang (bukan disimpan)

Menyimpan seluruh `rawDelta` ke accumulator justru **memicu** spiral. Dengan membuang kelebihannya, kita membatasi jumlah maksimum `update` per frame:

```
maxUpdatesPerFrame = ceil(MAX_DELTA / STEP)
                  = ceil(0.250 / 0.0166667)
                  = ceil(15) = 15 update per frame (paling banyak)
```

Batas 15 update/frame adalah pagar pengaman: meski lambat, kita tetap sempat `render` dan memberi napas ke browser.

### 3.4 Deteksi & Pemulihan Spiral

```pseudocode
VARIABLES:
  consecutiveClamps: Int = 0
  MAX_CONSECUTIVE_CLAMPS = 5

FUNCTION frame(currentTime):
  rawDelta = currentTime - previousTime
  previousTime = currentTime

  IF rawDelta > MAX_DELTA:
    consecutiveClamps += 1
    IF consecutiveClamps >= MAX_CONSECUTIVE_CLAMPS:
      // Sistem benar-benar tidak sanggup -> turunkan beban, bukan nambah update.
      onSustainedSlowdown()   // mis. kurangi jumlah partikel, nonaktifkan efek
    // Tetap clamp.
    delta = MAX_DELTA
  ELSE:
    consecutiveClamps = 0
    delta = rawDelta

  accumulator += delta
  // ... rest of loop (§2.3)
```

```typescript
// Saat slowdown berkepanjangan: turunkan kualitas, jaga FPS.
function onSustainedSlowdown(): void {
  qualityManager.dropTier();        // mis. 60->30 target, nonaktifkan glow
  particleSystem.setBudget(0.5);    // potong separuh partikel
  poolingManager.compactUnused();   // kembalikan objek idle ke pool
}
```

### 3.5 Tabel Konstanta Rekomendasi

| Konstanta | Nilai | Keterangan |
|-----------|-------|------------|
| `STEP` | `1/60` s | Langkah simulasi tetap (CDP). |
| `MAX_DELTA` | `0.250` s | Batas delta; cegah spiral. |
| `MAX_UPDATES_PER_FRAME` | `15` | `ceil(MAX_DELTA/STEP)`. |
| `MAX_CONSECUTIVE_CLAMPS` | `5` | Ambang turunkan kualitas. |
| `MIN_DELTA` | `0` | `delta` negatif (jam mundur) di-clamp ke 0. |

> **Asumsi eksplisit:** `MAX_DELTA = 250 ms` dipilih karena mencakup jeda tab-visible yang wajar (~100–200 ms) tanpa membiarkan lebih dari 15 update. Bila profil perangkat menunjukkan kebutuhan lain, nilai ini boleh disetel di `config/perf.ts` (lihat §11), bukan di-hardcode tersebar.

---

## 4. Interpolation / Render Alpha

### 4.1 Mengapa Interpolasi

Karena `update` berjalan pada langkah tetap (mungkin 0, 1, atau beberapa kali per frame rAF), posisi objek di simulasi **berloncatan** antar `STEP`. Bila kita menggambar posisi "mentah" terbaru, objek akan terlihat *nyut-nyut* (stutter) di layar, terutama bila FPS render ≠ kelipatan 60.

Interpolasi menggambar posisi di **antara** state-(N-1) dan state-N berdasarkan `alpha = accumulator / STEP`. Hasil: gerakan mulus di 60 FPS (dan 120/144 Hz).

### 4.2 Penyimpanan State Ganda

Setiap entitas yang bergerak menyimpan **dua snapshot posisi**: `previous` (state sebelum `update` terakhir) dan `current` (state sesudah). Saat `update` dijalankan, `previous = current` lalu `current` dihitung baru.

```typescript
export interface InterpolatedTransform {
  prevX: number; prevY: number;   // posisi sebelum update terakhir
  curX: number;  curY: number;    // posisi setelah update terakhir
  prevHp: number; curHp: number;  // (untuk bar HP halus, opsional)
}

export function render(ent: InterpolatedTransform, alpha: number): void {
  const x = ent.prevX + (ent.curX - ent.prevX) * alpha;
  const y = ent.prevY + (ent.curY - ent.prevY) * alpha;
  sprite.setPosition(x, y);
}
```

### 4.3 Pseudocode Update + Render Terinterpolasi

```pseudocode
// Saat fixedUpdate, geser snapshot:
FUNCTION fixedUpdateEntity(ent, step):
  ent.prevX = ent.curX
  ent.prevY = ent.curY
  // simulasi gerakan -> hitung curX, curY baru
  ent.curX = simulateX(ent, step)
  ent.curY = simulateY(ent, step)

// Saat render, interpolasi:
FUNCTION renderEntity(ent, alpha):
  x = LERP(ent.prevX, ent.curX, alpha)
  y = LERP(ent.prevY, ent.curY, alpha)
  ent.sprite.x = x
  ent.sprite.y = y
```

### 4.4 Render Extrapolation (Hati-hati)

*Extrapolation* (memperkirakan ke depan melebihi `cur`) kadang dipakai bila `alpha` mendekati 1 dan belum ada `update` baru. **Untuk RPG ini kita TIDAK pakai extrapolation** karena:
- Combat server-authoritative: prediksi salah akan terlihat aneh bila di-extrapolate.
- Idle RPG gerakannya lambat; interpolasi cukup.

Kita cukup menahan di `cur` bila `alpha` melebihi batas wajar (seharusnya tidak terjadi karena `alpha < 1` oleh konstruksi).

### 4.5 Alpha dan Resolusi Tinggi (120/144 Hz)

Di monitor 120 Hz, rAF dipanggil 2× lebih sering. `delta` lebih kecil, accumulator tumbuh lambat, sehingga beberapa frame `render(alpha)` dengan `alpha` berbeda tapi **tanpa** `update` di antaranya. Inilah mengapa interpolasi esensial: tanpa itu, di 120 Hz objek akan "beku" selama 2 frame lalu loncat. Dengan interpolasi, posisi digeser halus tiap frame.

```
120 Hz contoh (STEP=1/60):
  frame rAF #1: acc=8.3ms, alpha=0.50 -> render di tengah
  frame rAF #2: acc=16.6ms -> update! acc=0, alpha=0 -> render di cur
  frame rAF #3: acc=8.3ms, alpha=0.50 ...
```

### 4.6 Interpolasi untuk UI / Bar (HP, MP, EXP)

Bar status sebaiknya di-interpolasi juga agar tidak kelap-kelip saat angka berubah cepat (mis. damage beruntun). Gunakan pendekatan serupa: simpan `prevValue`/`curValue`, lerp di render. Khusus angka teks (integer), tampilkan `round(curValue)` (bukan interpolated) agar tidak menampilkan angka desimal; hanya *lebar bar* yang di-interpolasi.

---

## 5. Integrasi Phaser 3

### 5.1 Konteks: Phaser Sudah Membungkus rAF

Phaser 3 secara internal menjalankan loop berbasis `requestAnimationFrame` (melalui `Phaser.Game` + `TimeStep`). `Scene.update(time, delta)` dipanggil **sekali per frame rAF** oleh Phaser, dengan `delta` dalam **milidetik**.

Kita **tidak** membiarkan `Scene.update` menjalankan simulasi dengan `delta` mentah. Sebaliknya, kita gunakan `Scene.update` sebagai *driver* yang memanggil accumulator kita.

```
Phaser rAF  ->  Scene.update(time, delta)
                  |
                  v
              GameLoop.tick(time, delta)   // kita yang kelola accumulator
                  |
                  +---> while(acc>=STEP) fixedUpdate(STEP)
                  +---> render(acc/STEP)
```

### 5.2 Pola Integrasi (TypeScript)

```typescript
import Phaser from 'phaser';
import { GameLoop } from '../core/GameLoop';

export class BattleScene extends Phaser.Scene {
  private loop!: GameLoop;

  constructor() {
    super('BattleScene');
  }

  create(): void {
    this.loop = new GameLoop({
      step: 1 / 60,
      maxDelta: 0.25,
      systems: [
        this.combatPredictSystem,
        this.partyAISystem,
        this.idleWaveSpawner, // spawn wave, BUKAN idle ekonomi (lihat §7)
      ],
      renderables: [
        this.battleView,
        this.hudView,
      ],
    });

    // Pasang handler visibility (lihat §8).
    this.game.events.on(Phaser.Core.Events.BLUR, this.onBlur, this);
    this.game.events.on(Phaser.Core.Events.HIDDEN, this.onHidden, this);
    this.game.events.on(Phaser.Core.Events.FOCUS, this.onFocus, this);
    this.game.events.on(Phaser.Core.Events.VISIBLE, this.onVisible, this);
    document.addEventListener('visibilitychange', this.onVisibilityChange);
  }

  // Phaser memanggil ini SEKALI per frame. KITA JANGAN simulasikan di sini
  // dengan delta mentah — serahkan ke GameLoop.
  update(time: number, deltaMs: number): void {
    // deltaMs dari Phaser dalam ms -> detik.
    this.loop.tick(time / 1000, deltaMs / 1000);
  }

  // ... handler visibility di §8
}
```

### 5.3 Kapan Pakai Phaser Tween vs Manual Update

| Situasi | Gunakan | Alasan |
|---------|---------|--------|
| Animasi UI dekoratif (panel slide, fade) | **Phaser Tween** | Deklaratif, diurus Phaser, tak perlu masuk fixedUpdate. |
| Gerakan entitas tempur yang ikut simulasi | **Manual fixedUpdate + interpolasi** | Harus deterministik & synced ke server. |
| Damage number pop, floating text | **Phaser Tween** (atau partikel) | Visual murni, tak affect simulasi. |
| HP bar, MP bar (terikat state) | **Manual render(alpha)** | Perlu nge-LERP ke state. |
| Camera shake / flash | **Phaser Tween / Camera effect** | Visual murni. |
| Projectile / skill telegraph | **Manual fixedUpdate** | Bagian prediksi combat. |

**Prinsip:** Apa pun yang memengaruhi *simulasi* atau harus *deterministik* → manual fixedUpdate. Apa pun yang murni *kosmetik* → Phaser Tween (di luar loop deterministik kita, tapi tetap di rAF Phaser).

> ⚠️ **Jebakan:** Jangan letakkan logika gameplay di dalam callback Tween `onComplete` yang menentukan hasil combat. Tween berjalan pada delta Phaser yang tak deterministik. Hasil combat harus dari `fixedUpdate` + konfirmasi server.

### 5.4 requestAnimationFrame & Phaser

Phaser menangani rAF sendiri. Kita **tidak** membuat `requestAnimationFrame` kita sendiri (hindari dua loop berlomba). Cukup override `Scene.update`. Bila butuh kontrol sangat rendah (mis. mengganti FPS target), pakai `this.game.loop` (instance `Phaser.Core.TimeStep`):

```typescript
// Mengatur target FPS di level Phaser (jarang perlu; default 60).
this.game.loop.target = 60;
// forceSetTimeOut bila rAF tak tersedia (fallback PWA lama)
this.game.loop.forceSetTimeOut = false;
```

### 5.5 Pause Internal Phaser vs Pause Game Loop

- `this.scene.pause()` → menghentikan `update` scene tapi Phaser tetap rAF (scene lain jalan). Bukan yang kita pakai untuk idle handoff.
- Kita pause **GameLoop** kita sendiri (flag `running=false`) saat tab hidden, dan biarkan Phaser rAF berjalan tapi `tick()` di-no-op. Atau lebih baik: hentikan benar-benar lewat visibility (§8).

---

## 6. Budget Frame

### 6.1 Target Waktu

Satu frame di 60 FPS = **16.67 ms**. Kita menetapkan budget agar p99 (persentil ke-99, frame lambat sekalipun) tetap < 16.6 ms.

| Tahap | Budget (ms) | Keterangan |
|-------|------------|------------|
| Input sampling | 0.3 | Baca input/event queue. |
| `fixedUpdate` (N×STEP) | 6.0 | Simulasi; N bisa 0–15 (§3). |
| ECS systems (combat predict, AI, timers) | 3.0 | Di dalam fixedUpdate. |
| Interpolation + `render(alpha)` | 4.0 | Gambar + LERP. |
| Phaser internal (batcher, WebGL) | 2.0 | Di luar kendali langsung, diawasi. |
| **Sisa untuk browser/compositor/GC** | **≥ 1.3** | Napas agar tak dropout. |
| **TOTAL (target p99)** | **< 16.6** | — |

### 6.2 Breakdown Lebih Detail (Party 5, Wave Combat)

```text
FRAME (16.67 ms)
├─ [0.3] Input/Camera input poll
├─ [6.0] fixedUpdate block (jalankan sebanyak N kali)
│   ├─ CombatPredictSystem .... 1.5 ms
│   ├─ PartyAISystem (5 hero) . 1.0 ms
│   ├─ EnemyAISystem (wave) ... 1.0 ms
│   ├─ IdleWaveSpawner ........ 0.2 ms
│   ├─ TimerSystem (buffs/dots) 0.8 ms
│   ├─ ProjectileSystem ....... 1.0 ms
│   └─ NetworkConfirmApply .... 0.5 ms  (terapkan konfirmasi server)
├─ [3.0] ECS housekeeping (component sync ke sprite)
├─ [4.0] render(alpha)
│   ├─ Interpolation LERP ...... 0.5 ms
│   ├─ Sprite batch draw ...... 2.5 ms (atlas, lihat §9)
│   └─ HUD/Bar redraw ......... 1.0 ms
└─ [2.0] Phaser/WebGL driver
```

### 6.3 Teknik Menjaga Budget

1. **Profil berkala** dengan `performance.now()` di sekitar tiap blok (lihat §10).
2. **Culling**: jangan update/render sprite di luar viewport (§9).
3. **Pooling**: hindari `new`/`destroy` per frame (§9).
4. **Sistem kondisional**: AI musuh yang mati/sleep dilewati (early-out).
5. **ECS archetype iteration**: iterasi komponen contiguous, hindari pencarian `getComponent` berulang.
6. **Throttle mahal**: pathfinding (mis. A*) hanya setiap N step, bukan tiap step.
7. **Quality tiers**: bila p99 > 16.6 ms berturut-turut, turunkan tier (§3.4).

### 6.4 Fast Path vs Slow Path

```typescript
// Bila accumulator kecil (frame cepat), lewati blok update sama sekali.
tick(time: number, delta: number): void {
  this.accumulator += Math.min(delta, this.maxDelta);
  if (this.accumulator >= this.step) {
    // SLOW PATH: ada step yang harus dijalankan.
    do {
      this.fixedUpdate(this.step);
      this.accumulator -= this.step;
    } while (this.accumulator >= this.step);
  }
  // FAST PATH: render selalu (walau tanpa update -> alpha saja yang geser).
  this.render(this.accumulator / this.step);
}
```

> Di 120 Hz, ~setengah frame adalah FAST PATH (render tanpa update) — murah, aman.

### 6.5 GC Pressure Budget

Target: **GC pause < 1 ms/frame rata-rata**, dan tidak ada *major GC* saat combat sengit. Cara:
- `arena`/`pool` untuk event combat (hindari `new CombatEvent` tiap hit).
- Pre-alokasi array velocity/position seukuran party+waves maksimal.
- Jangan buat closure di dalam `render`/`update` (bocor ke heap).
- Gunakan `Float32Array` untuk data posisi massal (ribuan partikel).

---

## 7. Idle Real-Time Tick (5 detik)

### 7.1 Masalah

Game ini **idle RPG**: saat player tak aktif, ekonomi (gold, resource, exp offline) terus bertambah setiap **5 detik**. Pertanyaan kritis: jalankan tick 5 dtk di dalam game loop (fixedUpdate) atau pakai scheduler terpisah?

### 7.2 Rekomendasi: Scheduler Terpisah (di luar render loop)

**Jalankan idle tick di luar render loop.** Alasannya:

1. **Tab hidden → render loop berhenti** (§8). Bila idle tick ada di dalam fixedUpdate, maka saat tab hidden, akrual berhenti — padahal player idle *harus* terus dapat reward.
2. Idle tick **tidak perlu 60 Hz**; 5 dtk cukup. Menyelipkannya ke fixedUpdate membuang-buang slot akumulator.
3. Saat tab hidden, **server yang menghitung akrual** (offline accrual) — client hanya menyinkronkan saat kembali (§8).

### 7.3 Desain Dwi-Mode (Online vs Offline Accrual)

```text
┌─────────────────────┐        ┌──────────────────────────┐
│  TAB VISIBLE        │        │  TAB HIDDEN               │
│  (game loop jalan)  │        │  (render loop berhenti)   │
│                     │        │                           │
│ Idle tick lokal:    │        │ Client TIDAK hitung.      │
│ - setInterval 5s    │        │ Server hitung accrual.    │
│ - akumulasi gold/   │        │ Saat visible kembali:     │
│   exp/resource      │        │ - fetch server snapshot   │
│ - render indikator  │        │ - apply offline gains     │
│   (+X gold/5s)      │        │ - resume idle tick lokal  │
└─────────────────────┘        └──────────────────────────┘
```

### 7.4 Implementasi Scheduler Mandiri

```typescript
// IdleTickScheduler: BERDIRI SENDIRI, tidak di fixedUpdate.
export class IdleTickScheduler {
  private timer: number | null = null;
  private readonly intervalMs = 5000; // CDP: 5 detik
  private lastServerSync = 0;

  start(): void {
    if (this.timer !== null) return;
    // setInterval tetap jalan meski rAF berhenti (tab visible, game ringan).
    this.timer = window.setInterval(() => this.tick(), this.intervalMs);
  }

  stop(): void {
    if (this.timer !== null) {
      window.clearInterval(this.timer);
      this.timer = null;
    }
  }

  private tick(): void {
    if (document.visibilityState !== 'visible') {
      // Keamanan ganda: jangan akrual lokal saat hidden.
      return;
    }
    // Akrual lokal (hati-hati: ini prediksi; server adalah otoritas).
    economy.accrue(/* goldPerSec*5, expPerSec*5, ... */);
    hud.refreshIdleIndicator();
  }
}
```

### 7.5 Offline Accrual (Tab Hidden / App Closed)

Saat tab hidden, `setInterval` mungkin di-throttle browser ke >1 dtk (bahkan dijeda di beberapa browser untuk background tab). Maka kita **serahkan ke server**:

```typescript
// Saat pindah ke hidden:
onEnterHidden(): void {
  idleTickScheduler.stop();          // client berhenti akrual
  gameLoop.pause();                  // render loop berhenti
  // Server mulai menghitung accrual berdasarkan lastSeen timestamp.
  network.send({ type: 'IDLE_HANDOFF', at: Date.now() });
}

// Saat kembali visible:
onEnterVisible(): void {
  network.send({ type: 'IDLE_RESUME', at: Date.now() });
  const snapshot = await network.requestIdleSnapshot(); // gold/exp accrued
  economy.applyServerSnapshot(snapshot);                // terapkan gain offline
  gameLoop.resume();                 // reset accumulator (lihat §8)
  idleTickScheduler.start();         // lanjut akrual lokal
}
```

```pseudocode
// Server-side accrual (logika server, disarikan):
ON IDLE_HANDOFF(playerId, t0):
  store lastSeen[playerId] = t0

ON IDLE_RESUME(playerId, t1):
  elapsed = t1 - lastSeen[playerId]          // detik di background
  cycles = FLOOR(elapsed / 5)                // tick 5 dtk
  goldGain = cycles * goldPerTick
  expGain  = cycles * expPerTick
  RETURN IdleSnapshot(goldGain, expGain, cycles)
```

### 7.6 Catatan Kapasisme (Capping)

Biar player tak "farming" dengan cara menutup tab berulang kali tanpa batas, server menerapkan **cap** (mis. maksimal 8 jam akrual / maksimal N cycle). Ini di luar game loop tapi relevan sebagai asumsi:

```typescript
const MAX_OFFLINE_SECONDS = 8 * 3600; // cap 8 jam
const goldPerTick = computeGoldPerTick(player);
```

### 7.7 Hubungan Tick Idle dengan FixedUpdate

Idle tick **bukan** event di dalam `fixedUpdate`. Namun, *indikator visual* akrual (angka gold naik, partikel koin) di-render lewat `render(alpha)` seperti biasa. Jadi: perhitungan ekonomi = scheduler 5 dtk; penggambaran indikator = render loop. Pemisahan bersih.

---

## 8. Pause / Resume

### 8.1 Skenario Visibility

PWA dapat berjalan di background (tab lain, app ditutup ke recent, layar mati). Browser memanggil `visibilitychange` dan Phaser memancarkan event `HIDDEN`/`VISIBLE` + `BLUR`/`FOCUS`.

Kita definisikan dua state loop:

- **RUNNING**: `tick()` dijalankan tiap rAF.
- **PAUSED**: `tick()` di-no-op; render dihentikan; idle diserahkan ke server.

### 8.2 Handler visibilitychange

```typescript
export class GameLoop {
  private running = false;
  private accumulator = 0;
  private previousTime = 0;

  bindVisibility(): void {
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') this.pause();
      else this.resume();
    });
    // Phaser-level (jaga-jaga):
    this.game.events.on(Phaser.Core.Events.HIDDEN, () => this.pause());
    this.game.events.on(Phaser.Core.Events.VISIBLE, () => this.resume());
  }

  pause(): void {
    if (!this.running) return;
    this.running = false;
    idleTickScheduler.stop();        // (§7) client berhenti akrual
    network.send({ type: 'IDLE_HANDOFF', at: Date.now() });
    // Simpan state penting ke save (lihat §11) bila perlu.
    saveManager.flushIfDirty();
  }

  resume(): void {
    if (this.running) return;
    // SINKRONKAN idle gain dari server dulu.
    network.requestIdleSnapshot().then(s => economy.applyServerSnapshot(s));
    // RESET accumulator & previousTime agar tak meledak.
    this.accumulator = 0;
    this.previousTime = performance.now() / 1000; // mulai dari sekarang
    idleTickScheduler.start();
    this.running = true;
  }
}
```

### 8.3 Mengapa Reset Accumulator saat Resume (Penting!)

Bila kita **tidak** reset accumulator, maka `previousTime` masih menunjuk waktu sebelum tab hidden (mis. 30 detik lalu). Saat resume, `rawDelta = 30 s` → meski di-clamp ke 250 ms, itu tetap memicu 15 update sekaligus (bukan 1). Lebih parah: bila kita lupa clamp, 30 s → 1800 update → spiral-of-death instan.

**Aturan wajib:** Saat resume, set `previousTime = sekarang` DAN `accumulator = 0`. Maka frame pertama hanya menjalankan update secukupnya (kemungkinan 0–1), dan `alpha` mulai dari 0.

```pseudocode
FUNCTION resume():
  running = TRUE
  accumulator = 0                 // <-- wajib
  previousTime = NOW()            // <-- wajib, jangan pakai waktu lama
  idleTickScheduler.start()
  // (idle gain dari server sudah/sedang disinkronkan)
```

### 8.4 State Machine Pause/Resume

Loop terhubung ke State Machine global (lihat §11). Transisi:

```
                 visibilitychange(hidden)
   RUNNING ───────────────────────────────► PAUSED
      ▲                                      │
      │ visibilitychange(visible)            │ IDLE_HANDOFF ke server
      │ + reset accumulator                  ▼
   RUNNING ◄───────────────────────────── SERVER_ACCRUAL
```

State machine juga menangani pause eksplisit player (menu, inventory) — beda trigger tapi mekanisme loop sama: `pause()`/`resume()`.

### 8.5 Render Saat Paused

Saat PAUSED, kita **tidak** memanggil `render()` (hemat baterai, PWA ramah mobile). Layar membeku di frame terakhir. Bila perlu menampilkan overlay "Paused", itu diurus DOM/CSS di luar canvas, bukan di render loop.

```typescript
update(time: number, deltaMs: number): void {
  if (!this.loop.running) return; // no-op saat paused; canvas membeku
  this.loop.tick(time / 1000, deltaMs / 1000);
}
```

### 8.6 Throttled Background (Bila Browser Tetap Rendor)

Beberapa browser (terutama mobile) tetap memanggil rAF di background tapi dengan `delta` sangat besar (1 dtk+). Karena kita clamp (§3) dan reset saat visible (§8.3), ini aman. Namun untuk hemat baterai, **preferensi:** hentikan render saat hidden (lewat flag `running`), jangan andalkan throttle browser.

---

## 9. Strategi Render

### 9.1 Tujuan

Render harus murah dan stabil. Tiga pilar: **batching** (sedikit draw call), **culling** (gambar cuma yang kelihatan), **pooling** (tanpa alokasi per frame).

### 9.2 Batching Draw Calls

WebGL mahal saat banyak draw call kecil. Phaser mengelompokkan sprite yang pakai **texture atlas** sama ke dalam satu batch. Karena itu:

- **Gunakan texture atlas / spritesheet** untuk semua sprite tempur (hero, musuh, efek). Satu atlas = satu (atau sedikit) draw call.
- Jangan muat tiap frame sebagai texture terpisah.
- Urutkan render agar sprite ber-atlas sama berdekatan (Phaser melakukannya otomatis per-texture, tapi hindari ganti-ganti atlas tiap sprite).

```text
BURUK:  [heroTex][enemyTex][heroTex][fxTex][enemyTex] -> 5 draw call
BAIK:   [heroTex][heroTex] [enemyTex][enemyTex] [fxTex] -> 3 draw call (terbatas atlas)
TERBAIK: semua di 1 atlas -> 1 draw call
```

### 9.3 Texture Atlas & Spritesheet (Ref Asset Loading)

Semua aset karakter/effect di-build menjadi atlas saat *asset loading* (lihat §11). Di runtime, sprite mereferensi frame dalam atlas, bukan texture terpisah.

```typescript
// Load atlas sekali di BootScene (preload).
this.load.atlas('battle', 'assets/battle.png', 'assets/battle.json');

// Buat sprite dari frame atlas (murah, tak bikin texture baru).
const hero = this.add.sprite(x, y, 'battle', 'hero_knight_idle_01');
```

### 9.4 Culling Offscreen

Jangan update posisi atau gambar sprite di luar viewport kamera. Phaser punya culling otomatis untuk `TileSprite`/large groups, tapi untuk group custom kita lakukan manual:

```typescript
function isOnScreen(s: Phaser.GameObjects.Sprite, cam: Phaser.Cameras.Scene2D.Camera): boolean {
  return s.x >= cam.worldView.x - MARGIN &&
         s.x <= cam.worldView.right + MARGIN &&
         s.y >= cam.worldView.y - MARGIN &&
         s.y <= cam.worldView.bottom + MARGIN;
}

// Di render: skip culled.
for (const ent of entities) {
  if (!isOnScreen(ent.sprite, cam)) { ent.sprite.setVisible(false); continue; }
  ent.sprite.setVisible(true);
  renderEntity(ent, alpha);
}
```

### 9.5 Object Pooling

`new Sprite()` / `destroy()` per frame = alokasi + GC pressure (§6.5). Gunakan pool: ambil dari pool saat butuh, kembalikan saat tak dipakai.

```typescript
export class SpritePool {
  private free: Phaser.GameObjects.Sprite[] = [];
  constructor(private scene: Phaser.Scene, private texture: string) {}

  acquire(x: number, y: number): Phaser.GameObjects.Sprite {
    const s = this.free.pop() ?? this.scene.add.sprite(0, 0, this.texture);
    s.setActive(true).setVisible(true).setPosition(x, y);
    return s;
  }

  release(s: Phaser.GameObjects.Sprite): void {
    s.setActive(false).setVisible(false);
    this.free.push(s); // jangan destroy -> hindari GC
  }
}

// Dipakai untuk: damage numbers, projectile, particle burst, enemy spawn.
```

### 9.6 Hindari Alokasi per-Frame (GC Pressure)

```text
JANGAN (tiap frame):
  const v = new Vector2();           // alokasi heap
  entities.filter(e => e.alive)      // array baru tiap frame
  systems.map(s => s.update())       // closure + array baru

LAKUKAN:
  reuse vector scratch (this._tmp)
  for-loop dengan index (early-out bila !alive)
  panggil method langsung tanpa .map
```

```typescript
// Contoh: pakai scratch vector, bukan new.
private _tmp = new Phaser.Math.Vector2();
function moveEntity(ent: Entity, step: number): void {
  this._tmp.set(ent.vx * step, ent.vy * step); // reuse, no alloc
  ent.curX += this._tmp.x;
  ent.curY += this._tmp.y;
}
```

### 9.7 Depth & Z-Order

Atur `depth` sekali saat spawn (bukan tiap frame). Urutan: background → ground → entities → projectiles → fx → HUD. Gunakan `setDepth` konstan per layer; jangan hitung ulang tiap frame.

### 9.8 Particle & Effects

- Gunakan `Phaser.GameObjects.Particles.ParticleEmitter` dengan kuota tetap (pool internal Phaser).
- Batasi jumlah partikel via `quantity`/`frequency` sesuai quality tier (§3.4).
- Efek mahal (bloom, glow) hanya di high tier; low tier pakai sprite pre-baked.

### 9.9 Render Order dalam `render(alpha)`

```pseudocode
FUNCTION render(alpha):
  // 1. Update transform terinterpolasi (§4)
  FOR EACH ent IN visibleEntities:
    ent.sprite.x = LERP(ent.prevX, ent.curX, alpha)
    ent.sprite.y = LERP(ent.prevY, ent.curY, alpha)
  // 2. Update bar HP/MP (LERP width, round teks)
  FOR EACH bar IN statusBars:
    bar.setDisplayWidth(LERP(bar.prevW, bar.curW, alpha))
  // 3. Biarkan Phaser batch & gambar (otomatis di akhir update)
  //    KITA TIDAK panggil renderer manual; Phaser render queue-nya.
```

> Catatan: Di Phaser, `render` sebenarnya terjadi **setelah** `Scene.update` kembali (Phaser menggambar display list). Jadi di `update`/`render(alpha)` kita hanya **memperbarui posisi sprite**, Phaser yang menggambar. Ini konsisten: kita atur transform terinterpolasi, Phaser batch.

---

## 10. Verifikasi & Testing

### 10.1 Metrik Utama

| Metrik | Target | Alat |
|--------|--------|------|
| FPS rata-rata | 60 ± 1 | stats.js / custom |
| Frame time p99 | < 16.6 ms | Performance API |
| GC major pause | 0 saat combat | Chrome DevTools Memory |
| Delta clamp events | < 1% frame | Logger internal |
| Idle accuracy | ±1 tick (5 dtk) | Unit test scheduler |
| Akrual offline | == server (cap-respecting) | Integrasi test |

### 10.2 Mengukur FPS (stats.js)

```typescript
import Stats from 'stats.js';

const stats = new Stats();
stats.showPanel(0); // 0: fps, 1: ms, 2: mb
document.body.appendChild(stats.dom);

// Di awal update:
update(time: number, deltaMs: number): void {
  stats.begin();
  this.loop.tick(time / 1000, deltaMs / 1000);
  stats.end();
}
```

### 10.3 Mengukur Budget via Performance API

```typescript
const marks: Record<string, number> = {};

function profileFrame(): void {
  performance.mark('frame-start');
  // ... dalam tick:
  performance.mark('after-update');
  performance.mark('after-render');
  performance.measure('update-block', 'frame-start', 'after-update');
  performance.measure('render-block', 'after-update', 'after-render');
}

// Baca hasil:
const m = performance.getEntriesByName('update-block')[0];
if (m.duration > 6.0) console.warn('update block over budget', m.duration);
```

### 10.4 Test Case Kritis

#### TC-1: Stres Wave Banyak Sprite
- **Skenario:** Spawn 200+ musuh + 5 hero + 500 partikel damage.
- **Verifikasi:** p99 frame < 16.6 ms; FPS ≥ 55 di desktop, ≥ 30 di low-end.
- **Cek:** draw call ≤ 5 (atlas), pooling tak bocor (`free.length` stabil).

#### TC-2: Tab Hide/Show
- **Skenario:** Main → hidden 10 dtk → visible.
- **Verifikasi:**
  - Saat hidden: `loop.running == false`, tak ada `render`, `setInterval` idle berhenti/di-throttle.
  - Accumulator **reset** (bukan meledak); frame pertama pasca-visible = 0–1 update.
  - Akrual offline dari server == `floor(10/5) * perTick` (cap-respecting).
- **Cek:** tak ada spiral-of-death; gold bertambah tepat.

#### TC-3: Low-End Device (Throttle CPU 4× di DevTools)
- **Skenario:** CPU throttled, banyak GC.
- **Verifikasi:** delta clamp aktif; quality tier turun (§3.4); game tetap *playable* (FPS ≥ 20) tanpa crash.
- **Cek:** `consecutiveClamps` memicu `onSustainedSlowdown`.

#### TC-4: Determinisme Combat Predict
- **Skenario:** Jalankan prediksi client 2× dengan seed sama + input sama.
- **Verifikasi:** State akhir identik (bit-exact) → cocok dengan konfirmasi server.
- **Cek:** Tak pakai `Math.random()` di sistem combat.

#### TC-5: Idle Tick Akurasi
- **Skenario:** Biarkan jalan 60 dtk (12 tick) di tab visible.
- **Verifikasi:** gold naik persis `12 * goldPerTick`, tak lebih/kurang.
- **Cek:** `setInterval` jitter ±100 ms tak mengakumulasi drift (pakai timestamp absolut, bukan counter).

### 10.5 Otomatisasi (Contoh Test)

```typescript
// playwright / vitest contoh (disarikan)
test('accumulator reset on resume prevents spiral', () => {
  const loop = new GameLoop({ step: 1/60, maxDelta: 0.25 });
  loop.tick(0, 0.016);
  loop.pause();
  loop.resume();                       // harus reset accumulator & previousTime
  const updates = loop.tick(30 /*detik kemudian*/, 0.016); // rawDelta besar
  expect(loop.accumulator).toBeLessThan(loop.step);        // tak meledak
  expect(updates).toBeLessThanOrEqual(15);                 // clamp aktif
});
```

### 10.6 Telemetri Produksi (PWA)

Kirim metrik anonim ke server (sampled):
- median FPS, p99 frame ms, jumlah clamp/frame, device tier.
- Membantu setel `MAX_DELTA`/`quality tier` per segmen perangkat nyata.

---

## 11. Hubungan Dokumen Lain

Dokumen ini adalah satu dari satu set spesifikasi. Hubungannya:

| Dokumen | Hubungan dengan Game Loop |
|---------|---------------------------|
| `STATE_MACHINE.md` | Mendefinisikan state global (RUNNING/PAUSED/SERVER_ACCRUAL/MENU). Game Loop memanggil `pause()`/`resume()` sesuai transisi state (§8.4). |
| `ASSET_LOADING.md` | Mengatur pembuatan **texture atlas** & **spritesheet** yang dipakai strategi batching (§9.2–9.3) serta inisialisasi **object pool** (§9.5). |
| `SAVE_DATA.md` | Saat `pause()` (§8.2) kita `saveManager.flushIfDirty()`; akrual offline disinkronkan lewat snapshot (§7.5). Format save menyimpan `lastSeen` untuk server accrual. |
| `COMBAT_NETCODE.md` | Mendefinisikan **client predict / server confirm**. `fixedUpdate` menjalankan prediksi; `NetworkConfirmApply` (§6.2) menerapkan konfirmasi. Determinisme PRNG (§2.5) wajib di sini. |
| `ECS_ARCHITECTURE.md` | Mendefinisikan *systems* yang kita panggil di `fixedUpdate` (CombatPredict, PartyAI, EnemyAI, Timer, Projectile). Budget ECS (§6.3) merujuk ke arsitektur ini. |
| `PERF_CONFIG.md` (`config/perf.ts`) | Tempat konstanta `STEP`, `MAX_DELTA`, quality tiers, budget ms didefinisikan (bukan hardcode tersebar). Game Loop membacanya. |
| `PWA_MANIFEST.md` | Mendefinisikan service worker & offline. Relevan dengan offline accrual (§7.5) dan behavior saat app di background. |

### 11.1 Alumsi Lintas-Dokumen

- `COMBAT_NETCODE.md` menjamin server adalah otoritas; prediksi client boleh salah dan akan dikoreksi (reconcile). Game Loop tetap menjalankan prediksi tiap `fixedUpdate` tanpa menunggu server (antisipasi respons cepat).
- `SAVE_DATA.md` menjamin `lastSeen` tersimpan saat handoff, sehingga server bisa menghitung akrual saat app tertutup total (bukan cuma tab hidden).

---

## Lampiran A — Kelas `GameLoop` Lengkap (TypeScript)

```typescript
// core/GameLoop.ts
// Implementasi kanonik fixed-timestep accumulator untuk RPG Idle.
// CDP: STEP=1/60, MAX_DELTA=0.25, target 60 FPS, PWA visibility-aware.

export interface ISimSystem {
  fixedUpdate(step: number): void;
}

export interface IRenderView {
  render(alpha: number): void;
}

export interface GameLoopConfig {
  step?: number;          // default 1/60
  maxDelta?: number;      // default 0.25
  systems: ISimSystem[];
  renderables: IRenderView[];
  onSlowdown?: () => void;
}

export class GameLoop {
  readonly step: number;
  readonly maxDelta: number;
  private readonly systems: ISimSystem[];
  private readonly renderables: IRenderView[];
  private readonly onSlowdown?: () => void;

  private accumulator = 0;
  private previousTime = 0;
  private running = false;
  private consecutiveClamps = 0;
  private readonly maxClampsBeforeSlowdown = 5;

  constructor(cfg: GameLoopConfig) {
    this.step = cfg.step ?? 1 / 60;
    this.maxDelta = cfg.maxDelta ?? 0.25;
    this.systems = cfg.systems;
    this.renderables = cfg.renderables;
    this.onSlowdown = cfg.onSlowdown;
  }

  get isRunning(): boolean { return this.running; }

  /** Dipanggil sekali per frame dari Phaser Scene.update(time, deltaMs). */
  tick(currentTimeSec: number, deltaSec: number): void {
    if (!this.running) return; // no-op saat paused

    let delta = deltaSec;
    if (delta < 0) delta = 0; // jam mundur / anomali

    if (delta > this.maxDelta) {
      this.consecutiveClamps++;
      if (this.consecutiveClamps >= this.maxClampsBeforeSlowdown) {
        this.onSlowdown?.();
      }
      delta = this.maxDelta;
    } else {
      this.consecutiveClamps = 0;
    }

    this.accumulator += delta;

    // Jalankan update tetap sebanyak mungkin (dibatasi clamp).
    let steps = 0;
    while (this.accumulator >= this.step) {
      for (let i = 0; i < this.systems.length; i++) {
        this.systems[i].fixedUpdate(this.step);
      }
      this.accumulator -= this.step;
      if (++steps > 30) { this.accumulator = 0; break; } // pagar ekstra
    }

    const alpha = this.accumulator / this.step;
    for (let i = 0; i < this.renderables.length; i++) {
      this.renderables[i].render(alpha);
    }
  }

  pause(): void {
    this.running = false;
  }

  resume(): void {
    if (this.running) return;
    this.accumulator = 0;                 // WAJIB: cegah meledak
    this.previousTime = performance.now() / 1000; // WAJIB: mulai bersih
    this.consecutiveClamps = 0;
    this.running = true;
  }

  start(): void {
    this.previousTime = performance.now() / 1000;
    this.accumulator = 0;
    this.running = true;
  }

  /** Digunakan test: simulasikan tick dengan delta eksplisit. */
  tickManual(deltaSec: number): number {
    const before = this.accumulator;
    this.tick(this.previousTime + deltaSec, deltaSec);
    return this.accumulator - (before - this.step * Math.floor((before + deltaSec) / this.step));
  }
}
```

```typescript
// core/IdleTickScheduler.ts (ringkasan, lihat §7)
export class IdleTickScheduler {
  private timer: number | null = null;
  constructor(
    private readonly intervalMs = 5000,
    private readonly onTick: () => void,
  ) {}

  start(): void {
    if (this.timer !== null) return;
    this.timer = window.setInterval(() => {
      if (document.visibilityState === 'visible') this.onTick();
    }, this.intervalMs);
  }
  stop(): void {
    if (this.timer !== null) { window.clearInterval(this.timer); this.timer = null; }
  }
}
```

```typescript
// scenes/BattleScene.ts (ringkasan integrasi, lihat §5)
export class BattleScene extends Phaser.Scene {
  private loop!: GameLoop;
  private idle!: IdleTickScheduler;

  create(): void {
    this.loop = new GameLoop({
      systems: [combatPredict, partyAI, enemyAI, waveSpawner, timers, projectiles, netConfirm],
      renderables: [battleView, hudView],
      onSlowdown: () => qualityManager.dropTier(),
    });
    this.idle = new IdleTickScheduler(5000, () => economy.accrue());
    this.loop.start();
    this.idle.start();

    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'hidden') {
        this.loop.pause(); this.idle.stop();
        network.send({ type: 'IDLE_HANDOFF', at: Date.now() });
      } else {
        network.requestIdleSnapshot().then(s => economy.applyServerSnapshot(s));
        this.loop.resume(); this.idle.start(); // resume reset accumulator
      }
    });
  }

  update(time: number, deltaMs: number): void {
    // Phaser panggil sekali/frame. Serahkan ke loop kita.
    this.loop.tick(time / 1000, deltaMs / 1000);
  }
}
```

---

## Lampiran B — Glosarium

| Istilah | Arti |
|---------|------|
| **Game Loop** | Siklus update+render yang berjalan tiap frame. |
| **Fixed Timestep** | Langkah simulasi waktu tetap (1/60 s). |
| **Accumulator** | Penampung delta waktu yang belum disimulasikan. |
| **Spiral-of-Death** | Loop terjebak menjalankan update tanpa sempat render. |
| **Render Alpha** | Faktor interpolasi ∈ [0,1] untuk gambar halus. |
| **Interpolasi** | Menggambar di antara dua state simulasi. |
| **Clamp** | Membatasi nilai maksimal/minimal. |
| **rAF** | `requestAnimationFrame`, callback per frame browser. |
| **ECS** | Entity-Component-System, arsitektur simulasi. |
| **PWA** | Progressive Web App (installable, offline). |
| **Offline Accrual** | Akrual idle dihitung server saat client tak aktif. |
| **Client Predict** | Client simulasi duluan sebelum konfirmasi server. |
| **Server Confirm** | Server mengirim state otoritatif; client reconcile. |
| **Object Pool** | Kumpulan objek siap-pakai, hindari alokasi/GC. |
| **Draw Call** | Satu instruksi gambar ke GPU; sedikit = cepat. |
| **Atlas** | Satu texture berisi banyak frame (batch efisien). |
| **Culling** | Tak menggambar objek di luar layar. |
| **GC** | Garbage Collector; pembersih memori tak terpakai. |
| **p99** | Persentil ke-99; frame lambat sekalipun. |
| **Visibility** | Status tab `visible`/`hidden` via `visibilitychange`. |

---

## Lampiran C — Asumsi Eksplisit

Berikut asumsi yang diambil agar spesifikasi lengkap & tidak ambigu:

1. **Refresh rate target 60 Hz**; monitor 120/144 Hz didukung lewat interpolasi (§4.5) tanpa ubah `STEP`.
2. **`MAX_DELTA = 250 ms`** (§3.5) cukup untuk menutup jeda tab-visible wajar; bisa disetel di `config/perf.ts`.
3. **Idle tick 5 dtk dijalankan scheduler `setInterval` terpisah**, bukan di `fixedUpdate` (§7.2), karena tab hidden menghentikan render loop.
4. **Saat tab hidden, akrual diserahkan ke server** (offline accrual); client tak menghitung (§7.5).
5. **Accumulator di-reset saat resume** (`accumulator=0`, `previousTime=sekarang`) untuk cegah spiral (§8.3).
6. **Combat server-authoritative**: prediksi client di `fixedUpdate`, konfirmasi server diterapkan lewat `NetworkConfirmApply` (§6.2). Determinisme via PRNG seedable (§2.5).
7. **Phaser mengelola rAF**; kita tak buat loop rAF sendiri, hanya override `Scene.update` (§5.1).
8. **Party = 5 hero**, wave combat; budget ECS (§6) disetel untuk skala ini.
9. **Extrapolation tidak dipakai** (§4.4); kita tahan di `cur` bila perlu.
10. **Object pooling wajib** untuk entitas dinamis (damage number, projectile, enemy spawn) (§9.5).
11. **Texture atlas wajib** untuk seluruh sprite tempur guna batching (§9.2).
12. **Quality tiers** ada (high/medium/low); saat slowdown berkepanjangan, turunkan tier (§3.4).
13. **Offline accrual di-cap** (mis. 8 jam) di server agar tak abuse (§7.6).
14. **Save menyimpan `lastSeen`** saat handoff untuk akrual app-tertutup (§11.1).

---

> **Penutup:** Spesifikasi ini wajib dipatuhi seluruh subsistem yang menyentuh siklus permainan. Penyimpangan (mis. simulasi pakai delta mentah, atau idle tick di dalam `fixedUpdate`) harus disetujui lewat review lintas-dokumen (`STATE_MACHINE.md`, `COMBAT_NETCODE.md`, `PERF_CONFIG.md`). Determinisme dan stabilitas 60 FPS adalah kontrak, bukan opsional.
