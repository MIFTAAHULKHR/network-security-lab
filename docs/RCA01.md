# 🔍 Root Cause Analysis (RCA) — Core Challenges

Dokumen ini mencatat masalah arsitektural dan operasional utama yang dipecahkan selama pembangunan lab simulasi MITM (*Man-in-the-Middle*).

## Visualisasi Solusi Jaringan (Host vs Macvlan)

Di bawah ini adalah diagram arsitektur yang menunjukkan mengapa *spoofer* yang dijalankan di *Host* gagal, dan mengapa Docker Macvlan berhasil menjadi solusi.


```mermaid
sequenceDiagram
    autonumber
    
    box rgb(33, 33, 33) Korban & Target Asli
    participant U as Ubuntu VM<br/>(IP: .101 | MAC: Asli A)
    participant D as DVWA Server<br/>(IP: .20 | MAC: Asli B)
    end
    
    box rgb(80, 20, 20) Penyerang
    participant H as Bettercap<br/>(IP: .10 | MAC: Hacker C)
    end

    rect rgb(30, 40, 50)
    Note over U,D: FASE 1: Network Discovery (net.probe on)
    H->>U: ARP Request: Siapa yang punya IP .101?
    H->>D: ARP Request: Siapa yang punya IP .20?
    U-->>H: ARP Reply: Saya .101 di MAC Asli A
    D-->>H: ARP Reply: Saya .20 di MAC Asli B
    Note over H: Hacker mencatat MAC Address korban
    end

    rect rgb(50, 30, 30)
    Note over U,D: FASE 2: ARP Spoofing (arp.spoof on)
    H->>U: Spoofed ARP: Hai Ubuntu, IP .20 sekarang ada di MAC Hacker C!
    Note left of U: Cache ARP Teracuni:<br/>DVWA = MAC Hacker C
    H->>D: Spoofed ARP: Hai DVWA, IP .101 sekarang ada di MAC Hacker C!
    Note right of D: Cache ARP Teracuni:<br/>Ubuntu = MAC Hacker C
    Note over U,D: Posisi Hacker sekarang berada di tengah (Man-in-the-Middle)
    end

    rect rgb(30, 50, 30)
    Note over U,D: FASE 3: Credential Sniffing (net.sniff on)
    U->>H: HTTP POST /login.php [user=admin, pass=rahasia123]
    Note over H: Kredensial terbaca dalam<br/>plaintext oleh net.sniff!
    H->>D: Forward (Meneruskan) HTTP POST ke Server Asli
    D->>H: HTTP 302 Found / 200 OK (Login Sukses)
    H->>U: Forward HTTP Response ke Ubuntu
    Note left of U: Korban berhasil login<br/>dan tidak menyadari penyadapan
    end
