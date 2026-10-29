# AI Challenge — Keputusan dan Verifikasi

## 1. Keputusan final

Audit dilakukan pada project Minggu 6 yang ada di `06-week-6-authentication-security-fcm/campus_notify`. Dokumen ini adalah usulan refactor; kode `campus_notify` belum dipindahkan oleh AI Challenge ini.

| Area/file lama | Usulan awal | Keputusan final | Alasan teknis |
|---|---|---|---|
| `data/auth_repository.dart` | Pecah menjadi entity, interface domain, data source, dan implementasi | Pisahkan `AuthSession` ke `features/auth/domain/entities/`; letakkan interface `AuthRepository` di domain dan `AuthRepositoryImpl` di data. Implementasi boleh tetap memakai simulasi sampai backend tersedia. | Kontrak domain tidak mengikat UI pada backend, tetapi jangan mengklaim API real yang belum ada. |
| Login/refresh | Tambahkan use case `Login` dan `RefreshSession` | Use case dipakai bila memuat validasi/aturan sesi atau mengoordinasikan repository. Untuk operasi pass-through sederhana, notifier boleh memanggil interface repository langsung dan alasannya dicatat. | Tidak membuat kelas per operasi tanpa kebutuhan bisnis yang nyata. Refresh token dan aturan 401 lebih kompleks daripada CRUD satu baris. |
| `data/token_store.dart` | Pindahkan ke auth data source | Jadikan adapter secure storage di `features/auth/data/datasources/`. Abstraksi token hanya ditambahkan pada domain jika dibutuhkan oleh kebijakan bisnis atau lebih dari satu adapter. | Domain tidak boleh tahu package `flutter_secure_storage`; hindari interface tambahan yang tidak memberi substitusi/testing berguna. |
| `data/api_client.dart` | Pindahkan Dio ke core network | Tempatkan client/interceptor pada infrastructure/data dan injeksikan token reader serta refresh callback. Gunakan client terpisah untuk refresh bila perlu menghindari interceptor rekursif. | `buildApiClient` sekarang belum dipakai. Jangan mengaktifkan interceptor atau menambah endpoint sebagai bagian dari pemindahan folder saja. |
| `data/api_errors.dart` | Pindahkan pemetaan error ke core | Pertahankan mapper yang bergantung Dio di luar domain; repository menerjemahkan error teknis menjadi failure domain bila kontrak mulai memakai `Failure`. | Domain tidak boleh import Dio. Mapper global baru masuk akal bila beberapa fitur menggunakannya. |
| `pages/login_page.dart` | Pindahkan ke auth presentation | Pindahkan UI ke `features/auth/presentation/pages/`; UI berinteraksi melalui notifier/provider. | Memisahkan UI dari implementasi tanpa mengubah alur login. |
| `pages/home_page.dart` | Pindahkan ke home presentation | Pindahkan ke `features/home/presentation/pages/`. | Home adalah feature/shell presentation, bukan bagian repository auth. |
| `pages/announcement_page.dart` | Tambah entity, datasource, repository, use case | Untuk saat ini pindahkan hanya ke `features/announcements/presentation/pages/`; pertahankan isi statis. Tambah domain/data hanya saat announcement berasal dari sumber nyata atau memiliki aturan bisnis. | Repository fiktif untuk konten hard-coded adalah over-engineering dan berisiko mengarang behavior/API. |
| `messaging/push_service.dart` | Jadikan service/data adapter FCM dengan koordinasi route di app | Pisahkan adapter Firebase/local notification dari penanganan navigasi. Background callback tetap fungsi top-level; callback memberi event/route ke router melalui batas aplikasi yang terkontrol. | FCM SDK tidak masuk domain dan background isolate tidak boleh bergantung pada widget, `BuildContext`, atau provider UI yang belum diinisialisasi. |
| `pages/debug_fcm_page.dart` | Pindahkan ke announcements presentation | Pindahkan hanya bila dibutuhkan di build/demo; jika khusus debug, gate dari navigasi/konfigurasi production. | Hindari mencampur tool debug dengan fitur pengguna tanpa alasan. |
| `routes.dart`, `main.dart` | Buat `app/router.dart` dan bootstrap | Pindahkan route names/router ke app layer; pertahankan `main.dart` sebagai entry point dan isolasi startup Firebase/FCM jika semakin kompleks. | Startup FCM, auth guard, serta router harus tetap punya satu composition root yang jelas. |

### Kesimpulan trade-off

Gunakan struktur feature-first untuk auth dan announcements karena keduanya punya perubahan dan dependensi berbeda. Namun, jangan membangun lapisan announcement data/use case sebelum fitur statis tersebut benar-benar mengambil data. Interface repository, bila dibuat, tinggal di domain; implementasinya tinggal di data. Entity tetap Dart murni dan tidak memuat `toMap`/`fromJson`.

## 2. Pemetaan semua file sumber saat ini

Pemetaan ini berdasarkan file yang benar-benar ditemukan di `campus_notify/lib`:

| File saat ini | Tujuan |
|---|---|
| `main.dart` | Tetap menjadi entry point; startup dapat diekstrak ke bootstrap aplikasi. |
| `routes.dart` | `app/app_routes.dart` untuk konstanta route. |
| `data/api_client.dart` | Infrastruktur/data network; belum dipakai oleh file lain menurut pencarian `buildApiClient(`. |
| `data/api_errors.dart` | Mapper Dio di data/infrastructure; domain menerima failure netral bila kelak diperlukan. |
| `data/auth_repository.dart` | Dipecah menjadi `AuthSession` dan interface domain, serta implementasi repository data. |
| `data/token_store.dart` | Auth data source untuk secure storage. |
| `messaging/push_service.dart` | FCM/local-notification adapter; navigasi dikoordinasikan pada app/presentation. |
| `pages/announcement_page.dart` | `features/announcements/presentation/pages/`; saat ini hanya UI data hard-coded. |
| `pages/debug_fcm_page.dart` | Tool/debug page, atau dihapus hanya bila memang tidak diperlukan setelah dikonfirmasi pemilik project. |
| `pages/home_page.dart` | `features/home/presentation/pages/`. |
| `pages/login_page.dart` | `features/auth/presentation/pages/`. |
| `providers/auth_provider.dart` | `features/auth/presentation/providers/`; composition provider dapat ditata di sana atau app scope. |

Tidak ada `lib/widgets/` pada project saat ini.

## 3. Tiga pemeriksaan grep — hasil aktual sebelum migrasi

Semua pemeriksaan ini dijalankan pada project legacy Minggu 6, bukan diklaim sebagai bukti migrasi selesai. Hasilnya menunjukkan kondisi awal yang harus ditangani:

### Pemeriksaan 1 — akses data mentah di halaman

Perintah yang sesuai dengan struktur aktual (folder `lib/widgets` tidak ada):

```sh
rg -n "Dio\\(|openDatabase|getDatabasesPath|FlutterSecureStorage|SharedPreferences\\.getInstance|jsonDecode" lib/pages
```

**Hasil:** tidak ada kecocokan; exit code `1` (`lib/pages` tidak melakukan akses tersebut secara langsung).

### Pemeriksaan 2 — framework/plugin dalam layer non-domain lama

Project lama belum memiliki `lib/features/*/domain` atau `lib/core`, jadi pemeriksaan domain belum bisa membuktikan dependency rule. Sebagai audit batas awal, jalankan:

```sh
rg -n "import 'package:flutter|import 'package:dio|import 'package:sqflite|import 'package:firebase" lib/data lib/providers lib/messaging
```

**Hasil aktual:**

```text
lib/data/token_store.dart:1:import 'package:flutter_secure_storage/flutter_secure_storage.dart';
lib/data/api_client.dart:1:import 'package:dio/dio.dart';
lib/data/api_errors.dart:1:import 'package:dio/dio.dart';
lib/providers/auth_provider.dart:1:import 'package:flutter_riverpod/flutter_riverpod.dart';
lib/messaging/push_service.dart:1:import 'package:firebase_messaging/firebase_messaging.dart';
lib/messaging/push_service.dart:2:import 'package:flutter/foundation.dart';
lib/messaging/push_service.dart:3:import 'package:flutter_local_notifications/flutter_local_notifications.dart';
```

Ini bukan temuan domain karena folder domain belum ada; setelah migrasi, grep harus diarahkan ke domain/core dan tidak boleh menghasilkan import framework/plugin.

### Pemeriksaan 3 — mapping serialisasi dalam source lama

```sh
rg -n "toMap|fromMap|toJson|fromJson|jsonDecode" lib
```

**Hasil:** tidak ada kecocokan; exit code `1`. Source saat ini tidak memiliki mapping entity/model yang bisa dipindah. Jika model JSON ditambahkan saat API sungguhan tersedia, mapping harus tinggal di model/data source.

## 4. Koreksi prompt dan batas klaim

- Prompt awal mengatakan widget memanggil Dio, tetapi grep dan pemeriksaan import halaman tidak mendukung klaim tersebut. Tidak ada `lib/widgets/`; halaman login memakai Riverpod, sedangkan halaman announcement hanya menampilkan string statis.
- `buildApiClient` ditemukan hanya sebagai deklarasi, tanpa pemanggil. Sebelum mengintegrasikannya, tentukan apakah client tersebut memang bagian dari scope dan backend sudah tersedia.
- `AuthRepository.login` dan `.refresh` mensimulasikan respons. Migrasi arsitektur tidak mengubah simulasi menjadi auth server sungguhan.
- Checklist domain di atas adalah **belum diverifikasi setelah migrasi**, sebab migrasi campus_notify tidak termasuk perubahan kode pada AI Challenge ini.

## 5. Perintah verifikasi setelah migrasi

Jalankan dari folder `06-week-6-authentication-security-fcm/campus_notify` setelah perubahan arsitektur benar-benar diterapkan:

```sh
# Presentation tidak mengakses plugin/data mentah
rg -n "Dio\\(|openDatabase|getDatabasesPath|FlutterSecureStorage|SharedPreferences\\.getInstance|jsonDecode" lib/features/*/presentation

# Domain dan core tidak mengimpor framework atau SDK eksternal
rg -n "import 'package:(flutter|dio|sqflite|firebase)" lib/features/*/domain lib/core

# Entity tidak memuat serialisasi dan mapping
rg -n "toMap|fromMap|toJson|fromJson|jsonDecode" lib/features/*/domain

flutter analyze
flutter test
```

Temuan grep awal di atas bukan bukti refactor selesai. Setelah migrasi, catat ulang output aktual serta uji alur login, logout/refresh, navigasi notifikasi foreground/background/terminated, dan halaman announcement. Screenshot hanya ditambahkan setelah benar-benar diambil dari aplikasi.
