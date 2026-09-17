# Jarkom Modul 1 2026 — K-09

| Nama | NRP |
|---|---|
| Sultan Ahmad Maulana Bahyshidqi | 5027250170 |
| Muhammad Razzan Azizi Djauhari | 5027251086 |

## Serial Experiments Lain — The Wired

Pada praktikum Modul 1, kami membangun jaringan **The Wired** menggunakan GNS3. **Lain** berperan sebagai router utama yang menghubungkan tiga segmen jaringan, sedangkan **Alice, Mika, Chisa, Knights, dan Eiri** berperan sebagai client. Router Lain terhubung ke internet melalui node NAT pada interface `eth0`.

Prefix IP kelompok yang digunakan adalah **10.68.x.x** dengan pembagian subnet:

- Switch 1: `10.68.1.0/24`
- Switch 2: `10.68.2.0/24`
- Switch 3: `10.68.3.0/24`

### TOPOLOGI JARINGAN

<p align="center">
  <img src="assets/no1-topologi.png" width="900">
</p>

<p align="center">
  <i>Topologi jaringan The Wired pada GNS3.</i>
</p>

---

## 1. Membangun Topologi The Wired

Lain berperan sebagai router yang menghubungkan tiga switch. Switch 1 terhubung ke **Alice** dan **Mika**, Switch 2 terhubung ke **Chisa**, sedangkan Switch 3 terhubung ke **Knights** dan **Eiri**.

Node Linux dibuat menggunakan Docker image yang diperbolehkan pada praktikum, yaitu `ardhptr21/alpinet:latest` dan/atau `ardhptr21/debinet:latest`.

Pembagian interface pada router Lain:

```text
eth0 -> NAT / Internet
eth1 -> Switch 1
eth2 -> Switch 2
eth3 -> Switch 3
```

Skema alamat IP:

| Perangkat | Interface | IP Address | Gateway |
|---|---|---|---|
| Lain | eth0 | DHCP | Otomatis |
| Lain | eth1 | `10.68.1.1/24` | - |
| Alice | eth0 | `10.68.1.2/24` | `10.68.1.1` |
| Mika | eth0 | `10.68.1.3/24` | `10.68.1.1` |
| Lain | eth2 | `10.68.2.1/24` | - |
| Chisa | eth0 | `10.68.2.2/24` | `10.68.2.1` |
| Lain | eth3 | `10.68.3.1/24` | - |
| Knights | eth0 | `10.68.3.2/24` | `10.68.3.1` |
| Eiri | eth0 | `10.68.3.3/24` | `10.68.3.1` |

Konfigurasi router Lain:

```text
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 10.68.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 10.68.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 10.68.3.1
    netmask 255.255.255.0
```

Contoh konfigurasi client Alice:

```text
auto eth0
iface eth0 inet static
    address 10.68.1.2
    netmask 255.255.255.0
    gateway 10.68.1.1
```

Setelah konfigurasi dilakukan, alamat IP diverifikasi menggunakan:

```bash
ip -br a
```

<p align="center">
  <img src="assets/no1-lain-ip.png" width="700">
</p>

<p align="center">
  <i>Hasil verifikasi alamat IP pada router Lain. Interface eth1, eth2, dan eth3 masing-masing digunakan sebagai gateway untuk tiga subnet jaringan.</i>
</p>

---

## 2. Menghubungkan Router Lain ke Internet

Interface `eth0` pada Lain dihubungkan ke node **NAT** GNS3 dan menggunakan DHCP:

```text
auto eth0
iface eth0 inet dhcp
```

Dengan DHCP, Lain memperoleh alamat IP secara otomatis dari jaringan NAT. Konektivitas internet diuji dengan:

```bash
ping -c 3 8.8.8.8
```

Jika ping berhasil, maka router Lain sudah memiliki akses ke internet publik.

<p align="center">
  <img src="assets/no2-lain-internet.png" width="900">
</p>

<p align="center">
  <i>Pengujian koneksi internet pada router Lain menggunakan ping ke 8.8.8.8.</i>
</p>

---

## 3. Mengaktifkan Routing Antar-Subnet

Client berada pada tiga subnet berbeda sehingga komunikasi antar-subnet harus dilewatkan melalui Lain.

IP forwarding diaktifkan pada router Lain menggunakan:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Verifikasi:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Output yang diharapkan:

```text
1
```

Setelah itu dilakukan pengujian antar-subnet, misalnya:

```bash
# Dari Alice ke Chisa
ping -c 3 10.68.2.2

# Dari Alice ke Knights
ping -c 3 10.68.3.2
```

Jika ping berhasil, Lain telah meneruskan paket antar-interface dengan benar.

<p align="center">
  <img src="assets/no3-routing.png" width="900">
</p>

<p align="center">
  <i>Pengujian komunikasi antar-subnet melalui router Lain.</i>
</p>

---

## 4. Memberikan Akses Internet kepada Seluruh Client

Agar semua client dapat mengakses internet melalui Lain, digunakan **NAT Masquerade** dan forwarding pada `iptables`.

Konfigurasi pada Lain:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Setelah itu client diuji dengan:

```bash
ping -c 3 8.8.8.8
```

Untuk resolusi nama domain, DNS resolver dikonfigurasi. Pada lingkungan praktikum, resolver yang digunakan adalah:

```bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf
```

Kemudian dilakukan pengujian:

```bash
ping -c 3 google.com
```

Jika ping ke `8.8.8.8` dan `google.com` berhasil, maka routing, NAT, dan DNS client telah bekerja.

<p align="center">
  <img src="assets/no4-client-internet.png" width="900">
</p>

<p align="center">
  <i>Client berhasil mengakses internet dan melakukan resolusi DNS ke google.com.</i>
</p>

---

## 5. Membuat Konfigurasi Tetap Berfungsi Setelah Restart

Untuk menjaga konfigurasi router setelah restart, perintah penting dapat dijalankan kembali ketika interface aktif. Contoh pada interface `eth0` Lain:

```text
auto eth0
iface eth0 inet dhcp
    up sysctl -w net.ipv4.ip_forward=1
    up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    up iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
    up iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
    up iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
    up iptables -A FORWARD -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Pada router Lain dibuat script `/root/cek_status.sh`:

```bash
#!/bin/sh

echo "=== STATUS INTERFACE ==="
ip -br a

echo
echo "=== STATUS NAT ==="
iptables -t nat -L -v -n
```

Script dibuat executable:

```bash
chmod +x /root/cek_status.sh
```

Kemudian setelah Stop dan Start node, dilakukan pengecekan:

```bash
/root/cek_status.sh
```

Jika IP interface dan rule NAT tetap muncul serta client masih dapat mengakses internet, konfigurasi dinyatakan berhasil bertahan setelah restart.

<p align="center">
  <img src="assets/no5-script.png" width="900">
</p>

<p align="center">
  <i>Isi script /root/cek_status.sh untuk memeriksa interface dan konfigurasi NAT.</i>
</p>

<p align="center">
  <img src="assets/no5-after-restart.png" width="900">
</p>

<p align="center">
  <i>Hasil pengecekan setelah node Lain direstart menunjukkan IP interface dan NAT Masquerade tetap aktif.</i>
</p>

---

## 6. Analisis Traffic DNS dan ICMP pada Node Mika

File `traffic_protocol7.sh` dijalankan pada node Mika. Script tersebut menghasilkan traffic **ICMP** melalui `ping` serta traffic **DNS** melalui `nslookup` dan `dig`.

Script dijalankan dengan:

```bash
chmod +x /root/traffic_protocol7.sh
/root/traffic_protocol7.sh
```

Capture dilakukan pada link **Mika ↔ Switch 1** menggunakan Wireshark. Display filter yang digunakan:

```text
dns or icmp
```

Hasil capture menunjukkan:

- DNS query dan response untuk beberapa domain.
- ICMP Echo Request dan Echo Reply dari aktivitas ping.
- Total packet pada capture: **52 packet**.
- Packet yang lolos display filter: **48 packet (92.3%)**.

Contoh traffic yang terlihat:

```text
10.68.1.3 -> 8.8.8.8    DNS Query
8.8.8.8 -> 10.68.1.3    DNS Response

10.68.1.3 -> 1.1.1.1    ICMP Echo Request
1.1.1.1 -> 10.68.1.3    ICMP Echo Reply
```

<p align="center">
  <img src="assets/no6-wireshark.png" width="900">
</p>

<p align="center">
  <i>Hasil capture Wireshark pada Mika menggunakan display filter dns or icmp.</i>
</p>

---

## 7. Konfigurasi FTP Server pada Chisa

Chisa dikonfigurasi sebagai **FTP Server** menggunakan `vsftpd` dengan shared folder:

```text
/var/wired/data
```

Kebijakan akses:

| User | Hak Akses |
|---|---|
| alice | Read + Write |
| mika | Read-only |
| eiri | Blacklist / login ditolak |

Pada Alpinet, instalasi `vsftpd` dilakukan dengan:

```bash
apk update
apk add vsftpd
```

User dibuat dengan:

```bash
adduser alice
adduser mika
adduser eiri
```

Group untuk akses read-only Mika:

```bash
addgroup wiredftp
addgroup mika wiredftp
```

Folder FTP:

```bash
mkdir -p /var/wired/data
chown alice:wiredftp /var/wired/data
chmod 750 /var/wired/data
```

Konfigurasi penting `vsftpd`:

```text
listen=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES

chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/var/wired/data

userlist_enable=YES
userlist_deny=YES
userlist_file=/etc/vsftpd.user_list

seccomp_sandbox=NO
```

Eiri dimasukkan ke blacklist:

```bash
echo "eiri" > /etc/vsftpd.user_list
```

Karena container Alpinet tidak boot menggunakan OpenRC, `vsftpd` dijalankan langsung:

```bash
/usr/sbin/vsftpd /etc/vsftpd/vsftpd.conf &
```

### Pengujian Alice

Pada Alice dibuat file:

```bash
echo "signal from alice" > /root/signal_alice.txt
```

Alice terhubung ke Chisa:

```bash
ftp 10.68.2.2
```

Setelah login sebagai `alice`, file di-upload menggunakan:

```text
put /root/signal_alice.txt signal_alice.txt
```

Upload berhasil ditandai dengan:

```text
226 Transfer complete.
```

File dapat diverifikasi di Chisa:

```bash
ls -l /var/wired/data
```

### Pengujian Eiri

Eiri mencoba login ke FTP Server Chisa menggunakan akun `eiri`. Server dapat dihubungi, tetapi proses autentikasi menghasilkan:

```text
Login failed
```

Hal ini membuktikan bahwa blacklist user Eiri bekerja.

<p align="center">
  <img src="assets/no7-alice-upload.png" width="900">
</p>

<p align="center">
  <i>Alice berhasil mengunggah file signal_alice.txt ke FTP Server Chisa.</i>
</p>

<p align="center">
  <img src="assets/no7-eiri-denied.png" width="900">
</p>

<p align="center">
  <i>Login FTP Eiri ditolak karena user Eiri telah dimasukkan ke blacklist.</i>
</p>

---

## 8. Upload `knights_report.txt` dan Analisis FTP PASV

Knights terhubung sebagai FTP client ke Chisa menggunakan akun `alice`.

File yang di-upload adalah:

```text
knights_report.txt
```

Capture Wireshark dimulai pada link Knights sebelum proses upload.

Knights login:

```bash
ftp 10.68.2.2
```

Kemudian passive mode diaktifkan dan file dikirim:

```text
passive
put /root/knights_report.txt knights_report.txt
```

Pada Wireshark ditemukan:

```text
Request: PASV
Response: 227 Entering Passive Mode (10,68,2,2,106,111)
Request: STOR knights_report.txt
Response: 226 Transfer complete.
```

Perintah **STOR** digunakan oleh FTP untuk melakukan upload file.

Port data TCP pada mode PASV dihitung dari dua angka terakhir response `227`:

```text
Port = (106 × 256) + 111
     = 27136 + 111
     = 27247
```

Wireshark juga menunjukkan pembentukan koneksi TCP menuju port `27247`.

Kesimpulan:

- Perintah upload: **STOR**
- Status sukses: **226 Transfer complete**
- Port data PASV: **27247**


<p align="center">
  <img src="assets/no8-ftp-pasv.png" width="900">
</p>

<p align="center">
  <i>Capture Wireshark menunjukkan PASV, response 227, koneksi data pada port 27247, perintah STOR knights_report.txt, dan response 226 Transfer complete.</i>
</p>

---

## 9. Read-Only FTP untuk Mika

File yang digunakan adalah:

```text
protocol7_manifesto.txt
```

File ditempatkan pada FTP Server Chisa dan diberikan permission agar group `wiredftp` dapat membaca file:

```bash
chown alice:wiredftp /var/wired/data/protocol7_manifesto.txt
chmod 640 /var/wired/data/protocol7_manifesto.txt
```

Mika login ke FTP Server Chisa:

```bash
ftp 10.68.2.2
```

File di-download menggunakan:

```text
get protocol7_manifesto.txt
```

Download berhasil karena Mika memiliki hak baca.

Untuk membuktikan Mika tidak memiliki hak tulis, dibuat konfigurasi per-user pada vsftpd.

Pada `/etc/vsftpd/vsftpd.conf` ditambahkan:

```text
user_config_dir=/etc/vsftpd_user_conf
```

Kemudian dibuat:

```bash
mkdir -p /etc/vsftpd_user_conf
```

File `/etc/vsftpd_user_conf/mika` berisi:

```text
write_enable=NO
```

Setelah `vsftpd` dijalankan ulang, Mika mencoba upload:

```text
put /root/mika_test.txt mika_test.txt
```

Upload ditolak dengan:

```text
550 Permission denied
```

Hal ini membuktikan bahwa user Mika bersifat **read-only**: dapat melakukan download tetapi tidak dapat upload.

<p align="center">
  <img src="assets/no9-mika-download.png" width="900">
</p>

<p align="center">
  <i>Mika berhasil mengunduh protocol7_manifesto.txt dari FTP Server Chisa.</i>
</p>

<p align="center">
  <img src="assets/no9-mika-denied.png" width="900">
</p>

<p align="center">
  <i>Percobaan upload oleh Mika ditolak dengan response 550 Permission denied.</i>
</p>

---

## 10. Analisis ICMP Knights ke Chisa

Knights melakukan pengujian koneksi ke Chisa menggunakan:

```bash
ping -c 77 -s 128 -i 0.3 10.68.2.2
```

Parameter:

- `-c 77`: mengirim 77 packet.
- `-s 128`: payload ICMP sebesar 128 byte.
- `-i 0.3`: interval antar-packet 0,3 detik.

Hasil pengujian:

```text
77 packets transmitted, 77 received, 0% packet loss
rtt min/avg/max/mdev = 0.428/0.614/0.980/0.122 ms
```

Sehingga:

- Packet loss: **0%**
- RTT minimum: **0.428 ms**
- RTT rata-rata: **0.614 ms**
- RTT maksimum: **0.980 ms**

Pada Wireshark digunakan filter:

```text
icmp
```

Hasil analisis ICMP:

| Jenis | Type | Code |
|---|---:|---:|
| Echo Request | 8 | 0 |
| Echo Reply | 0 | 0 |

Echo Request dikirim dari Knights ke Chisa, sedangkan Echo Reply dikirim kembali dari Chisa ke Knights.

<p align="center">
  <img src="assets/no10-ping-result.png" width="900">
</p>

<p align="center">
  <i>Hasil pengiriman 77 paket ICMP dari Knights ke Chisa menunjukkan 0% packet loss dengan RTT min/avg/max sebesar 0.428/0.614/0.980 ms.</i>
</p>

<p align="center">
  <img src="assets/no10-icmp-request.png" width="900">
</p>

<p align="center">
  <i>ICMP Echo Request memiliki Type 8 dan Code 0.</i>
</p>

<p align="center">
  <img src="assets/no10-icmp-reply.png" width="900">
</p>

<p align="center">
  <i>ICMP Echo Reply memiliki Type 0 dan Code 0.</i>
</p>

---


