# Test Plan - Paper B: Nginx SPOF Mitigation

## Pendahuluan

Dokumen ini menjelaskan rencana pengujian untuk evaluasi mitigasi Single Point of Failure (SPOF) pada Nginx Reverse Proxy dalam arsitektur kontainer Docker.

## Tujuan Pengujian

1. Mengidentifikasi titik-titik arsitektural yang berpotensi menjadi SPOF
2. Menganalisis strategi mitigasi melalui parameter konfigurasi Nginx
3. Mengevaluasi efektivitas Docker healthcheck dalam memitigasi kegagalan

## Lingkungan Pengujian

### Komponen yang Diuji
| Komponen | Peran | SPOF Risk |
|----------|------|-----------|
| Nginx Reverse Proxy | Gerbang akses tunggal | **HIGH** |
| MariaDB | Database backend | HIGH |
| WordPress | Application server | MEDIUM |
| Docker Host | Infrastruktur | **CRITICAL** |

### Tools
| Tool | Fungsi |
|------|--------|
| JMeter | Load testing |
| Docker inspect | Container analysis |
| Prometheus | Metrics collection |
| Grafana | Visualization |

## Skenario Pengujian

### Skenario 1: Identifikasi SPOF

**Tujuan:** Memetakan titik-titik arsitektural yang berpotensi menjadi SPOF.

**Langkah Pengujian:**

1. **Architecture Walkthrough**
   - Inspeksi docker-compose.yml
   - Analisis network topology
   - Identifikasi single-instance components
   
2. **Configuration Snapshot**
   - Capture nginx.conf
   - Capture docker-compose.yml
   - Capture container inspect output

3. **Risk Assessment**
   - Kategorisasi risiko per komponen
   - Analisis dependensi
   - Identifikasi failure cascade potential

**Checklist:**
- [ ] Snapshot konfigurasi Nginx
- [ ] Snapshot konfigurasi Docker Compose
- [ ] Inspeksi container dependencies
- [ ] Analisis network topology
- [ ] Dokumentasi SPOF points

**Output:**
```markdown
## SPOF Identification Report

### Component: Nginx Reverse Proxy
- **Risk Level:** HIGH
- **Failure Mode:** Single instance, no failover
- **Impact:** 100% service unavailability
- **Mitigation:** None (current)

### Component: MariaDB
- **Risk Level:** HIGH
- **Failure Mode:** Single instance, no replication
- **Impact:** Database unavailable, WordPress down
- **Mitigation:** Docker restart policy

### Component: Docker Host
- **Risk Level:** CRITICAL
- **Failure Mode:** VM failure
- **Impact:** Complete service outage
- **Mitigation:** None (single VM)
```

---

### Skenario 2: Analisis Konfigurasi Nginx

**Tujuan:** Mengevaluasi efektivitas parameter timeout dan upstream dalam mitigasi SPOF.

**Parameter yang Diuji:**

| Test | max_fails | fail_timeout | proxy_connect_timeout | proxy_read_timeout |
|------|-----------|--------------|----------------------|-------------------|
| T1 (Default) | 1 | 10s | 60s | 60s |
| T2 (Basic) | 3 | 30s | 30s | 90s |
| T3 (Advanced)| 5 | 60s | 10s | 120s |

**Langkah Pengujian:**

1. **Baseline Measurement**
   - Deploy dengan config default
   - Ukur response time normal
   - Catat error rate baseline

2. **Timeout Variation Test**
   - Untuk setiap variasi timeout:
     - Apply konfigurasi
     - Restart Nginx container
     - Jalankan load test 5 menit
     - Ukur metrik

3. **Backend Failure Simulation**
   - Stop WordPress container
   - Ukur waktu deteksi failure
   - Ukur waktu recovery Nginx
   - Catat behavior

**Metrik yang Diukur:**

| Metrik | Deskripsi | Target |
|--------|-----------|--------|
| Detection Time | Waktu Nginx mendeteksi backend down | < 30s |
| Recovery Time | Waktu Nginx recovery setelah backend up | < 60s |
| Error Rate | Persentase request gagal selama failure | Track |
| Response Time | Latency selama operasi normal | Track |

**Checklist:**
- [ ] Test dengan config default
- [ ] Test dengan max_fails=3
- [ ] Test dengan max_fails=5
- [ ] Test dengan fail_timeout=30s
- [ ] Test dengan fail_timeout=60s
- [ ] Capture semua hasil
- [ ] Generate compliance matrix

**Output: Compliance Matrix**

```markdown
## Nginx Configuration Compliance Matrix

| Parameter | Current | Best Practice | Gap | Compliance |
|-----------|---------|--------------|-----|------------|
| max_fails | 1 | 3-5 | -2 to -4 | 20% |
| fail_timeout | 10s | 30s | -20s | 33% |
| proxy_connect_timeout | 60s | 10-30s | +30 to +50s | 33% |
| proxy_read_timeout | 60s | 30-90s | 0 to -30s | 67% |
| keepalive | 0 | 32-64 | -32 to -64 | 0% |

**Overall Compliance: 30.6%**
```

---

### Skenario 3: Efektivitas Healthcheck

**Tujuan:** Mengukur efektivitas restart policy dan healthcheck Docker dalam memitigasi kegagalan.

**Konfigurasi yang Diuji:**

| Test | Restart Policy | Healthcheck | Expected Behavior |
|------|----------------|-------------|-------------------|
| H1 | no | no | No recovery |
| H2 | on-failure | no | Conditional recovery |
| H3 | always | no | Unconditional recovery |
| H4 | unless-stopped | yes | Best practice|

**Langkah Pengujian:**

1. **Baseline Recording**
   - Catat state awal container
   - Catat healthcheck status
   - Catat uptime

2. **Failure Simulation**
   - Stop Nginx container: `docker stop cra-nginx`
   - Catat timestamp stop
   - Monitor setiap 2 detik

3. **Recovery Monitoring**
   - Tunggu recovery (jika ada)
   - Catat timestamp recovery
   - Hitung recovery time

4. **Repeat for Each Policy**
   - Ulangi untuk setiap restart policy
   - Catat recovery time masing-masing

**Metrik yang Diukur:**

| Metrik | Deskripsi | Formula |
|--------|-----------|---------|
| Recovery Time | Waktu dari failure ke recovery | t_recovery - t_stop |
| Detection Time | Waktu healthcheck detect failure | interval × retries |
| Availability Loss | Persentase downtime | (recovery_time / test_duration) × 100 |

**Checklist:**
- [ ] Test restart: no
- [ ] Test restart: on-failure
- [ ] Test restart: always
- [ ] Test restart: unless-stopped
- [ ] Capture recovery time masing-masing
- [ ] Compare effectiveness

**Output: Recovery Time Comparison**

```markdown
## Docker Restart Policy Effectiveness

| Policy | Recovery Time | Detection | Availability Loss |
|--------|---------------|-----------|-------------------|
| no | ∞ (No recovery) | N/A | 100% |
| on-failure | 45s | N/A | 7.5% |
| always | 12s | N/A | 2% |
| unless-stopped | 15s | 30s | 2.5% |

**Recommendation: unless-stopped with healthcheck**
```

## Evidence Collection

### Struktur Evidence per Skenario

```
evidence/scenario-{X}/
├── screenshots/
│   ├── grafana-overview-*.png
│   ├── grafana-upstream-*.png
│   └── docker-ps-*.png
├── config-snapshots/
│   ├── nginx-inspect-*.json
│   ├── nginx.conf-*.txt
│   ├── docker-compose-*.yml
│   └── container-status-*.txt
├── raw-data/
│   ├── timeout-test-*.csv
│   └── recovery-time-*.csv
└── MANIFEST.md
```

### MANIFEST.md Template

```markdown
# Evidence Manifest - Scenario {X}

**Generated:** {TIMESTAMP}
**Scenario:** {Scenario Name}
**Duration:** {Duration}

## Test Parameters
- Config: {nginx config variant}
- Restart Policy: {policy}
- Healthcheck: {enabled/disabled}

## Files Generated

### Screenshots
- [ ] Grafana Overview
- [ ] Grafana Upstream Status
- [ ] Docker PS Output

### Config Snapshots
- [ ] Nginx Inspect JSON
- [ ] Nginx Config
- [ ] Docker Compose

### Raw Data
- [ ] Timeout Test CSV
- [ ] Recovery Time CSV

## Observations
{Free-form observations}

## Findings
{Key findings from this scenario}
```

## Timeline

| Skenario | Durasi | Kegiatan |
|----------|--------|----------|
| Setup | 1 hari | Deploy testbed |
| Skenario 1 | 1 hari | Architecture analysis |
| Skenario 2 | 2 hari | Timeout variation tests |
| Skenario 3 | 1 hari | Healthcheck effectiveness |
| Analisis | 3 hari | Data processing & report |

## Checklist Keseluruhan

### Sebelum Pengujian
- [ ] Testbed deployed dan healthy
- [ ] Nginx config baseline documented
- [ ] Docker inspect output saved
- [ ] Evidence directories ready

### Selama Pengujian
- [ ] Screenshot each configuration
- [ ] Capture container status
- [ ] Log recovery times
- [ ] Document observations

### Setelah Pengujian
- [ ] Generate compliance matrix
- [ ] Create recommendation document
- [ ] Backup evidence
- [ ] Write analysis report