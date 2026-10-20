# Arm-memory — Robotic Arm Record & Playback

Robotic arm 5 servo yang **digerakkan manual pakai potensiometer**, **menyimpan posisi sudut** setiap servo dengan satu tombol, lalu **memutar ulang gerakan** tersebut secara otomatis dan halus, berulang terus (loop).

> Atur pose → simpan → atur pose berikutnya → simpan → … → tekan lama → lengan menari sendiri.

---

## Fitur

- **Manual mode** — setiap servo dikontrol langsung oleh satu potensiometer (0–180°).
- **Record snapshot** — simpan posisi kelima servo sekaligus, sampai **50 langkah**.
- **Play mode** — lengan bergerak dari satu langkah ke langkah berikutnya secara bertahap (1° per 20 ms), lalu kembali ke langkah awal (loop).
- **Satu tombol saja** — tekan singkat untuk simpan, tekan lama untuk ganti mode.
- **Kalibrasi terpisah** untuk servo MG90S dan HV23MG.
- **Log via Serial Monitor** (9600 baud).

---

## Komponen

| Komponen | Jumlah | Keterangan |
|---|---|---|
| Arduino (Uno / Nano / Mega) | 1 | Lihat catatan pin A4 di bawah |
| PCA9685 16-ch PWM Servo Driver | 1 | Alamat I²C `0x40` |
| Servo MG90S | 4 | Channel 0, 1, 2, 4 |
| Servo HV23MG | 1 | Channel 3 (sendi yang butuh torsi lebih) |
| Potensiometer 10 kΩ | 5 | Satu per servo |
| Push button | 1 | Record / Play |
| Power supply 5–6 V (≥ 3 A) | 1 | **Khusus servo**, ke terminal V+ PCA9685 |
| Rangka robotic arm, kabel jumper, breadboard | — | — |

---

## Wiring

### PCA9685 ↔ Arduino

| PCA9685 | Arduino |
|---|---|
| VCC | 5V |
| GND | GND |
| SDA | SDA (A4 di Uno/Nano, pin 20 di Mega) |
| SCL | SCL (A5 di Uno/Nano, pin 21 di Mega) |
| V+ (terminal hijau) | Power supply servo (+) |
| GND (terminal hijau) | Power supply servo (−) **dan** GND Arduino |

### Potensiometer → Servo

| Potensiometer (wiper) | Channel PCA9685 | Servo |
|---|---|---|
| A4 | 0 | MG90S |
| A3 | 1 | MG90S |
| A2 | 2 | MG90S |
| A1 | 3 | **HV23MG** |
| A0 | 4 | MG90S |

Kaki luar potensiometer ke **5V** dan **GND**.

### Tombol

| Tombol | Arduino |
|---|---|
| Kaki 1 | D2 |
| Kaki 2 | GND |

Tidak perlu resistor — sudah pakai `INPUT_PULLUP` internal.

> [!WARNING]
> **Konflik pin A4 di Arduino Uno/Nano.** Pada Uno/Nano, A4 adalah pin **SDA** I²C yang dipakai PCA9685, sementara kode juga membaca potensiometer di A4. Solusinya: pakai **Arduino Mega** (SDA/SCL di pin 20/21), atau pindahkan potensiometer pertama ke **A6/A7** (Nano) dan ubah `potPin` di kode.

> [!CAUTION]
> Jangan menyuplai servo dari pin 5V Arduino. Lima servo bisa menarik arus beberapa ampere dan membuat Arduino reset/brownout. Pakai power supply terpisah dan **satukan GND**-nya.

---

## Instalasi

1. Install [Arduino IDE](https://www.arduino.cc/en/software).
2. Buka **Library Manager** dan install **Adafruit PWM Servo Driver Library** (dependensi `Adafruit BusIO` ikut terinstall).
3. Clone repo ini:
   ```bash
   git clone https://github.com/fathoni-tr/Arm-memory.git
   ```
4. Buka `Robotic_arm.ino`, pilih board & port, lalu **Upload**.
5. Buka **Serial Monitor** di **9600 baud**.

---

## Cara Pakai

Saat dinyalakan, semua servo bergerak ke posisi **90°** dan Serial Monitor menampilkan:

```
System ready:
Short press = Record snapshot
Long press  = Toggle Play mode
```

| Aksi | Durasi tekan | Hasil |
|---|---|---|
| **Short press** | < 600 ms | Simpan posisi semua servo saat ini sebagai langkah baru |
| **Long press** | ≥ 600 ms | Masuk / keluar Play mode |

### Alur kerja

1. **Manual mode** (default): putar potensiometer untuk membentuk pose.
2. **Short press** → `Snapshot saved step 0`.
3. Bentuk pose berikutnya → **short press** → `Snapshot saved step 1`. Ulangi (maks. 50 langkah).
4. **Long press** → `Play Mode ON`. Lengan memutar urutan gerakan dari langkah 0 sampai terakhir, lalu mengulang.
5. **Long press** lagi → `Play Mode OFF (Manual)`. Kontrol kembali ke potensiometer.

> [!TIP]
> Saat kembali ke manual mode, servo langsung "melompat" ke posisi potensiometer. Samakan dulu posisi potensiometer dengan posisi lengan supaya tidak menyentak.

---

## Cara Kerja

```
   ┌─────────────┐   long press   ┌─────────────┐
   │ MANUAL MODE │ ─────────────► │  PLAY MODE  │
   │ pot → servo │ ◄───────────── │ step → step │
   └─────────────┘   long press   └─────────────┘
          │
     short press
          ▼
   recordSnapshot() → positions[arrayMax][0..4]
```

- **Manual mode** — `analogRead()` (0–1023) dipetakan ke 0–180°, lalu dikonversi ke pulsa PCA9685 lewat `angleToPulse()` / `angleToPulseHV23()`.
- **Record** — sudut kelima servo (`ist[]`) disalin ke buffer `positions[MAX_STEP][SERVO_COUNT]`.
- **Play** — `play_servo()` menggeser setiap servo **1° per siklus** (tiap 20 ms) menuju target `sol[]`. Jika semua servo sudah sampai, target pindah ke langkah berikutnya (`prepareStep()`); setelah langkah terakhir kembali ke langkah 0.
- **Tombol** — `checkButton()` mengukur lama tekan dari transisi HIGH→LOW sampai LOW→HIGH.

Kecepatan gerak: 1° / 20 ms ≈ **50°/detik**, jadi perpindahan 0°→180° butuh ±3,6 detik.

---

## Konfigurasi

Semua parameter ada di bagian atas `Robotic_arm.ino`:

| Konstanta | Default | Fungsi |
|---|---|---|
| `SERVO_COUNT` | `5` | Jumlah servo |
| `potPin[]` | `{A4, A3, A2, A1, A0}` | Pin potensiometer per channel |
| `SERVOMIN` / `SERVOMAX` | `100` / `500` | Batas pulsa MG90S (0° / 180°) |
| `SERVOMIN_HV23` / `SERVOMAX_HV23` | `100` / `500` | Batas pulsa HV23MG |
| `MAX_STEP` | `50` | Jumlah langkah maksimum |
| `buttonPin` | `2` | Pin tombol |
| `pwm.setPWMFreq()` | `60` | Frekuensi PWM servo (Hz) |
| `delay(20)` di `loop()` | `20` ms | Makin kecil = gerakan play makin cepat |
| `600` di `checkButton()` | `600` ms | Ambang short vs long press |

### Kalibrasi servo

Jika servo bergetar/mentok di ujung atau tidak mencapai 0°/180°, sesuaikan `SERVOMIN`/`SERVOMAX` sedikit demi sedikit sampai gerakan penuh tanpa servo berbunyi di ujung.

---

## Keterbatasan

- **Rekaman hilang saat reset/mati listrik** — data disimpan di RAM.
- **Tidak ada cara menghapus rekaman** selain reset board; jika penuh muncul `Buffer full!`.
- **Play mode tanpa rekaman** — jika long press sebelum merekam apa pun, semua servo bergerak ke 0° (`sol[]` masih 0). Rekam minimal satu langkah dulu.
- **Short press saat play mode** tetap menambah langkah (merekam posisi lengan saat itu).
- **Tanpa debounce** — tombol yang "bouncy" bisa memicu snapshot ganda.
- Pembacaan potensiometer tanpa filter, servo bisa sedikit jitter di manual mode.

---

## Roadmap

- [ ] Simpan rekaman ke EEPROM
- [ ] Hapus rekaman tanpa reset
- [ ] Debounce tombol & smoothing potensiometer
- [ ] Kecepatan play yang bisa diatur
- [ ] Gerakan sinkron (semua servo tiba bersamaan)
- [ ] Kontrol & upload urutan gerakan via Serial / Bluetooth

---

## Struktur

```
Arm-memory/
├── Robotic_arm.ino   # Firmware utama
├── IDEA.md           # Catatan ide proyek
└── README.md
```

---

## Lisensi

Bebas dipakai untuk belajar dan dikembangkan. Tambahkan file `LICENSE` (mis. MIT) jika ingin lisensi formal.
