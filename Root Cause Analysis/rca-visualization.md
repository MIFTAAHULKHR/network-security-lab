# 🔍 Root Cause Analysis (RCA) — Core Challenges

Dokumen ini mencatat masalah arsitektural dan operasional utama yang dipecahkan selama pembangunan lab simulasi MITM (*Man-in-the-Middle*).

---

## 1. Visualisasi Solusi Jaringan (Host vs Macvlan)

Di bawah ini adalah diagram arsitektur yang menunjukkan mengapa *spoofer* yang dijalankan di *Host* gagal, dan mengapa Docker Macvlan berhasil menjadi solusi.

```mermaid
graph TD
    subgraph HOST["❌ Setup Lama — Kali Host"]
        VBOX["vboxnet0\nMAC: 0a:00:27:00:00:00"]
    end

    subgraph MACVLAN["✅ Setup Baru — Docker Macvlan Network 192.168.56.0/24"]
        Attacker["🔴 bettercap Container\nIP: .56.10 | MAC: unik"]
        DVWA["🎯 DVWA\nIP: .56.20"]
        WG["🎯 WebGoat\nIP: .56.21"]
        JS["🎯 Juice Shop\nIP: .56.22"]
    end

    Victim["💻 Ubuntu VM\nIP: 192.168.56.101"]

    Victim -. "ARP poison diterima\ntraffic dikirim ke host\npacket di-DROP oleh kernel" .-> VBOX
    VBOX -. "same-interface routing\nproblem — tidak bisa forward" .-> Victim

    Victim == "ARP poison berhasil\ntraffic lewat attacker" ==> Attacker
    Attacker == "credential ter-capture\ntraffic di-forward" ==> DVWA
    Attacker == "forward" ==> WG
    Attacker == "forward" ==> JS
```

---

## 2. Alur Kegagalan — Kenapa Host Tidak Bisa Jadi MITM

```mermaid
sequenceDiagram
    participant V as Ubuntu VM (.101)
    participant K as Kali Host (vboxnet0)
    participant M as Metasploitable (.102)

    Note over K: bettercap aktif di host
    K->>V: ARP Reply — ".102 ada di MAC 0a:00:27"
    Note over V: ARP cache diupdate
    V->>K: traffic ke .102 dikirim ke MAC host
    Note over K: ❌ packet masuk & keluar<br/>dari interface yang sama<br/>kernel DROP
    Note over M: tidak pernah menerima traffic
    Note over V: internet mati — traffic hilang
```

---

## 3. Alur Sukses — Docker Macvlan MITM

```mermaid
sequenceDiagram
    participant V as Ubuntu VM (.101)
    participant A as bettercap Container (.10)
    participant D as DVWA Container (.20)

    Note over A: arp.spoof.fullduplex true<br/>arp.spoof.internal true
    A->>V: ARP Reply — ".20 ada di MAC container"
    A->>D: ARP Reply — ".101 ada di MAC container"
    Note over V: ARP cache: .20 → MAC attacker
    Note over D: ARP cache: .101 → MAC attacker

    V->>A: POST /login.php (username + password)
    Note over A: ✅ net.sniff capture<br/>username=admin&password=...
    A->>D: forward POST request
    D->>A: HTTP 302 redirect
    A->>V: forward response
```

---

## 4. Root Cause Map — Semua 9 Obstacle

```mermaid
graph LR
    START([🚀 Mulai Lab]) --> RC1

    RC1["RC-01\nTraffic tidak ter-capture\nDocker bridge ≠ vboxnet0"]
    RC2["RC-02\nCould not detect gateway\nHost-Only = isolated"]
    RC3["RC-03\nTarget IP salah subnet\n192.168.1.x bukan 56.x"]
    RC4["RC-04\nARP table tidak berubah\nbettercap di host"]
    RC5["RC-05\nInternet Ubuntu mati\nMAC 0a:00:27 = host interface"]
    RC6["RC-06\nHanya response\ntanpa credentials"]
    RC7["RC-07\n302 → login.php\nDatabase belum init"]
    RC8["RC-08\nLog entry duplikat\nfullduplex = 2 arah"]
    RC9["RC-09\nMAC mismatch\nDVWA container MAC berubah"]

    FIX1["✅ Switch ke macvlan"]
    FIX2["✅ Non-blocking\ngunakan internal mode"]
    FIX3["✅ Set target 192.168.56.x"]
    FIX4["✅ Pindah ke Docker container"]
    FIX5["✅ macvlan = MAC unik"]
    FIX6["✅ net.sniff.verbose true"]
    FIX7["✅ Jalankan setup.php"]
    FIX8["✅ Expected behavior"]
    FIX9["✅ Verify MAC via docker inspect"]

    SUCCESS(["🎯 Credentials Ter-capture"])

    RC1 --> FIX1
    RC2 --> FIX2
    RC3 --> FIX3
    RC4 --> FIX4
    RC5 --> FIX5
    RC6 --> FIX6
    RC7 --> FIX7
    RC8 --> FIX8
    RC9 --> FIX9

    START --> RC2
    START --> RC3
    FIX1 --> RC4
    FIX4 --> RC5
    FIX5 --> RC6
    FIX5 --> RC7
    FIX6 --> RC8
    FIX7 --> RC9

    FIX1 & FIX2 & FIX3 & FIX4 & FIX5 & FIX6 & FIX7 & FIX8 & FIX9 --> SUCCESS
```

---

## 5. Timeline Perubahan Arsitektur

```mermaid
timeline
    title Evolusi Arsitektur Lab MITM
    section Attempt 1
        bettercap di Kali host : Target IP salah subnet (192.168.1.x)
                               : bettercap di vboxnet0 langsung
                               : Hanya mDNS yang ter-capture
    section Attempt 2
        Perbaikan target IP : Set target 192.168.56.101 dan .102
                            : ARP spoof aktif tapi MAC vboxnet0 = host MAC
                            : Internet Ubuntu mati — traffic di-drop kernel
    section Attempt 3
        Docker macvlan : Container bettercap di macvlan network
                       : MAC unik per container
                       : ARP poison berhasil — HTTP response ter-capture
    section Attempt 4
        Konfigurasi net.sniff : verbose true + filter port 80
                              : Setup DVWA database via setup.php
                              : Verifikasi MAC container DVWA
    section Sukses
        Credentials ter-capture : POST /login.php ter-intercept
                                : username dan password terlihat di bettercap
                                : MITM lab Phase 1 selesai
```

---

## 6. Perbandingan Network Driver

```mermaid
graph TD
    subgraph BRIDGE["Docker Bridge — ❌ Gagal"]
        B_NET["172.17.0.x\nNAT sebelum keluar"]
        B_NOTE["Traffic di-NAT\nbettercap tidak bisa ARP spoof\nke subnet 192.168.56.x"]
    end

    subgraph HOST_NET["Docker Host Network — ❌ Gagal"]
        H_NET["Pakai MAC vboxnet0\n0a:00:27:00:00:00"]
        H_NOTE["Same-interface routing problem\nPacket di-drop kernel\nSama dengan masalah Attempt 1"]
    end

    subgraph MACVLAN_NET["Docker Macvlan — ✅ Berhasil"]
        M_NET["192.168.56.x langsung\nMAC unik tiap container"]
        M_NOTE["Container = participant nyata\ndi dalam jaringan\nARP spoof bekerja normal"]
    end

    BRIDGE --- HOST_NET --- MACVLAN_NET
```

---

## 7. Decision Tree — Kapan ARP Spoof Berhasil

```mermaid
graph TD
    Q1{Apakah attacker\npunya MAC unik\ndi subnet target?}
    Q2{Apakah attacker\nbisa forward\npacket?}
    Q3{Apakah victim\nbrowse ke target\nyang reachable?}
    Q4{Apakah target\npakai HTTP\nbukan HTTPS?}

    FAIL1["❌ Gagal\nARP poison tidak efektif\ntraffic tidak dialihkan"]
    FAIL2["❌ Gagal\nTraffic masuk attacker\ntapi di-drop, tidak di-forward\ninternet victim mati"]
    FAIL3["❌ Gagal\nTidak ada traffic\nuntuk di-capture"]
    FAIL4["⚠️ Partial\nTraffic ter-intercept\ntapi terenkripsi\nperlu SSL strip"]
    SUCCESS2["✅ Sukses\nCredentials ter-capture\ndalam plaintext"]

    Q1 -->|Tidak| FAIL1
    Q1 -->|Ya| Q2
    Q2 -->|Tidak| FAIL2
    Q2 -->|Ya| Q3
    Q3 -->|Tidak| FAIL3
    Q3 -->|Ya| Q4
    Q4 -->|HTTPS| FAIL4
    Q4 -->|HTTP| SUCCESS2
```
