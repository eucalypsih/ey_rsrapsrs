# drop

q:
instruksi tingkat rendah (low-level compiler intrinsic) di Rust yang mendesak kompiler untuk mengabaikan suatu objek dan melarang compiler untuk menjalankan kode pembersihan memori (drop) pada objek tertentu

Instruksi yang Anda maksud adalah **`std::mem::forget`**.

Fungsi ini memberi tahu kompiler Rust untuk mengambil kepemilikan (_ownership_) dari suatu objek dan mengabaikannya tanpa menjalankan fungsi pembersihan memori (`drop`).

**Detail Teknis** `std::mem::forget`

- **Fungsi Utama**: Memindahkan nilai keluar dari cakupan (_scope_) saat ini sehingga destruktor (`drop`) tidak pernah dipanggil.
- **Manfaat**: Membuka kendali penuh atas siklus hidup objek, terutama saat berinteraksi dengan kode C/C++ (_FFI_) atau mengelola alokasi memori manual.
- **Keamanan**: Fungsi ini **aman (safe)** digunakan di Rust. Rust menjamin keselamatan memori (_memory safety_), namun kebocoran memori (_memory leak_) tidak dianggap sebagai pelanggaran keselamatan memori oleh Rust.

**Contoh Penggunaan**
```rust
use std::mem;

struct SumberDaya {
    data: String,
}

impl Drop for SumberDaya {
    fn drop(&mut self) {
        println!("Memori dibersihkan!");
    }
}

fn main() {
    let objek = SumberDaya {
        data: String::from("Data Penting"),
    };

    // Memaksa kompiler mengabaikan objek dan melarang fungsi drop berjalan
    mem::forget(objek); 

    // Output "Memori dibersihkan!" TIDAK AKAN pernah tercetak
}
```

**Alternatif Modern**: `std::mem::ManuallyDrop`

Pada Rust modern, penggunaan `std::mem::ManuallyDrop` lebih disukai daripada `std::mem::forget`. `ManuallyDrop` adalah pembungkus (_wrapper_) yang secara bawaan mencegah fungsi `drop` dipanggil, kecuali jika Anda memanggilnya secara manual lewat kode.

Apakah Anda sedang merancang kode untuk **interoperabilitas dengan bahasa C (FFI)** atau memerlukan bantuan untuk **mengelola _pointer_ mentah (_raw pointers_)** menggunakan instruksi ini?

<br>

q:
penggunaan `std::mem::ManuallyDrop` lebih disukai daripada `std::mem::forget`. `ManuallyDrop` adalah pembungkus (_wrapper_) yang secara bawaan mencegah fungsi `drop` dipanggil, kecuali jika Anda memanggilnya secara manual lewat kode.

<br>

a-ai:
Betul sekali, analisis Anda sangat tepat. `std::mem::ManuallyDrop` adalah solusi yang jauh lebih bersih dan aman di Rust modern.

Berikut adalah alasan utama mengapa `ManuallyDrop` lebih disukai dan bagaimana ia bekerja:

**Mengapa** `ManuallyDrop` **Lebih Baik?**
- **Mencegah Panic Leak**: Jika kode Anda mengalami panic sebelum `mem::forget` dipanggil, objek tersebut akan tetap terkena `drop`. `ManuallyDrop` membungkus objek sejak awal, sehingga jika terjadi _panic_, objek di dalamnya dijamin aman dari `drop`.
- **Ekspresif & Jelas**: Tipe datanya langsung memberi tahu pembaca kode bahwa memori objek ini dikelola secara manual sejak dideklarasikan.
- **Fleksibilitas Tinggi**: Anda bisa mengambil kembali objek di dalamnya kapan saja menggunakan `ManuallyDrop::into_inner(slot)`.

**Contoh Penggunaan** `ManuallyDrop`
```rust
use std::mem::ManuallyDrop;

struct Senjata {
    nama: String,
}

impl Drop for Senjata {
    fn drop(&mut self) {
        println!("{} hancur dan dibersihkan!", self.nama);
    }
}

fn main() {
    // Membungkus objek agar fungsi drop tidak otomatis berjalan
    let mut pedang = ManuallyDrop::new(Senjata {
        nama: String::from("Excalibur"),
    });

    // ... lakukan operasi kompleks di sini ...

    // Jika ingin menghapusnya secara manual lewat kode:
    unsafe {
        ManuallyDrop::drop(&mut pedang); // Mencetak: "Excalibur hancur dan dibersihkan!"
    }
}

```

<br>

q-ai:
Apakah Anda ingin melihat contoh bagaimana `ManuallyDrop` digunakan dalam skenario **FFI (Foreign Function Interface)**, atau Anda ingin tahu **cara mengambil kembali nilai di dalamnya** tanpa memicu `drop`?

<br>

q-c:
Saya ingin tahu **cara mengambil kembali nilai di dalamnya** tanpa memicu `drop`?

<br>

a-ai:
Untuk mengambil kembali nilai di dalam `ManuallyDrop` tanpa memicu fungsi `drop`, Anda harus menggunakan fungsi **`ManuallyDrop::into_inner(slot)`**.

Fungsi ini akan mengekstrak nilai asli dan mengembalikan kepemilikannya (_ownership_) kepada Anda. Setelah diekstrak, nilai tersebut akan kembali tunduk pada aturan manajemen memori Rust yang normal.

Contoh Kode
```rust
use std::mem::ManuallyDrop;

struct Kotak {
    isi: String,
}

impl Drop for Kotak {
    fn drop(&mut self) {
        println!("Destruktor dipanggil: Kotak '{}' dihancurkan!", self.isi);
    }
}

fn main() {
    // 1. Bungkus objek ke dalam ManuallyDrop
    let kotak_aman = ManuallyDrop::new(Kotak {
        isi: String::from("Emas"),
    });

    // 2. Ambil kembali nilai di dalamnya
    // Ini benar-benar aman (Safe Code) dan tidak memerlukan blok unsafe!
    let nilai_asli = ManuallyDrop::into_inner(kotak_aman);

    println!("Berhasil mengambil objek dengan isi: {}", nilai_asli.isi);

    // 3. Di akhir scope main, `nilai_asli` akan di-drop secara otomatis oleh Rust
    println!("Akhir dari fungsi main.");
}

```

**Output dari Kode di Atas**
```
Berhasil mengambil objek dengan isi: Emas
Akhir dari fungsi main.
Destruktor dipanggil: Kotak 'Emas' dihancurkan!

```

**Mengapa Cara Ini Sangat Aman?**
- **Pindah Kepemilikan (_Move_)**: `into_inner` mengambil kepemilikan dari pembungkus `ManuallyDrop`. Karena pembungkusnya dikonsumsi (hancur), objek di dalamnya pindah ke variabel baru (`nilai_asli`).
- **Tanpa Blok unsafe**: Berbeda dengan fungsi `ManuallyDrop::drop` yang membutuhkan blok `unsafe`, fungsi `into_inner` adalah **fungsi aman (_safe function_)** karena Rust menjamin nilai tersebut hanya berpindah tempat dan tidak akan terduplikasi.

<br>

q-ai:
Jika Anda tertarik, saya bisa menunjukkan cara kerja `into_inner` ini dalam struktur data tingkat lanjut seperti **membuat implementasi `Vec` kustom** atau bagaimana menggunakannya bersama **pointer mentah (_raw pointer_)**. Mana yang ingin Anda pelajari lebih lanjut?

<br>

q:
Di akhir scope main, `nilai_asli` akan di-drop secara otomatis oleh Rust, apabila program dipaksa berhenti CTRL-C apakah nilai_asli masih belum didrop ?

<br>

a-ai:
**Benar, `nilai_asli` tidak akan di-_drop_** jika program dipaksa berhenti menggunakan CTRL-C (sinyal `SIGINT`).

Saat Anda menekan CTRL-C, sistem operasi akan langsung menghentikan proses program secara paksa. Akibatnya, Rust tidak sempat menjalankan kode pembersihan memori (_stack unwinding_ atau destruktor `drop`).

**Mengapa CTRL-C Membatalkan drop?**
- **Penghentian Paksa**: Sinyal `SIGINT` (CTRL-C) secara bawaan akan langsung membunuh proses aplikasi.
- **Alokasi OS yang Menghapus**: Semua memori yang digunakan oleh program (termasuk isi dari `nilai_asli`) tetap akan dibersihkan dan diambil kembali oleh **Sistem Operasi**, bukan oleh fungsi `drop` dari Rust.
- **Efek Samping Tertinggal**: Jika fungsi `drop` Anda bertugas menutup koneksi basis data, menghapus file temporer, atau menyimpan status ke disk, proses tersebut **tidak akan berjalan**.

**Cara Mengatasi Agar `drop` Tetap Berjalan Saat CTRL-C**
Jika Anda ingin program sempat melakukan pembersihan (_cleanup_) sebelum mati karena CTRL-C, Anda harus menangkap sinyal tersebut secara manual menggunakan _crate_ pihak ketiga seperti `ctrlc`.

Berikut adalah contoh penerapannya:
```rust
use std::mem::ManuallyDrop;
use std::process;

struct Kotak {
    isi: String,
}

impl Drop for Kotak {
    fn drop(&mut self) {
        println!("Destruktor dipanggil: Kotak '{}' BERHASIL dihancurkan!", self.isi);
    }
}

fn main() {
    let kotak_aman = ManuallyDrop::new(Kotak { isi: String::from("Emas") });
    let nilai_asli = ManuallyDrop::into_inner(kotak_aman);

    // Menangkap sinyal CTRL-C secara eksplisit
    ctrlc::set_handler(move || {
        println!("\nMenerima sinyal CTRL-C! Keluar dengan aman...");
        
        // Memaksa program keluar, namun Anda bisa memanggil cleanup di sini jika variabel berada di scope global/atomic.
        // Catatan: Jika nilai_asli ada di stack main, exit() biasa tetap tidak memanggil drop.
        process::exit(0); 
    }).expect("Gagal mengatur handler CTRL-C");

    println!("Program berjalan. Tekan CTRL-C untuk menguji...");
    
    // Simulasi program berjalan terus menerus
    loop {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}

```

<br>

q-ai:
Apakah Anda sedang membuat program yang **memiliki efek samping penting saat `drop`** (seperti menulis ke file) sehingga perlu memastikan pembersihan saat program mati mendadak?

<br>

q:
penjelasan, Jika nilai_asli ada di stack main, exit() biasa tetap tidak memanggil drop.

<br>

a-ai:
Pernyataan tersebut sangat akurat. Ketika Anda memanggil `std::process::exit()`, program akan **langsung berhenti seketika** di baris tersebut. Sistem operasi (OS) langsung mengambil alih untuk menarik semua memori, dan Rust tidak diberi kesempatan untuk membersihkan variabel yang ada di dalam _stack_ fungsi `main`.

Berikut adalah penjelasan mendalam mengapa hal ini terjadi dan bagaimana mekanisme internalnya bekerja.

**Mengapa `process::exit()` Melewati Proses `drop`?**
1. **Tidak Ada Stack Unwinding (Pembongkaran Stack)**:
Saat program keluar secara normal (mencapai akhir tanda kurung kurawal `}` fungsi `main`), Rust akan melakukan _stack unwinding_. Rust akan berjalan mundur di dalam _stack frame_ dan memanggil fungsi `drop` pada setiap variabel yang masih aktif satu per satu. Namun, `process::exit()` adalah instruksi tingkat sistem operasi yang memberi tahu OS: _"Hentikan proses ini sekarang juga."_ Akibatnya, proses pembongkaran _stack_ ini dilewati seluruhnya.
2. **Kendali Langsung ke OS**:
Fungsi `process::exit()` secara internal memanggil _system call_ milik OS (seperti `exit` di Linux/macOS atau `ExitProcess` di Windows). OS akan menghapus seluruh ruang memori (_address space_) milik program tersebut dari RAM. Memori Anda memang bersih, tetapi fungsi `drop` kustom Anda tidak pernah dieksekusi.

**Visualisasi Perbedaan Alur Program**
- **Alur Keluar Normal (`main` selesai)**:
`main()` dimulai \(\rightarrow \) Variabel dibuat \(\rightarrow \) Kode selesai \(\rightarrow \) **Rust membongkar stack (Drop dijalankan) \(\rightarrow \)** OS menutup program.
- **Alur Keluar via `process::exit()`**:
`main()` dimulai \(\rightarrow \) Variabel dibuat \(\rightarrow \) `process::exit()` dipanggil \(\rightarrow \) **OS langsung membunuh program (Drop LEWAT/TIDAK JALAN)**.

**Cara Mengatasi: Bagaimana agar `drop` tetap berjalan?**
Jika Anda harus menghentikan program di tengah jalan tetapi ingin variabel di _stack main_ tetap di-_drop_, ada dua strategi utama yang bisa Anda gunakan:

**Strategi 1: Gunakan Pola Arsitektur "Outer Main" (Sangat Direkomendasikan)**
Metode ini memindahkan seluruh logika program utama ke dalam fungsi terpisah (misalnya `run()`). Fungsi `main` hanya bertugas memeriksa status keluar dan memanggil `process::exit()`.
```rust
use std::process;

struct SumberDaya { name: String }
impl Drop for SumberDaya {
    fn drop(&mut self) { println!("{} berhasil dibersihkan!", self.name); }
}

fn main() {
    // Fungsi run() akan selesai sepenuhnya, sehingga semua variabel di dalamnya 
    // di-drop terlebih dahulu SEBELUM process::exit() dieksekusi.
    if let Err(code) = run() {
        process::exit(code);
    }
}

fn run() -> Result<(), i32> {
    let _nilai_asli = SumberDaya { name: String::from("Data_Stack") };
    
    let terjadi_error_fatal = true;
    if terjadi_error_fatal {
        println!("Terjadi kesalahan fatal!");
        return Err(1); // Keluar dari fungsi run(), memicu drop pada _nilai_asli
    }

    Ok(())
}

```
**Output:**
```
Terjadi kesalahan fatal!
Data_Stack berhasil dibersihkan!

```

**Strategi 2: Memicu _Panic_ Terkontrol (`panic=unwind`)**
Secara bawaan, jika program mengalami `panic!`, Rust akan melakukan pembongkaran _stack_ dan menjalankan fungsi `drop` sebelum program mati. Anda bisa menangkap _panic_ tersebut di tingkat atas.

_(Catatan: Strategi ini tidak akan bekerja jika Anda mengompilasi Rust dengan profil `panic = "abort"` di file `Cargo.toml`, karena opsi `abort` juga akan langsung mematikan program tanpa `drop`)._

<br>

q-ai:
Apakah program yang sedang Anda bangun membutuhkan mekanisme **penyelamatan status data terakhir** sebelum program ditutup paksa, atau Anda ingin mendiskusikan implementasi arsitektur **"Outer Main"** yang lebih kompleks?





<br>


