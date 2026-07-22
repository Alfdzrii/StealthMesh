# Post-Quantum Mesh VPN — Setup Guide (English)

## Overview

This guide describes the setup, configuration, and demonstration procedure for a **Post-Quantum Secure Mesh VPN** built with **WireGuard** and **Rosenpass** (ML-KEM key exchange), including a **dark network** firewall configuration, a **Grafana/Prometheus** monitoring dashboard, and forensic traffic validation using **tcpdump** and **Wireshark**.

The setup involves **three Ubuntu virtual machines (Laptop 1, Laptop 2, Laptop 3)**, each acting as a peer node in a mesh (point-to-point) VPN topology. Ensure all three VMs are running Ubuntu, have network mode set to **Bridged Adapter**, and can communicate with each other over the local network before starting.

| Node | Role |
|---|---|
| Laptop 1 | Client / monitoring dashboard host / forensic capture host |
| Laptop 2 | Secret file server (XAMPP) |
| Laptop 3 | Secondary mesh peer (proves point-to-point / no single central server) |

---

## Step 1 — Install Base Packages

**Run on Laptop 1, 2, and 3.**

Open a terminal on each Ubuntu VM and run the following commands to install WireGuard, monitoring tools, and Post-Quantum Cryptography (PQC) dependencies.

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install WireGuard, firewall, packet capture, and exporter tools
sudo apt install wireguard wireguard-tools ufw tcpdump prometheus-blackbox-exporter -y

# Install Cargo (Rust) to build Rosenpass
sudo apt install cargo -y
cargo install rosenpass
```

---

## Step 2 — Build the Dark Network (Firewall)

**Run on Laptop 1, 2, and 3.**

This step enforces **network obfuscation** so that the API/VPN ports are not visible to public scanners.

```bash
# Deny all incoming connections by default
sudo ufw default deny incoming

# Allow all outgoing connections
sudo ufw default allow outgoing

# Allow SSH so you are not locked out of remote sessions
sudo ufw allow ssh

# Allow the WireGuard port only
sudo ufw allow 51820/udp

# Enable the firewall
sudo ufw enable
```

---

## Step 3 — Generate WireGuard Keys

**Run on Laptop 1, 2, and 3.**

Each node requires a base cryptographic identity before Rosenpass adds post-quantum protection.

```bash
cd /etc/wireguard/
umask 077
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey
```

> **Note:** Run `cat publickey` on each laptop and save the output somewhere accessible, as you will need to exchange these public keys with the other nodes in the next step.

---

## Step 4 — Assemble the Mesh Architecture

**Run on Laptop 1, 2, and 3.**

Create the WireGuard configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

For a true mesh architecture, every laptop's `[Peer]` blocks must point to the **other two** laptops. Example for **Laptop 1 (Node A — 10.0.0.1)**:

```toml
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [LAPTOP_1_PRIVATE_KEY]

[Peer]
# Connection to Laptop 2
PublicKey = [LAPTOP_2_PUBLIC_KEY]
AllowedIPs = 10.0.0.2/32
Endpoint = [LAPTOP_2_WIFI_IP]:51820

[Peer]
# Connection to Laptop 3
PublicKey = [LAPTOP_3_PUBLIC_KEY]
AllowedIPs = 10.0.0.3/32
Endpoint = [LAPTOP_3_WIFI_IP]:51820
```

Apply the same pattern to Laptop 2 (`Address = 10.0.0.2/24`) and Laptop 3 (`Address = 10.0.0.3/24`), updating the endpoint and public key to point to the other two peers.

---

## Step 5 — Allow the Monitoring Exporter Path

**Run on Laptop 2 and Laptop 3 only.**

This allows the dashboard on Laptop 1 to safely pull ping metrics through the VPN tunnel without breaking the dark network rules.

```bash
sudo ufw allow in on wg0 to any port 9115
```

---

## Step 6 — Install the Grafana and Prometheus Dashboard

**Run on Laptop 1 only.**

Install the monitoring stack used to display live latency metrics.

```bash
# Install Prometheus
sudo apt install prometheus -y

# Install Grafana
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana -y

# Start and enable Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

---

## Step 7 — Configure the Prometheus Target

**Run on Laptop 1 only.**

Point Prometheus to ping the virtual VPN IPs of Laptop 2 and Laptop 3.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add the following block at the bottom of the file:

```yaml
- job_name: 'vpn_mesh_latency'
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets:
        - 10.0.0.2   # Laptop 2 VPN IP
        - 10.0.0.3   # Laptop 3 VPN IP
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:9115   # Local blackbox exporter
```

Restart Prometheus to apply the change:

```bash
sudo systemctl restart prometheus
```

---

## Step 8 — Activate the PQC VPN

**Run on Laptop 1, 2, and 3.**

Bring up the network interface and let Rosenpass secure the tunnel using post-quantum keys (ML-KEM).

```bash
# Bring up WireGuard
sudo wg-quick up wg0

# Open the Rosenpass exchange port
sudo ufw allow 9999/udp
```

Create the Rosenpass configuration file:

```bash
sudo nano /etc/wireguard/rp.toml
```

Copy the following template and adjust it for your node (example shown for **Laptop 1 / Node A**):

```toml
# ==========================================
# ROSENPASS CONFIGURATION — LAPTOP 1 (NODE A)
# ==========================================
secret_key = "/home/eji/vpn-keys/rp-secret.key"
public_key = "/home/eji/vpn-keys/rp-public.key"
listen = ["0.0.0.0:9999"]

# ------------------------------------------
# ROSENPASS CONNECTION TO LAPTOP 2 (NODE B)
# ------------------------------------------
[[peers]]
# Replace with the contents of Laptop 2's rp-public.key
public_key = "<LAPTOP_2_RP_PUBLIC_KEY>"
endpoint = "10.8.21.103:9999"
# This line injects the quantum-derived key into WireGuard
wireguard = { dev = "wg0", peer = "AEWL3SByr3PHHFsL2AP1NW6dRox7iI3VtXBDg/Ydww8=" }

# ------------------------------------------
# ROSENPASS CONNECTION TO LAPTOP 3 (NODE C)
# ------------------------------------------
[[peers]]
# Replace with the contents of Laptop 3's rp-public.key
public_key = "<LAPTOP_3_RP_PUBLIC_KEY>"
endpoint = "10.8.9.210:9999"
# This line injects the quantum-derived key into WireGuard
wireguard = { dev = "wg0", peer = "+YKW2VvHBf1Bytq3frmW+fa9FoLI3gN+GU8c1S1t9xI=" }
```

Start the Rosenpass exchange daemon, which reads the `wg0` configuration and performs automatic PQC key encapsulation:

```bash
sudo rosenpass exchange-config /etc/wireguard/rp.toml
```

---

## How to Run

### Part 1 — Build the Latency Graph in Grafana

1. **Log in to Grafana.** On Laptop 1, open `http://localhost:3000` in a browser.
   - Default username: `admin`
   - Default password: `admin`
   - Grafana will prompt you to set a new password on first login.
2. **Connect Grafana to Prometheus.**
   - In the left menu, click **Connections** (or the gear/Settings icon) → **Data Sources**.
   - Click **Add data source** → select **Prometheus**.
   - Set the URL field to `http://localhost:9090`.
   - Scroll down and click **Save & Test** (a green confirmation indicates success).
3. **Create a new dashboard.**
   - Click the **+ (Plus)** icon → **Dashboard**.
   - Click **Add a new panel**.
   - Set the panel title to `PQC VPN Latency`.
4. **Enter the query.** In the Metrics browser / Query field, enter exactly:
   ```
   probe_duration_seconds{job="vpn_mesh_latency"}
   ```
   Press **Shift + Enter** or click **Run query**. A line graph showing ping latency from Laptop 1 to Laptop 2 and Laptop 3 should appear.
5. **Polish the visualization.**
   - Under **Graph styles**, increase the **Line width** for visibility.
   - Under **Standard options → Unit**, set the unit to **Time → seconds** or **milliseconds**.
   - Click **Save** in the top-right corner.

### Part 2 — Capturing Forensic Evidence with tcpdump

This step proves that network traffic is genuinely encrypted by WireGuard and Rosenpass's post-quantum layer, rather than being plain HTTP/TCP.

**Important:** Capture traffic on the **physical WiFi/LAN interface**, not on `wg0`. Capturing on `wg0` shows already-decrypted traffic; capturing on the physical interface shows the ciphertext as it travels over the air.

1. Identify the physical network interface:
   ```bash
   ip a
   ```
   Look for the interface with your local WiFi IP (e.g. `192.168.x.x`) — commonly named `eth0`, `enp0s3`, `wlan0`, or `wlp2s0`.
2. Capture traffic on that interface (replace `enp0s3` with your interface name):
   ```bash
   sudo tcpdump -i enp0s3 port 51820 -X
   ```
   - `-i enp0s3` — capture only on this interface
   - `port 51820` — filter to WireGuard/PQC VPN traffic only
   - `-X` — display packet payload in hexadecimal and ASCII
3. While `tcpdump` is running, generate traffic (e.g. keep the Grafana dashboard open, or run `ping 10.0.0.2` from another terminal). The payload column should show no readable text (no `GET / HTTP`, filenames, or messages) — only random-looking bytes. This output is the evidence that data in transit is post-quantum-secured ciphertext.

### Optional — Visualizing Traffic with Wireshark

#### What will be shown

| Element | Filter | What is observed | What it demonstrates |
|---|---|---|---|
| WireGuard tunnel | `udp.port == 51820` | Thousands of UDP packets | VPN operates on a hidden port |
| Encrypted data (ciphertext) | Inspect payload on the WiFi interface | Hex text and random-looking symbols | Post-quantum encryption is wrapping the original data |
| Original traffic (plaintext) | Capture on `wg0` (via SSHDump) | Readable keywords such as `Echo (ping) request` or `HTTP GET` | VPN delivers data intact without corruption |

#### Execution steps

1. **Generate traffic** on Laptop 1: open a terminal and run `ping 10.0.0.2`, letting it run continuously.
2. **Open Wireshark** on the main capture machine and select the active network interface (Wi-Fi or Ethernet).
3. **Filter for the VPN traffic:** in the display filter bar, type `udp.port == 51820` and press Enter. Click a packet and inspect the raw data — it should be unreadable, showing that only ciphertext is visible on the wire.
4. **Switch to the internal view (SSHDump):**
   - Stop the current capture.
   - Go to **Capture → Options**.
   - Under **SSH remote capture: sshdump**, click the **Settings (gear)** icon.
5. **Configure SSHDump:**
   - **Server** tab: enter the WiFi IP of the target Ubuntu VM and port `22`.
   - **Authentication** tab: enter the Ubuntu login username and password.
   - **Capture** tab: set **Remote interface** to `wg0`.
   - Click **Save**, enable the sshdump entry, then click **Start**.
6. **Visualize the decrypted traffic:** type `icmp` in the filter bar and press Enter. You should see `Echo (ping) request` and `Echo (ping) reply` entries with readable payloads — confirming the tunnel delivers data correctly end to end.

---

## Presentation / Demo Script

### Role Assignment

- **Laptop 1 (yours):** Acts as the client accessing the file, and as the forensic capture station (`tcpdump`).
- **Laptop 2 (Teammate 1):** Acts as the "secret company server" — runs XAMPP and hosts the sample file.
- **Laptop 3 (Teammate 2):** Acts as a remote branch client, proving the mesh is truly point-to-point rather than a two-node connection.

### Stage 1 — Preparation (Laptop 2, the file server)

Complete this before evaluation begins.

**1. Prepare the bait document**

Create a PDF file named `Data_Penting_Perusahaan.pdf`.

**2. Start XAMPP (Apache)**

- Open the **XAMPP Control Panel**.
- Click **Start** next to **Apache** and confirm it turns green (running).
- Open the file manager, navigate to the XAMPP install directory (e.g. `/opt/lampp/htdocs/`).
- Copy `Data_Penting_Perusahaan.pdf` into the `htdocs` folder.

**3. Build the dark network (UFW configuration)**

Run these commands in order on Laptop 2:

```bash
# Deny direct access from the campus/public network
sudo ufw deny in on enp0s8 to any port 80

# Allow access only through the VPN tunnel
sudo ufw allow in on wg0 to any port 80

# Enable the firewall
sudo ufw enable
```

Optional check:

```bash
sudo ufw status
```

Confirm `enp0s8` shows `DENY` and `wg0` shows `ALLOW`.

### Stage 2 — Live Demonstration (Laptop 1)

**Act 1 — Demonstrate stealth mode (expected failure)**

1. Open Google Chrome on Laptop 1.
2. Navigate to Laptop 2's **public WiFi IP** plus the filename, e.g.:
   `http://192.168.1.50/Data_Penting_Perusahaan.pdf`
3. Show the evaluator that the browser hangs and eventually times out.
4. Narrative: explain that the UFW firewall on the physical interface (`enp0s8`) blocks external scans, so the server is invisible from the public network.

**Act 2 — Demonstrate P2P mesh VPN access (success)**

1. Open a new browser tab.
2. Navigate to Laptop 2's **WireGuard (VPN) IP**, e.g.:
   `http://10.0.0.2/Data_Penting_Perusahaan.pdf`
3. Show the evaluator that the PDF opens and can be downloaded.
4. Narrative: explain that access succeeds because Laptop 1 holds a valid key into the mesh tunnel, allowing it to reach the file over the private intranet IP.

**Act 3 — Demonstrate quantum resistance (Rosenpass)**

1. Leave the PDF open. Open a terminal on Laptop 1.
2. Run the forensic capture command:
   ```bash
   sudo tcpdump -i enp0s8 port 51820 -X
   ```
3. Show the evaluator the stream of random-looking characters filling the terminal.
4. Narrative: explain that despite the file being downloaded successfully, anyone intercepting the physical network interface sees only this randomized ciphertext — the ML-KEM algorithm in Rosenpass secures the data against future "store now, decrypt later" attacks by quantum computers.

### Stage 3 — Final Proof (Laptop 3)

To demonstrate enterprise-grade P2P architecture:

1. Have Teammate 2 (Laptop 3) open a browser.
2. Have them access Laptop 2's WireGuard IP: `http://10.0.0.2/Data_Penting_Perusahaan.pdf`.
3. Closing narrative: explain that Laptop 3 can access the same data directly, proving there is no single central server (no single point of failure) — every node can exchange data directly, hidden from the outside world, and protected by post-quantum cryptography.

---
---

# Panduan Setup Post-Quantum Mesh VPN (Bahasa Indonesia)

## Gambaran Umum

Dokumen ini menjelaskan proses instalasi, konfigurasi, dan demonstrasi untuk **Post-Quantum Secure Mesh VPN** yang dibangun menggunakan **WireGuard** dan **Rosenpass** (pertukaran kunci ML-KEM), lengkap dengan konfigurasi firewall **dark network**, *dashboard* pemantauan **Grafana/Prometheus**, serta validasi forensik lalu lintas jaringan menggunakan **tcpdump** dan **Wireshark**.

Setup ini melibatkan **tiga virtual machine Ubuntu (Laptop 1, Laptop 2, Laptop 3)**, yang masing-masing berperan sebagai *node* dalam topologi VPN *mesh* (point-to-point). Pastikan ketiga VM sudah menjalankan Ubuntu, mode jaringan diatur ke **Bridged Adapter**, dan dapat saling terhubung melalui jaringan lokal sebelum memulai.

| Node | Peran |
|---|---|
| Laptop 1 | Klien / host dashboard pemantauan / host penyadapan forensik |
| Laptop 2 | Server file rahasia (XAMPP) |
| Laptop 3 | Peer mesh kedua (membuktikan koneksi point-to-point / tanpa server pusat tunggal) |

---

## Langkah 1 — Instalasi Paket Dasar

**Lakukan di Laptop 1, 2, dan 3.**

Buka terminal di masing-masing VM Ubuntu dan jalankan perintah berikut untuk memasang WireGuard, alat pemantauan, dan dependensi Post-Quantum Cryptography (PQC).

```bash
# Perbarui sistem
sudo apt update && sudo apt upgrade -y

# Instal WireGuard, firewall, alat penyadapan paket, dan exporter
sudo apt install wireguard wireguard-tools ufw tcpdump prometheus-blackbox-exporter -y

# Instal Cargo (Rust) untuk memasang Rosenpass
sudo apt install cargo -y
cargo install rosenpass
```

---

## Langkah 2 — Membangun Dark Network (Firewall)

**Lakukan di Laptop 1, 2, dan 3.**

Langkah ini memenuhi syarat **Network Obfuscation** agar port API/VPN tidak terlihat oleh *scanner* publik.

```bash
# Blokir semua koneksi masuk secara default
sudo ufw default deny incoming

# Izinkan semua koneksi keluar
sudo ufw default allow outgoing

# Buka port SSH agar tidak terkunci dari akses remote
sudo ufw allow ssh

# Buka port khusus WireGuard
sudo ufw allow 51820/udp

# Aktifkan firewall
sudo ufw enable
```

---

## Langkah 3 — Membuat Kunci WireGuard

**Lakukan di Laptop 1, 2, dan 3.**

Setiap *node* membutuhkan identitas kriptografi dasar sebelum Rosenpass menambahkan perlindungan pasca-kuantum.

```bash
cd /etc/wireguard/
umask 077
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey
```

> **Catatan:** Jalankan `cat publickey` di masing-masing laptop dan simpan hasilnya, karena kunci publik ini perlu ditukar antar *node* pada langkah berikutnya.

---

## Langkah 4 — Merakit Mesh Architecture

**Lakukan di Laptop 1, 2, dan 3.**

Buat file konfigurasi WireGuard:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Agar terbentuk arsitektur *mesh* yang sesungguhnya, setiap laptop harus memiliki blok `[Peer]` yang mengarah ke **dua laptop lainnya**. Contoh untuk **Laptop 1 (Node A — 10.0.0.1)**:

```toml
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [PRIVATE_KEY_LAPTOP_1]

[Peer]
# Koneksi ke Laptop 2
PublicKey = [PUBLIC_KEY_LAPTOP_2]
AllowedIPs = 10.0.0.2/32
Endpoint = [IP_WIFI_LAPTOP_2]:51820

[Peer]
# Koneksi ke Laptop 3
PublicKey = [PUBLIC_KEY_LAPTOP_3]
AllowedIPs = 10.0.0.3/32
Endpoint = [IP_WIFI_LAPTOP_3]:51820
```

Terapkan pola yang sama untuk Laptop 2 (`Address = 10.0.0.2/24`) dan Laptop 3 (`Address = 10.0.0.3/24`), sesuaikan endpoint dan public key agar mengarah ke dua peer lainnya.

---

## Langkah 5 — Mengizinkan Jalur Monitoring Exporter

**Lakukan HANYA di Laptop 2 dan Laptop 3.**

Langkah ini memungkinkan *dashboard* di Laptop 1 menarik data *ping* secara aman melalui terowongan VPN tanpa melanggar aturan *dark network*.

```bash
sudo ufw allow in on wg0 to any port 9115
```

---

## Langkah 6 — Instalasi Dashboard Grafana dan Prometheus

**Lakukan HANYA di Laptop 1.**

Pasang perangkat pemantauan untuk menampilkan *live latency metrics*.

```bash
# Instal Prometheus
sudo apt install prometheus -y

# Instal Grafana
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana -y

# Jalankan dan aktifkan Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

---

## Langkah 7 — Konfigurasi Target Prometheus

**Lakukan HANYA di Laptop 1.**

Arahkan Prometheus agar melakukan *ping* ke IP virtual VPN milik Laptop 2 dan Laptop 3.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Tambahkan blok berikut di baris paling bawah:

```yaml
- job_name: 'vpn_mesh_latency'
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets:
        - 10.0.0.2   # IP VPN Laptop 2
        - 10.0.0.3   # IP VPN Laptop 3
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:9115   # Blackbox exporter lokal
```

Restart Prometheus untuk menerapkan perubahan:

```bash
sudo systemctl restart prometheus
```

---

## Langkah 8 — Menyalakan PQC VPN

**Lakukan di Laptop 1, 2, dan 3.**

Nyalakan antarmuka jaringan dan biarkan Rosenpass mengamankan terowongan menggunakan kunci pasca-kuantum (ML-KEM).

```bash
# Nyalakan WireGuard
sudo wg-quick up wg0

# Buka port pertukaran Rosenpass
sudo ufw allow 9999/udp
```

Buat file konfigurasi Rosenpass:

```bash
sudo nano /etc/wireguard/rp.toml
```

Salin *template* berikut, lalu sesuaikan dengan *node* kalian (contoh untuk **Laptop 1 / Node A**):

```toml
# ==========================================
# KONFIGURASI ROSENPASS — LAPTOP 1 (NODE A)
# ==========================================
secret_key = "/home/eji/vpn-keys/rp-secret.key"
public_key = "/home/eji/vpn-keys/rp-public.key"
listen = ["0.0.0.0:9999"]

# ------------------------------------------
# KONEKSI ROSENPASS KE LAPTOP 2 (NODE B)
# ------------------------------------------
[[peers]]
# Ganti dengan isi file rp-public.key milik Laptop 2
public_key = "<RP_PUBLIC_KEY_LAPTOP_2>"
endpoint = "10.8.21.103:9999"
# Baris ini menyuntikkan kunci kuantum ke dalam WireGuard
wireguard = { dev = "wg0", peer = "AEWL3SByr3PHHFsL2AP1NW6dRox7iI3VtXBDg/Ydww8=" }

# ------------------------------------------
# KONEKSI ROSENPASS KE LAPTOP 3 (NODE C)
# ------------------------------------------
[[peers]]
# Ganti dengan isi file rp-public.key milik Laptop 3
public_key = "<RP_PUBLIC_KEY_LAPTOP_3>"
endpoint = "10.8.9.210:9999"
# Baris ini menyuntikkan kunci kuantum ke dalam WireGuard
wireguard = { dev = "wg0", peer = "+YKW2VvHBf1Bytq3frmW+fa9FoLI3gN+GU8c1S1t9xI=" }
```

Jalankan daemon pertukaran Rosenpass, yang akan membaca konfigurasi `wg0` dan melakukan enkapsulasi kunci PQC secara otomatis:

```bash
sudo rosenpass exchange-config /etc/wireguard/rp.toml
```

---

## Cara Menjalankan

### Bagian 1 — Membuat Grafik Latency di Grafana

1. **Login ke Grafana.** Di Laptop 1, buka `http://localhost:3000` melalui *browser*.
   - Username bawaan: `admin`
   - Password bawaan: `admin`
   - Grafana akan meminta kalian membuat *password* baru setelah login pertama kali.
2. **Hubungkan Grafana ke Prometheus.**
   - Di menu kiri, klik **Connections** (atau ikon roda gigi/Settings) → **Data Sources**.
   - Klik **Add data source** → pilih **Prometheus**.
   - Isi kolom URL dengan `http://localhost:9090`.
   - Scroll ke bawah dan klik **Save & Test** (notifikasi hijau menandakan berhasil).
3. **Membuat dashboard baru.**
   - Klik ikon **+ (Plus)** → pilih **Dashboard**.
   - Klik **Add a new panel**.
   - Ubah judul panel menjadi `PQC VPN Latency`.
4. **Masukkan query.** Pada kolom Metrics browser / Query, masukkan persis seperti ini:
   ```
   probe_duration_seconds{job="vpn_mesh_latency"}
   ```
   Tekan **Shift + Enter** atau klik **Run query**. Grafik garis kecepatan *ping* dari Laptop 1 ke Laptop 2 dan Laptop 3 akan muncul.
5. **Percantik tampilan.**
   - Pada pengaturan **Graph styles**, perbesar **Line width** agar terlihat jelas.
   - Pada **Standard options → Unit**, ubah menjadi **Time → seconds** atau **milliseconds**.
   - Klik **Save** di pojok kanan atas.

### Bagian 2 — Mengeksekusi tcpdump (Bukti Forensik)

Langkah ini krusial untuk membuktikan bahwa lalu lintas jaringan benar-benar dikunci oleh WireGuard dan enkripsi *Post-Quantum* Rosenpass, bukan teks biasa HTTP/TCP.

**Perhatian:** Lakukan penyadapan pada **kartu jaringan WiFi/LAN asli**, bukan pada `wg0`. Menyadap di `wg0` akan menampilkan data yang sudah didekripsi; menyadap di kartu jaringan asli menampilkan bentuk *ciphertext* saat data melintas di udara.

1. Cari nama kartu jaringan asli:
   ```bash
   ip a
   ```
   Cari *interface* yang memiliki IP lokal WiFi kalian (misalnya `192.168.x.x`), biasanya bernama `eth0`, `enp0s3`, `wlan0`, atau `wlp2s0`.
2. Jalankan perintah penyadapan pada *interface* tersebut (ganti `enp0s3` sesuai nama kartu jaringan kalian):
   ```bash
   sudo tcpdump -i enp0s3 port 51820 -X
   ```
   - `-i enp0s3` — mengarahkan penyadapan hanya ke kartu jaringan ini
   - `port 51820` — memfilter agar hanya menyadap lalu lintas WireGuard/PQC VPN
   - `-X` — menampilkan isi (*payload*) paket data dalam bentuk heksadesimal dan ASCII
3. Saat `tcpdump` berjalan, biarkan ada data yang mengalir (misalnya dashboard Grafana tetap menyala, atau jalankan `ping 10.0.0.2` dari terminal lain). Kolom *payload* seharusnya tidak menampilkan teks yang bisa dibaca (tidak ada `GET / HTTP`, nama file, atau pesan) — hanya karakter acak. Output ini adalah bukti bahwa data yang melintas sudah berupa *ciphertext* yang diamankan secara pasca-kuantum.

### Opsional — Memvisualisasikan dengan Wireshark

#### Apa yang akan divisualisasikan

| Elemen | Cara memfilter di Wireshark | Apa yang terlihat | Arti visualisasi |
|---|---|---|---|
| Terowongan WireGuard | `udp.port == 51820` | Ribuan paket UDP | Membuktikan VPN berjalan di port rahasia |
| Enkripsi data (ciphertext) | Cek panel data pada antarmuka WiFi | Teks heksadesimal dan simbol acak | Membuktikan enkripsi Post-Quantum berhasil membungkus data asli |
| Lalu lintas asli (plaintext) | Menyadap antarmuka `wg0` (via SSHDump) | Kata kunci terbaca jelas seperti `Echo (ping) request` | Membuktikan VPN tidak merusak data dan berhasil mengantarkannya utuh |

#### Langkah eksekusi

1. **Siapkan pemicu data:** di VM Ubuntu Laptop 1, jalankan `ping 10.0.0.2` dan biarkan berjalan terus-menerus.
2. **Buka Wireshark** dan pilih kartu jaringan aktif (Wi-Fi atau Ethernet).
3. **Filter jalur VPN:** pada kolom *Apply a display filter*, ketik `udp.port == 51820` lalu tekan Enter. Klik salah satu paket dan periksa *raw data* — isinya seharusnya berupa simbol acak yang tidak terbaca.
4. **Pindah ke skenario internal (SSHDump):**
   - Hentikan penyadapan yang sedang berjalan.
   - Buka menu **Capture** → **Options**.
   - Pada opsi **SSH remote capture: sshdump**, klik ikon **Roda Gigi (Settings)**.
5. **Konfigurasi SSHDump:**
   - Tab **Server:** isi IP lokal WiFi dari VM Ubuntu target dan port `22`.
   - Tab **Authentication:** isi *username* dan *password* login Ubuntu.
   - Tab **Capture:** pada kolom *Remote interface*, ketik `wg0`.
   - Klik **Save**, centang entri *sshdump* tersebut, lalu klik **Start**.
6. **Visualisasikan hasil dekripsi:** ketik filter `icmp` lalu tekan Enter. Baris data `Echo (ping) request` dan `Echo (ping) reply` akan terlihat dengan payload yang terbaca — membuktikan terowongan berfungsi sempurna secara end-to-end.

---

## Skrip Presentasi / Demo

### Pembagian Peran

- **Laptop 1 (milik Anda):** Bertindak sebagai klien yang mengakses file, sekaligus sebagai stasiun penyadapan forensik (`tcpdump`).
- **Laptop 2 (Teman 1):** Bertindak sebagai "server rahasia perusahaan" — menjalankan XAMPP dan menyimpan file contoh.
- **Laptop 3 (Teman 2):** Bertindak sebagai klien cabang lain, membuktikan bahwa jaringan ini benar-benar *mesh*, bukan sekadar koneksi dua komputer.

### Tahap 1 — Persiapan (Laptop 2, server file)

Lakukan langkah ini sebelum sesi penilaian dimulai.

**1. Menyiapkan dokumen umpan**

Buat sebuah file PDF bernama `Data_Penting_Perusahaan.pdf`.

**2. Menyalakan XAMPP (Apache)**

- Buka aplikasi **XAMPP Control Panel**.
- Klik **Start** pada baris **Apache**, pastikan statusnya berjalan (berwarna hijau).
- Buka *File Manager*, navigasikan ke direktori instalasi XAMPP (misalnya `/opt/lampp/htdocs/`).
- Salin file `Data_Penting_Perusahaan.pdf` ke dalam folder `htdocs` tersebut.

**3. Membangun dark network (konfigurasi UFW)**

Jalankan perintah berikut secara berurutan di Laptop 2:

```bash
# Menutup akses langsung dari jaringan kampus/publik
sudo ufw deny in on enp0s8 to any port 80

# Membuka akses hanya melalui terowongan VPN
sudo ufw allow in on wg0 to any port 80

# Mengaktifkan firewall
sudo ufw enable
```

Pemeriksaan opsional:

```bash
sudo ufw status
```

Pastikan status `enp0s8` adalah `DENY` dan `wg0` adalah `ALLOW`.

### Tahap 2 — Eksekusi Presentasi (Laptop 1)

**Babak 1 — Mendemonstrasikan mode siluman (gagal)**

1. Buka Google Chrome di Laptop 1.
2. Ketikkan **IP WiFi publik** milik Laptop 2, diikuti nama file, contoh:
   `http://192.168.1.50/Data_Penting_Perusahaan.pdf`
3. Tunjukkan ke dosen bahwa *browser* terus *loading* dan akhirnya gagal (*time out*).
4. Narasi: jelaskan bahwa firewall UFW pada antarmuka fisik (`enp0s8`) memblokir pemindaian dari luar, sehingga server tidak terlihat dari jaringan publik.

**Babak 2 — Mendemonstrasikan akses P2P Mesh VPN (berhasil)**

1. Buka *tab* baru di Chrome.
2. Ketikkan **IP WireGuard (VPN)** milik Laptop 2, contoh:
   `http://10.0.0.2/Data_Penting_Perusahaan.pdf`
3. Tunjukkan ke dosen bahwa file PDF terbuka dan bisa diunduh.
4. Narasi: jelaskan bahwa akses berhasil karena Laptop 1 memiliki kunci masuk ke dalam terowongan mesh, sehingga dapat mengakses file melalui IP intranet.

**Babak 3 — Mendemonstrasikan kekebalan kuantum (Rosenpass)**

1. Biarkan PDF tetap terbuka. Buka terminal di Laptop 1.
2. Jalankan perintah penyadapan forensik:
   ```bash
   sudo tcpdump -i enp0s8 port 51820 -X
   ```
3. Tunjukkan ke dosen bahwa layar terminal dipenuhi teks acak yang bergerak cepat.
4. Narasi: jelaskan bahwa meskipun file berhasil diunduh melalui *browser*, siapa pun yang menyadap antarmuka fisik jaringan hanya akan melihat data acak ini — algoritma ML-KEM pada Rosenpass melindungi data dari teknik dekripsi *"store now, decrypt later"* oleh komputer kuantum di masa depan.

### Tahap 3 — Pukulan Pamungkas (Laptop 3)

Untuk membuktikan arsitektur P2P tingkat *enterprise*:

1. Minta Teman 2 (Laptop 3) membuka *browser*.
2. Minta mereka mengakses IP WireGuard Laptop 2: `http://10.0.0.2/Data_Penting_Perusahaan.pdf`.
3. Narasi penutup: jelaskan bahwa Laptop 3 juga bisa mengakses data tersebut secara langsung. Dalam arsitektur P2P mesh, tidak ada server pusat tunggal (*no single point of failure*) — setiap *node* dapat saling bertukar data secara langsung, tertutup dari dunia luar, dan terlindungi secara kriptografi kuantum
