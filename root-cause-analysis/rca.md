# Root Cause Analysis Log

Dokumentasi lengkap setiap obstacle yang ditemukan selama membangun lab MITM, beserta analisis penyebab dan solusinya. Ditulis sebagai referensi belajar, bukan sekadar troubleshooting log.

---

## RCA-01 — bettercap tidak capture traffic apapun (sesi pertama)

**Gejala:** bettercap berjalan di Kali host via `vboxnet0`, ARP spoof aktif, tapi `net.sniff` tidak menampilkan HTTP traffic sama sekali. Hanya mDNS query yang muncul.

**Analisis:**
Docker container (DVWA, WebGoat, Juice Shop) berjalan di Docker bridge network `172.17.0.x`. bettercap memonitor `vboxnet0` di subnet `192.168.56.x`. Dua network ini sepenuhnya terisolir — traffic DVWA tidak pernah melewati `vboxnet0`, sehingga bettercap tidak punya apa-apa untuk di-capture.

mDNS muncul karena itu adalah broadcast traffic yang dikirim otomatis oleh Ubuntu VM ke seluruh subnet, bukan karena MITM berhasil.

**Root cause:** Arsitektur network salah — target berada di network yang berbeda dengan interface yang dimonitor.

**Fix:** Pindahkan container target ke macvlan network di subnet `192.168.56.x`, sehingga traffic target melewati `vboxnet0`.

---

## RCA-02 — Warning "Could not detect gateway"

**Gejala:** Setiap kali bettercap dijalankan dengan `-iface vboxnet0`, muncul warning `[war] Could not detect gateway`.

**Analisis:**
Host-Only network di VirtualBox adalah isolated network by design — tidak ada default gateway karena tidak ada koneksi ke luar. bettercap mendeteksi ini dan memperingatkan bahwa ARP spoof dalam mode gateway (victim ↔ internet) tidak bisa dilakukan.

**Root cause:** Bukan bug. Ini perilaku yang benar untuk Host-Only network.

**Dampak:** Non-blocking untuk skenario MITM antar-VM. ARP spoof antar-host dalam satu subnet tetap bisa berjalan tanpa gateway.

**Fix:** Tidak perlu difix. Gunakan `set arp.spoof.internal true` untuk spoof antar-host tanpa gateway.

---

## RCA-03 — Target IP di script awal salah subnet

**Gejala:** Script menggunakan `set arp.spoof.targets 192.168.1.5, 192.168.1.1` tapi lab berjalan di `192.168.56.x`.

**Root cause:** Copy-paste dari contoh tutorial luar yang menggunakan subnet berbeda.

**Fix:** Sesuaikan dengan subnet lab: `set arp.spoof.targets 192.168.56.101,192.168.56.102`.

---

## RCA-04 — ARP table Ubuntu tidak berubah meski bettercap aktif

**Gejala:** Setelah `arp.spoof on`, cek `arp -n` di Ubuntu menunjukkan MAC asli Metasploitable, bukan MAC Kali.

**Analisis:**
bettercap berjalan di Kali host dan menggunakan MAC `vboxnet0` (`0a:00:27:00:00:00`) sebagai source ARP reply. Packet ini dikirim dari host ke VM melalui virtual switch VirtualBox. VirtualBox Host-Only adapter memproses packet ini di level hypervisor, dan karena source-nya adalah host interface itu sendiri, packet tidak selalu ter-deliver ke VM dengan cara yang mengupdate ARP cache secara efektif.

**Root cause:** Kali adalah host fisik yang mengelola vboxnet0 — bukan participant jaringan yang setara dengan VM. Posisi topologinya di atas jaringan, bukan di dalam jaringan.

**Fix:** Pindahkan bettercap ke dalam container dengan macvlan network, sehingga ia punya posisi topologi yang benar: di dalam jaringan bersama VM-VM lain.

---

## RCA-05 — MAC `0a:00:27:00:00:00` membuat internet Ubuntu mati

**Gejala:** Saat bettercap aktif, ARP cache Ubuntu berubah — tapi internet Ubuntu mati. Saat bettercap dihentikan, MAC kembali ke asli dan internet pulih.

**Analisis:**
ARP poison berhasil mengubah cache Ubuntu: entry `.102` menunjuk ke MAC `0a:00:27:00:00:00` (MAC vboxnet0 Kali host). Ubuntu kemudian mengirim traffic ke Kali. Tapi Kali menerima packet ini di `vboxnet0`, dan karena traffic masuk dan keluar dari interface yang sama, kernel Linux menolak forwarding (same-interface routing problem). Packet di-drop.

Ketika bettercap berhenti, Metasploitable mengirim gratuitous ARP reply aslinya, menimpa cache Ubuntu, dan koneksi pulih.

**Root cause:** Host tidak bisa berfungsi sebagai MITM relay ketika traffic masuk dan keluar dari interface yang sama.

**Fix:** macvlan container — container bettercap punya MAC unik sendiri, traffic dari Ubuntu masuk ke container via satu path, keluar ke Metasploitable via path berbeda. Kernel bisa forward dengan benar.

---

## RCA-06 — Hanya HTTP response yang ter-capture, credentials tidak muncul

**Gejala:** bettercap menampilkan `net.sniff.http.response` tapi tidak ada `net.sniff.http.request`. Credentials tidak terlihat.

**Analisis:**
Username dan password dikirim dalam HTTP POST request body dari Ubuntu ke DVWA. bettercap tanpa `net.sniff.verbose true` hanya menampilkan summary response (status code, ukuran, content-type). POST body tidak ditampilkan.

**Root cause:** `net.sniff.verbose` tidak diaktifkan.

**Fix:**
```
set net.sniff.verbose true
set net.sniff.filter port 80
net.sniff on
```

---

## RCA-07 — Login DVWA selalu gagal (302 → login.php)

**Gejala:** Setiap percobaan login DVWA dari Ubuntu menghasilkan `302 Found` dengan `Location: login.php` — redirect kembali ke form login, bukan ke dashboard.

**Analisis:**
`302 → index.php` = login berhasil. `302 → login.php` = login gagal. DVWA membutuhkan database MySQL yang sudah diinisialisasi dengan tabel `users`. Tanpa inisialisasi, tidak ada user yang bisa divalidasi sehingga semua login ditolak.

**Root cause:** Setup database DVWA belum dijalankan.

**Fix:** Buka `http://192.168.56.20/setup.php` dari Ubuntu, klik "Create / Reset Database", kemudian login dengan `admin` / `password`.

---

## RCA-08 — Setiap log entry muncul dua kali

**Gejala:** Semua output `net.sniff` tampil duplikat — setiap baris muncul persis dua kali berurutan.

**Analisis:**
`set arp.spoof.fullduplex true` memposisikan bettercap sebagai MITM di dua arah: Ubuntu→DVWA dan DVWA→Ubuntu. Setiap packet yang melintas di-capture dua kali — sekali dari perspektif tiap arah.

**Root cause:** Perilaku expected dari `fullduplex` mode.

**Dampak:** Tidak mempengaruhi hasil. Credentials tetap ter-capture, hanya tampil dua kali.

---

## RCA-09 — Serangan final gagal karena MAC mismatch pada DVWA container

**Gejala:** Setup sudah benar, bettercap di macvlan, tapi credentials masih tidak ter-capture dari DVWA.

**Analisis:**
MAC address yang dikonfigurasi pada target DVWA di ARP spoof config tidak cocok dengan MAC aktual container DVWA di macvlan network. bettercap mengirim ARP poison menggunakan MAC yang salah, sehingga victim tidak ter-redirect ke attacker.

**Root cause:** MAC address container berubah setiap kali container di-recreate (kecuali di-pin secara eksplisit). Konfigurasi ARP spoof menggunakan MAC lama.

**Fix:** Verifikasi MAC aktual tiap container sebelum serangan:
```bash
docker inspect dvwa | grep MacAddress
```
Kemudian sesuaikan konfigurasi, atau gunakan IP-based targeting di bettercap (bettercap akan resolve MAC sendiri via ARP).

**Status:** ✅ Resolved — serangan berhasil setelah MAC disesuaikan.
