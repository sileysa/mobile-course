# AI Challenge — Prompt

Prompt yang digunakan untuk meminta draf arsitektur:

```text
Project Flutter saya: campus_notify (auth + FCM + daftar pengumuman).
Kondisi kini: folder lib/{data, providers, pages, messaging},
repository tercampur dengan implementasi, widget memanggil Dio langsung.
Tugas:
1. Usulkan struktur feature-first Clean Architecture
   (presentation/domain/data) untuk fitur auth + announcements.
2. Untuk tiap file lama, sebutkan tujuan barunya (pindah/pecah/hapus).
3. Tandai bagian yang over-engineering bila diterapkan ke CRUD sederhana,
   dan kapan use case benar-benar dibutuhkan vs repository langsung.
4. Tunjukkan wiring DI dengan Riverpod (tanpa package DI tambahan).
Jelaskan trade-off setiap keputusan.
```

## Pemeriksaan sebelum menerima prompt

Prompt sengaja dipertahankan sebagai input challenge, tetapi deskripsi kondisi awal perlu diverifikasi terhadap repository:

- `lib/widgets/` tidak ada pada `campus_notify`.
- Tidak ada halaman yang mengimpor atau memanggil Dio secara langsung. `lib/pages/login_page.dart` memanggil `authStateProvider`; `lib/pages/announcement_page.dart` hanya menampilkan data statis.
- `buildApiClient` hanya dideklarasikan di `lib/data/api_client.dart` dan tidak memiliki pemanggil di `lib/`.
- `AuthRepository` saat ini adalah simulasi login/refresh, bukan repository yang sudah berbicara dengan backend.

Karena itu, output awal AI harus dikoreksi berdasarkan bukti kode. Jangan mengarang pemanggilan API pengumuman atau memindahkan widget yang tidak ada.
