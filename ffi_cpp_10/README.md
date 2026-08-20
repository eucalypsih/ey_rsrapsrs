# ey_rsrapsrs/ffi_cpp_2


**membuat program C++ yang melakukan hal yang sama persis seperti yang dilakukan `index.js`**, yaitu:

1. Membaca *path* file biner `.so` secara dinamis menggunakan *Current Working Directory* (seperti `process.cwd()`).
2. Memuat (*load*) library `.so` tersebut secara dinamis saat program berjalan (*runtime FFI*), bukan saat kompilasi.
3. Mencari fungsi `contoh_fungsi` di dalam `.so` tersebut.
4. Menyiapkan buffer string biner manual dan mengeksekusinya.

Di C++ (terutama di Linux), teknik memuat library secara dinamis saat program berjalan ini menggunakan fungsi bawaan `<dlfcn.h>` (Dynamic Linking).

Berikut adalah perubahan total pada `main.cpp` untuk meniru 100% logika dinamis dari `index.js`:

Kode `main.cpp` Baru (Logika FFI Dinamis)
```cpp
#include <iostream>
#include <string>
#include <unistd.h>  // Untuk getcwd()
#include <dlfcn.h>   // Untuk dlopen, dlsym, dlclose (Fungsi FFI milik C++)

// 1. Definisikan tipe fungsi (Signature) yang sama seperti di Rust/C
typedef void (*ContohFungsiPtr)(const char*);

int main() {
    std::cout << "[C++] Memulai program..." << std::endl;

    // 2. Tiru process.cwd() di Node.js untuk mendapatkan path absolut
    char cwd[1024];
    if (getcwd(cwd, sizeof(cwd)) == nullptr) {
        std::cerr << "[C++] Gagal mendapatkan Current Working Directory" << std::endl;
        return 1;
    }

    // 3. Gabungkan path seperti path.join() di Node.js
    std::string libraryPath = std::string(cwd) + "/target/release/libnixhelpers.so";
    std::cout << "[C++] Jalur biner SO: " << libraryPath << std::endl;

    std::cout << "[C++] Memanggil fungsi Rust via Dinamis FFI..." << std::endl;

    // 4. Tiru ffi.load() - Memuat file .so secara dinamis di runtime
    void* handle = dlopen(libraryPath.c_str(), RTLD_LAZY);
    if (!handle) {
        std::cerr << "[C++] Gagal memanggil fungsi FFI (Library tidak ditemukan):\n" 
                  << dlerror() << std::endl;
        return 1;
    }

    // Clear error yang tersisa jika ada
    dlerror();

    // 5. Tiru pencarian nama fungsi: funcName: 'contoh_fungsi'
    ContohFungsiPtr contoh_fungsi = (ContohFungsiPtr)dlsym(handle, "contoh_fungsi");
    
    char* error = dlerror();
    if (error != nullptr) {
        std::cerr << "[C++] Gagal menemukan fungsi 'contoh_fungsi':\n" << error << std::endl;
        dlclose(handle);
        return 1;
    }

    try {
        // 6. Siapkan data yang dikirim (C++ string sudah otomatis berakhiran \0)
        std::string targetFile = "contoh.txt";
        
        // Eksekusi fungsi biner mentah
        contoh_fungsi(targetFile.c_str());
        
        std::cout << "[C++] Selesai dengan sukses!" << std::endl;
    } catch (...) {
        std::cerr << "[C++] Terjadi kesalahan saat mengeksekusi fungsi." << std::endl;
    }

    // 7. Bersihkan memori library setelah selesai digunakan
    dlclose(handle);
    std::cout << "[C++] Selesai!" << std::endl;

    return 0;
}

```

Cara Kompilasi
Karena sekarang C++ memuat `.so` secara dinamis (FFI murni saat runtime), Anda **tidak perlu lagi** melakukan linking `-lnixhelpers` saat kompilasi. Anda hanya perlu menambahkan flag `-ldl` untuk mengaktifkan fitur Dynamic Linking milik sistem operasi.
```bash
# Kompilasi biasa (tanpa perlu tahu letak libnixhelpers.so saat kompilasi)
g++ main.cpp -o program_cpp -ldl

# Pastikan file target/release/libnixhelpers.so sudah di-build oleh Rust, lalu jalankan:
./program_cpp

```

Poin Penting Perubahan:
-   `dlopen` bertindak sama seperti fungsi `load()` di `ffi-rs`. Jika file `.so` tidak ada di folder tersebut, program akan memunculkan *error message* yang mirip seperti Node.js.
-   `dlsym` bertindak untuk mencari properti `funcName` di dalam library yang dimuat.
-   `-ldl` adalah library standar Linux/Unix untuk memanipulasi *shared library* secara dinamis.

<br>

q-ai:
Apakah struktur dynamic loading seperti ini yang Anda maksud, atau ada bagian penanganan buffer data spesifik dari `index.js` yang ingin Anda bedah lebih dalam?

<br>


Jika Anda ingin memastikan kode Anda benar-benar aman dan sesuai dengan standar C++ modern, Anda bisa menambahkan beberapa parameter tambahan berikut:
```cpp
clang++ -std=c++17 -Wall -Wextra main.cpp -o program_cpp -ldl
```

Penjelasan flag:
-   `-std=c++17`: Memastikan compiler menggunakan standar C++17 (atau Anda bisa ganti ke `c++20` atau `c++23`).
-   `-Wall -Wextra`: Menampilkan semua peringatan (*warnings*) jika ada kode yang berpotensi menyebabkan *bug* atau tidak aman.
-   `-ldl`: Flag wajib agar `clang++` bisa mengenali fungsi `dlopen`, `dlsym`, dan `dlclose` dari pustaka sistem Linux/POSIX.

(Jangan lupa sesuaikan juga nama file di kode C++ Anda dari `.so` menjadi `.dylib` jika dijalankan di macOS).

<br>

q:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <dlfcn.h>

// 1. Ubah return type signature menjadi 'int' bukan 'void'
typedef int (*ContohFungsiPtr)(const char*);

int main() {
    // ... (Bagian getcwd dan dlopen tetap sama seperti sebelumnya) ...

    ContohFungsiPtr contoh_fungsi = (ContohFungsiPtr)dlsym(handle, "contoh_fungsi");
    
    // ... (Bagian validasi dlsym tetap sama) ...

    const char* targetFile = "contoh.txt";
    
    // 6. Eksekusi dan tangkap kode error dari Rust
    int status = contoh_fungsi(targetFile);
    
    // 7. Evaluasi penanganan error spesifik berdasarkan return value Rust
    if (status == 0) {
        printf("[C] Proses FFI selesai dengan sukses!\n");
    } else if (status == -1) {
        fprintf(stderr, "[C] Error Spesifik FFI: Menerima pointer NULL!\n");
    } else if (status == -2) {
        fprintf(stderr, "[C] Error Spesifik FFI: File '%s' TIDAK DITEMUKAN di sistem!\n", targetFile);
    } else if (status == -3) {
        fprintf(stderr, "[C] Error Spesifik FFI: File ada, tapi gagal dibuka (Masalah Hak Akses/Permission)!\n");
    } else {
        fprintf(stderr, "[C] Error Spesifik FFI: Terjadi kesalahan tidak dikenal (Kode: %d)\n", status);
    }

    dlclose(handle);
    printf("[C] Selesai!\n");
    return (status == 0) ? 0 : 1;
}

```
membuat program C++ yang melakukan hal yang sama persis seperti yang dilakukan C dan Saya ingin menambahkan **mekanisme penanganan error yang lebih spesifik** jika file `contoh.txt` tersebut tidak ditemukan di sistem

<br>

a-ai:
Berikut adalah kode program `main.cpp` lengkap yang menerapkan logika penanganan status eror spesifik dari Rust (menggunakan nilai kembalian integer) menggunakan fitur standar C++ modern seperti std::string, `std::cout, dan std::cerr`.

Kode `main.cpp` (Bahasa C++ Modern)
```cpp
#include <iostream>
#include <string>
#include <unistd.h>  // Untuk getcwd
#include <dlfcn.h>   // Untuk dlopen, dlsym, dlclose

// 1. Definisikan tipe fungsi (Signature) dengan tipe kembalian 'int' sesuai fungsi Rust
typedef int (*ContohFungsiPtr)(const char*);

int main() {
    std::cout << "[C++] Memulai program..." << std::endl;

    // 2. Mendapatkan Current Working Directory seperti process.cwd()
    char cwd[1024];
    if (getcwd(cwd, sizeof(cwd)) == nullptr) {
        std::cerr << "[C++] Gagal mendapatkan Current Working Directory" << std::endl;
        return 1;
    }

    // 3. Menggabungkan path menggunakan std::string
    std::string libraryPath = std::string(cwd) + "/target/release/libnixhelpers.so";
    std::cout << "[C++] Jalur biner SO: " << libraryPath << std::endl;

    std::cout << "[C++] Memanggil fungsi Rust via Dinamis FFI..." << std::endl;

    // 4. Memuat file .so secara dinamis di runtime
    void* handle = dlopen(libraryPath.c_str(), RTLD_LAZY);
    if (!handle) {
        std::cerr << "[C++] Gagal memanggil fungsi FFI (Library tidak ditemukan):\n" 
                  << dlerror() << std::endl;
        return 1;
    }

    // Bersihkan error yang tersisa jika ada
    dlerror();

    // 5. Mencari fungsi 'contoh_fungsi' di dalam library
    ContohFungsiPtr contoh_fungsi = (ContohFungsiPtr)dlsym(handle, "contoh_fungsi");
    
    char* error = dlerror();
    if (error != nullptr) {
        std::cerr << "[C++] Gagal menemukan fungsi 'contoh_fungsi':\n" << error << std::endl;
        dlclose(handle);
        return 1;
    }

    // 6. Nama file target yang akan diperiksa oleh Rust
    std::string targetFile = "contoh.txt";
    
    // 7. Eksekusi fungsi Rust dan tangkap kode status/error nilainya
    int status = contoh_fungsi(targetFile.c_str());
    
    // 8. Evaluasi penanganan error spesifik berdasarkan return value dari Rust
    if (status == 0) {
        std::cout << "[C++] Proses FFI selesai dengan sukses!" << std::endl;
    } else if (status == -1) {
        std::cerr << "[C++] Error Spesifik FFI: Menerima pointer NULL!" << std::endl;
    } else if (status == -2) {
        std::cerr << "[C++] Error Spesifik FFI: File '" << targetFile << "' TIDAK DITEMUKAN di sistem!" << std::endl;
    } else if (status == -3) {
        std::cerr << "[C++] Error Spesifik FFI: File ada, tapi gagal dibuka (Masalah Hak Akses/Permission)!" << std::endl;
    } else {
        std::cerr << "[C++] Error Spesifik FFI: Terjadi kesalahan tidak dikenal (Kode: " << status << ")" << std::endl;
    }

    // 9. Bersihkan memori biner .so setelah selesai
    dlclose(handle);
    std::cout << "[C++] Selesai!" << std::endl;

    return (status == 0) ? 0 : 1;
}

```

Keunggulan Kode C++ Ini:

-   Keamanan String: Penggunaan `std::string` mengeliminasi risiko *buffer overflow* saat menggabungkan jalur direktori aktif dengan nama file `.so`.
-   Aliran Output Terpisah: Menggunakan `std::cout` untuk pesan log normal dan `std::cerr` untuk pesan galat (*error message*) agar mempermudah proses *debugging* atau *logging redirection*.
-   Sinkronisasi dengan Kode Rust Kedua: Kode ini otomatis menangkap nilai `-2` secara spesifik saat kode Rust Anda melemparkan `std::io::ErrorKind::NotFound`.

<br>

q-ai:
Apakah Anda ingin melengkapi program C++ ini agar **otomatis membuat file `contoh.txt` secara mandiri** menggunakan `std::ofstream` jika fungsi Rust mengembalikan kode error `-2` tersebut?

<br>

q-c:
Saya ingin melengkapi program C++ ini agar **otomatis membuat file `contoh.txt` secara mandiri** menggunakan `std::ofstream` jika fungsi Rust mengembalikan kode error `-2` tersebut?

<br>

a-ai:
Berikut adalah pembaruan kode `main.cpp` yang menggunakan `std::ofstream` dari pustaka `<fstream>`.

Program ini dirancang cerdas: jika eksekusi pertama ke Rust mengembalikan kode `-2` (file tidak ditemukan), C++ akan **otomatis membuat file tersebut**, mengisi teks di dalamnya, lalu **memanggil ulang fungsi Rust** agar proses FFI selesai dengan sukses.

Kode `main.cpp` (Dengan Pembuatan File Otomatis via `std::ofstream`)
```cpp
#include <iostream>
#include <string>
#include <fstream>   // WAJIB: Untuk std::ofstream
#include <unistd.h>  // Untuk getcwd
#include <dlfcn.h>   // Untuk dlopen, dlsym, dlclose

// 1. Definisikan tipe fungsi (Signature)
typedef int (*ContohFungsiPtr)(const char*);

int main() {
    std::cout << "[C++] Memulai program..." << std::endl;

    // 2. Dapatkan Current Working Directory
    char cwd[1024];
    if (getcwd(cwd, sizeof(cwd)) == nullptr) {
        std::cerr << "[C++] Gagal mendapatkan Current Working Directory" << std::endl;
        return 1;
    }

    // 3. Gabungkan path ke biner .so
    std::string libraryPath = std::string(cwd) + "/target/release/libnixhelpers.so";
    std::cout << "[C++] Jalur biner SO: " << libraryPath << std::endl;

    // 4. Memuat file .so secara dinamis
    void* handle = dlopen(libraryPath.c_str(), RTLD_LAZY);
    if (!handle) {
        std::cerr << "[C++] Gagal memanggil fungsi FFI (Library tidak ditemukan):\n" 
                  << dlerror() << std::endl;
        return 1;
    }

    dlerror(); // Bersihkan error log

    // 5. Cari fungsi 'contoh_fungsi'
    ContohFungsiPtr contoh_fungsi = (ContohFungsiPtr)dlsym(handle, "contoh_fungsi");
    char* error = dlerror();
    if (error != nullptr) {
        std::cerr << "[C++] Gagal menemukan fungsi 'contoh_fungsi':\n" << error << std::endl;
        dlclose(handle);
        return 1;
    }

    std::string targetFile = "contoh.txt";

    // 6. Eksekusi pertama fungsi Rust
    std::cout << "[C++] Mencoba memanggil fungsi Rust..." << std::endl;
    int status = contoh_fungsi(targetFile.c_str());

    // 7. Evaluasi penanganan error spesifik
    if (status == -2) {
        std::cerr << "[C++] Error Spesifik FFI: File '" << targetFile << "' TIDAK DITEMUKAN!" << std::endl;
        std::cout << "[C++] Membuat file '" << targetFile << "' secara mandiri via std::ofstream..." << std::endl;

        // Menggunakan std::ofstream untuk membuat file baru
        std::ofstream berkasBaru(targetFile);
        
        if (!berkasBaru) {
            std::cerr << "[C++] Gagal membuat file secara otomatis!" << std::endl;
            dlclose(handle);
            return 1;
        }

        // Tulis teks ke dalam file baru
        berkasBaru << "File ini dibuat otomatis oleh program C++ menggunakan std::ofstream." << std::endl;
        berkasBaru.close(); // WAJIB ditutup agar data tersimpan dan lock file dilepas

        std::cout << "[C++] File berhasil dibuat! Mencoba memanggil ulang fungsi Rust..." << std::endl;
        
        // Panggil ulang fungsi Rust setelah file dipastikan ada
        status = contoh_fungsi(targetFile.c_str());
    }

    // 8. Evaluasi status akhir setelah penanganan otomatis
    if (status == 0) {
        std::cout << "[C++] Proses FFI selesai dengan sukses!" << std::endl;
    } else if (status == -1) {
        std::cerr << "[C++] Error Spesifik FFI: Menerima pointer NULL!" << std::endl;
    } else if (status == -3) {
        std::cerr << "[C++] Error Spesifik FFI: File ada, tapi gagal dibuka (Masalah Hak Akses/Permission)!" << std::endl;
    } else {
        std::cerr << "[C++] Error Spesifik FFI: Terjadi kesalahan tidak dikenal (Kode: " << status << ")" << std::endl;
    }

    // 9. Bersihkan memori biner
    dlclose(handle);
    std::cout << "[C++] Selesai!" << std::endl;

    return (status == 0) ? 0 : 1;
}

```

Penjelasan Logika Alur Baru:
-   `std::ofstream` berkasBaru(targetFile): Membuka jalur aliran data keluar (_output file stream_). Jika file `contoh.txt` tidak ada, konstruktor ini akan **otomatis membuat file baru** di direktori aktif.
-   `berkasBaru << ...`: Memasukkan string teks ke dalam file tersebut dengan operator *stream* khas C++.
-   `berkasBaru.close()`: Menutup *stream*. Langkah ini krusial di C++ agar sistem operasi mengosongkan buffer memori RAM ke harddisk, sehingga saat fungsi Rust dipanggil ulang, file tersebut sudah siap dibaca secara fisik.

<br>

q-ai:
Apakah sistem *retry* otomatis ini sudah sesuai dengan kebutuhan arsitektur program Anda, atau Anda ingin menambahkan **fitur pembacaan isi file** dari Rust agar C++ bisa menerima teks isinya?

<br>

q-c:
Apakah sistem *retry* otomatis ini sudah sesuai dengan kebutuhan arsitektur program Anda, atau Anda ingin menambahkan **fitur pembacaan isi file** dari Rust agar C++ bisa menerima teks isinya?

<br>
a-ai:
Ya, mari kita tambahkan **fitur pembacaan isi file dari Rust lalu mengirimkan teksnya kembali ke program C++**.

Mengirim data string (teks) dari Rust ke C++ melalui FFI memerlukan sedikit ketelitian. Karena manajemen memori (*garbage collection/allocation*) Rust dan C++ berbeda, cara paling aman dan standar industri adalah **mengalokasikan buffer memori (array karakter) di sisi C++, lalu membiarkan Rust mengisi buffer tersebut**.

Berikut adalah implementasi lengkap untuk Rust dan C++ menggunakan arsitektur dynamic loading:

1. Perubahan di Sisi Rust (`src/lib.rs`)
Rust akan menerima pointer buffer dari C++ (`*mut c_char`) beserta kapasitas maksimum buffer tersebut (`buf_len`) agar tidak terjadi *buffer overflow*.
```rust
use std::ffi::CStr;
use std::os::raw::{c_char, c_int};
use std::fs::File;
use std::io::Read;

#[no_mangle]
pub unsafe extern "C" fn contoh_fungsi(
    pathname: *const c_char,
    out_buffer: *mut c_char,
    buf_len: c_int
) -> c_int {
    // 1. Validasi pointer null
    if pathname.is_null() || out_buffer.is_null() {
        return -1; // Pointer Null
    }

    let c_str = CStr::from_ptr(pathname);
    
    match c_str.to_str() {
        Ok(file_path) => {
            // 2. Mencoba membuka file
            match File::open(file_path) {
                Ok(mut file) => {
                    let mut isi_file = String::new();
                    // 3. Membaca isi file ke dalam string Rust
                    if file.read_to_string(&mut isi_file).is_err() {
                        return -3; // Gagal membaca file (Masalah enkripsi/corrupt)
                    }

                    // 4. Salin isi teks Rust ke buffer milik C++ yang sudah disediakan
                    let bytes = isi_file.as_bytes();
                    let max_copy = (buf_len as usize - 1).min(bytes.len()); // Sisakan 1 byte untuk '\0'

                    // Salin memori mentah secara aman
                    std::ptr::copy_nonoverlapping(bytes.as_ptr(), out_buffer as *mut u8, max_copy);
                    
                    // Wajib beri akhiran null terminator (\0) agar C++ tahu stringnya berakhir
                    *out_buffer.add(max_copy) = 0; 

                    0 // Sukses
                }
                Err(e) => {
                    if e.kind() == std::io::ErrorKind::NotFound {
                        -2 // File tidak ditemukan
                    } else {
                        -3 // Masalah permission / hak akses
                    }
                }
            }
        }
        Err(_) => -4, // String path tidak valid
    }
}

```

2. Perubahan di Sisi C++ (`main.cpp`)
Di sisi C++, kita perbarui *typedef signature* fungsi agar menerima parameter buffer, lalu menyiapkan array `char` lokal untuk menampung teks dari Rust.
```cpp
#include <iostream>
#include <string>
#include <fstream>
#include <unistd.h>
#include <dlfcn.h>

// 1. Perbarui Signature: Menambah pointer buffer output dan ukuran panjangnya
typedef int (*ContohFungsiPtr)(const char*, char*, int);

int main() {
    std::cout << "[C++] Memulai program..." << std::endl;

    char cwd[1024];
    if (getcwd(cwd, sizeof(cwd)) == nullptr) {
        std::cerr << "[C++] Gagal mendapatkan Current Working Directory" << std::endl;
        return 1;
    }

    std::string libraryPath = std::string(cwd) + "/target/release/libnixhelpers.so";
    
    void* handle = dlopen(libraryPath.c_str(), RTLD_LAZY);
    if (!handle) {
        std::cerr << "[C++] Gagal memuat biner FFI:\n" << dlerror() << std::endl;
        return 1;
    }

    dlerror();
    ContohFungsiPtr contoh_fungsi = (ContohFungsiPtr)dlsym(handle, "contoh_fungsi");
    if (dlerror() != nullptr) {
        std::cerr << "[C++] Gagal menemukan fungsi 'contoh_fungsi'!" << std::endl;
        dlclose(handle);
        return 1;
    }

    std::string targetFile = "contoh.txt";

    // 2. Siapkan Buffer di sisi C++ untuk menampung teks dari Rust (misal maks 512 karakter)
    const int BUFFER_SIZE = 512;
    char textBuffer[BUFFER_SIZE] = {0}; // Isi dengan 0 (null) di awal

    // 3. Eksekusi pertama fungsi Rust dengan menyertakan buffer
    std::cout << "[C++] Mencoba memanggil fungsi Rust..." << std::endl;
    int status = contoh_fungsi(targetFile.c_str(), textBuffer, BUFFER_SIZE);

    // 4. Logika otomatisasi retry jika file hilang
    if (status == -2) {
        std::cerr << "[C++] Error: File '" << targetFile << "' TIDAK DITEMUKAN!" << std::endl;
        std::cout << "[C++] Membuat file secara mandiri via std::ofstream..." << std::endl;

        std::ofstream berkasBaru(targetFile);
        if (!berkasBaru) {
            std::cerr << "[C++] Gagal membuat file otomatis!" << std::endl;
            dlclose(handle);
            return 1;
        }

        // Tulis teks rahasia yang nantinya akan dibaca Rust
        berkasBaru << "Halo dari C++! Teks ini dibaca oleh Rust dan dikembalikan lagi.";
        berkasBaru.close(); 

        std::cout << "[C++] File berhasil dibuat! Mencoba memanggil ulang fungsi Rust..." << std::endl;
        
        // Panggil ulang dan isi buffer kembali
        status = contoh_fungsi(targetFile.c_str(), textBuffer, BUFFER_SIZE);
    }

    // 5. Evaluasi hasil akhir dan cetak isi teks yang dikirim Rust
    if (status == 0) {
        std::cout << "[C++] Proses FFI selesai dengan sukses!" << std::endl;
        
        // Konversi buffer mentah menjadi std::string C++ yang bersih
        std::string isiTeksDariRust(textBuffer);
        std::cout << "[C++] ISI FILE YANG DIBACA RUST: \"" << isiTeksDariRust << "\"" << std::endl;
    } else {
        std::cerr << "[C++] FFI gagal mengeksekusi dengan kode status: " << status << std::endl;
    }

    dlclose(handle);
    std::cout << "[C++] Selesai!" << std::endl;

    return (status == 0) ? 0 : 1;
}

```

-   Bebas Memory Leak: Anda tidak perlu memusingkan siapa yang bertanggung jawab melakukan `free()` atau `drop()` pada pointer teks tersebut, karena memori dialokasikan di stack C++ (`char textBuffer[512]`) yang akan hancur otomatis saat fungsi `main` selesai.
- Mencegah Crash: Batasan `buf_len` yang dikirim ke Rust memastikan bahwa meskipun isi file berukuran 10 MB, Rust hanya akan menyalin maksimal 511 byte saja, sehingga mencegah galat *segmentation fault (buffer overflow*) di C++.

<br>

q-ai:
Apakah Anda ingin melanjutkan eksperimen ini dengan mencoba **mengirim data berstruktur (Struct)** atau ada skenario arsitektur data lain yang ingin Anda terapkan?


<br>

