# Pra-Praktikum Modul 2
## HTTP Server, DNS Server, dan Reverse Proxy

**Pre-requisite:**
a. GNS3 VM dan GNS3-GUI sudah terinstall.
b. Appliance `netics-pc` / `netics-server` dan `netics-pc-desktop`.

**Cakupan Modul:** HTTP Server (nginx), DNS Server (BIND), Integrasi DNS dengan DHCP, Reverse Proxy Berbasis Path, Reverse Proxy Multi-Backend Lintas Port.

> **Note:** Semua problem dan bagian wajib dikerjakan sebagai bahan persiapan untuk praktikum hari H, pengerjaan laporan mengikuti instruksi yang ada di readme github. IP Prefix disesuaikan dengan pembagian yang sudah diumumkan sebelumnya.

---

### Pemetaan Node

| Node | Appliance | Peran | IP Address |
| :--- | :--- | :--- | :--- |
| **Router** | `netics-pc` | Menyediakan akses internet dari NAT | `eth0`: Auto (DHCP)<br>`eth1`: `[IP Prefix].200.1/24` (statis) |
| **Sargon, Iberia, Higashi** | `netics-pc` | Client uji command line (`curl`) | Auto (DHCP) |
| **Ognisko** | `netics-pc-desktop` | Client uji browser (VNC + Chromium) | Auto (DHCP) |
| **Aegir** | `netics-pc` | DHCP Server | `[IP Prefix].200.10/24` (statis)<br>Gateway: `[IP Prefix].200.1`<br>Pool DHCP: `[IP Prefix].200.100-150`<br>*Tambahkan di konfigurasi dhcpd nya:*<br>`option router [IP Prefix].200.1` |
| **Kazdel** | `netics-pc` atau `netics-server` | Web Server (Bagian A) -> Reverse Proxy (Bagian C) | `[IP Prefix].200.11/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| **Chernobog** | `netics-pc` atau `netics-server` | DNS Server (BIND) | `[IP Prefix].200.12/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| **web1** | `netics-pc` atau `netics-server` | Web1, backend HTTP server | `[IP Prefix].200.20/24` (statis), port 8080 (statis)<br>Gateway: `[IP Prefix].200.1` |
| **web2** | `netics-pc` atau `netics-server` | Web2, backend HTTP server + aplikasi Python (Flask) | `[IP Prefix].200.21/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |

---

### 0. Pre

1. Konfigurasi Router sesuai dengan tabel pemetaan node diatas, kemudian hubungkan `eth0` router ke NAT.
2. Masuk ke console node Router, seharusnya secara otomatis akan mendapat IP untuk interface `eth0` seperti gambar dibawah.
   ```text
   Trying ::1...
   telnet: connect to address ::1: Connection refused
   Trying 127.0.0.1...
   Connected to localhost.
   Escape character is '^]'.
   router console is now available... Press RETURN to get started.
   udhcpc: started, v1.36.1
   udhcpc: broadcasting discover
   udhcpc: broadcasting discover
   udhcpc: broadcasting select for 192.168.124.189, server 192.168.124.1
   udhcpc: lease of 192.168.124.189 obtained from 192.168.124.1, lease time 3600
   
   AlpiNet - Lightweight Alpine-based Networking Toolbox
   NETICS-PC for praktikum JARKOM
   Type "alpinet-tools" for available utilities
   ```
3. Kemudian jalankan perintah berikut:
   ```bash
   $ apk update
   $ apk add iptables
   $ iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0 -s [IP Prefix].0.0/16
   ```
4. Pastikan kalian bisa ping `google.com` dari node router.
5. Di node DHCP Server (Aegir), jalankan:
   ```bash
   $ echo "nameserver 8.8.8.8" > /etc/resolv.conf
   ```
   Kemudian coba ping `google.com` dari node tersebut.
6. Konfigurasikan DHCP Server seperti pada modul 1 sebelumnya dan sesuai dengan ketentuan pool yang ada dalam tabel dengan menambahkan konfigurasi pada `dhcpd.conf` nya.
   ```text
   option router [IP Prefix].200.1
   ```

---

## A. Bagian A - HTTP Server

### a. Tujuan Pembelajaran
1. Memahami HTTP server sebagai salah satu bentuk konkret aplikasi jaringan (network application) di Application Layer.
2. Memasang dan menjalankan web server nginx untuk keperluan information sharing sederhana.

### b. Tools yang Dibutuhkan
1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. Nginx.
3. `curl` (di Sargon) dan browser Chromium (di Ognisko).

**Skenario Topologi**
Topologi yang dipakai pada bagian ini terdiri dari satu ethernet switch yang menghubungkan:
a. Sargon, Iberia, Higashi, dan Ognisko (semuanya ip config: auto, mendapat IP dari DHCP server).
b. Aegir sebagai DHCP server (`[IP Prefix].200.10/24`, pool `[IP Prefix].200.100-150`).
c. Kazdel sebagai web/HTTP server dengan IP statis `[IP Prefix].200.11/24`.

Dan satu node router (`netics-pc`) yang berfungsi sebagai penyedia internet melalui NAT.
Web server sengaja diberi IP statis (bukan DHCP) karena alamatnya perlu tetap dan dapat diprediksi, baik untuk diakses langsung oleh client, maupun untuk nantinya dipetakan oleh DNS server pada Bagian B.

*Gambar A.1, Topologi Bagian A: client DHCP, DHCP server (Aegir), dan web/HTTP server statis (Kazdel)*

### c. Problem A.1 - Setup HTTP Server

**Soal:**
Lakukan setup HTTP server/web server menggunakan nginx pada Kazdel dengan IP `[IP Prefix].200.11/24`.

Konvensi direktori yang dipakai sepanjang praktikum ini:

| Direktori | Fungsi |
| :--- | :--- |
| `~/root/myconfig~` | Lokasi file konfigurasi (`nginx.conf`, start script) |
| `~/root/myweb~` | Lokasi file web (`index.html`) |
| `~/root/mylogs~` | Lokasi pencatatan log |

**Langkah Kerja:**
1. Setup IP address Kazdel ke `[IP Prefix].200.11/24`, dilakukan lewat edit konfigurasi network interface pada node (bukan `ip addr add` sementara), agar IP tetap statis setiap kali node dinyalakan ulang. Pada panel Kazdel interfaces, uncomment dan sesuaikan bagian static config:
   ```text
   # Static config for eth0
   auto eth0
   iface eth0 inet static
   address [IP Prefix].200.11
   netmask 255.255.255.0
   gateway [IP Prefix].200.1
   ```
2. Install nginx menggunakan:
   ```bash
   $ echo "nameserver 8.8.8.8" > /etc/resolv.conf
   $ apk update
   $ apk add nginx
   ```
3. Buat struktur direktori:
   ```bash
   $ mkdir -p /root/myconfig
   $ mkdir -p /root/myweb
   $ mkdir -p /root/mylogs
   ```
4. Buat file `index.html` di `/root/myweb`:
   ```html
   <html>
   <head>
       <title>This is My Web</title>
   </head>
   <body>
       <p>
       This is my web
       </p>
   </body>
   </html>
   ```
5. Buat file `nginx.conf` di `/root/myconfig`:
   ```nginx
   user root;
   worker_processes auto;
   worker_cpu_affinity auto;
   pid /tmp/nginx.pid;
   error_log /root/mylogs/error.log;
   
   events { worker_connections 768; }
   
   http {
       server {
           listen 80;
           server_name _;
           root /root/myweb;
           index index.html;
           
           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```
6. Buat file `start_http.sh` di `/root/myconfig`:
   ```bash
   #!/bin/bash
   nginx -c /root/myconfig/nginx.conf -g 'daemon off;'
   ```
7. Jalankan web server dengan perintah:
   ```bash
   $ nginx -c /root/myconfig/nginx.conf
   ```
8. Tambahkan baris berikut ini di `/root/init.sh` agar menjalankan script diatas secara otomatis saat node dinyalakan:
   ```bash
   #!/bin/bash
   nginx -c /root/myconfig/nginx.conf
   ```

**Indikator Keberhasilan:**
1. **Verifikasi lewat browser:**
   a. Buka console pada Ognisko menggunakan VNC viewer.
   b. Klik kanan pada layar desktop untuk memunculkan menu, lalu pilih Desktops -> chromium.
   c. Pada Chromium, ketik alamat URL `http://[IP Prefix].200.11`.
   d. Akan terlihat tampilan web yang bertuliskan "This is my web".
   
   *Gambar A.3, Console Ognisko via VNC: memilih chromium, lalu mengakses http://[IP Prefix].200.11*

2. **Verifikasi lewat command line:**
   a. Dari Sargon, gunakan mode Console Command Line Interface, lalu jalankan:
      ```bash
      $ curl -vL http://[IP Prefix].200.11
      ```
      *Gambar A.4, Hasil curl -vL http://[IP Prefix].200.11 dari Sargon*

   b. Contoh hasil yang diharapkan (rekonstruksi teks dari output di atas):
      ```http
      * Trying [IP Prefix].200.11:80...
      * Connected to [IP Prefix].200.11 ([IP Prefix].200.11) port 80
      * using HTTP/1.x
      > GET / HTTP/1.1
      > Host: [IP Prefix].200.11
      > User-Agent: curl/8.14.1
      > Accept: */*
      * Request completely sent off
      < HTTP/1.1 200 OK
      < Server: nginx/1.26.3
      < Date: Sat, 27 Sep 2025 13:31:41 GMT
      < Content-Type: text/html
      < Content-Length: 111
      < Last-Modified: Sat, 27 Sep 2025 07:46:19 GMT
      < Connection: keep-alive
      < ETag: "68d7964b-6f"
      < Accept-Ranges: bytes
      <
      <html>
      <head>
          <title>This is My Web</title>
      </head>
      <body>
          <p>
          This is my web
          </p>
      </body>
      </html>
      * Connection #30 to host [IP Prefix].200.11 left intact
      ```

---

## B. Bagian B - DNS Server

### a. Tujuan Pembelajaran
1. Memahami DNS sebagai mekanisme mapping nama domain ke IP address.
2. Memasang DNS server (BIND) dan mengintegrasikannya dengan DHCP agar seluruh client otomatis mengetahui alamat DNS server.

### b. Tools yang Dibutuhkan
1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. BIND (`named`).
3. DHCP server yang sudah aktif (untuk diintegrasikan dengan opsi DNS).

### c. Problem B.1 - Setup DNS Server

**Soal:**
Pasang DNS server pada Chernobog (IP `[IP Prefix].200.12/24`) untuk domain `netics.my.id`, dan integrasikan dengan DHCP server agar seluruh client otomatis mengetahui alamat DNS server ini.

**Langkah Kerja:**
1. Setup IP address Chernobog ke `[IP Prefix].200.12/24` dan set gateway address nya ke `[IP Prefix].200.1`.
2. Buat struktur direktori:
   ```bash
   $ mkdir -p /root/dns
   $ mkdir -p /root/dnsdata
   ```
3. Install bind menggunakan:
   ```bash
   $ echo "nameserver 8.8.8.8" > /etc/resolv.conf
   $ apk update
   $ apk add bind
   ```
4. Buat file `named.conf` di `/root/dns`:
   ```text
   options {
       directory "/root/dns";
       listen-on { any; };
       allow-query { any; };
   };
   
   logging {
       channel default_log {
           file "/root/dns/named.log" versions 3 size 5m;
           severity info;
           print-time yes;
           print-severity yes;
           print-category yes;
       };
       category default { default_log; };
       category queries { default_log; };
   };
   
   zone "localhost" {
       type master;
       file "db.localhost";
   };
   
   zone "netics.my.id" {
       type master;
       file "db.netics.my.id";
   };
   ```
5. Buat file `db.localhost`:
   ```text
   $TTL 86400
   @ IN SOA localhost. root.localhost. (
           1       ; serial
           3600    ; refresh
           1800    ; retry
           604800  ; expire
           86400 ) ; minimum
   
       IN NS localhost.
       IN A  127.0.0.1
   ```
6. Buat file `db.netics.my.id`, memetakan `www.netics.my.id` ke IP web server (Kazdel):
   ```text
   $TTL 86400
   @ IN SOA ns1.netics.my.id. admin.netics.my.id. (
           1       ; serial
           3600    ; refresh
           1800    ; retry
           604800  ; expire
           86400 ) ; minimum
   
       IN NS ns1.netics.my.id.
   ns1 IN A  [IP Prefix].200.12
   www IN A  [IP Prefix].200.11
   ```
7. Buat file `start_dns.sh` di `/root/dns` dan jalankan dns servernya:
   ```bash
   #!/bin/sh
   named -g -c /root/dns/named.conf
   ```
   Untuk melihat progress/aktivitas dari DNS Server, pantaulah di file `/root/dns/named.log` (gunakan tmux), dengan perintah:
   ```bash
   $ tail -f /root/dns/named.log
   ```
8. Tambahkan file bash di `/root/init.sh` untuk Chernobog agar DHCP otomatis dijalankan saat node di start/restart:
   ```bash
   #!/bin/bash
   chmod +x /root/dns/start_dns.sh
   /root/dns/start_dns.sh
   ```
9. Buka Console pada node Chernobog (klik kanan node -> Console) untuk memantau proses named berjalan.
   
   *Gambar B.1, Membuka Console Chernobog dan log named saat pertama kali dijalankan*

**Langkah Kerja, Integrasi dengan DHCP:**
Agar seluruh client otomatis mengetahui alamat DNS server tanpa perlu diset manual satu per satu:
1. Edit file `dhcpd.conf` pada DHCP server yang sudah ada, tambahkan baris:
   ```text
   option dns [IP Prefix].200.12 8.8.8.8
   ```
2. Reload kembali DHCP server, lalu restart seluruh client agar setting DNS otomatis ter-set.
3. Verifikasi isi `dhcpd.conf` di Aegir setelah diedit.
   
   *Gambar B.2, Isi dhcpd.conf di Aegir (dhcp-server) setelah ditambahkan baris option dns*

**Indikator Keberhasilan:**
1. Di Sargon, cek `/etc/resolv.conf`:
   ```bash
   $ cat /etc/resolv.conf
   ```
   Pastikan terdapat baris domain dan nameserver yang mengarah ke DNS server (`[IP Prefix].200.12`)
2. Lakukan ping dan curl memakai nama domain (bukan IP):
   ```bash
   $ ping www.netics.my.id
   $ curl -vL http://www.netics.my.id
   ```
3. Dari Ognisko, akses `http://www.netics.my.id` lewat Firefox.
4. Ulangi pengecekan dari netics-pc lain untuk memastikan konsisten di seluruh jaringan.

   *Gambar B.3, Verifikasi lengkap dari Sargon: /etc/resolv.conf, ping www.netics.my.id, curl -vL, dan akses via browser*

### d. Problem B.2 - Multi-Zone DNS dan Multi-Domain Web Server

**Soal:**
Perluas DNS server agar melayani dua zona sekaligus (`netics.my.id` dan `netics.org`), dan perluas web server agar melayani nama domain berbeda dengan lokasi index yang relevan: `http://www.netics.my.id/` dan `http://web.netics.org`.

*Gambar B.4, Topologi Bagian B setelah DNS server ditambahkan: Kazdel (web, dua domain), Chernobog (DNS, dua zona), Aegir (DHCP)*

**Langkah Kerja, DNS Server (Chernobog):**
1. Tambahkan definisi zona baru di `named.conf`:
   ```text
   zone "netics.org" {
       type master;
       file "db.netics.org";
   };
   ```
2. Buat file `db.netics.org`, dengan record web mengarah ke IP web server:
   ```text
   $TTL 86400
   @ IN SOA ns1.netics.org. admin.netics.org. (
           1       ; serial
           3600    ; refresh
           1800    ; retry
           604800  ; expire
           86400 ) ; minimum
   
       IN NS ns1.netics.org.
   ns1 IN A  [IP Prefix].200.12
   web IN A  [IP Prefix].200.11
   ```
3. Restart node Chernobog (stop lalu start kembali) agar konfigurasi baru terbaca.
4. Sebagai indikator keberhasilan, cek dari Sargon dan yang lain dengan ping ke `ns1.netics.org`.
   ```bash
   # ping ns1.netics.org
   ```

**Langkah Kerja, Web Server (Kazdel):**
1. Buat direktori baru untuk konten `web.netics.org`:
   ```bash
   $ mkdir -p /root/myweb-netics-org
   ```
2. Buat file `index.html` di direktori tersebut:
   ```html
   <html>
   <head>
       <title>Hello</title>
   </head>
   <body>
       <p>
       This is NETICS-ORG web
       </p>
   </body>
   </html>
   ```
3. Ubah blok http pada `nginx.conf` agar melayani dua nama domain sekaligus dari root direktori yang berbeda:
   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;
           root /root/myweb;
           index index.html;
           
           location / {
               try_files $uri $uri/ =404;
           }
       }
       
       server {
           listen 80;
           server_name web.netics.org;
           root /root/myweb-netics-org;
           index index.html;
           
           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```
4. Restart node Kazdel (stop lalu start kembali).

**Indikator Keberhasilan:**
1. Dari Sargon dan netics-pc lain: ping `web.netics.org` dan ping `www.netics.my.id`, keduanya berhasil resolve.
   
   *Gambar B.5, Ping web.netics.org dan www.netics.my.id dari Sargon, keduanya resolve ke [IP Prefix].200.11*

2. Dari Iberia (atau client lain):
   ```bash
   $ curl -vL http://www.netics.my.id
   $ curl -vL http://web.netics.org
   ```
   *Gambar B.6, Hasil curl -vL untuk www.netics.my.id dan web.netics.org dari Iberia, masing-masing dengan konten dan ETag berbeda*

3. Dari Ognisko, cek kedua alamat lewat Firefox.
   
   *Gambar B.7, Kedua domain diakses lewat Chromium di Ognisko: netics.my.id ("This is my web") dan web.netics.org ("This is NETICS-ORG web")*

---

## C. Bagian C - Reverse Proxy

### a. Tujuan Pembelajaran
1. Memahami pemisahan fungsi antara reverse proxy dan real server pada arsitektur HTTP server.
2. Mengonfigurasi reverse proxy yang meneruskan request berdasarkan path URL ke backend yang berbeda, termasuk backend yang berjalan di port berbeda.

### b. Tools yang Dibutuhkan
1. GNS3 appliance `netics-pc` dan `netics-pc-desktop`.
2. Nginx (reverse proxy dan real server).
3. Python3 dan Flask (untuk backend aplikasi dinamis).

*Gambar C.1, Arsitektur reverse proxy: internet users mengakses www.netics.my.id, diteruskan ke real server (Nginx, Nginx, Python) di belakang layar*

### c. Problem C.1 - Reverse Proxy di Depan Dua Web Server

**Soal:**
Implementasikan reverse proxy di depan dua web server berbeda:
a. melayani base URL `http://www.netics.my.id`, meneruskan `/web1` ke Web1 (`http://[IP Prefix].200.20:8080`).
b. `/web2` ke Web2 (`http://[IP Prefix].200.21:8080`).

*Gambar C.2, Topologi setelah Web1 dan Web2 ditambahkan: reverse proxy (Kazdel) meneruskan /web1 dan /web2 ke masing-masing backend*

**Langkah Kerja, Web1 (web1):**
1. Setup IP address ke `[IP Prefix].200.20/24`
   ```text
   # Static config for eth0
   auto eth0
   iface eth0 inet static
   address [IP Prefix].200.20
   netmask 255.255.255.0
   gateway [IP Prefix].200.1
   ```
   *Gambar C.3, Konfigurasi interface statis Web1 (web1), address [IP Prefix].200.20*

2. Buat struktur direktori (`myconfig`, `myweb`, `mylogs`) seperti Bagian A.
3. Jalankan command yang sama untuk instalasi nginx seperti pada bagian A.
4. Buat `index.html` seperti yang ada pada bagian A tapi berisi teks "Ini WEB-1".
5. Buat `nginx.conf` seperti yang ada pada bagian A, dan modifikasi blok http, kali ini nginx listen di port `8080` (bukan 80):
   ```nginx
   http {
       server {
           listen 8080;
           server_name _;
           root /root/myweb;
           index index.html;
           
           location / {
               try_files $uri $uri/ =404;
           }
       }
   }
   ```
6. Buat `start_http.sh` seperti yang ada di bagian A sebelumnya, dan set start command node.
7. Restart node Web1, verifikasi dari Sargon dan Ognisko:
   ```bash
   $ curl http://[IP Prefix].200.20:8080
   ```
   *Gambar C.4, Verifikasi Web1: curl http://[IP Prefix].200.20:8080 dan akses via browser, menampilkan "Ini WEB-1"*

**Langkah Kerja, Web2 (web2):**
1. Ulangi langkah yang sama persis seperti Web1, dengan perbedaan:
   a. IP address `[IP Prefix].200.21/24`.
   b. Isi `index.html`: "Ini WEB-2".
   
   *Gambar C.5, Konfigurasi interface statis Web2 (web2), address [IP Prefix].200.21*

2. Isi `index.html` di Web2:
   ```html
   <html>
   <head>
       <title>This is My Web</title>
   </head>
   <body>
       <p>
       Ini WEB-2
       </p>
   </body>
   </html>
   ```
3. Isi `nginx.conf` dan `start_http.sh` identik dengan Web1 (hanya root/direktori yang menyesuaikan `/root/myweb` milik Web2 sendiri).
4. Restart node Web2, verifikasi dari Sargon dan Ognisko:
   ```bash
   $ curl http://[IP Prefix].200.21:8080
   ```
   *Gambar C.6, Verifikasi Web2: curl http://[IP Prefix].200.21:8080 dan akses via browser, menampilkan "Ini WEB-2"*

**Langkah Kerja, Reverse Proxy (Kazdel):**
1. IP address tetap `[IP Prefix].200.11/24`, `server_name` tetap `www.netics.my.id`.
   
   *Gambar C.7, Konfirmasi interface Kazdel tetap [IP Prefix].200.11 (tidak berubah dari Bagian A/B)*

2. Ubah `nginx.conf` di `/root/myconfig` menjadi konfigurasi reverse proxy (menggantikan konfigurasi Bagian A/B):
   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;
           
           location /web1 {
               proxy_pass http://[IP Prefix].200.20:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
           
           location /web2 {
               proxy_pass http://[IP Prefix].200.21:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
       }
   }
   ```
3. Restart node Kazdel.

**Indikator Keberhasilan:**
1. Dari Sargon:
   ```bash
   $ curl http://www.netics.my.id/web1
   $ curl http://www.netics.my.id/web2
   ```
   Masing-masing menghasilkan "Ini WEB-1" dan "Ini WEB-2", padahal keduanya diakses dari satu domain dan port (80) yang sama.
2. Dari Ognisko, cek juga kedua path lewat firefox.
   
   *Gambar C.8, Verifikasi reverse proxy: curl -vL untuk /web1 dan /web2 via www.netics.my.id, serta akses via browser*

### d. Problem C.2 - Menambahkan Backend Aplikasi di Port Berbeda

**Soal:**
Terdapat aplikasi kalender berbasis Python yang ditempatkan di Web2, berjalan di port 5000. Perluas reverse proxy agar `/app` diarahkan ke aplikasi ini `http://[IP Prefix].200.21:5000`, selain `/web1` dan `/web2` yang sudah ada.

*Gambar C.9, Topologi setelah aplikasi Python (port 5000) ditambahkan di Web2, dan reverse proxy meneruskan /app ke aplikasi tersebut*

**Langkah Kerja, Aplikasi di Web2 (web2):**
1. Buat direktori baru:
   ```bash
   $ mkdir -p /root/aplikasi
   ```
2. Install python menggunakan:
   ```bash
   $ echo "nameserver 8.8.8.8" > /etc/resolv.conf
   $ apk update
   $ apk add python3
   ```
3. Buat file `program.py` pada direktori `/root/aplikasi`:
   ```python
   import calendar
   from datetime import datetime
   from http.server import BaseHTTPRequestHandler, HTTPServer
   
   PORT = 5000
   
   class CalendarRequestHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           # 1. Get current year and month
           now = datetime.now()
           year = now.year
           month = now.month
           
           # 2. Generate calendar data
           cal = calendar.Calendar(firstweekday=6) # 6 = Sunday start. Change to 0 for Monday.
           month_days = cal.monthdayscalendar(year, month)
           month_name = calendar.month_name[month]
           
           # 3. Build HTML response
           html = f"""<!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>{month_name} {year} Calendar</title>
       <style>
           body {{
               font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
               background-color: #f4f7f6;
               display: flex;
               justify-content: center;
               align-items: center;
               height: 100vh;
               margin: 0;
           }}
           .calendar-container {{
               background: #ffffff;
               padding: 24px;
               border-radius: 12px;
               box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
               width: 100%;
               max-width: 500px;
           }}
           h2 {{
               text-align: center;
               color: #2c3e50;
               margin-top: 0;
               margin-bottom: 20px;
           }}
           table {{
               width: 100%;
               border-collapse: collapse;
           }}
           th {{
               background-color: #3498db;
               color: white;
               font-weight: 600;
               padding: 12px 0;
               width: 14.28%;
               border-radius: 4px;
           }}
           td {{
               text-align: center;
               padding: 16px 0;
               color: #333;
               font-size: 16px;
               font-weight: 500;
           }}
           .today {{
               background-color: #e8f4fd;
               color: #3498db;
               border-radius: 50%;
               font-weight: bold;
           }}
           .empty {{
               color: #ccc;
           }}
       </style>
   </head>
   <body>
       <div class="calendar-container">
           <h2>{month_name} {year}</h2>
           <table>
               <thead>
                   <tr>
                       <th>Sun</th>
                       <th>Mon</th>
                       <th>Tue</th>
                       <th>Wed</th>
                       <th>Thu</th>
                       <th>Fri</th>
                       <th>Sat</th>
                   </tr>
               </thead>
               <tbody>
   """
           # 4. Construct table rows from calendar data
           for week in month_days:
               html += "<tr>"
               for day in week:
                   if day == 0:
                       html += '<td class="empty">&bull;</td>'
                   elif day == now.day:
                       html += f'<td class="today">{day}</td>'
                   else:
                       html += f"<td>{day}</td>"
               html += "</tr>"
               
           html += """
               </tbody>
           </table>
       </div>
   </body>
   </html>
   """
           # 5. Send HTTP Response headers and content
           self.send_response(200)
           self.send_header("Content-type", "text/html")
           self.end_headers()
           self.wfile.write(html.encode("utf-8"))
   
   def run():
       server_address = ("", PORT)
       httpd = HTTPServer(server_address, CalendarRequestHandler)
       print(f"Server running at http://localhost:{PORT}/")
       try:
           httpd.serve_forever()
       except KeyboardInterrupt:
           print("\\nServer stopped.")
   
   if __name__ == "__main__":
       run()
   ```
4. Jalankan aplikasi:
   ```bash
   $ python3 /root/aplikasi/program.py
   ```
   *Gambar C.10, Aplikasi Flask berjalan di Web2, melayani di http://127.0.0.1:5000*
5. Verifikasi awal dari Ognisko:
   ```bash
   $ curl http://[IP Prefix].200.21:5000
   ```
   *Gambar C.11, Verifikasi langsung curl -vL http://[IP Prefix].200.21:5000, menampilkan HTML kalender dari Flask (Server: Werkzeug/Python)*

**Langkah Kerja, Reverse Proxy (Kazdel):**

*Gambar C.12, Konfirmasi interface Kazdel masih [IP Prefix].200.11 sebelum menambahkan location /app*

1. Tambahkan satu blok `location` baru pada `nginx.conf`:
   ```nginx
   http {
       server {
           listen 80;
           server_name www.netics.my.id;
           
           location /web1 {
               proxy_pass http://[IP Prefix].200.20:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
           
           location /web2 {
               proxy_pass http://[IP Prefix].200.21:8080/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
           
           location /app {
               proxy_pass http://[IP Prefix].200.21:5000/;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
       }
   }
   ```
2. Restart node Kazdel.

**Indikator Keberhasilan:**
1. Dari Sargon:
   ```bash
   $ curl http://www.netics.my.id/web1
   $ curl http://www.netics.my.id/web2
   $ curl http://www.netics.my.id/app
   ```
   `/app` menghasilkan tampilan kalender tahun dari aplikasi Flask.
2. Dari Ognisko, cek ketiga path lewat Chromium.
   
   *Gambar C.14, Verifikasi akhir: curl -vL untuk /web1 dan /app via www.netics.my.id, serta akses ketiga path lewat browser (termasuk tampilan kalender /app)*

---

### e. Verifikasi Menyeluruh

Setelah Problem C.1 dan C.2 selesai, bagian ini menguji seluruh komponen (DNS, HTTP server, reverse proxy) sebagai satu kesatuan, dari berbagai titik di jaringan.

| No. | Item Verifikasi | Perintah / Cara | Dari Node |
| :---: | :--- | :--- | :--- |
| 1. | Client otomatis memiliki alamat DNS server | `cat /etc/resolv.conf` | Sargon |
| 2. | Resolusi nama domain berhasil | `ping www.netics.my.id`<br>`ping web.netics.org` | Sargon, netics-pc lain |
| 3. | Akses langsung ke tiap backend (tanpa proxy) masih berfungsi | `curl http://[IP Prefix].200.20:8080`<br>`curl http://[IP Prefix].200.21:8080`<br>`curl http://[IP Prefix].200.21:5000` | Sargon |
| 4. | Reverse proxy meneruskan ke backend yang benar | `curl http://www.netics.my.id/web1`<br>`/web2`, `/app` | Sargon, Iberia |
| 5. | Multi-domain web server tetap berfungsi | `curl http://www.netics.my.id`<br>`curl http://web.netics.org` | Sargon |
| 6. | Verifikasi visual lewat browser | Firefox ke seluruh alamat pada baris 2-5 | Ognisko |
| 7. | Header diteruskan dengan benar (Host, X-Real-IP) | `curl -vL` ke tiap path, amati header request/response | Sargon |

---

## f. Tugas Mandiri Bagian C

**Pemetaan Node**

| Node | Appliance | Peran | IP Address |
| :--- | :--- | :--- | :--- |
| **Router** | `netics-pc` | Menyediakan akses internet dari NAT | `eth0`: Auto (DHCP)<br>`eth1`: `[IP Prefix].200.1/24` (statis) |
| **Sargon, Iberia, Higashi** | `netics-pc` | Client uji command line (`curl`) | Auto (DHCP) |
| **Ognisko** | `netics-pc-desktop` | Client uji browser (VNC + Chromium) | Auto (DHCP) |
| **Aegir** | `netics-pc` | DHCP Server | `[IP Prefix].200.10/24` (statis)<br>Gateway: `[IP Prefix].200.1`<br>pool DHCP: `[IP Prefix].200.100-150`<br>*Tambahkan di konfigurasi dhcpd nya:*<br>`option router [IP Prefix].200.1` |
| **Kazdel** | `netics-pc` atau `netics-server` | Web Server (Bagian A) -><br>Reverse Proxy (Bagian C) | `[IP Prefix].200.11/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| **Chernobog** | `netics-pc` atau `netics-server` | DNS Server (BIND) | `[IP Prefix].200.12/24` (statis)<br>Gateway: `[IP Prefix].200.1` |
| **web1** | `netics-pc` atau `netics-server` | Web1, backend HTTP server | `[IP Prefix].200.20/24` (statis), port 8080 (statis)<br>Gateway: `[IP Prefix].200.1` |
| **web2** | `netics-pc` atau `netics-server` | Web2, backend HTTP server + aplikasi Python (Flask) | `[IP Prefix].200.21/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |
| **web3** | `netics-pc` atau `netics-server` | Web3, backend HTTP server | `[IP Prefix].200.22/24` (statis), port 8080 & 5000 (statis)<br>Gateway: `[IP Prefix].200.1` |

**Instruksi:**
1. Konfigurasi mengikuti bagian sebelumnya (Bagian A hingga E).
   **Tambahan:** tunjukkan juga untuk soal 1 topologi yang kamu buat di GNS3.
2. Tambahkan Web3 (backend HTTP baru) dan integrasikan ke reverse proxy lewat path `/web3`.
3. Daftarkan nama domain baru di DNS server (misalnya `app.netics.my.id`) yang mengarah ke reverse proxy, lalu ubah konfigurasi reverse proxy agar juga bisa diakses lewat nama domain baru ini (name-based virtual host), selain `www.netics.my.id`.
4. Buktikan dengan `curl` dan capture (`termshark`) pada Kazdel (gunakan tmux untuk dapat menjalankan nginx dan termshark bersamaan) bahwa `X-Real-IP` pada request ke Web3 sesuai dengan IP address client yang sesungguhnya mengakses.
