# AI Challenge — Output Awal

Berikut draf awal usulan arsitektur yang perlu dinilai, bukan instruksi migrasi yang otomatis benar.

## Usulan struktur

```text
lib/
├── app/
│   ├── app.dart
│   └── router.dart
├── core/
│   ├── errors/
│   │   └── failures.dart
│   ├── network/
│   │   ├── dio_client.dart
│   │   └── api_error_mapper.dart
│   └── providers.dart
├── features/
│   ├── auth/
│   │   ├── data/
│   │   │   ├── datasources/auth_remote_data_source.dart
│   │   │   ├── datasources/secure_token_data_source.dart
│   │   │   └── repositories/auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── entities/auth_session.dart
│   │   │   └── repositories/auth_repository.dart
│   │   └── presentation/
│   │       ├── pages/login_page.dart
│   │       └── providers/auth_providers.dart
│   └── announcements/
│       ├── data/
│       │   └── datasources/announcement_remote_data_source.dart
│       ├── domain/
│       │   └── entities/announcement.dart
│       └── presentation/
│           └── pages/announcement_page.dart
├── features/home/presentation/pages/home_page.dart
└── main.dart
```

## Pemetaan file lama

| File lama | Tujuan yang diusulkan | Tindakan |
|---|---|---|
| `lib/main.dart` | `lib/main.dart` dan bootstrap aplikasi | Pertahankan entry point; isolasi inisialisasi Firebase, notification handler, dan provider scope ke bootstrap bila bootstrap bertambah kompleks. |
| `lib/routes.dart` | `lib/app/app_routes.dart` | Pindahkan konstanta route; router GoRouter berada di `lib/app/router.dart`. |
| `lib/data/api_client.dart` | `lib/core/network/dio_client.dart` atau auth data network | Pindahkan konfigurasi Dio dan interceptor. Hindari core yang bergantung pada fitur auth; suntikkan callback refresh/token ke interceptor. |
| `lib/data/api_errors.dart` | `lib/core/network/api_error_mapper.dart` atau auth data | Pertahankan pemetaan Dio-specific di luar domain; konversikan ke failure domain pada batas repository. |
| `lib/data/auth_repository.dart` | `features/auth/domain/repositories/auth_repository.dart`, `features/auth/domain/entities/auth_session.dart`, dan `features/auth/data/repositories/auth_repository_impl.dart` | Pecah kontrak, entity session, dan implementasi simulasi/API. Jangan menyebut implementasi simulasi sebagai API sungguhan. |
| `lib/data/token_store.dart` | `features/auth/data/datasources/secure_token_data_source.dart` | Pindahkan implementasi `FlutterSecureStorage` ke data. Bila domain perlu mengendalikan token, expose abstraksi dari domain; jangan impor secure-storage package ke domain. |
| `lib/providers/auth_provider.dart` | `features/auth/presentation/providers/auth_providers.dart` | Pindahkan state notifier. Provider menjadi composition point untuk repository dan use case. |
| `lib/pages/login_page.dart` | `features/auth/presentation/pages/login_page.dart` | Pindahkan UI; pertahankan validasi/form behavior, tanpa membuat repository atau client HTTP. |
| `lib/pages/home_page.dart` | `features/home/presentation/pages/home_page.dart` | Pindahkan layar home; navigasi menggunakan route feature. |
| `lib/pages/announcement_page.dart` | `features/announcements/presentation/pages/announcement_page.dart` | Pindahkan halaman. Data saat ini hard-coded, jadi jangan membuat repository/API sampai sumber data pengumuman benar-benar ada. |
| `lib/pages/debug_fcm_page.dart` | `features/announcements/presentation/pages/debug_fcm_page.dart` atau tooling debug | Pindahkan hanya bila halaman tetap diperlukan; jika hanya untuk debugging lokal, jangan ikut ke production navigation. |
| `lib/messaging/push_service.dart` | FCM adapter/service di announcements data, dengan koordinasi navigasi di app/presentation | Pisahkan integrasi Firebase/local notification dari routing UI; handler background tetap top-level. |

`lib/widgets/` tidak memiliki file pada project saat ini sehingga tidak ada file yang dipetakan dari folder tersebut. `AuthSession` saat ini hanya value object di file repository; ia belum memiliki mapping JSON. `buildApiClient` belum digunakan.

## Aturan use case dan DI

- Login dan refresh melibatkan kebijakan autentikasi/token sehingga dapat berada di `Login` dan `RefreshSession` use case bila aturan tersebut berkembang. Jika use case hanya meneruskan satu pemanggilan repository, memilih repository langsung dari notifier juga dapat dibenarkan.
- Jangan menambah use case untuk getter/setter token atau operasi CRUD satu baris hanya demi jumlah file.
- Jangan membuat announcement repository/use case/data source untuk halaman statis sebelum ada sumber data dan perilaku bisnis yang perlu diuji.
- Provider Riverpod merangkai secure token data source, auth repository implementation, use case, notifier, dan layanan FCM. Domain hanya melihat interface/entity Dart murni.
- `Dio` dan secure storage tidak boleh diimpor domain. FCM background service tidak boleh memanggil widget atau `BuildContext`.

Contoh wiring berikut menunjukkan arah dependensi (tipe adapter/use case adalah target usulan, bukan file yang sudah ada):

```dart
final tokenDataSourceProvider = Provider<TokenDataSource>((ref) {
  return SecureTokenDataSource();
});

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepositoryImpl(
    tokenDataSource: ref.watch(tokenDataSourceProvider),
    remoteDataSource: ref.watch(authRemoteDataSourceProvider),
  );
});

final loginProvider = Provider<Login>((ref) {
  return Login(ref.watch(authRepositoryProvider));
});

final authStateProvider =
    AsyncNotifierProvider<AuthNotifier, bool>(AuthNotifier.new);
```

Wiring lengkap hanya boleh ditambahkan setelah kontrak dan implementasinya benar-benar dibuat. Jika backend belum tersedia, pertahankan remote data source simulasi yang sudah ada. Ketika auth API kelak menggunakan Dio, gunakan client login/refresh yang tidak memasang interceptor refresh dirinya sendiri; client terautentikasi boleh mendelegasikan refresh melalui callback agar tidak membentuk siklus dependency `Dio -> AuthRepository -> Dio`.

## Catatan penilaian draf

Draf struktur di atas adalah kandidat, bukan hasil final. Sumber kode saat ini tidak membuktikan klaim "widget memanggil Dio langsung" atau "daftar pengumuman berasal dari API". Keputusan final dalam `ai-decisions-and-verification.md` membatasi migrasi agar tidak menambahkan layer data palsu untuk fitur yang masih statis.
