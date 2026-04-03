# ARIA: Acoustic Road Identification & Attribution

**An ultrasonic vehicle identification system for automated horn violation enforcement in urban silent zones.**

ARIA embeds a unique inaudible acoustic signature into every vehicle's horn signal. When the horn is pressed, a roadside unit decodes the ultrasonic ID, identifies the vehicle, and issues a challan automatically — no manual policing, no blurry camera footage, no human intervention.

---

## Problem Statement

Horn noise pollution is a severe public health issue in Indian cities. The WHO Environmental Noise Guidelines (2018) link chronic traffic noise exposure to ischemic heart disease, hypertension, sleep disturbance, and cognitive impairment. Research by Vijay et al. (2015) found that honking alone adds 2–5 dB(A) to ambient traffic noise in Indian urban environments.

Indian law designates 100-metre zones around hospitals, schools, and courts as "silent zones" under the Noise Pollution (Regulation and Control) Rules, 2000. Section 194F of the Motor Vehicles Act prescribes fines of Rs.1,000 (first offence) and Rs.2,000 (repeat) for unnecessary honking in these zones. Yet enforcement is nearly non-existent — manual policing is impractical, and existing noise camera systems like France's Hydra suffer from fundamental limitations (NYC's pilot found 38% of licence plate captures were too blurry to read).

ARIA solves this by shifting from passive noise detection to active vehicle identification through ultrasonic acoustic tagging.

---

## How ARIA Works

Every registered vehicle is fitted with a small electronic module connected to its horn circuit. When the driver presses the horn, two things happen simultaneously: the audible horn sounds normally (safety preserved), and the module emits a 25 kHz ultrasonic signal encoding the vehicle's unique 40-bit binary ID. This signal is inaudible to humans but detectable by the roadside unit's ultrasonic microphone.

The roadside unit captures the signal, isolates the ultrasonic band, decodes the binary ID, validates it against the national vehicle database, and — if the vehicle is in a silent zone — generates an enforcement record automatically.

If a vehicle's module has been tampered with or removed, no ultrasonic signal is emitted. The roadside unit detects a horn sound with no accompanying ID and falls back to ANPR (Automatic Number Plate Recognition) camera identification, issuing a higher Rs.5,000 tampering fine.

---

## System Architecture

```
TIER 1 — VEHICLE                TIER 2 — ROADSIDE UNIT              TIER 3 — BACKEND
========================        ============================        ====================

Horn button pressed              Audible mic detects horn            DB lookup by
        |                               |                           acoustic ID
  STM32L0 wakes up               Ultrasonic mic captures                |
        |                         25 kHz OOK signal                 Vehicle identified
  Reads 32-bit ID                       |                               |
  from flash memory              Bandpass filter                    Challan decision
        |                         20-45 kHz isolation               engine runs
  Appends CRC-8                         |                               |
  (40-bit total)                 Envelope detection                 Deduplication check
        |                               |                               |
  Piezo transducer               NCC preamble search                Fine calculated
  emits at 25 kHz                       |                           (Rs.1000 or Rs.5000)
        |                        OOK demodulation                       |
  Signal propagates               40 bits decoded                   Challan record
  through air                           |                           written to DB
                                 CRC-8 validation                       |
                                        |                           SMS notification
                                 Database lookup ──────────────────→ to vehicle owner
```

---

## Module Breakdown

The project is implemented as 5 Jupyter notebooks executed sequentially. Each module reads from the previous module's output and writes to the next module's input. The shared `acoustic_id.db` SQLite database is the central data store.

### Module 1 — ID Generation & Database

**Purpose:** Register vehicles and generate unique acoustic IDs.

**Input:** Excel or JSON file with columns: `plate_number`, `owner_name`, `owner_contact`, `owner_address`.

**Process:** Each plate number is normalised, validated against Indian registration format (standard SS DD LL NNNN and Bharat Series YY BH NNNN LL), then hashed with SHA-256. The first 32 bits of the hash become the acoustic ID. A CRC-8 checksum (polynomial 0x07) is appended to produce the 40-bit transmission ID. Collision avoidance slides the 32-bit window across the 256-bit hash if needed.

**Output:** `acoustic_id.db` containing the `vehicles` table with all registration and ID data.

**Key functions:** `compute_crc8()`, `normalise_plate()`, `validate_plate()`, `generate_acoustic_id()`, `register_vehicle()`, `register_vehicles_batch()`, `lookup_by_plate()`, `lookup_by_acoustic_id()`, `lookup_by_transmission_id()`

### Module 2 — Signal Encoder

**Purpose:** Convert each vehicle's 40-bit transmission ID into an ultrasonic audio file.

**Input:** Reads `transmission_id` from `acoustic_id.db`.

**Process:** On-Off Keying (OOK) modulation at 25 kHz carrier frequency. Bit `1` = 5ms sine wave burst, Bit `0` = 5ms silence. The transmission frame is structured as: 10ms guard silence → 4-bit preamble (1010) → 32-bit acoustic ID → 8-bit CRC → 10ms guard silence. Total duration: 240ms. Sample rate: 96 kHz (Nyquist requirement for 25 kHz signal).

**Output:** One `.wav` file per vehicle in `acoustic_signals/`. Database updated with `wav_path` and `wav_generated` flag.

**Key functions:** `generate_ook_signal()`, `save_wav()`, `load_wav()`, `encode_vehicle()`, `encode_all()`, `plot_waveform()`, `plot_spectrogram()`

### Module 2B — Noise Simulator

**Purpose:** Generate realistic noisy test signals for decoder validation.

**Input:** Clean `.wav` files from `acoustic_signals/`.

**Process:** 7 noise profiles × 4 SNR levels = 28 files per vehicle. Six additive profiles layer noise on top of the clean OOK signal (ultrasonic ID preserved). One replacement profile (tamper) discards the OOK signal entirely and substitutes a horn tone + ambient noise with zero ultrasonic content.

**Noise profiles:**

| Profile | Frequency Range | Scenario |
|---|---|---|
| Traffic | 80–6000 Hz layered | Rush hour, major intersection |
| Rain | Full spectrum, rolloff >10 kHz | Heavy monsoon |
| DJ | 60–2000 Hz bass-heavy | DJ event outside hospital |
| Procession | 80–8000 Hz broadband, 2 Hz beat | Wedding dhol/band procession |
| Hawker | 300–3400 Hz voice band | Street vendor with loudspeaker |
| Crowd | 300–4000 Hz multi-layer | Rally, protest, market |
| Tamper | 420 Hz horn + traffic noise | Tampered/removed module |

**SNR levels:**

| Label | SNR Condition | Meaning |
|---|---|---|
| 60 dB | +20 dB | Signal 10× louder than noise |
| 70 dB | +10 dB | Signal 3× louder than noise |
| 80 dB | 0 dB | Signal equals noise |
| 90 dB | −20 dB | Noise 10× louder than signal |

**Output:** 28 `.wav` files per vehicle in `noisy_signals/`. Zero database interaction.

**Key functions:** `bandlimited_noise()`, `mix_noise()`, `generate_horn_tone()`, `synthesise_traffic()`, `synthesise_rain()`, `synthesise_dj()`, `synthesise_procession()`, `synthesise_hawker()`, `synthesise_crowd()`, `synthesise_tamper()`, `generate_noisy_files()`, `generate_all()`, `verify_noisy_dir()`

### Module 3 — Signal Decoder

**Purpose:** Decode noisy audio files and identify vehicles.

**Input:** `.wav` files from `noisy_signals/`.

**Process:** 8-stage pipeline per file:

1. **Load WAV** — verify 96 kHz sample rate
2. **Ambient noise measurement** — RMS dB of full signal, sets context flag (`CONTEXTUAL_REVIEW` if >85 dB)
3. **Bandpass filter** — 4th-order Butterworth, 20–45 kHz, SOS form. Strips all audible noise.
4. **Envelope detection** — rectify + low-pass at 400 Hz, clamp negatives, peak-normalise
5. **Preamble search** — normalised cross-correlation (NCC) against `1010` template, threshold 0.7, constrained to first `GUARD_SAMPLES + 2 × SAMPLES_PER_BIT` samples
6. **OOK demodulation** — 40 consecutive 5ms windows, adaptive threshold at midpoint of max/min energy
7. **CRC-8 validation** — recompute and compare
8. **Database lookup** — retrieve vehicle record by acoustic ID

**Detection modes:**

| Mode | Meaning | Challan Path |
|---|---|---|
| `ACOUSTIC_ID` | Full decode success | Rs.1,000 honking challan |
| `ANPR_FALLBACK` | No preamble found — tampered module | Rs.5,000 tampering challan |
| `ACOUSTIC_ID_CORRUPTED` | Preamble found but CRC failed | Held for human review |
| `UNREGISTERED_MODULE` | CRC passed but ID not in database | Rs.5,000 unregistered challan |

**Output:** Decode results written to `acoustic_id.db` (`last_decode_status`, `last_decode_time`, `context_flag`). 7×4 accuracy table for the research paper.

**Key functions:** `bandpass_filter()`, `detect_envelope()`, `measure_ambient_db()`, `find_preamble()`, `demodulate_ook()`, `decode_wav()`, `update_decode_result()`, `run_reliability_test()`

### Module 4 — Challan Decision Engine

**Purpose:** Read decode results and issue enforcement records.

**Input:** `last_decode_status` and `context_flag` from `acoustic_id.db` (written by Module 3).

**Process:** Decision tree applies fine amounts based on detection mode and context flag. 15-minute deduplication prevents multiple challans for the same vehicle in the same zone within a short window. 24-hour repeat offence check doubles the fine from Rs.1,000 to Rs.2,000 for acoustic ID violations.

**Decision tree:**

| Detection Mode | Context Flag | Fine | Status |
|---|---|---|---|
| `ACOUSTIC_ID` | `NORMAL` | Rs.1,000 (Rs.2,000 repeat) | issued |
| `ACOUSTIC_ID` | `CONTEXTUAL_REVIEW` | Rs.1,000 | held_for_review |
| `ANPR_FALLBACK` | any | Rs.5,000 | issued |
| `UNREGISTERED_MODULE` | any | Rs.5,000 | issued |
| `ACOUSTIC_ID_CORRUPTED` | any | Rs.1,000 | held_for_review |

**Output:** `challans` table in `acoustic_id.db` with UUID, plate, fine, status, GPS coordinates, evidence reference, and offence count.

**Key functions:** `is_duplicate()`, `is_repeat_offender()`, `decide_action()`, `issue_challan()`, `process_all_pending()`, `show_challans()`

---

## Signal Design

### Why OOK at 25 kHz

On-Off Keying was chosen over FSK and PSK for three reasons: the energy contrast between bit 1 (carrier present) and bit 0 (silence) is essentially infinite, making threshold-based demodulation trivially robust; OOK requires only a single carrier frequency, simplifying transducer design; and the demodulator can be implemented with basic energy detection rather than phase-locked loops.

The 25 kHz carrier sits above all human hearing (>20 kHz), below the range where atmospheric attenuation becomes severe (>40 kHz), and within the operating range of standard automotive piezoelectric parking sensors — hardware that already exists on most vehicles.

### Transmission Frame

```
[10ms guard] [1010 preamble] [32-bit acoustic ID] [8-bit CRC] [10ms guard]
   silence      4 bits              from SHA-256      integrity    silence
                 20ms                  160ms             40ms        10ms

Total: 240ms — fits within a 500ms minimum horn press
```

### CRC-8 Error Detection

Polynomial 0x07 (standard CRC-8/SMBUS). Detects all single-bit errors, all double-bit errors, and all burst errors up to 8 bits in length. A corrupted signal fails CRC validation before any database lookup occurs, preventing false vehicle identification.

---

## Results

### Test Configuration

- **Vehicles:** 500 registered across 32 Indian state codes (including BH-series)
- **Test files:** 14,000 (500 vehicles × 7 profiles × 4 SNR levels)
- **Carrier frequency:** 25 kHz
- **Sample rate:** 96 kHz
- **Bit duration:** 5 ms
- **Preamble:** 1010 (4-bit alternating pattern)

### Acoustic ID Decode Accuracy

| Profile | +20 dB SNR | +10 dB SNR | 0 dB SNR | −20 dB SNR |
|---|---|---|---|---|
| Traffic | 100% | 100% | 100% | 0% |
| Rain | 100% | 100% | 100% | 0% |
| DJ | 100% | 100% | 98% | 0% |
| Procession | 100% | 100% | 100% | 8% |
| Hawker | 100% | 100% | 100% | 0% |
| Crowd | 100% | 100% | 100% | 17% |
| Tamper* | 100% | 99% | 100% | 100% |

*Tamper column shows % correct ANPR_FALLBACK detection (not acoustic decode accuracy).*

The system achieves 100% vehicle identification accuracy at SNR conditions of +20 dB to 0 dB across all six urban noise profiles, with 99–100% tamper detection accuracy. The sharp accuracy drop at −20 dB SNR (noise 10× louder than signal) represents the system's honest operational boundary.

Procession (8%) and crowd (17%) profiles retain partial decodability at −20 dB because their noise energy is concentrated in the audible band (80–8000 Hz), allowing the 20–45 kHz bandpass filter to remove a larger proportion compared to spectrally broader profiles like rain.

### Challan Summary

- **Total challans issued:** 500
- **Acoustic ID challans (Rs.1,000):** vehicles with successful decode
- **Tamper challans (Rs.5,000):** vehicles flagged as ANPR_FALLBACK
- **Total fine revenue:** Rs.16,08,000
- **Deduplication events:** 0 (single decode per vehicle in test)
- **Held for review:** 0 (no CONTEXTUAL_REVIEW flags in test dataset)

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.11+ |
| Signal processing | NumPy, SciPy |
| Audio I/O | SoundFile |
| Database | SQLite3 (standard library) |
| Data ingestion | openpyxl (Excel), json (standard library) |
| Visualization | Matplotlib |
| Environment | Jupyter Notebook |

No external APIs, no cloud dependencies, no GPU required. Fully offline operation.

---

## How to Run

### Prerequisites

```bash
pip install numpy scipy soundfile matplotlib openpyxl
```

### Execution Order

**Step 1 — Module 1: Register vehicles**
- Place `vehicles.xlsx` (or `.json`) in the notebook directory
- Set `INPUT_FILE = "vehicles.xlsx"` in Step 9
- Run all cells
- Output: `acoustic_id.db` created

**Step 2 — Module 2: Encode signals**
- Ensure `acoustic_id.db` exists in the same directory
- Run all cells
- Output: `acoustic_signals/` folder with one `.wav` per vehicle

**Step 3 — Module 2B: Generate noisy test files**
- Ensure `acoustic_signals/` exists with `.wav` files
- Run all cells
- Output: `noisy_signals/` folder with 28 `.wav` files per vehicle
- Run `verify_noisy_dir()` to confirm completeness

**Step 4 — Module 3: Decode and test**
- Ensure `acoustic_id.db` and `noisy_signals/` exist
- Run all cells
- Output: 7×4 accuracy table printed, decode results written to DB

**Step 5 — Module 4: Issue challans**
- Ensure Module 3 has populated `last_decode_status` in DB
- Run all cells
- Output: `challans` table populated, summary printed
- Run `show_challans(conn)` to view all records

---

## Folder Structure

```
ARIA/
├── Module1_ARIA_1_0.ipynb          # ID generation & database
├── Module2_ARIA_1_0.ipynb          # Signal encoder
├── Module2B_ARIA_1_0.ipynb         # Noise simulator
├── Module3_ARIA_1_0.ipynb          # Signal decoder
├── Module4_ARIA_1_0.ipynb          # Challan decision engine
├── vehicles.xlsx                    # Input data (500 vehicles)
├── acoustic_id.db                   # SQLite database (generated)
├── acoustic_signals/                # Clean .wav files (generated)
│   ├── MH12AB1234_acoustic_id.wav
│   ├── DL08GH3456_acoustic_id.wav
│   └── ...
├── noisy_signals/                   # Noisy test files (generated)
│   ├── MH12AB1234_traffic_60db.wav
│   ├── MH12AB1234_tamper_90db.wav
│   └── ...
└── README.md
```

---

## Hardware Architecture (Theoretical)

This project is a software simulation. The hardware described below is specified theoretically for real-world deployment.

### Vehicle Module (Per Vehicle — Rs.240 at manufacturing scale)

| Component | Model | Purpose |
|---|---|---|
| Microcontroller | STM32L0 (ARM Cortex-M0+) | Stores acoustic ID, generates OOK signal, manages ECU heartbeat |
| Secure element | ATECC608A | Tamper-proof cryptographic key storage for ECU pairing |
| Ultrasonic transducer | Automotive-grade Bosch piezo (40 kHz) | Emits OOK signal — same component as parking sensors |
| CAN bus transceiver | TJA1050 | Connects to vehicle ECU for heartbeat and horn event detection |
| Power regulator | AMS1117-3.3 | Steps down 12V horn circuit to 3.3V for module electronics |
| Casing | Epoxy potting compound | Tamper-evident — intrusion detection wire loop, self-destructs on breach |

### Roadside Unit (Per Installation — Rs.53,000 at volume)

| Component | Model | Purpose |
|---|---|---|
| Edge processor | NVIDIA Jetson Nano | Runs CNN horn classifier, DSP pipeline, and ANPR OCR simultaneously |
| Ultrasonic mic array | 3× Knowles SPU1410 MEMS | Captures 25 kHz OOK signal, triangular array for TDOA direction sensing |
| Audible microphone | Electret condenser | Detects horn sound to trigger ultrasonic processing |
| ANPR camera | Dahua ITC337 (4K, IR, global shutter) | Reads licence plates — fallback identification for tampered modules |
| GPS module | u-blox NEO-M9N (±2m accuracy) | Silent zone boundary enforcement and timestamp synchronisation |
| 4G modem | Quectel EC21 (TRAI certified) | Transmits challan data to backend server |
| Power | 20W solar + 20Ah LiFePO4 battery | Self-powered, 48+ hours autonomy without sunlight |

---

## Security & Tamper Detection

The system uses a three-layer security model inspired by Apple's Parts Pairing architecture:

**Layer 1 — Cryptographic ECU Pairing:** At installation, the module's ATECC608A secure element generates a 256-bit ECC key pair. The public key is registered with the vehicle's ECU. The private key cannot be extracted — it is hardware-protected and self-destructs on physical intrusion.

**Layer 2 — 60-Second Heartbeat:** The ECU sends an encrypted challenge to the module every 60 seconds via CAN bus. The module signs it with its private key. Signature mismatch (wrong module, tampered module, missing module) triggers an alert.

**Layer 3 — Dashboard Warning:** On tamper detection, the ECU displays a persistent warning on the instrument cluster/infotainment system: "Acoustic ID Module Tamper Detected — Visit RTO Authorised Centre." This warning cannot be dismissed by the driver and persists across ignition cycles.

### Attack Scenarios

| Attack | Module Emits? | Who Gets Caught | How |
|---|---|---|---|
| Module physically removed | No | Correct vehicle via ANPR camera | ECU heartbeat fails → dashboard alert |
| Signal jammed | No | Correct vehicle via ANPR camera | Roadside detects horn without ID |
| Replaced with stolen module | No — ECU key mismatch | Criminal via ANPR | Module refuses to emit in unpaired vehicle |
| Replaced with fake module | No — key not in RTO DB | Flagged as unregistered | Database lookup fails |
| Water/rat/physical damage | Corrupted signal | ANPR fallback | CRC check fails → signal discarded |

In every scenario, tampering is self-defeating — the driver is worse off than if the module were intact.

---

## Legal Framework

### Applicable Law

- **Noise Pollution (Regulation and Control) Rules, 2000** — defines silent zones as 100m around hospitals, educational institutions, and courts
- **Section 194F, Motor Vehicles Act, 1988** — Rs.1,000 first offence, Rs.2,000 repeat offence for use of horn or sound-emitting devices in violation of rules
- **Exception:** "Reasonable cause" (emergency avoidance of accident) overrides the silent zone restriction

### Enforcement Model

ARIA follows an "issue-then-dispute" model aligned with the existing Parivahan e-Challan system:

1. Challan is automatically generated for every detected violation
2. Driver receives notification within 24 hours
3. 15-day dispute window opens automatically
4. Driver selects reason: pedestrian danger, vehicle obstruction, medical emergency, road obstruction, other
5. Evidence upload: dashcam footage, photos, written explanation
6. Human traffic officer reviews and approves/rejects
7. If rejected: driver may escalate to Traffic Court before a Magistrate

### Contextual Auto-Suppression

If the roadside unit detects sustained ambient noise above 85 dB for more than 30 continuous seconds (indicating an abnormal event — procession, accident, protest), all challans generated during that window are automatically held for human review rather than auto-issued. The driver bears zero burden — the system recognises the context itself.

---

## Future Work

- **Emergency Mode Signal:** Triple horn press within 2 seconds triggers a special encoding that flags the event as `EMERGENCY_REVIEW` instead of `VIOLATION`, routing directly to human review without driver action
- **ML-Enhanced Bit Classification:** Scikit-learn Random Forest or SVM trained on bit-window features to improve decode accuracy at extreme sub-zero SNR conditions
- **Real-Time Audio Stream Processing:** PyAudio/sounddevice integration for live microphone decode — closest simulation to Jetson Nano deployment
- **Beamforming:** MUSIC/ESPRIT algorithms on the 3-mic array for spatial separation of simultaneous horn events
- **Time-Varying Codes:** Rolling code system where the acoustic ID changes with each horn press using a synchronized PRNG — prevents replay attacks
- **City Pilot Proposal:** Phased deployment plan starting with 10 silent zones in Pune, integration with existing CCTV and traffic management infrastructure

---

## Deployment Cost Estimate

| Item | Retail | At Scale |
|---|---|---|
| Acoustic ID module per vehicle | Rs.898 | Rs.240 |
| Roadside unit per installation | Rs.83,000 | Rs.53,000 |
| Per silent zone (3 units) | Rs.2,49,000 | Rs.1,59,000 |
| National deployment (320M vehicles + 10,000 zones) | Rs.5,363 crore | Rs.2,358 crore |

At 1,000 challans per zone per year × Rs.1,000 = Rs.100 crore annual revenue. Full cost recovery in under 3 years.

---

## References

The full research paper accompanying this repository cites 24 papers across 8 categories: urban noise pollution and health effects, horn detection datasets and ML models, noise radar and enforcement systems, smart horn suppression via IoT, acoustic telemetry, ultrasonic signal technology, signal processing techniques, and Indian traffic policy. Key references include WHO Environmental Noise Guidelines (2018), HornBase dataset (Dim et al. 2024), MELAUDIS dataset (Parineh et al. 2025), Hydra noise radar (Mietlicki et al. 2022), and the Mosquito ultrasonic device (Stapleton 2005).

---

## Author

**Dishant Sasane**
- Vidyavardhini College of Engineering & Technology
- dishants0605@gmail.com
- https://www.linkedin.com/in/dishants039102315

---

## License

This project is developed as part of a Master's thesis / research application. All rights reserved. Contact the author for permissions regarding reproduction or deployment.
