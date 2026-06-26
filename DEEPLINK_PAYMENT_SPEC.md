# Spesifikasi Deeplink Pembayaran — Dompet Kampus Global

Dokumen ini menjelaskan format URL, konfigurasi platform, dan alur end-to-end agar aplikasi **Dompet Kampus Global** dapat menerima link pembayaran dari merchant atau aplikasi pihak ketiga (deeplink), menampilkan halaman konfirmasi pembayaran, serta menyelesaikan transaksi menggunakan verifikasi **PIN + 2FA** sesuai konfigurasi keamanan masing-masing pengguna.

---

## Daftar Isi

1. Format URL Deeplink
2. Parameter Deeplink
3. Validasi Parameter
4. Konfigurasi Platform

   * Android
   * iOS
   * Flutter Dependency
5. Komponen di Sisi Flutter
6. Alur End-to-End
7. Mapping 2FA
8. Contoh Link Lengkap
9. Testing
10. Skenario Pengujian
11. Catatan Keamanan

---

# 1. Format URL Deeplink

Aplikasi mendukung dua jenis deeplink:

| Tipe           | Format                             |
| -------------- | ---------------------------------- |
| Custom Scheme  | `dompetkampus://pay?...`           |
| HTTPS App Link | `https://dompetkampus.app/pay?...` |

### Custom Scheme

```text
dompetkampus://pay?
merchant_id=MCH001&
merchant_name=Kantin%20Kampus&
amount=25000&
description=Makan%20Siang&
reference=INV-2026-0001&
callback=https%3A%2F%2Fmerchant.example.com%2Freturn
```

### HTTPS App Link

```text
https://dompetkampus.app/pay?
merchant_id=MCH001&
merchant_name=Kantin%20Kampus&
amount=25000
```

---

# 2. Parameter Deeplink

| Parameter     | Wajib | Tipe   | Keterangan                                                                     |
| ------------- | ----- | ------ | ------------------------------------------------------------------------------ |
| merchant_id   | Ya    | string | ID unik merchant. Tidak boleh kosong.                                          |
| merchant_name | Ya    | string | Nama merchant yang ditampilkan ke user.                                        |
| amount        | Ya    | number | Nominal pembayaran dalam Rupiah. Harus lebih besar dari 0.                     |
| description   | Tidak | string | Keterangan transaksi.                                                          |
| reference     | Tidak | string | Nomor referensi atau invoice merchant.                                         |
| callback      | Tidak | URL    | URL callback merchant. Saat ini hanya disimpan dan tidak di-redirect otomatis. |

### Contoh Nilai

```text
merchant_id=MCH001
merchant_name=Kantin Kampus
amount=25000
description=Makan Siang
reference=INV-001
```

---

# 3. Validasi Parameter

Parsing dilakukan oleh:

```text
lib/core/services/deeplink_service.dart
```

melalui:

```dart
DeeplinkPaymentData.fromUri()
```

Validasi yang dilakukan:

* `merchant_id` wajib ada
* `merchant_id` tidak boleh kosong
* `merchant_name` wajib ada
* `merchant_name` tidak boleh kosong
* `amount` wajib ada
* `amount` harus berupa angka
* `amount` harus lebih besar dari 0

Jika validasi gagal, aplikasi akan membuka halaman `/pay` dalam mode error dan menampilkan pesan yang sesuai dalam Bahasa Indonesia.

---

# 4. Konfigurasi Platform

## Android

File:

```text
android/app/src/main/AndroidManifest.xml
```

Tambahkan intent-filter berikut pada `MainActivity`.

### Custom Scheme (Deep Link (dompetkampus://pay?...))

```xml
<intent-filter android:autoVerify="false">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <data
        android:scheme="dompetkampus"
        android:host="pay"/>
</intent-filter>
```

### HTTPS App Link (https://dompetkampus.app/pay?... -->)

```xml
<intent-filter android:autoVerify="false">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <data
        android:scheme="https"
        android:host="dompetkampus.app"
        android:pathPrefix="/pay"/>
</intent-filter>
```

### Production App Link

Agar App Link HTTPS aktif:

1. Hosting file assetlinks.json:

```text
https://dompetkampus.app/.well-known/assetlinks.json
```

2. Ubah:

```xml
android:autoVerify="false"
```

menjadi

```xml
android:autoVerify="true"
```

3. Reinstall APK.

---

## iOS

File:

```text
ios/Runner/Info.plist
```

Tambahkan:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.kampus.dompetKampusGlobal.payment</string>

        <key>CFBundleURLSchemes</key>
        <array>
            <string>dompetkampus</string>
        </array>
    </dict>
</array>
```

### Catatan

Saat ini iOS hanya mendukung:

```text
dompetkampus://
```

Jika ingin menggunakan HTTPS App Link di iOS, perlu konfigurasi:

* Associated Domains
* `applinks:dompetkampus.app`
* `apple-app-site-association`

---

## Flutter Dependency

Tambahkan package:

```yaml
dependencies:
  app_links: ^6.3.2
```

---

# 5. Komponen di Sisi Flutter

| File                                                      | Peran                                    |
| --------------------------------------------------------- | ---------------------------------------- |
| lib/core/services/deeplink_service.dart                   | Parser, validator, dan listener deeplink |
| lib/core/router/app_router.dart                           | Routing `/pay`                           |
| lib/presentation/pages/payment/payment_deeplink_page.dart | Halaman konfirmasi pembayaran            |
| lib/presentation/pages/payment/pin_page.dart              | PIN dan OTP/TOTP                         |
| lib/main.dart                                             | Inisialisasi DeeplinkService             |

---

# 6. Alur End-to-End

## 1. User membuka deeplink

```text
dompetkampus://pay?merchant_id=...&amount=...
```

## 2. Android/iOS meneruskan URI ke aplikasi

* Cold Start → `getInitialLink()`
* App Running → `uriLinkStream`

## 3. DeeplinkService memproses URI

Jika valid:

```dart
router.go('/pay', extra: data);
```

Jika tidak valid:

```dart
router.go('/pay', extra: errorMessage);
```

## 4. PaymentDeeplinkPage

Menampilkan:

* Merchant
* Nominal
* Deskripsi
* Referensi
* Informasi keamanan

Kemudian user menekan:

```text
Bayar Rp xxx
```

## 5. PinPage Step 1

User memasukkan PIN.

### Topup

Langsung submit.

### Transfer / Payment / Deeplink

Masuk ke Step 2.

## 6. PinPage Step 2 (2FA)

Metode dibaca dari:

```dart
SecureStorageDatasource.get2faMethod()
```

## 7. Submit OTP/TOTP

```dart
PaymentTransferRequested(
  amount,
  description,
  otpCode,
  otpType,
)
```

Request:

```http
POST /v1/payment/transfer
```

## 8. Respon Backend

### Berhasil

```http
200 OK
```

Navigasi:

```text
/success
```

### OTP Salah

```http
401 INVALID_OTP
```

### Saldo Tidak Cukup

```http
400 INSUFFICIENT_BALANCE
```

### Error Lain

Menampilkan pesan backend.

---

# 7. Mapping 2FA

| k2faMethod | UI                              | otp_type |
| ---------- | ------------------------------- | -------- |
| totp       | Input kode Google Authenticator | totp     |
| smtp       | Kirim OTP Email                 | email    |
| notif      | Kirim OTP Firebase              | firebase |

### Default

Jika user belum pernah memilih metode 2FA:

```text
totp
```

akan digunakan sebagai default.

---

# 8. Contoh Link Lengkap

## Custom Scheme

```text
dompetkampus://pay?merchant_id=MCH001&merchant_name=Kantin%20Kampus%20A&amount=25000&description=Makan%20Siang%20Paket%202&reference=INV-20260616-0001
```

## HTTPS App Link

```text
https://dompetkampus.app/pay?merchant_id=MCH002&merchant_name=Koperasi%20Mahasiswa&amount=15000
```

---

# 9. Testing

## Android

### Membuka Deeplink

```bash
adb shell am start -a android.intent.action.VIEW \
-d "dompetkampus://pay?merchant_id=MCH001&merchant_name=Kantin%20Kampus&amount=25000&description=Makan%20Siang&reference=INV-001"
```

### HTTPS App Link

```bash
adb shell am start -a android.intent.action.VIEW \
-d "https://dompetkampus.app/pay?merchant_id=MCH001&merchant_name=Kantin%20Kampus&amount=25000"
```

---

## iOS Simulator

```bash
xcrun simctl openurl booted \
"dompetkampus://pay?merchant_id=MCH001&merchant_name=Kantin%20Kampus&amount=25000&description=Makan%20Siang&reference=INV-001"
```

---

# 10. Skenario Pengujian

| No | Skenario                   | Hasil Yang Diharapkan           |
| -- | -------------------------- | ------------------------------- |
| 1  | Cold Start via Deeplink    | Langsung membuka halaman `/pay` |
| 2  | Deeplink saat app berjalan | Navigasi ke `/pay`              |
| 3  | amount kosong              | Halaman error                   |
| 4  | amount bukan angka         | Halaman error                   |
| 5  | merchant_id kosong         | Halaman error                   |
| 6  | 2FA TOTP                   | Input Authenticator             |
| 7  | 2FA SMTP                   | OTP Email terkirim              |
| 8  | 2FA Firebase               | OTP Push Notification terkirim  |
| 9  | OTP salah                  | Banner error + reset input      |
| 10 | Pembayaran berhasil        | Halaman success                 |

---

# 11. Catatan Keamanan

## Wajib Validasi di Backend

Semua data berikut harus divalidasi ulang oleh backend:

* merchant_id
* amount
* reference
* saldo user
* status akun

Jangan mengandalkan validasi frontend.

---

## Wajib PIN dan 2FA

Setiap transaksi pembayaran harus melewati:

1. Verifikasi PIN
2. Verifikasi OTP/TOTP

Tidak boleh ada bypass.

---

## Callback Tidak Di-Redirect Otomatis

Parameter:

```text
callback=
```

saat ini hanya disimpan pada payload.

Aplikasi **tidak melakukan redirect otomatis** ke URL callback untuk mencegah:

* Open Redirect
* Phishing
* URL Injection
* Pengalihan ke domain yang tidak dipercaya

Jika di masa depan callback diaktifkan, harus menggunakan whitelist domain merchant yang telah diverifikasi.

---

## Token Authorization

Pembayaran melalui deeplink harus tetap menggunakan JWT user yang tersimpan pada Secure Storage.

Jika aplikasi dibuka melalui deeplink saat kondisi cold start, token wajib dipulihkan sebelum request pembayaran dikirim ke backend.
