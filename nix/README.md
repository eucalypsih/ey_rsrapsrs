# nix

q:
```rust
Cargo.toml

[dependencies]
nix = "0.26.1"
```

<br>

a-ai:
Secara default, crate `nix` menggunakan sistem **_opt-in_ fitur (feature flags)**. Jika Anda hanya menuliskan dependensi seperti di atas, hampir **seluruh fungsi API tidak akan aktif** saat dikompilasi, sehingga fungsi seperti `nix::unistd` atau `nix::sys` tidak bisa dipanggil.

**Rekomendasi Konfigurasi**

Agar crate `nix` dapat digunakan, Anda harus menentukan fitur apa saja yang ingin diaktifkan di dalam berkas `Cargo.toml` Anda.

1. **Mengaktifkan Fitur Secara Spesifik (Disarankan)**
Aktifkan hanya modul atau sistem API Unix/Linux yang benar-benar Anda butuhkan. Pendekatan ini membuat waktu kompilasi (_compile time_) menjadi jauh lebih cepat.
```rust
[dependencies]
nix = { version = "0.26.1", features = ["fs", "user", "process", "signal"] }

```
2. **Mengaktifkan Semua Fitur (_Full_)**
Jika Anda sedang melakukan eksperimen, membuat prototipe cepat, atau membutuhkan banyak API Unix sekaligus, Anda bisa langsung mengaktifkan semua fitur yang tersedia:
```rust
[dependencies]
nix = { version = "0.26.1", features = ["full"] }

```

**Daftar Fitur Populer di Crate** `nix`
Berikut adalah beberapa _feature flags_ yang paling sering digunakan beserta modul API yang diaktifkannya:
- `fs`: Manajemen berkas dan direktori (misal: `nix::fcntl`, `nix::sys::stat`).
- `user`: Manajemen pengguna dan grup sistem (misal: `nix::unistd::Uid`, `nix::unistd::Gid`).
- `process`: Manajemen proses OS (misal: `nix::unistd::fork`, `nix::sched`).
- `signal`: Penanganan sinyal POSIX (misal: `nix::sys::signal`).
- `net`: Fungsi jaringan tingkat rendah (misal: `nix::sys::socket`).
- `ioctl`: Manipulasi perangkat keras/I/O khusus (macro `ioctl!`).

**Contoh Penggunaan di Kode Rust (`src/main.rs`)**
Setelah memperbarui `Cargo.toml` (misalnya dengan fitur `["user", "process"]`), Anda dapat menggunakan kode Unix native seperti berikut:
```rust
use nix::unistd::{getuid, getpid};

fn main() {
    // Mengambil ID Proses saat ini
    let pid = getpid();
    // Mengambil ID Pengguna saat ini
    let uid = getuid();

    println!("PID Proses Saat Ini: {}", pid);
    println!("UID Pengguna Saat Ini: {}", uid);
}

```

<br>

q-ai:
Jika Anda ingin menyesuaikan kode ini dengan fungsionalitas Unix tertentu, beri tahu saya **API atau operasi sistem apa yang ingin Anda jalankan** agar saya bisa membantu menentukan daftar fitur yang tepat!

<br>

q:
use nix::unistd::lseek;

<br>

a-ai:
Untuk menggunakan fungsi `nix::unistd::lseek`, Anda wajib mengaktifkan fitur `fs` di dalam berkas `Cargo.toml`.

Secara default, fungsi-fungsi manipulasi _file descriptor_ tingkat rendah (seperti `lseek`, `read`, dan `write`) di dalam modul `unistd` dikelompokkan ke dalam fitur sistem berkas (_file system_).

**Konfigurasi Cargo.toml**
Perbarui dependensi proyek Anda menjadi seperti ini:
```toml
[dependencies]
nix = { version = "0.26.1", features = ["fs"] }

```

**Contoh Kode Penggunaan** `lseek`
Fungsi `lseek` digunakan untuk mengubah posisi _read/write offset_ pada sebuah _file descriptor_. Fungsi ini membutuhkan enum `Whence` untuk menentukan titik acuan pergeseran.

Berikut adalah contoh cara membaca isi file dari posisi tertentu menggunakan `lseek`:
```rust
use nix::unistd::{lseek, Whence};
use std::fs::File;
use std::os::unix::io::AsRawFd;
use std::io::Read;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Buka file contoh
    let file = File::open("contoh.txt")?;
    let fd = file.as_raw_fd();

    // 2. Geser posisi kursor sebanyak 5 byte dari awal file (SeekFromStart)
    // lseek mengembalikan posisi offset terbaru jika berhasil
    let posisi_baru = lseek(fd, 5, Whence::SeekBuf)?;
    println!("Posisi kursor saat ini: {} byte", posisi_baru);

    // 3. Baca file dari posisi baru tersebut
    let mut buffer = String::new();
    let mut file_ref = &file;
    file_ref.read_to_string(&mut buffer)?;
    
    println!("Isi file setelah lompat 5 byte: {}", buffer);

    Ok(())
}

```

**Opsi Variasi `Whence` yang Tersedia:**
- **`Whence::SeekBuf`**: Menggeser offset dihitung dari **awal file** (setara dengan `SEEK_SET` di C atau `SeekFrom::Start` di Rust standar).
- **`Whence::SeekCur`**: Menggeser offset dihitung dari **posisi kursor saat ini** (setara dengan `SEEK_CUR` atau `SeekFrom::Current`).
- **`Whence::SeekEnd`**: Menggeser offset dihitung dari **akhir file** (setara dengan `SEEK_END` atau `SeekFrom::End`).

<br>

q-ai:
Apakah Anda berencana menggunakan `lseek` untuk **mengetahui ukuran file** atau melakukan **operasi baca/tulis acak (random access)**? Beritahu saya jika Anda membutuhkan contoh kode yang lebih spesifik!

<br>

q:
```
File::open("contoh.txt")?;

menggunakan PathBuff
```

<br>

q-ai:
Untuk menggunakan `PathBuf`, fungsi `File::open` dapat **langsung menerima variabel `PathBuf`** secara instan tanpa perlu dikonversi menjadi string atau tipe data lainnya.

Tipe data `PathBuf` mengimplementasikan _trait_ `AsRef<Path>` [io::Seek functions seek() and tell() value types do not match · Issue #20202 · rust-lang/rust], yang mana merupakan tipe input yang diminta oleh fungsi `File::open`.

_(Catatan perbaikan kode sebelumnya: Mohon maaf, enum untuk memposisikan awal file yang benar pada crate `nix` versi 0.26 adalah **`Whence::SeekSet`**, bukan `SeekBuf` [Whence in nix::unistd - Rust])_.

**Contoh Kode Menggunakan `PathBuf` dan `lseek`**
Berikut adalah cara memperbarui kode Anda agar menggunakan `PathBuf`:
```rust
use nix::unistd::{lseek, Whence};
use std::fs::File;
use std::os::unix::io::AsRawFd;
use std::io::Read;
// 1. Import PathBuf dari standard library
use std::path::PathBuf; 

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 2. Inisialisasi PathBuf
    let mut path = PathBuf::from("direktori_kamu");
    path.push("contoh.txt"); // Menghasilkan path: "direktori_kamu/contoh.txt"

    // 3. Masukkan variabel PathBuf langsung ke File::open
    // Anda bisa menggunakan token &path atau path langsung
    let file = File::open(&path)?; 
    let fd = file.as_raw_fd();

    // 4. Geser posisi kursor sebanyak 5 byte dari AWAL file menggunakan SeekSet
    let posisi_baru = lseek(fd, 5, Whence::SeekSet)?;
    println!("Posisi kursor saat ini: {} byte", posisi_baru);

    // 5. Baca sisa isi file
    let mut buffer = String::new();
    let mut file_ref = &file;
    file_ref.read_to_string(&mut buffer)?;
    
    println!("Isi file setelah lompat 5 byte: {}", buffer);

    Ok(())
}

```

**Hal Penting yang Perlu Diperhatikan:**
- **Ownership (`&path`)**: Sangat disarankan untuk menuliskan `File::open(&path)` menggunakan tanda referensi (`&`). Jika ditulis tanpa `&`, variabel `path` milik Anda akan hangus (_moved_) dan tidak bisa digunakan lagi di baris kode bawahnya.
- **Validasi**: Keunggulan `PathBuf` adalah mempermudah Anda memanipulasi lokasi file secara dinamis melalui metode `.push()`, `.pop()`, atau `.set_extension()`.

<br>

q-ai:
Apakah `PathBuf` yang Anda miliki didapatkan dari **input pengguna (_arguments_)**, hasil pemindaian direktori, atau ada skenario pembuatan path dinamis lainnya?

<br>

q:
```
let mut path = PathBuf::from(CStr::from_ptr(pathname));
```

<br>

a-ai:
Baris kode Anda tidak akan berhasil dikompilasi (_compile error_). Penyebab utamanya adalah `PathBuf::from()` tidak mengenali atau tidak memiliki implementasi bawaan untuk menerima tipe data `&CStr` secara langsung.

Selain itu, karena `CStr::from_ptr` adalah fungsi FFI untuk membaca pointer mentah C (`*const c_char`), Anda wajib membungkus baris kode tersebut di dalam blok **`unsafe`**.

Berikut adalah solusi dan cara yang benar untuk mengubah string pointer C menjadi `PathBuf` berdasarkan kebutuhan sistem operasi Anda:

**Solusi Terbaik untuk Unix/Linux (Zero-Copy & Aman untuk Path)**
Di sistem operasi berbasis Unix, sebuah _path_ berkas pada dasarnya hanyalah deretan bita (_bag of bytes_) tanpa keharusan berupa format UTF-8 yang valid.

Cara terbaik, tercepat, dan paling aman adalah mengubah bita dari `CStr` langsung ke objek `OsStr` melalui ekstensi Unix, kemudian memasukkannya ke `PathBuf`:
```rust
use std::ffi::CStr;
use std::os::unix::ffi::OsStrExt; // 1. Wajib import trait ekstensi Unix ini
use std::os::raw::c_char;
use std::path::{Path, PathBuf};

fn konversi_path(pathname: *const c_char) -> PathBuf {
    // 2. Operasi pointer FFI wajib berada di dalam blok unsafe
    let c_str = unsafe { CStr::from_ptr(pathname) };
    
    // 3. Ambil bita mentah dari CStr, ubah ke OsStr, lalu ke PathBuf
    let os_str = std::ffi::OsStr::from_bytes(c_str.to_bytes());
    
    PathBuf::from(os_str)
}

```

**Solusi Alternatif (Menggunakan String UTF-8)**
Jika Anda yakin bahwa string path dari C tersebut selalu berupa teks UTF-8 yang valid, Anda bisa mengubahnya menjadi `&str` terlebih dahulu:
```rust
use std::ffi::CStr;
use std::os::raw::c_char;
use std::path::PathBuf;

fn konversi_path_utf8(pathname: *const c_char) -> PathBuf {
    let c_str = unsafe { CStr::from_ptr(pathname) };
    
    // Konversi ke &str (bisa memicu panic jika bukan UTF-8 valid)
    let rust_str = c_str.to_str().expect("Path harus berupa UTF-8 valid");
    
    PathBuf::from(rust_str)
}

```

💡 **Tips Optimalisasi untuk Crate** `nix`
Jika tujuan akhir Anda membuat `PathBuf` ini hanya untuk dimasukkan kembali ke fungsi-fungsi sistem milik crate `nix` (seperti `nix::sys::stat::stat` atau `nix::fcntl::open`), **Anda sebenarnya tidak perlu mengubahnya ke `PathBuf`**.

Banyak API fungsi di crate `nix` yang menerima objek bertipe **`&CStr`** secara langsung! Anda bisa langsung memakai variabel `c_str` hasil dari `CStr::from_ptr(pathname)` tanpa melakukan alokasi memori tambahan.

<br>

q-ai:
Apakah fungsi `pathname` ini Anda dapatkan dari **parameter fungsi FFI (extern "C")** atau dari interop library C lainnya?

<br>

q:
```
fn to_string(s: &CStr) -> String {
    s.to_string_lossy().into_owned()
}

let name: String = to_string(CStr::from_ptr(pathname));
```

<br>

a-ai:
Baris kode Anda sudah menggunakan logika konversi yang aman dengan `.to_string_lossy()`. Namun, kode tersebut **`belum bisa dikompilasi`** karena ada dua aturan ketat Rust yang terlanggar:
1. **Aturan `unsafe`**: Fungsi `CStr::from_ptr` membaca _raw pointer_ C (`*const c_char`), sehingga wajib dibungkus di dalam blok `unsafe`.
2. **Kecocokan Tipe Data (_Type Mismatch_)**: Fungsi `to_string` Anda menerima referensi `&CStr`, sedangkan Anda memasukkan objek `CStr` hasil dari `from_ptr` secara langsung tanpa tanda referensi (`&`).

Berikut adalah perbaikan kode yang benar agar bisa langsung dikompilasi:
```rust
use std::ffi::CStr;
use std::os::raw::c_char;

// Fungsi pembantu Anda sudah benar
fn to_string(s: &CStr) -> String {
    s.to_string_lossy().into_owned()
}

fn contoh_fungsi(pathname: *const c_char) {
    // Perbaikan: Tambahkan blok `unsafe` dan tanda ampersand `&`
    let name: String = to_string(unsafe { CStr::from_ptr(pathname) });
    
    println!("Nama path berhasil dikonversi: {}", name);
}

```

**Penjelasan Perbaikan:**
- **`unsafe { ... }`**: Memberitahu _compiler_ Rust bahwa Anda bertanggung jawab memastikan pointer `pathname` tersebut valid (tidak _null_ dan menunjuk ke memori yang benar).
- **`&` sebelum `unsafe`**: Mengubah objek `CStr` yang dihasilkan oleh `from_ptr` menjadi referensi `&CStr` agar cocok dengan parameter fungsi `to_string(s: &CStr)`.


**Alternatif Lebih Ringkas (Tanpa Fungsi Pembantu)**
Jika Anda ingin kodenya lebih pendek tanpa perlu membuat fungsi `to_string` terpisah, Anda bisa langsung menulisnya seperti ini:
```rust
let name: String = unsafe { CStr::from_ptr(pathname) }
    .to_string_lossy()
    .into_owned();

```

<br>

q-ai:
Apakah string `name` ini nantinya akan Anda modifikasi atau hanya digunakan untuk **dicetak (`logging`)** dan **dibuka filenya**? Jika Anda punya rencana spesifik untuk variabel string ini, beri tahu saya agar kita bisa mengoptimasi alokasi memorinya!

<br>

q:
```
fn contoh_fungsi(pathname: *const c_char)

cara penggunaanya
```

<br>

a-ai:
Untuk menggunakan fungsi yang menerima parameter berupa _raw pointer_ C (`*const c_char`), Anda harus membuat string Rust terlebih dahulu, mengubahnya menjadi string yang dipahami oleh bahasa C (`CString`), lalu mengambil pointernya [how to pass `String` or `&str` to `*const c_char` - Help - Rust Users].

Berikut adalah beberapa cara menggunakan fungsi `contoh_fungsi(pathname: *const c_char)` tersebut, tergantung dari mana asal string yang Anda miliki:

1. **Menggunakan String Literal (Teks Statis)**
Jika path sudah pasti dan ditulis langsung di dalam kode, gunakan makro `c_""` (tersedia mulai Rust 1.77) [New c"" literal style syntax - Help - Rust Users]. Ini adalah cara paling efisien karena tidak ada alokasi memori tambahan saat program berjalan.
```rust
use std::os::raw::c_char;

// Fungsi yang ingin dipanggil
fn contoh_fungsi(pathname: *const c_char) {
    // ... isi fungsi Anda ...
}

fn main() {
    // Gunakan huruf 'c' di depan string literal untuk membuat CStr konstan
    let c_path = c"contoh.txt"; 
    
    // Ambil pointernya menggunakan .as_ptr()
    contoh_fungsi(c_path.as_ptr());
}

```

2. **Menggunakan Variabel `String` atau `&str` Dinamis**
Jika path berupa variabel dinamis (misal hasil input pengguna, pembacaan file, atau dari `PathBuf`), Anda harus mengubahnya menjadi `CString`. Fungsi `CString::new` akan menambahkan karakter _null-terminator_ (`\0`) di akhir string secara otomatis agar kompatibel dengan C.
```rust
use std::ffi::CString;
use std::os::raw::c_char;

fn main() {
    // Anggap ini string dinamis dari Rust
    let rust_string = String::from("folder_baru/file_rahasia.txt");

    // Konversi String Rust menjadi CString
    let c_string = CString::new(rust_string).expect("Gagal konversi ke CString");

    // PENTING: Panggil .as_ptr() dan masukkan ke fungsi
    contoh_fungsi(c_string.as_ptr());
    
    // CATATAN: Variabel `c_string` harus tetap hidup selama `contoh_fungsi` berjalan.
    // Jangan sampai `c_string` hancur sebelum pointernya selesai dibaca.
}

```

3. **Menggunakan Objek** `PathBuf`
Jika saat ini Anda sedang memegang objek `PathBuf`, Anda bisa mengubahnya menjadi bita mentah (_bytes_) terlebih dahulu menggunakan ekstensi Unix, lalu mengubahnya menjadi `CString`:
```rust
use std::ffi::CString;
use std::os::unix::ffi::OsStrExt; // Wajib diimport untuk sistem Unix
use std::path::PathBuf;

fn main() {
    let mut path = PathBuf::from("dokumen");
    path.push("data.bin");

    // Ambil bita mentah dari PathBuf dan ubah ke CString
    let c_string = CString::new(path.as_os_str().as_bytes())
        .expect("Path mengandung karakter null internal");

    contoh_fungsi(c_string.as_ptr());
}

```

⚠️ **Aturan Keselamatan Memori (Penting!)**
Saat menggunakan `.as_ptr()` dari sebuah `CString`, pastikan Anda **tidak menulis kode dalam satu baris pintas** seperti ini:
```rust
// ❌ SALAH DAN BERBAHAYA (Dangling Pointer)
let ptr = CString::new("file.txt").unwrap().as_ptr();
contoh_fungsi(ptr); 

```
Kode di atas salah karena `CString` akan langsung dihapus dari memori (_dropped_) tepat setelah baris pertama selesai [how to pass `String` or `&str` to `*const c_char` - Help - Rust Users]. Akibatnya, `ptr` akan menunjuk ke memori kosong yang sudah tidak valid saat dimasukkan ke dalam `contoh_fungsi`. Selalu simpan `CString` ke dalam variabel lokal terlebih dahulu seperti pada contoh nomor 2 dan 3.

<br>

q-ai:
Apakah Anda berniat memanggil fungsi ini dari **fungsi `main` di Rust**, atau fungsi ini merupakan bagian dari **arsitektur FFI / library** yang akan dipanggil oleh bahasa lain (seperti C atau Python)?

<br>

q:
```
pub unsafe contoh_fungsi(pathname: *const c_char)

membuat crate staticlib
```

<br>

a-ai:
Untuk membuat sebuah proyek Rust menjadi pustaka statis (_static library_ atau file `.a`), Anda harus mengubah konfigurasi jenis _crate_ di berkas `Cargo.toml` dan menambahkan penanda `#[no_mangle]` pada fungsi Anda agar namanya tidak diubah oleh _compiler_ Rust saat diekspor ke bahasa C.

Berikut adalah langkah-langkah lengkap untuk membuatnya:

**Langkah 1: Konfigurasi** `Cargo.toml`
Buka berkas `Cargo.toml` Anda, lalu tambahkan blok `[lib]` dan tentukan `crate-type` menjadi `["staticlib"]`.
```toml
[package]
name = "my_unix_helpers"
version = "0.1.0"
edition = "2021"

[lib]
name = "my_unix_helpers"
crate-type = ["staticlib"] # <-- Menginstruksikan Rust untuk membuat file .a

[dependencies]
nix = { version = "0.26.1", features = ["fs"] }

```

**Langkah 2: Menulis Kode Fungsi di** `src/lib.rs`
Ubah nama berkas utama Anda dari `src/main.rs` menjadi `src/lib.rs` karena kita membuat pustaka, bukan program yang bisa dieksekusi langsung.

Tambahkan atribut `#[no_mangle]` tepat di atas fungsi dan gunakan konversi `CStr` yang aman seperti yang telah kita bahas:
```rust
use std::ffi::CStr;
use std::os::raw::c_char;
use std::fs::File;
use std::os::unix::io::AsRawFd;
use nix::unistd::{lseek, Whence};

/// # Safety
/// Fungsi ini tidak aman karena melakukan dereferensi terhadap `pathname` yang merupakan *raw pointer*.
/// Pemanggil fungsi dari sisi C wajib memastikan pointer tidak null dan menunjuk ke string C yang valid.
#[no_mangle] // <-- Wajib agar fungsi bisa dipanggil oleh C/C++ dengan nama yang sama
pub unsafe extern "C" fn contoh_fungsi(pathname: *const c_char) {
    // 1. Validasi pointer null demi keamanan dasar
    if pathname.is_null() {
        eprintln!("Error: Pointer pathname tidak boleh null!");
        return;
    }

    // 2. Konversi raw pointer ke &CStr (Aman karena sudah di dalam fungsi unsafe)
    let c_str = CStr::from_ptr(pathname);
    
    // 3. Konversi ke String Rust untuk keperluan log atau manipulasi
    let path_string = c_str.to_string_lossy().into_owned();
    println!("Membuka berkas: {}", path_string);

    // 4. Contoh implementasi operasi file dengan nix::unistd::lseek
    match File::open(&path_string) {
        Ok(file) => {
            let fd = file.as_raw_fd();
            // Geser kursor ke akhir file untuk mengetahui ukurannya
            match lseek(fd, 0, Whence::SeekEnd) {
                Ok(ukuran) => println!("Ukuran berkas '{}' adalah {} byte", path_string, ukuran),
                Err(err) => eprintln!("Gagal melakukan lseek: {}", err),
            }
        }
        Err(err) => eprintln!("Gagal membuka berkas '{}': {}", path_string, err),
    }
}

```

**Langkah 3: Melakukan Kompilasi**
Jalankan perintah berikut di terminal Anda untuk mengompilasi proyek menjadi pustaka statis:
```bash
cargo build --release

```
Setelah proses kompilasi selesai, hasil pustaka statis Anda akan berada di direktori: `target/release/libmy_unix_helpers.a`

**Cara Penggunaan di Kode C / C++**
Untuk menguji atau menggunakan pustaka statis tersebut di bahasa C, Anda bisa membuat berkas `main.c` seperti ini:
```c
// main.c
#include <stdio.h>

// Deklarasikan fungsi eksternal dari Rust
void contoh_fungsi(const char *pathname);

int main() {
    // Memanggil fungsi Rust dari kode C
    contoh_fungsi("contoh.txt");
    return 0;
}

```

**Cara melakukan linking (kompilasi bersama file C):**
Di sistem operasi Linux/Unix, Anda harus menautkan pustaka statis Rust bersama dengan pustaka sistem yang dibutuhkan oleh Rust (`libdl`, `librt`, `libpthread`, dll.):
```bash
gcc main.c -L./target/release -lmy_unix_helpers -lpthread -ldl -lrt -o program_c

```

Untuk menautkan (_link_) pustaka statis Rust (`.a`) dengan kode C menggunakan `clang`, Anda harus menyertakan file pustaka Rust tersebut serta **pustaka sistem dasar** (seperti pustaka manajemen memori, _threading_, dan matematika) yang diwajibkan oleh Rust _runtime_.

Berikut adalah perintah langsung untuk melakukan kompilasi:
```bash
clang main.c -L. -l:libnamapustaka.a -lrt -ldl -lpthread -lm -o program_utama

```

**Penjelasan Bendera Kumpulan Perintah (Flags)**
- **`main.c`**: File sumber kode C Anda.
- **`-L.`**: Memberitahu `clang` untuk mencari pustaka di direktori saat ini.
- **`-l:libnamapustaka.a`**: Menautkan langsung file pustaka statis Rust Anda (ganti `libnamapustaka.a` dengan nama file asli Anda).
- **`-lrt`**: Menyediakan fungsi _real-time_ (seperti pewaktu sistem).
- **`-ldl`**: Memungkinkan pemuatan pustaka dinamis (_dynamic loading_).
- **`-lpthread`**: Mengaktifkan dukungan multi-utas (_multi-threading_) POSIX.
- **`-lm`**: Menautkan pustaka matematika standar.
- **`-o program_utama`**: Menentukan nama file hasil eksekusi akhir.

**Cara Mengetahui Pustaka Sistem Secara Tepat**
Jika Anda menggunakan fungsi spesifik di Rust (seperti fitur jaringan atau kriptografi) yang membutuhkan pustaka sistem tambahan, Anda bisa meminta Rust untuk menjabarkan daftar pustaka yang ia butuhkan secara otomatis.

Jalankan perintah ini di terminal proyek Rust Anda:
```bash
cargo rustc -- --print native-static-libs

```
Perintah di atas akan memunculkan teks panduan di terminal berisi seluruh bendera pustaka sistem yang wajib Anda salin dan tempel ke dalam perintah `clang` Anda.

<br>

q-ai:
Apakah Anda ingin membuat fungsi Rust ini mengembalikan sebuah nilai (seperti angka ukuran file atau status _error_) ke kode C, atau ada **kebutuhan fungsi FFI tambahan** lainnya yang ingin Anda buat?

Agar proses ini berjalan lancar, pastikan fungsi di sisi Rust sudah menggunakan atribut `#[no_mangle]` dan `pub extern "C"`. Jika Anda menemui kendala saat kompilasi, beri tahu saya **pesan galat (_error message_)** yang muncul atau **sistem operasi** yang Anda gunakan agar saya bisa memberikan solusi yang lebih spesifik.




<br>


