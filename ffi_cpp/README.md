# ey_rsrapsrs/ffi_cpp
q:
program Saya ini nantinya akan dipanggil oleh kode program bertipe **C++** menggunakan compiler clang++

<br>

a-ai:
Jika program Rust Anda akan dipanggil oleh **C++ menggunakan Clang++**, integrasinya akan menjadi sangat mulus. Clang++ memiliki dukungan yang sangat baik untuk berinteraksi dengan pustaka (pustaka dinamis `.so` atau statis `.a`) yang dibuat dengan Rust.

Berikut adalah beberapa penyesuaian dan poin penting agar kode Rust Anda dapat dipanggil dengan aman dan optimal dari sisi C++:
1. File Header C++ (`libnixhelpers.hpp`)
Agar Clang++ tahu cara memanggil fungsi Rust Anda, Anda perlu membungkus deklarasi fungsinya di dalam blok `extern "C"` di C++. Ini mencegah Clang++ melakukan _name mangling_ (perubahan nama fungsi secara internal).

Buat file header baru, misalnya `libnixhelpers.hpp`:
```cpp
#pragma once

// Beritahu kompiler C++ bahwa fungsi ini menggunakan konvensi pemanggilan C
extern "C" {
    /**
     * @brief Membuka berkas dan menampilkan ukurannya via lseek.
     * @param pathname Pointer ke string C (const char*) yang valid.
     */
    void contoh_fungsi(const char* pathname);
}

```

2. Cara Memanggil dari C++ (`main.cpp`)
Di dalam kode C++, Anda cukup memasukkan file header tersebut dan memanggil fungsinya seperti fungsi C/C++ biasa:
```cpp
#include <iostream>
#include "libnixhelpers.hpp"

int main() {
    std::cout << "[C++] Memulai program..." << std::endl;

    // Memanggil fungsi Rust dengan string C++ (.c_str())
    std::string target_file = "test.txt";
    contoh_fungsi(target_file.c_str());

    // Uji coba penanganan keselamatan jika pointer null
    std::cout << "[C++] Menguji pointer null..." << std::endl;
    contoh_fungsi(nullptr); 

    return 0;
}

```

3. Cara Kompilasi dengan Clang++
Pastikan Anda sudah melakukan `cargo build --release` di Rust terlebih dahulu. Setelah file `.so` atau `.a` terbentuk di folder `target/release/`, Anda bisa mengompilasinya dengan Clang++.

Opsi A: Menggunakan Dynamic Library (`.so`)
Jika di `Cargo.toml` Anda menggunakan `crate-type = ["cdylib"]`:
```bash
clang++ main.cpp -L./target/release -lnixhelpers $(cargo rustc -- --print native-static-libs 2>&1 | grep "native-static-libs:" | sed 's/note: native-static-libs: //') -o main_program
```
> _(Catatan: Saat menjalankan `main_program`, pastikan path `.so` terdaftar di `LD_LIBRARY_PATH` Termux Anda)._

Opsi B: Menggunakan Static Library (`.a`) — _Sangat Direkomendasikan_
Jika di `Cargo.toml` Anda menggunakan `crate-type = ["staticlib"]`. Cara ini menggabungkan semua kode Rust langsung ke dalam file _executable_ C++, sehingga hasilnya mandiri dan tidak ketergantungan file luar:
```bash
clang++ main.cpp -L./target/release -lnixhelpers $(cargo rustc -- --print native-static-libs 2>&1 | grep "native-static-libs:" | sed 's/note: native-static-libs: //') -o main_program
```

💡 Tips Keamanan Tambahan untuk C++
-   Gunakan `std::string::c_str()`: Di sisi C++, pastikan Anda selalu melewatkan string menggunakan `.c_str()`, bukan melewatkan objek `std::string` secara langsung. Fungsi Rust Anda mengekspektasikan `*const c_char` (string berakhiran null), bukan objek string C++.
-   Atasi Duplikasi Log: Baik Rust (`println!`) maupun C++ (`std::cout`) menggunakan buffer internal yang berbeda untuk mencetak teks ke layar. Jika urutan teks di terminal terlihat acak-acakan (teks Rust muncul duluan atau belakangan), tambahkan `std::flush` atau `std::endl` di C++ sebelum memanggil fungsi Rust.

<br>

q:
cara memastikan path `.so` terdaftar di `LD_LIBRARY_PATH` Termux Saya, Sebelum menjalankan `program_utama` ?

<br>

a-ai:
Untuk memastikan path file `.so` terdaftar di `LD_LIBRARY_PATH` Termux sebelum menjalankan `program_utama`, Anda memiliki dua pilihan: **cara sementara** (hanya berlaku untuk sesi terminal aktif saat ini) atau **cara permanen** (otomatis aktif setiap kali Anda membuka Termux).

Berikut adalah cara-cara yang bisa Anda gunakan:

Opsi 1: Cara Sementara (Direkomendasikan untuk Uji Coba)
Anda bisa mengekspor _path_ langsung di terminal sebelum mengeksekusi program. Jalankan perintah berikut di dalam folder proyek Anda:
```bash
export LD_LIBRARY_PATH=$PREFIX/lib:$(pwd)/target/release
```
-   `$PREFIX/lib`: Harus tetap dimasukkan agar Termux tidak kehilangan akses ke library bawaan sistemnya.
-   `$(pwd)/target/release`: Otomatis mengambil path folder rilis Rust Anda saat ini.
Setelah itu, Anda bisa langsung menjalankan program:
```bash
./program_utama
```
Opsi 2: Sekali Jalan (One-Liner)
Jika Anda tidak ingin mengubah variabel lingkungan terminal, Anda bisa memasukkan `LD_LIBRARY_PATH` tepat di depan perintah eksekusi program:
```bash
LD_LIBRARY_PATH=$PREFIX/lib:$(pwd)/target/release ./program_utama
```
Opsi 3: Cara Permanen (Tidak Perlu Ketik Ulang)
Jika Anda sering menguji coba program ini, Anda bisa memasukkan konfigurasi tersebut ke dalam file profil shell Termux Anda (`.bashrc` atau `.zshrc`).
1. Buka file konfigurasi menggunakan text editor (misalnya `nano`):
```bash
nano ~/.bashrc
```
2. Tambahkan baris berikut di bagian paling bawah file (ganti `/absolute/path/to/` dengan path asli folder proyek Anda):
```bash
export LD_LIBRARY_PATH=$PREFIX/lib:/data/data/com.termux/files/home/ey_rsrapsrs/nix/libnixhelpers/target/release
```
3. Simpan file (di Nano: tekan `Ctrl + O`, lalu `Enter`, kemudian `Ctrl + X` untuk keluar).
4. Perbarui sesi terminal Anda saat ini dengan perintah
```bash
source ~/.bashrc
```

💡 Alternatif Cerdas: Trik Kompilasi Tanpa `LD_LIBRARY_PATH`
Jika Anda tidak ingin repot mengatur `LD_LIBRARY_PATH` saat menjalankan program, Anda bisa menyisipkan lokasi library `.so` langsung ke dalam file _executable_ program C++ saat proses kompilasi menggunakan bendera `-rpath`.

Kompilasi dengan cara ini di Clang++:

```bash
clang++ main.cpp -L./target/release -lnixhelpers -Wl,-rpath,'$ORIGIN/target/release' -o program_utama
```
Dengan benderan `-Wl,-rpath,'$ORIGIN/target/release'`, program C++ akan otomatis mencari file `.so` di folder relatif `target/release` tanpa membutuhkan `LD_LIBRARY_PATH` lagi.









<br>
