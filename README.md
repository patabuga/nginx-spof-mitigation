# Nginx SPOF Mitigation

**Paper B - Evaluasi Mitigasi Single Point of Failure pada Nginx Reverse Proxy dalam Arsitektur Kontainer Docker**

Repository ini berisi skenario pengujian, analisis, dan dokumentasi untuk Penelitian Rizal tentang mitigasi SPOF pada Nginx Reverse Proxy.

## Ringkasan Penelitian

Penelitian ini mengevaluasi strategi mitigasi **Single Point of Failure (SPOF)** pada Nginx Reverse Proxy dalam arsitektur kontainer Docker, dengan fokus pada:

1. **Identifikasi SPOF** - Titik arsitektural yang rentan kegagalan
2. **Analisis Konfigurasi** - Parameter Nginx yang mempengaruhi resiliensi
3. **Efektivitas Mitigasi** - Evaluasi restart policy dan healthcheck Docker

## Metodologi Studi Kasus

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      METODOLOGI STUDI KASUS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐│
│   │  SCENARIO 1  │    │  SCENARIO 2  │    │  SCENARIO 3  │               │
│   │ IDENTIFIKASI │    │   TIMEOUT    │    │ HEALTHCHECK  │               │
│   │    SPOF      │───▶│  EVALUATION  │───▶│  EFFECTIVENSS│               │
│   │              │    │              │    │              │               │
│   │ Architecture │    │ Nginx Config │    │ Docker Policy│               │
│   │ Walkthrough  │    │ max_fails    │    │ restart      │               │
│   │              │    │ fail_timeout  │    │ healthcheck  │               │
│   └──────────────┘    └──────────────┘    └──────────────┘               │
│          │                   │                    │                       │
│          ▼                   ▼                    ▼                       │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐│
│   │ SPOF Matrix  │    │ Compliance   │    │ Recovery Time │               │
│   │Risk Level│    │ Rate %       │    │ Effectiveness│               │
│   └──────────────┘    └──────────────┘    └──────────────┘               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Parameter Nginx yang Dianalisis

| Parameter | Default | Best Practice | Fungsi |
|-----------|---------|---------------|--------|
| max_fails | 1 | 3-5 | Jumlah kegagalan sebelum server ditandai down |
| fail_timeout | 10s | 30s | Durasi server ditandai tidak tersedia |
| proxy_connect_timeout | 60s | 10-30s | Timeout koneksi ke backend |
| proxy_read_timeout | 60s | 30-90s | Timeout membaca response dari backend |
| proxy_send_timeout | 60s | 30-60s | Timeout mengirim request ke backend |
| keepalive | - | 32-64 | Jumlah koneksi persistent ke upstream |

## Docker Restart Policy

| Policy | Deskripsi | Rekomendasi |
|--------|------------|-------------|
| no | Tidak restart (default) | ❌ Tidak |
| on-failure | Restart jika exit code non-zero | ⚠️ Dengan batasan |
| always | Selalu restart | ✅ Ya |
| unless-stopped | Restart kecuali dihentikan manual | ✅ Ya (Recommended) |

## Struktur Direktori

```
nginx-spof-mitigation/
├── docs/
│   ├── TEST_PLAN.md          # Rencana pengujian lengkap
│   ├── METHODOLOGY.md        # Metodologi studi kasus
│   ├── LITERATURE_REVIEW.md  # C5 analysis tinjauan pustaka
│   └── SPOF-CRITERIA.md      # Kriteria identifikasi SPOF
│
├── test-scenarios/
│   ├── scenario-1-architecture/  # Identifikasi SPOF
│   ├── scenario-2-nginx-config/   # Analisis timeout
│   └── scenario-3-docker-healthcheck/ # Efektivitas healthcheck
│
├── evidence/
│   ├── scenario-1/
│   │   ├── screenshots/
│   │   ├── config-snapshots/
│   │   └── MANIFEST.md
│   ├── scenario-2/
│   └── scenario-3/
│
├── analysis/
│   ├── spof-identification.py
│   ├── compliance-matrix.py
│   └── hardening-recommendations.py
│
├── configs/
│   ├── nginx-default.conf     # Config default (baseline)
│   ├── nginx-hardened.conf # Config hardened (rekomendasi)
│   ├── docker-compose-default.yml
│   └── docker-compose-hardened.yml
│
└── report/
    ├── BAB-I-PENDAHULUAN.md
    ├── BAB-II-TINJAUAN-PUSTAKA.md
    ├── BAB-III-METODE.md
    ├── BAB-IV-HASIL.md
    └── BAB-V-KESIMPULAN.md
```

## Testbed

Testbed shared tersedia di repository:
- **container-resilience-analysis**: https://github.com/patabuga/container-resilience-analysis

## Skenario Pengujian

### Skenario 1: Identifikasi SPOF

**Tujuan:** Memetakan titik-titik arsitektural yang berpotensi menjadi SPOF.

**Langkah:**
1. Analisis arsitektur kontainer
2. Identifikasi komponen tanpa redundansi
3. Evaluasi dependensi antar layanan
4. Dokumentasi risiko dan mitigasi existing

**Output:**
- SPOF Matrix
- Architecture diagram with risk annotations
- Risk level assessment

### Skenario 2: Analisis Konfigurasi Nginx

**Tujuan:** Mengevaluasi efektivitas parameter timeout dan upstream.

**Langkah:**
1. Test dengan config default
2. Variasi max_fails: 1, 3, 5
3. Variasi fail_timeout: 10s, 30s, 60s
4. Variasi proxy_*_timeout
5. Ukur response time dan error rate

**Output:**
- Compliance matrix terhadap best practices
- Timeout impact analysis
- Configuration recommendations

### Skenario 3: Efektivitas Healthcheck

**Tujuan:** Mengukur efektivitas restart policy dan healthcheck Docker.

**Langkah:**
1. StopNginx container
2. Ukur waktu recovery dengan restart: no
3. Ukur waktu recovery dengan restart: always
4. Ukur waktu recovery dengan restart: unless-stopped
5. Bandingkan recovery time

**Output:**
- Recovery time comparison
- Healthcheck effectiveness score
- Restart policy recommendation

## Prerequisites

- Testbed sudah ter-deploy
- Akses SSH ke VM
- Docker CLI configured

## Menjalankan Pengujian

```bash
# Clone testbed
git clone https://github.com/patabuga/container-resilience-analysis.git

# Skenario 1: Identifikasi SPOF
./scripts/run-spof-tests.sh arch

# Skenario 2: Analisis Timeout
./scripts/run-spof-tests.sh timeout

# Skenario 3: Healthcheck Test
./scripts/run-spof-tests.sh healthcheck

# Collect evidence
./scripts/collect-evidence.sh
```

## Compliance Matrix Template

```markdown
## Nginx Configuration Compliance

| Parameter | Current | Best Practice | Status | Gap |
|-----------|---------|---------------|--------|-----|
| max_fails | 1 | 3-5 | ❌ Not Compliant | Need to increase |
| fail_timeout | 10s | 30s | ❌ Not Compliant | Need to increase |
| ... | ... | ... | ... | ... |

**Compliance Rate: X%**
```

## License

MIT License - See [LICENSE](LICENSE) for details.

## Related Repositories

| Repository | Deskripsi |
|------------|-----------|
| [container-resilience-analysis](https://github.com/patabuga/container-resilience-analysis) | Shared Testbed |
| [resource-contention-analysis](https://github.com/patabuga/resource-contention-analysis) | Paper A - Dimas |

## Penulis

**Paper B**: Rizal
**Testbed**: Patabuga Research Team