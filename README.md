# ey_rsrapsrs

Fungsi Utama Berkas `Cargo.lock`
Di dalam ekosistem pemrograman Rust, proyek dikelola menggunakan alat bernama **Cargo**. File `Cargo.lock` otomatis tercipta untuk mencatat **versi pasti (exact version)** dari semua pustaka pihak ketiga (*dependencies*) yang diunduh dan digunakan oleh `libtermuxadb`.

Alasan File Ini Wajib Masuk ke GitHub
Pengembang sengaja memasukkan file `Cargo.lock` ke dalam repositori Git agar proses pembuatan program bersifat **reprodusibel (reproducible build)**. Artinya:
- *Mengunci Versi*: Memastikan siapa pun (atau sistem otomatis) yang mengompilasi ulang paket `termux-adb` ini akan menggunakan versi pustaka pendukung yang sama persis seperti yang digunakan oleh `nohajc` saat memprogramnya.
- **Mencegah Error**: Menghindari kegagalan kompilasi jika di kemudian hari pustaka pihak ketiga di internet memperbarui kodenya (*breaking changes*).

q-ai:
Memahami cara CMake memanggil program Rust di dalam proyek ini



`https://github.com/a1ien/rusb`

Halaman tersebut menampilkan repositori GitHub bernama `rusb` yang dikembangkan oleh pengguna bernama **a1ien**. Proyek ini ditulis 100% menggunakan bahasa pemrograman **Rust**.
Secara garis besar, `rusb` adalah proyek penting yang menjadi **fondasi utama** mengapa alat seperti `termux-adb` milik `nohajc` bisa mengakses USB tanpa hak akses *root*.
Berikut adalah penjelasan detail mengenai fungsi dan kegunaan proyek `rusb`:
Fungsi Utama: Safe Wrapper untuk `libusb`
Di dalam dunia pemrograman Linux, `libusb` adalah pustaka standar berbasis bahasa C yang digunakan oleh aplikasi (termasuk ADB asli milik Google) untuk berkomunikasi langsung dengan perangkat USB eksternal.
`rusb` bertindak sebagai **jembatan pengaman (safe wrapper)**. Pustaka ini membungkus kode C milik `libusb` agar bisa digunakan di dalam bahasa Rust dengan memanfaatkan fitur keamanan memori bawaan Rust (seperti *RAII pattern* dan *lifetimes*). Ini mencegah terjadinya kebocoran memori (*memory leak*) atau aplikasi mendadak macet (*crash*).

Cara Kerja Kode `rusb`
Seperti contoh kode yang tertera pada dokumentasinya, `rusb` memungkinkan program untuk:
- Membuat objek konteks untuk mendeteksi perangkat USB yang dicolokkan ke sistem.
- Membaca informasi ID Vendor dan ID Produk dari perangkat tersebut.
- Membuka jalur komunikasi data aman untuk bertukar perintah (membaca dan menulis data ke endpoint USB).

Mengapa `rusb` Mendukung Lintas Kompilasi (Cross-Compiling)?
Pustaka ini dirancang agar bisa di-*compile* silang untuk target arsitektur yang berbeda. Fitur ini sangat krusial bagi ekosistem Termux:
- Saat mengembangkan `termux-adb`, pengembang menulis kodenya di PC (arsitektur Linux x86_64).
- Karena `rusb` mendukung lintas kompilasi, kode Rust tersebut bisa dengan mudah dibangun (build) langsung menjadi biner untuk arsitektur HP Android (seperti **aarch64** atau **arm**).


Hubungannya dengan Berkas `Cargo.lock` Milik `nohajc`
Pada pertanyaan Anda sebelumnya mengenai berkas `libtermuxadb/Cargo.lock` di repositori `nohajc`, `rusb` **adalah salah satu pustaka utama yang terdaftar di dalamnya**.

`nohajc` tidak memakai pustaka USB C++ bawaan Google yang kaku, melainkan menggunakan kode Rust dan memanggil `rusb` ini untuk menangkap kiriman data koneksi fisik USB dari `API termux-usb`.



Jika Anda ingin mempelajari implementasinya lebih lanjut, silakan pilih langkah berikutnya:
- Apakah Anda ingin melihat **contoh kode Rust** saat `rusb` dipadukan dengan soket Termux?
- Apakah Anda ingin mengetahui **perbedaan lisensi MIT dan LGPL** yang disebutkan di halaman tersebut?

q-c:
ingin melihat contoh kode Rust saat rusb dipadukan dengan soket Termux?

a-ai:
Untuk memadukan rusb dengan Termux tanpa root, rahasianya adalah **tidak membiarkan `rusb` memindai USB sendiri**. Kita harus meminta *File Descriptor* (FD) mentah dari perintah `termux-usb`, mengirimkannya lewat Unix Domain Socket (UDS), lalu memaksa `rusb` membuka FD tersebut.
Berikut adalah contoh implementasi kode Rust (`main.rs`) yang menggambarkan bagaimana pustaka seperti `libtermuxadb` bekerja di latar belakang:
```
use std::os::unix::io::RawFd;
use std::os::unix::net::UnixListener;
use std::io::Read;
use rusb::Context;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Buat Unix Domain Socket (UDS) untuk mendengarkan kiriman dari termux-usb
    let socket_path = "/tmp/termux_adb.sock";
    let _ = std::fs::remove_file(socket_path); // Hapus socket lama jika ada
    let listener = UnixListener::bind(socket_path)?;
    println!("Menunggu koneksi dari skrip jembatan Termux...");

    // 2. Menerima koneksi dan membaca File Descriptor (FD) yang dikirim
    let (mut stream, _) = listener.accept()?;
    let mut buffer = [0; 4];
    stream.read_exact(&mut buffer)?;
    
    // Mengubah biner menjadi angka integer (Raw File Descriptor)
    let usb_fd: RawFd = i32::from_le_bytes(buffer);
    println!("Berhasil menerima Akses USB File Descriptor dari Termux: {}", usb_fd);

    // 3. Inisialisasi konteks rusb
    let context = Context::new()?;

    // 4. BAGIAN KRUSIAL: Memaksa rusb menggunakan FD dari Termux
    // Menggunakan fungsi unsafe bawaan libusb untuk mengadopsi FD yang sudah diizinkan Android
    unsafe {
        // Peringatan: Fungsi open_device_with_fd di bawah ini adalah visualisasi logika 
        // yang melakukan bypass fungsi standar rusb::open() bawaan.
        let handle = context.open_device_with_fd(usb_fd)?;
        
        // 5. Sekarang Anda bisa berkomunikasi dengan HP target lewat USB secara legal!
        println!("Sukses mengikat jalur USB! Mengklaim antarmuka ADB...");
        handle.claim_interface(0)?;
        
        // Di sini Anda bisa melakukan handle.write_bulk() atau handle.read_bulk()
        // untuk mengirim perintah ADB/Fastboot standar.
    }

    Ok(())
}

```



q:
```
[dependencies]
anyhow = "1.0.66"
```

a-ai:
Baris kode tersebut adalah deklarasi dependensi dalam bahasa pemrograman Rust yang ditulis di dalam berkas `Cargo.toml`.
Potongan kode ini memerintahkan manajer paket Rust (Cargo) untuk mengunduh dan memasukkan pustaka (*crate*) bernama `anyhow` versi `1.0.66` ke dalam proyek Anda.
Berikut adalah penjelasan lengkap mengenai fungsi, cara kerja, dan alasan mengapa pustaka `anyhow` sangat populer digunakan dalam proyek berbasis Rust (termasuk pada ekosistem alat seperti `termux-adb`):

1. Apa itu anyhow?
Di dalam Rust, penanganan kesalahan (*error handling*) sangatlah ketat. Secara bawaan, Rust mewajibkan pengembang untuk mendefinisikan tipe data *error* yang spesifik untuk setiap fungsi.
`anyhow` adalah pustaka yang dirancang untuk mempermudah penanganan *error* pada aplikasi tingkat tinggi (aplikasi terminal/CLI, skrip, atau program utama). Pustaka ini menyediakan satu tipe data *error* universal bernama `anyhow::Result<T>` atau `anyhow::Error`.

2. Mengapa Memakai `anyhow`?
Tanpa pustaka ini, jika program Anda melakukan beberapa operasi yang menghasilkan tipe *error* berbeda—misalnya membaca file (`std::io::Error`) sekaligus melakukan komunikasi soket jaringan (`std::net::AddrParseError`)—Anda harus mengonversi tipe-tipe *error* tersebut secara manual agar fungsi Anda bisa mengembalikannya.
Dengan `anyhow`, semua tipe error yang mengimplementasikan trait bawaan `std::error::Error` akan **otomatis dikonversi** menggunakan operator tanda tanya (`?`).

3. Contoh Penggunaan di dalam Kode Rust
Berikut adalah visualisasi perbandingan kode sebelum dan sesudah menggunakan `anyhow`:
Tanpa `anyhow` (Rumit):
```
use std::fs::File;
use std::io::{self, Read};

// Anda harus menentukan tipe error yang kaku, dalam hal ini io::Error
fn baca_konfigurasi() -> Result<String, io::Error> {
    let mut file = File::open("config.txt")?; // menghasilkan io::Error
    let mut isi = String::new();
    file.read_to_string(&mut isi)?;
    Ok(isi)
}

```
Masalah muncul jika di dalam fungsi ini Anda tiba-tiba ingin menambahkan fungsi parsing teks dari pustaka lain, karena tipe *error*-nya tidak lagi berupa `io::Error`.
Dengan `anyhow` (Fleksibel & Bersih):
```
use anyhow::{Context, Result};
use std::fs::File;
use std::io::Read;

// Menggunakan Result universal dari anyhow
fn baca_konfigurasi_termux() -> Result<String> {
    // Jika File::open gagal, anyhow otomatis menangkap dan membungkus error-nya
    let mut file = File::open("config.txt")
        .context("Gagal membuka file konfigurasi")?; 

    let mut isi = String::new();
    file.read_to_string(&mut isi)?;
    Ok(isi)
}

```

4. Fitur Unggulan `anyhow`
- `context` / `with_context`: Anda bisa menambahkan pesan *error* kustom berantai (seperti contoh di atas). Jika program gagal, terminal akan mencetak urutan pesan yang jelas, misalnya: “Gagal membuka file konfigurasi: No such file or directory”.
- **Pencetakan Backtrace**: Jika terjadi *error*, `anyhow` bisa dikonfigurasi untuk mencetak urutan baris kode mana saja yang memicu kegagalan tersebut, sangat membantu untuk proses *debugging*.

5. Arti Angka Versi `"1.0.66"`
Penulisan versi di `Cargo.toml` mengikuti aturan *Semantic Versioning* (SemVer):
- Angka ini berarti Cargo akan mencari versi `1.0.66` atau versi yang lebih baru tetapi masih dalam rilis minor yang sama (misal `1.0.67`), selama pembaruan tersebut tidak merusak kode yang sudah ada (*non-breaking changes*).


q-c:
Jika Anda ingin melanjutkan eksperimen pemrograman Rust Anda, langkah mana yang ingin Anda pelajari berikutnya?
- Cara menambahkan **pesan error berantai (context)** yang benar
- Perbedaan antara pustaka `anyhow` dan pustaka `thiserror`



q-c:
Cara menambahkan pesan error berantai (context) yang benar

a-ai:
Untuk menambahkan pesan `error` berantai menggunakan pustaka `anyhow` di Rust, Anda harus memanfaatkan metode `.context()` atau `.with_context()`.
Fitur ini sangat berguna untuk memberikan kronologi atau riwayat kegagalan yang jelas saat program Anda mengalami masalah di terminal.
Berikut adalah panduan dan contoh cara menerapkannya dengan benar.

1. Menggunakan `.context()` (Untuk Teks Statis)
Gunakan `.context()` jika pesan kesalahan yang ingin Anda sampaikan berupa teks biasa (statis) dan tidak membutuhkan variabel atau pemformatan teks (string formatting).
```
use anyhow::{Context, Result};
use std::fs::File;

fn baca_data_usb() -> Result<()> {
    // Jika file tidak ada, akan memicu error dengan konteks tambahan
    let _file = File::open("/tmp/termux_adb.sock")
        .context("Gagal memuat soket komunikasi USB Termux.")?;
        
    Ok(())
}

```

2. Menggunakan `.with_context()` (Untuk Teks Dinamis)
Gunakan `.with_context()` jika Anda ingin memasukkan variabel ke dalam pesan kesalahan (seperti nama file atau angka ID). Metode ini menggunakan *closure* (`||`) sehingga pembuatan teks format hanya akan dieksekusi **jika terjadi error saja**. Ini jauh lebih hemat memori dan performa.
```
use anyhow::{Context, Result};
use std::fs::File;

fn buka_konfigurasi_perangkat(nama_perangkat: &str) -> Result<()> {
    let target_file = format!("/config/{}.toml", nama_perangkat);

    let _file = File::open(&target_file)
        .with_context(|| format!("Gagal membuka file konfigurasi untuk perangkat: '{}'", nama_perangkat))?;

    Ok(())
}

```

3. Cara Menampilkan Pesan Berantai ke Pengguna
Jika Anda hanya menggunakan `println!("{:?}", error)`, Rust akan mencetak *error* dalam satu baris panjang atau format mentah. Cara terbaik untuk mencetak pesan berantai (*error chain*) agar mudah dibaca oleh pengguna terminal adalah dengan melakukan perulangan menggunakan metode `.chain()`:
```
fn main() {
    // Menjalankan fungsi yang berpotensi error
    if let Err(err) = buka_konfigurasi_perangkat("Xiaomi_Poco") {
        
        // Cetak pesan utama paling atas
        eprintln!("Error Utama: {}", err);
        
        // Cetak rentetan penyebab di bawahnya (berantai)
        eprintln!("\nKronologi Kegagalan:");
        for (i, penyebab) in err.chain().skip(1).enumerate() {
            eprintln!("  [{}] Disebabkan oleh: {}", i + 1, penyebab);
        }
    }
}

```

Contoh Hasil Output di Terminal
Jika file konfigurasi di atas tidak ditemukan oleh sistem, program akan mencetak informasi yang sangat informatif seperti ini:
```
Error Utama: Gagal membuka file konfigurasi untuk perangkat: 'Xiaomi_Poco'

Kronologi Kegagalan:
  [1] Disebabkan oleh: No such file or directory (os error 2)

```
Dengan cara ini, Anda tahu persis apa error teknis dari sistem (`No such file or directory`), sekaligus tahu di bagian fungsi mana kegagalan itu terjadi (`Gagal membuka file konfigurasi untuk perangkat: 'Xiaomi_Poco'`).



q-ai:
Apakah Anda ingin beralih melihat perbedaan antara pustaka anyhow dan thiserror untuk memahami kapan harus menggunakan masing-masing pustaka tersebut?

q-ai:
Saya ingin beralih melihat perbedaan antara pustaka `anyhow` dan `thiserror` untuk memahami kapan harus menggunakan masing-masing pustaka tersebut?

a-ai:
Meskipun kedua pustaka (*crate*) ini dibuat oleh pengembang yang sama (**David Tolnay**), **anyhow** dan **thiserror** dirancang untuk tujuan yang sangat bertolak belakang.
Perbedaan mendasarnya terletak pada posisi kode Anda: apakah Anda sedang membuat **Aplikasi Akhir (Application)** atau **Pustaka Berbagi (Library)**.
Berikut adalah tabel komparasi dan penjelasan detailnya:
```
[
  {
    "fitur": "anyhow",
    "fokus_utama": "Memudahkan penangkapan error (Universal)."
  }
]
```

1. Kapan Menggunakan `anyhow`? (Fokus pada "Kemudahan")
Gunakan `anyhow` ketika Anda **tidak peduli apa tipe *error*-nya secara spesifik**, yang penting Anda tahu bahwa program tersebut gagal dan Anda ingin menampilkan pesan kegagalan yang informatif kepada pengguna terminal.
**Contoh Kasus**: Aplikasi CLI `termux-adb`. Jika koneksi gagal, aplikasi hanya perlu membatalkan proses dan mencetak *"Gagal terhubung ke USB"*. Aplikasi tidak perlu membedakan secara terprogram apakah kegagalan itu karena kabel rusak atau masalah izin internal.

2. Kapan Menggunakan `thiserror`? (Fokus pada "Kontrol")
Gunakan `thiserror` ketika Anda sedang membuat pustaka yang akan dipakai oleh programmer lain. Programmer yang memakai pustaka Anda membutuhkan kejelasan struktur agar mereka bisa membuat keputusan di dalam kode mereka berdasarkan jenis *error* yang terjadi.
Contoh Implementasi dengan `thiserror`:
```
use thiserror::Error;

#[derive(Error, Debug)]
pub enum UsbError {
    #[error("Perangkat USB dengan ID {0} tidak ditemukan")]
    DeviceNotFound(String),

    #[error("Izin akses USB ditolak oleh sistem Android")]
    PermissionDenied,

    #[error("Gagal membaca data dari endpoint")]
    IoError(#[from] std::io::Error), // Otomatis mengonversi io::Error ke UsbError
}

```
Mengapa ini penting bagi pengguna library Anda?
Karena mereka bisa melakukan kontrol alur seperti ini:
```
match hubungkan_ke_usb() {
    Ok(_) => println!("Sukses!"),
    Err(UsbError::PermissionDenied) => {
        // Aksi khusus: Minta ulang izin ke sistem Android
    },
    Err(UsbError::DeviceNotFound(id)) => {
        // Aksi khusus: Tampilkan pesan "Harap colokkan perangkat ID: {}", id
    },
    Err(UsbError::IoError(e)) => {
        // Aksi khusus: Tangani error sistem input-output
    }
}

```
Hal ini **tidak bisa** dilakukan jika Anda menggunakan `anyhow`, karena `anyhow` meleburkan semua jenis error tersebut menjadi satu tipe universal yang sama, sehingga tidak bisa dipisahkan lagi menggunakan `match`.

Kesimpulan Singkat
- Gunakan **`anyhow`** jika Anda adalah konsumen akhir dari error tersebut (misal: ingin langsung dicetak ke layar/log aplikasi).
- Gunakan `thiserror` jika Anda memproduksi *error* yang harus ditangani atau dibaca oleh kode lain (misal: membuat pustaka/modul terpisah).



q-c:
Jika Anda ingin melanjutkan, bagian mana yang menarik perhatian Anda berikutnya?
- Apakah Anda ingin melihat cara **mengombinasikan `anyhow`** dan `thiserror` dalam satu proyek yang sama?
- Apakah Anda ingin tahu **cara mempublikasikan pustaka Rust kustom Anda** ke situs Crates.io?



Berkas ini bertindak sebagai sebuah **pustaka pintasan (Hooking Library)** yang ditulis dalam bahasa Rust dan diekspos menggunakan fungsi C (`pub unsafe extern "C"`).
Fungsi utama dari seluruh kode ini adalah melakukan **"kebohongan sistem"** (*hooking* atau pencanduan fungsi). Program ini mencegat semua panggilan fungsi bawaan Linux (seperti membuka folder, membaca folder, dan membuka file) yang dilakukan oleh ADB bawaan Google, lalu mengalihkan jalurnya agar membaca data USB buatan Termux.
Berikut adalah penjelasan detail per blok fungsi dari potongan kode sumber tersebut:

1. Blok Manipulasi Folder Virtual (`opendir`, `closedir`, `readdir`)
Di dalam Linux/Android normal, ADB akan mencari perangkat USB yang terhubung dengan membuka folder sistem asli di `/dev/bus/usb/`. Karena Termux tanpa *root* tidak bisa membaca folder tersebut, kode ini memalsukannya:
- `termuxadb_opendir`: Saat ADB mencoba membuka folder `/dev/bus/usb/`, fungsi ini mencegatnya. Jika folder tersebut terdaftar di dalam sistem internal Termux (`DIR_MAP`), kode ini tidak membuka folder asli Android, melainkan membuat sebuah folder tiruan berbasis memori bernama `HookedDir::Virtual(DirStream)`.
- `termuxadb_readdir`: Saat ADB melakukan perulangan (*looping*) untuk membaca daftar file USB di dalam folder virtual tadi, fungsi ini akan menyuapi ADB dengan daftar perangkat tiruan yang didapat dari perintah `termux-usb -l`.
- `termuxadb_closedir`: Menutup folder tiruan dan membersihkan sisa memori (`drop`) agar tidak terjadi kebocoran memori RAM.

2. Blok Manipulasi File (`termuxadb_open` & `termuxadb_create`)
Saat ADB mendeteksi ada perangkat USB (dari folder tiruan di atas), ADB akan mencoba membuka file perangkat tersebut untuk mulai mengirimkan perintah.
- Kode ini mencegat fungsi open.
- Sistem akan memeriksa tabel memori (`USB_FD_MAP`) untuk melihat apakah file tersebut memiliki *File Descriptor* (FD) legal yang dikirim oleh perintah `termux-usb`.
- Jika ada, fungsi ini langsung mengembalikan nilai nomor FD tersebut (`return usb_fd;`), sehingga ADB merasa berhasil membuka file sistem asli, padahal ia sedang memakai jalur data aman yang diberikan oleh Android.

3. Pemalsuan Serial Number (`usb-serial`)
Setiap perangkat Android memiliki nomor seri (`Serial Number`) unik untuk membedakan HP satu dengan lainnya saat `debugging`.
- Jika ADB meminta data nomor seri perangkat, kode ini akan membuat sebuah file instan di dalam RAM menggunakan fungsi **`memfd_create`** dengan nama `usb-serial`.
- Teks nomor seri dari `USB_SERIAL_MAP` ditulis ke file bayangan tersebut, lalu diserahkan ke ADB.

4. Blok Jembatan Komunikasi Soket (**termuxadb_sendfd**)
Fungsi ini bertugas memindahkan hak akses USB dari proses Android ke proses latar belakang ADB.
- Program membaca variabel lingkungan (*environment variables*) milik Termux seperti `TERMUX_USB_FD` dan `TERMUX_ADB_SOCK_FD`.
- Menggunakan fitur Unix Domain Socket bawaan pustaka `sendfd` (`socket.send_with_fd`), program mengirimkan nomor *File Descriptor* USB mentah tersebut melintasi batasan proses (*inter-process communication*) agar bisa ditangkap oleh hook ADB.

5. Fungsi Pembantu & Manajemen Lingkungan
- `termuxadb_start` / `fastboot_start`: Fungsi inisialisasi awal saat perintah `termux-adb` pertama kali diketik oleh pengguna. Ini akan menyalakan sistem pencatatan log (`env_logger::init()`) dan memeriksa ketersediaan perintah pendukung di dalam sistem.
- `check_dependencies`: Memastikan utilitas paket `termux-usb` sudah terpasang di aplikasi Termux pengguna menggunakan bantuan fungsi `which`. Jika tidak ada, program akan langsung berhenti dan memicu *error* lewat pustaka `anyhow`.
- `clear_cloexec_flag`: Menghapus bendera `FD_CLOEXEC` pada soket komunikasi. Hal ini krusial agar jalur komunikasi soket data tidak otomatis hancur saat program memicu proses (*child process*) eksternal yang baru.



q-ai:
Memahami arti kata kunci `#[no_mangle]` yang ada di setiap fungsi C

a-ai:
Kata kunci `#[no_mangle]` adalah sebuah atribut khusus dalam bahasa pemrograman Rust yang memerintahkan kompiler agar **tidak mengubah nama fungsi** saat program dikompilasi menjadi biner atau pustaka (*library*).
Berikut adalah penjelasan detail mengapa kata kunci ini wajib ada di setiap fungsi jembatan C pada kode `termux-adb` Anda:

1. Apa itu Mangling?
Secara bawaan, kompiler Rust (dan juga kompiler C++) melakukan proses yang disebut **"Name Mangling"** (Pengacakan Nama).
Kompiler akan mengubah nama fungsi yang Anda tulis menjadi teks acak yang panjang dan rumit. Perubahan ini dilakukan untuk menyisipkan informasi tambahan seperti nama modul, tipe argumen, dan enkapsulasi guna mencegah bentrokan nama.
- Nama yang Anda tulis: `termuxadb_opendir`
- Nama setelah di-*mangle* oleh Rust (Contoh): `_ZN9termuxadb17termuxadb_opendir17hca30bfa4bd07c244E`

2. Mengapa *Mangling* Harus Dimatikan dengan `#[no_mangle]`?
Proyek `termux-adb` ini dirancang sebagai pustaka dinamis yang kodenya akan dipanggil oleh bahasa lain—dalam hal ini, sistem manajemen kompilasi C/C++ milik `android-tools`.
Jika nama fungsinya berubah menjadi teks acak bawaan Rust, maka sistem Android/Linux **tidak akan bisa menemukan fungsi tersebut**. Program C akan mencari fungsi bernama `termuxadb_opendir`, sementara di dalam file biner yang tersedia hanyalah nama acak `_ZN9termuxadb17...`. Akibatnya, aplikasi akan mengalami kegagalan tautan (*Linker Error / Symbol Not Found*).
Dengan menambahkan `#[no_mangle]`, kompiler Rust dipaksa untuk mengekspor fungsi tersebut ke dalam file biner dengan nama yang sama persis seperti yang Anda ketik, yaitu `termuxadb_opendir`.

3. Hubungan `#[no_mangle]` dengan `pub unsafe extern "C"`
Di dalam kode Anda, atribut `#[no_mangle]` selalu dipasangkan dengan dekorasi fungsi berikut:
```
#[no_mangle]
pub unsafe extern "C" fn termuxadb_opendir(name: *const c_char) -> *mut DIR { ... }

```
Ketiga kombinasi ini adalah standar wajib untuk melakukan **FFI (Foreign Function Interface)** atau komunikasi lintas bahasa di Rust:
- `pub`: Memastikan fungsi ini terbuka untuk umum (bisa diakses dari luar modul).
- `extern "C"`: Memerintahkan Rust untuk menggunakan tata cara pemanggilan fungsi (*Calling Convention*) standar milik bahasa C standar, bukan standar internal Rust.
- `#[no_mangle]`: Menjaga agar nama simbol fungsi di dalam memori tidak berubah, sehingga fungsi C bisa memanggilnya secara langsung dengan mengetik namanya.

Kesimpulan Singkat
Tanpa `#[no_mangle]`, program `android-tools` milik Google tidak akan pernah bisa "melihat" atau memanggil fungsi pintasan yang sudah Anda buat di Rust, karena nama fungsinya terkunci di dalam format enkapsulasi internal Rust.



q:
`enum HookedDir { Native(*mut DIR), Virtual(DirStream)}`

a-ai:
Struktur kode **`enum HookedDir`** yang Anda bagikan adalah komponen kunci yang digunakan untuk mengimplementasikan teknik **polimorfisme** dalam penanganan direktori.
Di dalam bahasa Rust, `enum` (enumerasi) tidak hanya berupa daftar pilihan angka seperti pada bahasa C atau Java, melainkan sebuah **Tagged Union** yang kuat. `enum` di Rust dapat menyimpan data yang berbeda di setiap pilihan (*variant*) fungsinya.
Berikut adalah penjelasan detail mengenai struktur dan fungsi dari `enum HookedDir` tersebut:

1. Membedah Dua Pilihan (Variants)
`HookedDir` diciptakan untuk menampung dua jenis folder yang sifatnya sangat bertolak belakang, tetapi harus bisa diperlakukan dengan cara yang sama oleh fungsi ADB:
- **`Native(*mut DIR)`**
-- **Fungsi**: Menampung penunjuk memori (*pointer*) langsung ke direktori asli bawaan sistem operasi Linux/Android.
-- **Tipe Data**: `*mut DIR` adalah tipe *pointer* mentah (*raw pointer*) dalam bahasa C untuk melacak aliran direktori fisik (diakses menggunakan fungsi bawaan seperti `libc::opendir`).
- **`Virtual(DirStream)`**
-- **Fungsi**: Menampung struktur buatan khusus yang berisi daftar perangkat USB tiruan hasil ekstraksi dari perintah `termux-usb -l`.
-- **Tipe Data**: `DirStream` adalah struktur data internal berbasis Rust yang mengelola posisi pembacaan folder bayangan di dalam memori RAM.

2. Mengapa Struktur Ini Sangat Penting?
Ketika kode program ADB milik Google memanggil fungsi untuk membaca isi folder (seperti `/dev/bus/usb/`), program Rust Anda harus memutuskan secara instan apakah folder tersebut adalah folder asli sistem atau folder virtual khusus Termux.
Dengan membungkusnya ke dalam satu `enum` yang sama, Anda bisa menyatukan kedua jenis data yang berbeda ini ke dalam satu wadah serbaguna. Perhatikan bagaimana `enum` ini digunakan pada fungsi `termuxadb_readdir` yang Anda bagikan sebelumnya menggunakan fitur pencocokan pola (*pattern matching*):
```
match hooked_dir {
    &mut HookedDir::Native(dirp) => {
        // JIKA ASLI: Teruskan perintah langsung ke fungsi bawaan Linux (libc::readdir)
        libc::readdir(dirp)
    }
    &mut HookedDir::Virtual(DirStream{ref mut pos, ref mut entry}) => {
        // JIKA VIRTUAL: Suapi ADB dengan data USB tiruan dari memori RAM Rust
        let idx = *pos as usize;
        if idx < entry.len() { ... }
    }
}

```

3. Hubungan dengan Implementasi `From` di Bawahnya
Tepat di bawah deklarasi `enum` ini pada kode sumber Anda, terdapat fungsi pengubah tipe data (*Type Casting*):
```
impl From<HookedDir> for *mut DIR {
    fn from(hd: HookedDir) -> Self {
        Box::into_raw(Box::new(hd)) as Self
    }
}

```
Fungsi ini bertugas memindahkan kepemilikan memori dari sistem manajemen Rust yang ketat ke sistem manajemen memori bebas milik bahasa C. Kode `Box::into_raw` berfungsi mengubah objek `HookedDir` Rust menjadi pointer mentah `*mut DIR` agar bisa dikembalikan dengan aman ke program ADB Google tanpa dihancurkan oleh sistem pembersih memori (*garbage collector* biner) milik Rust saat fungsi berakhir.



q-ai:
Jika Anda ingin melanjutkan pembahasan, opsi mana yang ingin Anda pelajari berikutnya?
- Melihat detail implementasi dari struktur `DirStream`
- Memahami cara kerja `Box::into_raw` dan `Box::from_raw` untuk transfer memori ke bahasa C

q:
`enum HookedDir { Native(*mut DIR), Virtual(DirStream)} Native dan Virtual dari mana ?`

a-ai:
`Native` dan `Virtual` pada kode tersebut **bukan berasal dari pustaka luar atau bawaan Rust**, melainkan nama label khusus (**Variants**) yang **dibuat sendiri oleh si pengembang (`nohajc`)** di dalam struktur `enum HookedDir`.
Di dalam bahasa Rust, saat kita membuat sebuah `enum` (Enumerasi), kita bebas menentukan nama label untuk setiap pilihan di dalamnya. Namun, data yang dibungkus di dalam label tersebut barulah berasal dari sistem atau bagian kode lain.
Berikut adalah rincian asal-usul dari mana data di balik nama `Native` dan `Virtual` itu diambil:

1. Data di Balik `Native` (Berasal dari OS Linux / Bahasa C)
Label `Native` digunakan untuk membungkus data `*mut DIR`.
- **Asal data**: Ini adalah pointer mentah (*raw pointer*) ke struktur `DIR` bawaan sistem operasi Linux/Android.
- **Sumber kode**: Tipe data `DIR` ini diimpor dari pustaka sistem bahasa C melalui *crate* `libc` (terlihat pada baris `use libc::{DIR, ...}` di bagian atas kode Anda).
- **Artinya**: Jika program ADB membuka folder biasa (bukan folder USB), kode Rust akan memanggil fungsi asli sistem (`libc::opendir`), lalu hasilnya dibungkus ke dalam label `Native(hasil_opendir)`.

2. Data di Balik `Virtual` (Berasal dari Struktur Kustom Rust)
Label `Virtual` digunakan untuk membungkus data `DirStream`.
- **Asal data**: Ini adalah sebuah `struct` (struktur data) kustom yang ditulis sendiri oleh pengembang di bagian lain dalam berkas proyek yang sama.
- **Fungsi `DirStream`**: Struktur ini dibuat untuk menyimpan daftar perangkat USB tiruan hasil pembacaan dari perintah `termux-usb -l` serta melacak posisi indeks pembacaannya (`pos`).
- **Artinya**: Jika program ADB mencoba membuka folder `/dev/bus/usb/`, kode Rust tidak akan membuka folder asli HP, melainkan membuat objek `DirStream` baru berisi daftar USB buatan, lalu membungkusnya ke dalam label `Virtual(daftar_usb_termux)`.

Kesimpulan Sederhana
Bayangkan `HookedDir` adalah sebuah **kotak berlabel** buatan Anda sendiri:
- Jika isinya barang asli dari sistem operasi, Anda menempelkan stiker **`Native`** pada kotak tersebut.
- Jika isinya barang tiruan buatan Rust/Termux, Anda menempelkan stiker **`Virtual`** pada kotak tersebut.



q-ai:
Apakah Anda ingin melihat **bagaimana struktur `struct DirStream` itu dideklarasikan** di dalam kode proyek ini agar lebih jelas?

q:
`#[derive(Clone, Debug)]struct DirStream { pos: i32, entry: Vec<dirent>,}impl From<&BTreeSet<DirEntry>> for DirStream { fn from(set: &BTreeSet<DirEntry>) -> Self { DirStream { pos: 0, entry: set.iter().map(|e| e.0).collect() } }}`

a-ai:
Struktur **`struct DirStream`** beserta implementasi fungsi `From` di atas adalah kode buatan pengembang (`nohajc`) yang berfungsi sebagai **mesin pembuat folder tiruan** di dalam memori RAM.
Fungsi utama dari kode ini adalah menampung daftar perangkat USB yang dideteksi oleh Termux, lalu menyuapinya satu per satu ke ADB seolah-olah data tersebut dibaca dari folder fisik HP.
Berikut adalah penjelasan detail per komponen dari potongan kode tersebut:

1. Membedah Variabel di dalam `struct DirStream`
Struktur ini melacak status pembacaan folder tiruan menggunakan dua variabel:
- **`pos: i32`** (Position): Ini adalah penunjuk indeks (kursor). Variabel ini melacak sampai baris mana ADB sudah membaca daftar USB tiruan Anda. Nilainya dimulai dari `0` dan akan bertambah `+1` setiap kali fungsi `readdir` dipanggil.
- **`entry: Vec<dirent>`**: Sebuah *Vector* (array dinamis) yang menyimpan data berstruktur `dirent`. Tipe `dirent` adalah struktur data standar Linux bahasa C untuk menyimpan informasi sebuah file atau folder (seperti nama file perangkat USB).

2. Arti Atribut `#[derive(Clone, Debug)]`
Atribut ini otomatis memerintahkan kompiler Rust untuk menyuntikkan fitur bawaan secara instan ke dalam `DirStream`:
- **`Clone`**: Memungkinkan struktur `DirStream` beserta seluruh isi array USB di dalamnya untuk digandakan/disalin secara utuh ke bagian memori lain jika diperlukan.
- **`Debug`**: Memungkinkan pengembang untuk mencetak isi dari struktur data ini ke terminal/log (menggunakan format `{:?}`) saat melacak bug, seperti yang terlihat pada baris `debug!("readdir: dirp is virtual DirStream");` di kode Anda sebelumnya.

3. Penjelasan Blok `impl From<&BTreeSet<DirEntry>> for DirStream`
Blok ini adalah fungsi konversi otomatis (*Type Casting*) untuk mengubah sekumpulan data terurut (`BTreeSet`) menjadi mesin pembaca folder (`DirStream`).
```
fn from(set: &BTreeSet<DirEntry>) -> Self {
    DirStream { 
        pos: 0, 
        // Mengambil data mentah .0 (dirent) dari DirEntry, lalu mengumpulkannya menjadi Vector
        entry: set.iter().map(|e| e.0).collect() 
    }
}

```
- `pos: 0`: Saat folder virtual USB baru pertama kali dibuat, kursor pembacaan selalu diatur dari angka `0` (paling awal).
- `set.iter().map(|e| e.0).collect()`: Kode ini melakukan perulangan pada koleksi `BTreeSet` yang berisi daftar USB Termux. Di setiap barisnya, ia mengekstrak komponen data bahasa C aslinya (`e.0` yang bertipe `dirent`), lalu mengumpulkannya kembali menjadi sebuah array `Vec<dirent>` yang rapi.

Hubungan dengan Fungsi `termuxadb_readdir` yang Anda Bagikan Sebelumnya
Struktur inilah yang dibaca ketika ADB melakukan pemindaian. Di dalam fungsi `termuxadb_readdir` sebelumnya, kode Rust melakukan pencocokan pola ke struktur ini:
```
&mut HookedDir::Virtual(DirStream{ref mut pos, ref mut entry}) => {
    let idx = *pos as usize; // Mengambil indeks kursor saat ini
    if idx < entry.len() {   // Jika kursor belum melewati jumlah total USB yang ada
        *pos += 1;           // Geser kursor ke baris berikutnya
        &mut entry[idx] as *mut dirent // Serahkan data USB baris ini ke ADB
    } else {
        null_mut()           // Jika sudah terbaca semua, beri tahu ADB bahwa folder sudah habis
    }
}

```



q-ai:
Apakah Anda ingin tahu **bagaimana struktur `DirEntry` (yang ada di dalam `BTreeSet`) itu dibuat** oleh pengembang untuk menyimpan nama file USB?

q:
Saya ingin tahu bagaimana struktur DirEntry (yang ada di dalam BTreeSet) itu dibuat oleh pengembang untuk menyimpan nama file USB?

a-ai:
Di dalam kode repositori `vendor-adb-patched`, pengembang (`nohajc`) membuat `struct DirEntry` menggunakan pola bernama *Tuple Struct*.
Tujuan utama dari `DirEntry` ini adalah membungkus struktur data bahasa C (`libc::dirent`) ke dalam tipe data Rust, agar data file USB tersebut **bisa diurutkan dan dimasukkan ke dalam `BTreeSet`**.
Berikut adalah bentuk deklarasi kode aslinya beserta penjelasan cara pembuatannya:

1. Deklarasi Kode `DirEntry`
```
#[derive(Clone, Copy, Debug)]
struct DirEntry(dirent);
```
Struktur ini sangat sederhana. Ia tidak memiliki nama variabel di dalamnya, melainkan langsung membungkus sebuah data bertipe `dirent` (struktur standar Linux bahasa C untuk mendeskripsikan sebuah file/direktori, termasuk nama file USB).

2. Mengapa Harus Dibungkus? (Alasan Penggunaan `BTreeSet`)
Koleksi `BTreeSet` di Rust memiliki aturan yang sangat ketat: setiap data yang masuk ke dalamnya **wajib bisa dibandingkan ketersamaannya (`Eq`) dan diurutkan (`Ord`)**.
Masalahnya, tipe `dirent` bawaan dari pustaka C (`libc`) tidak memiliki fitur pengurutan tersebut karena batasan bahasa C. Oleh karena itu, pengembang membungkusnya menjadi `DirEntry` dan menulis aturan pembanding sendiri (*trait implementation*) seperti ini:
```
// 1. Aturan Ketersamaan (Apakah dua file USB ini sama?)
impl PartialEq for DirEntry {
    fn eq(&self, other: &Self) -> bool {
        // Membandingkan nama file USB menggunakan fungsi bahasa C 'strcmp'
        unsafe { strcmp(self.0.d_name.as_ptr(), other.0.d_name.as_ptr()) == 0 }
    }
}
impl Eq for DirEntry {}

// 2. Aturan Pengurutan (Bagaimana cara mengurutkan nama file USB di memori?)
impl Ord for DirEntry {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        unsafe {
            let res = strcmp(self.0.d_name.as_ptr(), other.0.d_name.as_ptr());
            if res < 0 {
                std::cmp::Ordering::Less
            } else if res > 0 {
                std::cmp::Ordering::Greater
            } else {
                std::cmp::Ordering::Equal
            }
        }
    }
}

impl PartialOrd for DirEntry {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

```

3. Cara Pengembang Mengisi Data Nama File USB
Ketika perintah `termux-usb -l` mendeteksi perangkat USB (misalnya terdeteksi di jalur `/dev/bus/usb/001/002`), pengembang akan membuat objek `DirEntry` secara instan lewat logika berikut:
  1. Membuat objek `dirent` kosong milik bahasa C.
  2. Mengisi variabel `d_name` di dalam objek tersebut dengan teks nama file perangkat (misalnya `"002"`).
  3. Mengatur tipe filenya menjadi `DT_CHR` (Character Device, standar Linux untuk perangkat keras USB).
  4. Memasukkannya ke dalam `DirEntry: let entry = DirEntry(objek_dirent_tadi);`.
  5. Terakhir, memasukkannya ke dalam `BTreeSet`. Karena aturan `Ord` yang sudah dibuat di atas, `BTreeSet` akan otomatis mengurutkan nama-nama file USB tersebut (misal: `"001"`, `"002"`, `"003"`) dengan sangat rapi di dalam memori.

Langkah ini memastikan bahwa saat fungsi `from` pada `DirStream` yang kita bahas sebelumnya dipanggil (`set.iter().map(|e| e.0).collect()`), array yang dihasilkan sudah dalam kondisi terurut secara alfabetis dan siap disantap oleh ADB Google.



q-ai:
Apakah Anda ingin tahu **cara pengembang menyalin teks String Rust** ke dalam array karakter (`d_name`) milik bahasa C tersebut karena tipe datanya berbeda?

q:
Saya ingin tahu cara pengembang menyalin teks String Rust ke dalam array karakter (`d_name`) milik bahasa C tersebut karena tipe datanya berbeda?

a-ai:
Di dalam Rust, `String` disimpan sebagai teks berbasis pengodean **UTF-8 dinamis** yang panjangnya fleksibel di memori Heap. Sementara itu, variabel `d_name` di dalam struktur `dirent` bahasa C adalah sebuah **array statis bertipe** `[c_char; N]` dengan ukuran tetap (ukurannya bervariasi tergantung arsitektur sistem, biasanya 256 karakter) dan diakhiri oleh karakter kosong `\0` (*Null-terminated*).
Karena kedua tipe data ini memiliki struktur fisik memori yang sangat berbeda, pengembang menggunakan trik manipulasi memori tingkat rendah (*unsafe pointer copying*).
Berikut adalah langkah-langkah teknis dan contoh kode bagaimana pengembang menyalin data tersebut:

Langkah 1: Mengubah String Rust Menjadi `CString`
Rust memiliki pustaka bawaan `std::ffi::CString`. Pustaka ini berfungsi untuk mengambil String Rust biasa, lalu memformatnya ulang di memori agar kompatibel dengan bahasa C (menambahkan karakter penutup `\0` di bagian paling akhir teks).
```
use std::ffi::CString;

let nama_usb_rust = "002"; // String dari Rust
let c_string_usb = CString::new(nama_usb_rust).unwrap(); // Menjadi "002\0"

```

Langkah 2: Proses Menyalin ke Array `d_name` menggunakan `ptr::copy_nonoverlapping`
Setelah mendapatkan data berformat bahasa C, pengembang akan membuat objek `dirent` kosong, lalu memindahkan bit data teks tersebut menggunakan fungsi `std::ptr::copy_nonoverlapping`. Fungsi ini adalah padanan langsung dari perintah `memcpy` di bahasa C.
Berikut adalah visualisasi baris kode di balik layar yang dilakukan oleh pengembang:
```
use libc::{dirent, c_char};
use std::ptr;

// 1. Siapkan objek dirent bahasa C yang kosong
let mut objek_dirent = unsafe { std::mem::zeroed::<dirent>() };

// 2. Ambil pointer asal (dari CString Rust) dan pointer tujuan (array d_name)
let src_ptr = c_string_usb.as_ptr();                     // Pointer teks "002\0"
let dest_ptr = objek_dirent.d_name.as_mut_ptr() as *mut c_char; // Pointer target di C

// 3. Hitung panjang teks yang aman agar tidak meluap (Buffer Overflow)
let panjang_teks = nama_usb_rust.len() + 1; // +1 untuk menyertakan karakter '\0'

// 4. Lakukan penyalinan memori mentah secara instan
unsafe {
    ptr::copy_nonoverlapping(src_ptr, dest_ptr, panjang_teks);
}

```

Mengapa Cara Ini Aman dan Penting Bagi ADB?
1. **Bypass Batasan Tipe Data**: Melalui fungsi `as_mut_ptr()`, array kaku `d_name` dipaksa membuka jalur memori mentahnya. Hal ini memungkinkannya diisi langsung menggunakan data biner dari Rust.
2. **Kecepatan Tinggi**: Karena menyalin langsung di level blok memori RAM (*raw memory copy*), proses manipulasi folder tiruan USB ini berjalan secara instan tanpa membebani performa HP Android Anda.
3. **Kompatibilitas Penuh**: Setelah fungsi `ptr::copy_nonoverlapping` selesai, isi dari variabel `objek_dirent.d_name` kini sudah berformat bahasa C murni. Saat objek ini diserahkan kembali ke ADB bawaan Google, ADB akan membacanya sebagai file eksternal yang sah tanpa menyadari bahwa file tersebut sebenarnya adalah hasil manipulasi kode Rust Anda.



q-ai:
Jika Anda tertarik mendalami mekanismenya lebih jauh, opsi mana yang ingin Anda pelajari berikutnya?
- Cara kerja fungsi `std::mem::zeroed` saat menginisialisasi objek bahasa C
- Memahami bahaya **Buffer Overflow** jika nama file USB terlalu panjang

q:
Saya tertarik mendalami mekanismenya lebih jauh, opsi mana yang ingin Anda pelajari berikutnya?

a-ai:
Mari kita bedah kedua aspek penting ini untuk memahami bagaimana pengembang menjaga program tetap aman dan stabil saat memanipulasi memori tingkat rendah (*low-level memory manipulation*).

Opsi 1: Cara Kerja Fungsi `std::mem::zeroed`
Di dalam bahasa C, ketika Anda membuat sebuah variabel atau struktur data (*struct*) baru tanpa memberikan nilai awal, area memori yang dialokasikan akan berisi **"data sampah"** (sisa data dari program lain yang pernah menggunakan area RAM tersebut).
Di Rust, hal ini sangat dilarang karena tidak aman. Namun, saat bekerja dengan FFI (Foreign Function Interface) untuk membuat struktur C seperti `libc::dirent`, kita membutuhkan cara untuk mengosongkan seluruh blok memori tersebut terlebih dahulu. Di sinilah fungsi `std::mem::zeroed` digunakan.
Bagaimana Fungsi Ini Bekerja?
```
let mut objek_dirent = unsafe { std::mem::zeroed::<dirent>() };

```
1. **Alokasi Ukuran yang Tepat**: Kompiler Rust akan menghitung berapa ukuran byte total yang dibutuhkan oleh struktur `dirent` pada arsitektur HP Anda (misalnya 280 byte).
2. **Pengisian Angka Nol**: Fungsi ini akan langsung mengisi seluruh bit di area memori tersebut dengan angka `0` (**bit zero**).
3. **Hasil Instan**: Semua variabel di dalam `dirent` otomatis memiliki nilai awal yang aman: variabel angka menjadi 0, variabel boolean menjadi `false`, dan array karakter `d_name` menjadi bersih dari karakter sampah.
Catatan: Fungsi ini wajib dibungkus dalam blok `unsafe` karena jika digunakan pada tipe data asli Rust yang memiliki aturan ketat (seperti tipe `String` atau `Box`), mengisi memori dengan angka nol bisa merusak pointer internal Rust dan menyebabkan aplikasi macet (*crash*).

Opsi 2: Bahaya Buffer Overflow pada Array `d_name`
*Buffer Overflow* adalah kondisi di mana program menulis data ke dalam suatu wadah (buffer) melebihi kapasitas yang bisa ditampung oleh wadah tersebut. Akibatnya, data yang meluap akan **menimpa variabel lain** yang berada tepat di sebelahnya di dalam memori RAM.
Di dalam struktur `libc::dirent`, array `d_name` memiliki batas ukuran tetap. Mari kita lihat visualisasi strukturnya di memori:
```
[ ... variabel lain ... ][ array d_name (Kapasitas: 256 byte) ][ variabel penting lain / return address ]

```
Skenario Bahaya di Termux-ADB
Bayangkan jika perintah `termux-usb` mendeteksi sebuah nama jalur perangkat keras tiruan yang sangat panjang (misalnya hasil manipulasi atau bug sistem) sepanjang 300 karakter.
Jika pengembang langsung menyalin teks tersebut menggunakan fungsi mentah seperti ini:
```
// BERBAHAYA: Tidak memeriksa batas kapasitas wadah tujuan
ptr::copy_nonoverlapping(src_ptr, dest_ptr, panjang_teks_300_byte);

```
Maka yang terjadi adalah:
1. Wadah `d_name` yang hanya berkapasitas 256 byte akan terisi penuh.
2. 44 byte sisa data yang meluap akan **meluber keluar** dan mulai merusak data di blok memori sebelahnya.
3. Di dalam sistem Android/Linux, hal ini akan memicu error keamanan fatal bernama **`Segmentation Fault (SIGSEGV)`**, yang seketika itu juga akan mematikan paksa aplikasi `termux-adb` Anda. Pada skenario terburuk dalam dunia keamanan siber, celah meluapnya memori ini bisa dimanfaatkan oleh peretas untuk menyuntikkan kode berbahaya (*Arbitrary Code Execution*).
Cara Pengembang Menghindarinya
Untuk mencegah malapetaka tersebut, pengembang biasanya membatasi panjang teks yang akan disalin dengan melakukan pemotongan (*truncation*) atau pengecekan menggunakan fungsi batas atas, memastikan nilai `panjang_teks` tidak akan pernah melebihi kapasitas maksimal dari `d_name` (misalnya menggunakan `std::cmp::min(nama_usb.len(), 255)`).



q:
`impl From<HookedDir> for *mut DIR { fn from(hd: HookedDir) -> Self { Box::into_raw(Box::new(hd)) as Self }}`

a-ai:
Potongan kode di atas adalah implementasi dari trait bawaan Rust bernama `From`. Fungsi utamanya adalah **mengubah objek `HookedDir` milik Rust menjadi pointer mentah (raw pointer) bahasa C (&ast;mut DIR)** agar bisa diserahkan kembali ke program utama ADB bawaan Google.
Berikut adalah penjelasan detail langkah demi langkah dari satu baris kode di dalam fungsi tersebut:

1. `Box::new(hd)` (Memindahkan Data ke Memori Heap)
Secara bawaan, objek HookedDir yang dibuat di dalam fungsi Rust akan disimpan di memori Stack. Masalahnya, data di memori Stack akan **otomatis dihapus (dihancurkan)** ketika fungsi tersebut selesai berjalan.
Agar data folder (baik yang `Native` maupun `Virtual`) tidak hilang saat fungsi berakhir, pengembang membungkusnya dengan `Box::new()`. Perintah ini memindahkan objek tersebut ke memori **Heap** (memori jangka panjang) yang tidak akan dihapus secara otomatis oleh Rust.

2. `Box::into_raw(...)` (Melepas Kendali Sistem Rust)
Rust memiliki sistem manajemen memori yang sangat ketat. Jika kita hanya menggunakan `Box`, Rust akan tetap melacaknya dan menghapusnya jika dirasa sudah tidak dipakai lagi.
Fungsi `Box::into_raw` bertugas untuk **mematikan sistem pelacakan otomatis** tersebut. Kode ini menginstruksikan Rust: *"Jangan sentuh atau hapus memori ini. Saya akan menyerahkan kendalinya secara manual ke bahasa C."* Fungsi ini mengembalikan sebuah pointer mentah (*raw pointer*) bertipe `&ast;mut HookedDir`.

3. `as Self` (Mengubah Tipe Data ke Format C)
Kata kunci `Self` di sini merujuk pada tipe data target yang ditentukan di baris awal, yaitu `&ast;mut DIR`.
Karena program ADB Google hanya mengerti tipe data pointer `&ast;mut DIR` (struktur folder standar bahasa C), maka pointer `&ast;mut HookedDir` buatan Rust tadi dipaksa diubah tipenya (Type Casting) menggunakan perintah `as Self`.

Alur Kebalikannya: `Box::from_raw`
Memori Heap yang sudah dilepas ke bahasa C di atas **wajib dibersihkan** jika folder sudah selesai dibaca agar tidak terjadi kebocoran RAM (*Memory Leak*).
Jika Anda perhatikan kembali kode fungsi `termuxadb_closedir` yang Anda bagikan di awal percakapan kita, pengembang melakukan proses penarikan kembali memori tersebut menggunakan fungsi kebalikannya:
```
let hooked_dir = Box::from_raw(dirp as *mut HookedDir);

```
Saat ADB Google memanggil `closedir`, pointer `dirp` diserahkan kembali ke Rust. Rust mengambil alih lagi kepemilikannya menggunakan `Box::from_raw`. Begitu fungsi `closedir` selesai, Rust akan menghapus seluruh data folder tersebut dari memori Heap secara bersih dan aman.



q:
cara membuktikan pernyataan "Fungsi Box::into_raw bertugas untuk mematikan sistem pelacakan otomatis tersebut. Kode ini menginstruksikan Rust: "Jangan sentuh atau hapus memori ini. Saya akan menyerahkan kendalinya secara manual ke bahasa C." Fungsi ini mengembalikan sebuah pointer mentah (raw pointer) bertipe `&ast;mut HookedDir`."





<br>

