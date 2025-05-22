# Installation and Configuration: Apache Cloudstack Private Cloud

## Contributors
Kelompok 17:
- Raihan Muhammad Ihsan
- Farhan Nuzul Noufendri
- Ivan Yuantama Pradipta
- Fathia Zulfa Alfajr

[toc]

---
## Pengantar

### Apa itu Cloudstack?
Apache CloudStack adalah sebuah platform open-source untuk penyediaan layanan cloud computing berbasis *Infrastructure as a Service* (IaaS). CloudStack memungkinkan organisasi atau penyedia layanan untuk membangun dan mengelola infrastruktur cloud mereka sendiri, mirip dengan layanan yang ditawarkan oleh penyedia cloud publik seperti Amazon Web Services (AWS), Microsoft Azure, atau Google Cloud. Platform ini dirancang untuk menyederhanakan penyediaan dan pengelolaan sumber daya komputasi seperti mesin virtual, penyimpanan, dan jaringan secara otomatis.

CloudStack mendukung berbagai hypervisor populer seperti KVM, VMware, dan XenServer, serta menyediakan antarmuka pengguna berbasis web, command-line interface (CLI), dan RESTful API. Selain itu, CloudStack juga menawarkan fitur-fitur canggih seperti multi-tenant isolation, manajemen IP publik dan privat, virtual router, elastic load balancing, hingga manajemen pengguna dan kuota. Dengan kemampuannya tersebut, CloudStack sangat cocok digunakan dalam implementasi private cloud maupun hybrid cloud.

## Installasi dan Konfigurasi Cloudstack
### Spesifikasi Hardware
Spesifikasi yang digunakan untuk instalasi Apache CloudStack ini adalah sebagai berikut:
- **CPU**: Intel Core i5 Generasi ke-8  
- **RAM**: 24 GB  
- **Storage**: 256 GB SSD/HDD  
- **Network**: Ethernet 100 Gb/s  
- **Operating System**: Ubuntu Server 22.04 LTS

## Installasi dan Konfigurasi Cloudstack
### Konfigurasi IP Statis dan Bridge Network di`(/etc/netplan)`
Pada tahap ini, dilakukan konfigurasi jaringan agar server memiliki IP statis dan menggunakan bridge network `(cloudbr0)` yang dibutuhkan oleh CloudStack untuk komunikasi antar VM dan manajemen. Proses dilakukan menggunakan tool bawaan Ubuntu, yaitu Netplan.
- Masuk ke direktori konfigurasi jaringan Netplan
![cloudcomp1](https://hackmd.io/_uploads/SyS-iaZlel.jpg)
Di dalam direktori ini terdapat file konfigurasi default bernama `50-cloud-init.yaml`.
- Backup file konfigurasi lama
![cc1](https://hackmd.io/_uploads/ry4XebFele.jpg)
File lama di-rename untuk menghindari konflik, sekaligus menjaga konfigurasi awal jika perlu dikembalikan.
- Membuat file konfigurasi baru
![cc2](https://hackmd.io/_uploads/BykPlbFlge.jpg)
File baru ini digunakan untuk menyusun konfigurasi IP statis dan bridge `cloudbr0`.
- Isi file konfigurasi (`01-netplan.yaml`)
![cloudcomp2](https://hackmd.io/_uploads/HkQzia-ggx.jpg)
Dimana:
    -  `cloudbr0` adalah jembatan jaringan (bridge) yang menghubungkan VM dengan jaringan fisik.
    - IP 192.168.107.84 akan digunakan sebagai IP statis untuk server.
    - Interface enp1s0 adalah kartu jaringan fisik yang di-bridge.
- Generate dan terapkan konfigurasi jaringan
![cc3](https://hackmd.io/_uploads/rk7lb-Fgeg.jpg)
![cc4](https://hackmd.io/_uploads/HJkMb-Fggg.jpg)
Command ini akan membaca file YAML dan menerapkan konfigurasi baru, menggantikan mode DHCP ke IP statis dengan bridge aktif.

### Update Sistem dan Instalasi Utilitas Dasar
Setelah konfigurasi jaringan selesai, tahap selanjutnya adalah memastikan sistem berada dalam kondisi terbaru dan stabil. Hal ini dilakukan dengan melakukan pembaruan daftar repository dan meng-upgrade seluruh paket yang terpasang. Selain itu, juga diinstal beberapa utilitas tambahan yang dibutuhkan selama proses instalasi CloudStack.
- Masuk sebagai root user dan update daftar paket
![cc5](https://hackmd.io/_uploads/SJ17G-Kxll.jpg)
`sudo su` digunakan agar semua perintah selanjutnya dapat dijalankan tanpa perlu menambahkan sudo di setiap langkah. Sedangkan, update paket akan mengambil daftar versi terbaru dari seluruh paket yang tersedia di repository, termasuk repository CloudStack yang sudah ditambahkan sebelumnya.
- Upgrade seluruh paket sistem
![cc6](https://hackmd.io/_uploads/SkqOG-tglg.jpg)
Setelah update selesai, dilakukan upgrade seluruh paket yang dapat diperbarui, termasuk:
    - `Kernel` (linux-image, linux-headers)
    - `Bootloader` (grub-efi, grub2-common)
    - Service penting lainnya seperti cloud-init, apparmor, dan ubuntu-drivers-common
- Instalasi utilitas monitoring dan jaringan
![Screenshot 2025-05-07 at 22.59.20](https://hackmd.io/_uploads/SJBP7bYlgl.png)
Dimana:
    - `htop`: alat pemantauan proses dan penggunaan resource sistem.
    - `lynx`: browser berbasis CLI yang dapat digunakan untuk mengakses halaman web melalui terminal.
    - `duf`: alat modern untuk melihat penggunaan disk space.
- Instalasi bridge-utils
![Screenshot 2025-05-07 at 23.01.04](https://hackmd.io/_uploads/Hy2amZteel.png)
Paket ini digunakan untuk membuat dan mengelola bridge jaringan seperti `cloudbr0` yang telah dikonfigurasi sebelumnya melalui Netplan. Bridge network ini diperlukan agar virtual machine dapat terhubung dengan jaringan luar.
- Instalasi tools sistem tambahan
![Screenshot 2025-05-07 at 23.01.54](https://hackmd.io/_uploads/rygbEbtgxx.png)
Dimana:
    - `openntpd`: digunakan untuk sinkronisasi waktu server.
    - `openssh-server`: agar server dapat diakses melalui SSH dari perangkat lain.
    - `sudo`: memastikan perintah administratif dapat dijalankan oleh user non-root jika diperlukan.
    - `vim`: editor teks berbasis terminal yang umum digunakan dalam konfigurasi sistem.
    - `tar`: utilitas untuk mengekstrak dan membuat file arsip .tar.gz.

### Konfigurasi Waktu dan Zona Waktu Server
Agar waktu server sinkron dengan zona lokal (Asia/Jakarta), serta sesuai dengan kebutuhan sistem CloudStack dan troubleshooting log yang akurat, dilakukan pengaturan timezone serta sinkronisasi waktu menggunakan NTP (Network Time Protocol).
- Cek informasi waktu saat ini
![Screenshot 2025-05-07 at 23.07.48](https://hackmd.io/_uploads/ry-vBZKgxl.png)
Perintah ini menampilkan status waktu sistem, termasuk timezone dan apakah sudah sinkron atau belum. Default-nya masih menggunakan `Etc/UTC`.
- Lihat daftar timezone yang tersedia
![Screenshot 2025-05-07 at 23.08.01](https://hackmd.io/_uploads/BkTwHZKlge.png)
Daftar ini digunakan untuk mencari nama zona waktu yang sesuai, dalam hal ini Asia/Jakarta.
- Set timezone ke Asia/Jakarta
![Screenshot 2025-05-07 at 23.08.18](https://hackmd.io/_uploads/H1A_SWFgex.png)
Setelah diatur, ulangi perintah `timedatectl` untuk memastikan waktu lokal sudah berubah ke WIB (UTC+7).

###  Konfigurasi Sinkronisasi Waktu (NTP)
Sinkronisasi waktu penting agar layanan seperti database dan cloud service berjalan akurat dan terkoordinasi. Karena sebelumnya sempat terinstal `openntpd`, pada tahap ini dilakukan penggantian dengan `systemd-timesyncd` untuk sinkronisasi waktu default Ubuntu.
- Edit file konfigurasi `timesyncd.conf` (opsional)
![Screenshot 2025-05-07 at 23.12.12](https://hackmd.io/_uploads/ByKwIZFelg.png)
Isi file dapat diatur sesuai kebutuhan, seperti mengganti server NTP default jika diperlukan (misal:     `ntp.ubuntu.com`).
- Install `systemd-timesyncd`
![Screenshot 2025-05-07 at 23.12.29](https://hackmd.io/_uploads/ryhOUWYllx.png)
Paket ini akan menggantikan `openntpd` dan menjadi default NTP client di Ubuntu.
- Aktifkan dan mulai layanan NTP
![Screenshot 2025-05-07 at 23.12.49](https://hackmd.io/_uploads/H1CFL-teex.png)
- Aktifkan sinkronisasi NTP secara eksplisit
![Screenshot 2025-05-07 at 23.13.06](https://hackmd.io/_uploads/Syki8-tgee.png)
Saat memeriksa kembali status sinkronisasi, hasilnya berhasil, ditunjukkan pada bagian `NTP service:` akan berubah menjadi `active`.

### Konfigurasi Akses Root melalui SSH
Untuk keperluan remote akses dan pengelolaan server, dilakukan pengaturan agar user `root` dapat login melalui SSH, serta pengaturan ulang password `root`.
- Install paket tambahan dan update microcode Intel
![Screenshot 2025-05-07 at 23.16.32](https://hackmd.io/_uploads/B1RwDWKgle.png)
Paket ini disarankan oleh sistem untuk optimalisasi prosesor Intel.
- Set password baru untuk user root
![Screenshot 2025-05-07 at 23.17.02](https://hackmd.io/_uploads/Hk9KDbYegl.png)
Perintah ini akan meminta pengisian password baru untuk user `root`.
- Izinkan root login melalui SSH
Menggunakan `sed` untuk mengubah konfigurasi SSH:
![Screenshot 2025-05-07 at 23.17.18](https://hackmd.io/_uploads/By95DbFllx.png)
![Screenshot 2025-05-07 at 23.17.34](https://hackmd.io/_uploads/Hk5oD-Fexg.png)
Perintah di atas menambahkan baris `PermitRootLogin` yes di bawah baris `PermitRootLogin prohibit-password`.
- Restart layanan SSH agar perubahan berlaku
![Screenshot 2025-05-07 at 23.17.58](https://hackmd.io/_uploads/HJbTPZYxeg.png)

### Import Repositori CloudStack
Untuk bisa menginstal CloudStack, kita perlu menambahkan repositori resmi dari ShapeBlue.
- Masuk sebagai root dan buat direktori keyrings
![Screenshot 2025-05-07 at 23.25.37](https://hackmd.io/_uploads/BJ0FtWtlee.png)
- Ambil file GPG key repositori ShapeBlue dan simpan ke keyring
![Screenshot 2025-05-07 at 23.26.00](https://hackmd.io/_uploads/BkNitbYeeg.png)
- Tambahkan repositori CloudStack ke daftar sumber APT
![Screenshot 2025-05-07 at 23.26.18](https://hackmd.io/_uploads/Bkr3FZKegx.png)
- Verifikasi isi file repositori
![Screenshot 2025-05-07 at 23.26.39](https://hackmd.io/_uploads/rkjatZtgxl.png)
Memastikan ada satu baris seperti ini:
![Screenshot 2025-05-07 at 23.27.52](https://hackmd.io/_uploads/HyNG5Wteee.png)

### Installing dan configuring Mysql
- Setelah repositori ditambahkan, lanjutkan dengan proses instalasi:
![Screenshot 2025-05-07 at 23.29.26](https://hackmd.io/_uploads/S1z_5bYxge.png)
![Screenshot 2025-05-07 at 23.29.37](https://hackmd.io/_uploads/ByAdq-Fxgg.png)
CloudStack akan mengatur dependensi tambahan, dan MySQL 8.x akan ikut terinstall. Setelah selesai, kernel akan meminta reboot karena update.
- Edit konfigurasi utama MySQL
![Screenshot 2025-05-07 at 23.30.51](https://hackmd.io/_uploads/Byw6qWYlgg.png)
Tambahkan atau pastikan konfigurasi ini:
![Screenshot 2025-05-07 at 23.31.08](https://hackmd.io/_uploads/SJFCqZKllg.png)
- Saat mencoba menjalankan kembali MySQL:
![Screenshot 2025-05-07 at 23.35.08](https://hackmd.io/_uploads/rkYaiWFxll.png)
![Screenshot 2025-05-07 at 23.36.02](https://hackmd.io/_uploads/HJ1-2-Kxle.png)
Error ini mengindikasikan adanya masalah permission atau konflik sisa instalasi lama, kemungkinan dari MariaDB/MySQL sebelumnya.
- Solusi (Based on StackOverflow)
Mengikuti beberapa referensi StackOverflow yang sudah dicoba namun gagal:
    - https://stackoverflow.com/questions/42317139
    - https://stackoverflow.com/questions/70813122
- Akhirnya solusi yang berhasil adalah:
![Screenshot 2025-05-07 at 23.37.29](https://hackmd.io/_uploads/H1B83Wtxge.png)
Perintah di atas benar-benar menghapus jejak instalasi lama dan memberikan fresh state untuk MySQL agar bisa diinstal ulang tanpa konflik permission.
- Setelah menjalankan ulang:
![Screenshot 2025-05-07 at 23.38.07](https://hackmd.io/_uploads/BJhdn-Fgxl.png)
MySQL akhirnya berhasil aktif tanpa error. Sehingga proses bisa lanjut ke tahap deploy database.

### Inisialisasi Database CloudStack dan Setup Primary dan Secondary Storage (NFS)
- Deploy database dan buat user `cloud`
![Screenshot 2025-05-07 at 23.39.53](https://hackmd.io/_uploads/rJIkpbteel.png)
Ini untuk inisialisasi database dan bikin user cloud. IP-nya pakai yang udah diset sebelumnya di `cloudbr0` (IP manajemen server).
- Install NFS Server dan quota
![Screenshot 2025-05-07 at 23.41.18](https://hackmd.io/_uploads/BJ94pbtleg.png)
NFS ini dibutuhin supaya bisa jadi storage buat CloudStack, nanti VM bakal nyimpen data-nya di sini.
- Edit file ekspor NFS
![Screenshot 2025-05-07 at 23.41.47](https://hackmd.io/_uploads/rJu8TZtllx.png)
Kita set supaya folder `/export` bisa dishare ke jaringan. Ini wajib buat akses storage dari CloudStack.
- Buat direktori ekspor
![Screenshot 2025-05-07 at 23.42.18](https://hackmd.io/_uploads/H1wuaZFllg.png)
Folder `primary` dan `secondary` ini nanti bakal dipakai sebagai tempat simpan disk VM dan template ISO.
- Beberapa konfigurasi NFS Tambahan
![Screenshot 2025-05-07 at 23.44.42](https://hackmd.io/_uploads/ByLb0ZYlex.png)
![Screenshot 2025-05-07 at 23.45.06](https://hackmd.io/_uploads/rJkmCZKlel.png)
![Screenshot 2025-05-07 at 23.45.28](https://hackmd.io/_uploads/SJI4A-Fegx.png)
Ini buat nge-set port biar NFS-nya ga random dan lebih stabil saat dipakai.
- Restart layanan NFS
![Screenshot 2025-05-07 at 23.47.33](https://hackmd.io/_uploads/Sk730-Fglx.png)


### Konfigurasi dan Instalasi Agent VM
-  Install `qemu-kvm` dan `cloudstack-agent`
![image](https://hackmd.io/_uploads/Hk2dfqtgex.png)
paket sudah terinstall sebelumnya, jadi tidak ada yang perlu di-upgrade. Udah ready buat jalanin VM nanti.
-  Edit `/etc/libvirt/qemu.conf` buat enable akses VNC dari luar
![image](https://hackmd.io/_uploads/H1W9zctlge.png)
Supaya nanti kita bisa akses GUI VM-nya dari jaringan luar, bukan cuma localhost.
- Enable libvirtd buat listen
![image](https://hackmd.io/_uploads/rkMifcYell.png)
![image](https://hackmd.io/_uploads/ryrhM5Yxgg.png)
kita buat libvirtd supaya aktif dan bisa nerima koneksi dari network.
- Tambah beberapa baris ke `/etc/libvirt/libvirtd.conf`
![image](https://hackmd.io/_uploads/HkUJXqFlle.png)
Ini biar agent bisa connect ke host via TCP dan tidak butuh TLS.
- Aktifkan dan restart socket libvirtd
![image](https://hackmd.io/_uploads/HyMl7qYgex.png)
Socket default kita matiin biar tidak bentrok. Yang penting sekarang kita pakai port TCP-nya.
- Konfigurasi Netfilter Bridge di Kernel
![image](https://hackmd.io/_uploads/B1XVQ5Ylel.png)
Kita menonaktifkan pemrosesan packet bridge oleh iptables, agar lalu lintas jaringan antar VM atau antar bridge nggak diblokir tanpa sengaja oleh firewall Linux. Ini langkah umum dan penting untuk memastikan VM yang pakai virtual bridge (seperti `cloudbr0`) bisa komunikasi tanpa terganggu.
- Install `UUID` dan Update Paket Tambahan
![image](https://hackmd.io/_uploads/Byer7qYegg.png)
Paket ini penting untuk generate `UUID` host. Proses ini juga otomatis menarik beberapa paket dependencies tambahan, termasuk update dari `libuuid1`, `libsnapd-glib`, dan lainnya.
- Tambah `host_uuid` ke `libvirt`
![image](https://hackmd.io/_uploads/S1SyNqtegx.png)
Digunakan buat generate `UUID` unik dari host, lalu ditulis ke file konfigurasi `libvirt` (libvirtd.conf). UUID ini bakal dibaca CloudStack supaya VM bisa dikenali di host.
- Edit `/etc/libvirt/libvirtd.conf`
![image](https://hackmd.io/_uploads/SkQQ45Klle.png)
Di sini kita nyalain koneksi TCP (tanpa TLS) untuk komunikasi antara CloudStack dan agent host via port 16509. Kita juga set `host_uuid` supaya CloudStack bisa ngenalin host ini secara unik.
- Restart Service Libvirt
![image](https://hackmd.io/_uploads/rJMjLcFele.png)
Dipakai untuk menerapkan semua perubahan konfigurasi yang sebelumnya dilakukan di file `libvirtd.conf`, `default/libvirtd`, dan lainnya. Tanpa restart, libvirt gak akan baca setting baru, jadi langkah ini wajib setelah modifikasi.
- Membuka Port di Firewall dengan `iptables`
![image](https://hackmd.io/_uploads/HJP7wcFgxl.png)
    Tujuannya adalah mengizinkan akses masuk ke port-port penting yang digunakan oleh CloudStack, NFS, KVM/libvirt, dan layanan pendukung lainnya.
Beberapa port yang dibuka:
    - `111` (port RPC)
    - `2049` (NFS)
    - `32803`, `892`, `875`, `662`, `2020` → (port internal NFS dan quota)
    - `8080` → akses UI CloudStack
    - `8443`, `8250`, `9090` → untuk manajemen service CloudStack
    - `16509` → koneksi TCP ke libvirt dari management server
    - `3128` → proxy/Squid (opsional tergantung setup)
Tanpa membuka port-port ini, komunikasi antar komponen di arsitektur CloudStack gak akan bisa jalan. Ini salah satu step wajib dalam hardening dan fungsionalitas jaringan CloudStack.
- Simpan Konfigurasi Firewall 
![image](https://hackmd.io/_uploads/ByI9PcKell.png)
Pertama, kita buat direktori `/etc/iptables` kalau belum ada. Kedua, kita simpan semua aturan iptables yang sudah dibuat (termasuk open port sebelumnya) ke file `rules.v4`.  Tujuannya, agar semua aturan firewall tidak hilang saat reboot, dan bisa langsung diterapkan ulang saat sistem menyala. Langkah ini akan dilengkapi lagi dengan instalasi `iptables-persistent`.
- Konfirmasi Simpan Aturan `iptables`
![image](https://hackmd.io/_uploads/BycaP5Kelx.png)
Tanpa ini, semua rule iptables akan hilang setelah reboot. Jadi ini langkah penting biar firewall tetap konsisten.
- Install `iptables-persistent`
![image](https://hackmd.io/_uploads/S1YPucFxle.png)
Ini untuk mengaktifkan fitur penyimpanan otomatis aturan firewall, yaitu menyimpan file rules.v4 (dan rules.v6 kalau ada) yang akan di-load tiap kali sistem dinyalakan.
![image](https://hackmd.io/_uploads/Sk6ZO5Kllg.png)
Ini adalah kelanjutan dari step sebelumnya saat kita `iptables-save`, dan menjadi penjamin bahwa konfigurasi firewall yang udah kita buat gak akan hilang setelah reboot.
- Nonaktifkan AppArmor untuk Libvirt
![image](https://hackmd.io/_uploads/rJtUOqKlle.png)
Tujuannya adalah agar AppArmor tidak membatasi akses libvirt, terutama dalam mengakses resource jaringan atau storage yang dibutuhkan CloudStack.
- Menjalankan CloudStack Management Server
![image](https://hackmd.io/_uploads/Byeod5Yxxl.png)
Ini adalah proses initial setup untuk CloudStack Management Server. Script ini akan mengonfigurasi environment dan service `cloudstack-management` agar siap dijalankan.
- Akses Dashboard CloudStack via Web UI
![image](https://hackmd.io/_uploads/Bkf-tcYglg.png)
Setelah semua konfigurasi berhasil dan service `cloudstack-management` sudah berjalan, kita bisa mengakses tampilan login CloudStack melalui browser. Tampilan ini menandakan bahwa CloudStack berhasil berjalan dan siap digunakan.


### Konfigurasi Infrastruktur CloudStack
- Login Pertama & Ganti Password Admin
![Screenshot 2025-05-08 at 09.41.37](https://hackmd.io/_uploads/S16y5cFeex.png)
Setelah berhasil login menggunakan akun default (`admin` / `password`), CloudStack langsung meminta pengguna untuk mengubah password awal. Hal ini dilakukan untuk alasan keamanan agar administrator mengganti kredensial default dan mencegah akses tidak sah ke dashboard manajemen.
- Menambahkan Zone Baru di CloudStack
![Screenshot 2025-05-08 at 09.42.55](https://hackmd.io/_uploads/Sk9455Yxxe.png)
Kita pilih opsi "Core" karena kita akan membangun infrastruktur lengkap, bukan skenario edge. Zone ini akan menjadi kerangka besar tempat Pod, Cluster, Host, dan storage akan dimasukkan nantinya.
- Memilih Tipe Core Zone
![Screenshot 2025-05-08 at 09.43.56](https://hackmd.io/_uploads/HJudccKllx.png)
Kita pakai Advanced, karena kita butuh kemampuan setup network internal (misalnya private IP, NAT, routing, dsb), bukan cuma satu flat network. Opsi Security Groups tidak dicentang karena belum dibutuhkan untuk segmentasi VM berbasis group rules.
- Konfigurasi Zone Details
![Screenshot 2025-05-08 at 09.46.04](https://hackmd.io/_uploads/Byvgi5txeg.png)
Field yang diisi antara lain:
    - Name: KELOMPOK17-ZONE → nama zone buatan sendiri.
    - IPv4 DNS1: 8.8.8.8 → menggunakan DNS publik dari Google.
    - Internal DNS1: 192.168.106.1 → diarahkan ke gateway internal jaringan kita (biasanya IP router atau DHCP server internal).
    - Hypervisor: KVM → sesuai dengan stack virtualisasi yang digunakan di environment kita.
- Network Configuration 
![Screenshot 2025-05-08 at 09.47.32](https://hackmd.io/_uploads/r11LscYlee.png)
Field yang diisi adalah:
    - Network name: Physical Network 1 → ini cuma nama label, bisa dikasih nama lain sesuai kebutuhan.
    - Isolation method: VLAN → dipakai agar masing-masing jenis trafik (management, guest, public) terpisah dan tidak saling ganggu.
    - Traffic types: Guest → untuk trafik dari dan ke VM user, Management → untuk komunikasi CloudStack ke host/agent, Public → jalur keluar internet dari VM user.
- Konfigurasi Public Traffic untuk Zone
![Screenshot 2025-05-08 at 09.49.01](https://hackmd.io/_uploads/r1_sjcFele.png)
    - Gateway: 192.168.106.1 → ini pintu keluar trafik VM ke jaringan luar.
    - Netmask: 255.255.254.0 → menentukan seberapa besar subnetnya.
    - Start IP & End IP: 192.168.106.201 sampai 192.168.106.210 → ini blok IP publik yang akan dialokasikan buat VM via NAT.
- Konfigurasi Pod Pertama dalam Zone
![Screenshot 2025-05-08 at 09.50.02](https://hackmd.io/_uploads/B1Iy25tlgx.png)
    - Pod Name: KELOMPOK17_POD → nama unik buat identifikasi pod.
    - Reserved system gateway & netmask: 192.168.106.1 & 255.255.254.0 → ini jalur internal buat komunikasi sistem.
    - Start & End reserved system IP: 192.168.106.211 sampai 192.168.106.220 → IP-IP ini dicadangkan buat traffic manajemen internal (bukan dipakai user).
- Konfigurasi Guest Traffic VLAN/VNI Range
![Screenshot 2025-05-08 at 09.51.29](https://hackmd.io/_uploads/r1TE39Ylee.png)
Guest traffic adalah komunikasi antar VM (instance) yang dibuat user di cloud. Di sini, kita diminta menentukan range VLAN atau VNI. Range ini dipakai untuk memisahkan traffic antar guest network biar nggak saling ganggu.
- Tambahkan Cluster ke Dalam Pod
![Screenshot 2025-05-08 at 09.53.05](https://hackmd.io/_uploads/Skacn9Kggg.png)
Cluster ini nanti berisi host-host (server virtual/VM) yang menjalankan hypervisor dan terhubung ke storage yang sama. Semua host di dalam satu cluster harus berada dalam subnet yang sama dan bisa akses storage bareng. Intinya, cluster adalah cara untuk ngelompokkan resource fisik (host dan storage) biar CloudStack bisa kelola secara efisien.
- Menambahkan Host ke Cluster
![Screenshot 2025-05-08 at 09.54.11](https://hackmd.io/_uploads/SJR0n9Kllx.png)
Host ini adalah mesin fisik/VM yang nanti menjalankan virtual machine milik user melalui hypervisor. Jadi step ini penting karena tanpa host, CloudStack nggak bisa deploy instance.
- Menambahkan Primary Storage
![Screenshot 2025-05-08 at 09.55.09](https://hackmd.io/_uploads/H1tf69tgle.png)
Langkah ini penting karena primary storage adalah tempat nyimpan semua volume dari VM yang aktif (misalnya disk utama virtual machine). Tanpa ini, VM gak akan bisa jalan.
- Menambahkan Secondary Storage
![Screenshot 2025-05-08 at 09.56.03](https://hackmd.io/_uploads/ByRra9txlg.png)
Jadi meskipun VM jalan di primary storage, semua "aset pendukung" disimpan di sini. Kedua storage ini sama-sama penting supaya CloudStack bisa berfungsi penuh.
- Launching the Zone
![Screenshot 2025-05-08 at 09.56.53](https://hackmd.io/_uploads/HJeFTqYlgx.png)
Langkah ini bakal mengaktifkan semua konfigurasi yang udah kita buat sebelumnya. Jadi mulai dari sekarang, sistem CloudStack akan bisa ngejalanin instance VM dan layanan cloud lain dalam zone tersebut.
- Menambahkan ISO Template ke CloudStack
![Screenshot 2025-05-08 at 09.57.41](https://hackmd.io/_uploads/B1Z369Fgxx.png)
ISO ini adalah file image yang berisi OS installer. Dengan mendaftarkannya ke CloudStack, kita bisa membuat instance baru (VM) yang menjalankan OS dari ISO tersebut.
- Memantau Proses Download ISO Template
![Screenshot 2025-05-08 at 09.58.30](https://hackmd.io/_uploads/rybkR5Kgle.png)
CloudStack tidak bisa menggunakan ISO sebagai sumber pembuatan VM sebelum ISO tersebut tersimpan lokal di secondary storage. Jadi, langkah ini memastikan file ISO tersedia di sistem sebelum lanjut provisioning VM.
- Mengaktifkan static NAT agar VM dapat terkoneksi dengan internet
![image](https://i.imgur.com/8MoyrAq.png)
- Mengkonfiguraasi Firewall
![image](https://i.imgur.com/ANOmn0f.png)
- Mengakses VM Instance via Console dan Melakukan Tes Konektivitas Jaringan
![image](https://i.imgur.com/IriplRE.png)
Disini, kita berhasil masuk ke VM instance, artinya instalasi ISO Ubuntu berjalan sukses. Lalu, kita mencoba melakukan ping ke beberapa alamat IP:
    - 8.8.8.8 → Google DNS, hasilnya: 100% packet loss, artinya tidak terkoneksi ke internet.
    - 192.168.107.84 dan 192.168.106.1 → hasilnya Reply received, artinya koneksi antar internal VM dan ke gateway lokal berhasil.

















