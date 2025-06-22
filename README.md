# FUSecure

Yuadi adalah seorang developer brilian yang sedang mengerjakan proyek top-secret. Sayangnya, temannya Irwandi yang terlalu penasaran memiliki kebiasaan buruk menyalin jawaban praktikum sistem operasi Yuadi tanpa izin setiap kali deadline mendekat. Meskipun sudah diperingatkan berkali-kali dan bahkan ditegur oleh asisten dosen karena sering ketahuan plagiat, Irwandi tetap tidak bisa menahan diri untuk mengintip dan menyalin jawaban praktikum Yuadi. Merasa kesal karena prestasinya terus-menerus dicuri dan integritasnya dipertaruhkan, Yuadi yang merupakan ahli keamanan data memutuskan untuk mengambil tindakan sendiri dengan membuat FUSecure, sebuah file system berbasis FUSE yang menerapkan akses read-only dan membatasi visibilitas file berdasarkan permission user.

## Deskripsi Tugas

Setelah frustrasi dengan kebiasaan plagiat Irwandi yang merugikan prestasi akademiknya, Yuadi memutuskan untuk merancang sistem keamanan yang sophisticated. Dia akan membuat sistem file khusus yang memungkinkan mereka berdua berbagi materi umum, namun tetap menjaga kerahasiaan jawaban praktikum masing-masing.

### a. Setup Direktori dan Pembuatan User

Langkah pertama dalam rencana Yuadi adalah mempersiapkan infrastruktur dasar sistem keamanannya.

1. Buat sebuah "source directory" di sistem Anda (contoh: `/home/shared_files`). Ini akan menjadi tempat penyimpanan utama semua file.

2. Di dalam source directory ini, buat 3 subdirektori: `public`, `private_yuadi`, `private_irwandi`. Buat 2 Linux users: `yuadi` dan `irwandi`. Anda dapat memilih password mereka.
   | User | Private Folder |
   | ------- | --------------- |
   | yuadi | private_yuadi |
   | irwandi | private_irwandi |

Yuadi dengan bijak merancang struktur ini: folder `public` untuk berbagi materi kuliah dan referensi yang boleh diakses siapa saja, sementara setiap orang memiliki folder private untuk menyimpan jawaban praktikum mereka masing-masing.

### b. Akses Mount Point

Selanjutnya, Yuadi ingin memastikan sistem filenya mudah diakses namun tetap terkontrol.

FUSE mount point Anda (contoh: `/mnt/secure_fs`) harus menampilkan konten dari `source directory` secara langsung. Jadi, jika Anda menjalankan `ls /mnt/secure_fs`, Anda akan melihat `public/`, `private_yuadi/`, dan `private_irwandi/`.

### c. Read-Only untuk Semua User

Yuadi sangat kesal dengan kebiasaan Irwandi yang suka mengubah atau bahkan menghapus file jawaban setelah menyalinnya untuk menghilangkan jejak plagiat. Untuk mencegah hal ini, dia memutuskan untuk membuat seluruh sistem menjadi read-only.

1. Jadikan seluruh FUSE mount point **read-only untuk semua user**.
2. Ini berarti tidak ada user (termasuk `root`) yang dapat membuat, memodifikasi, atau menghapus file atau folder apapun di dalam `/mnt/secure_fs`. Command seperti `mkdir`, `rmdir`, `touch`, `rm`, `cp`, `mv` harus gagal semua.

"Sekarang Irwandi tidak bisa lagi menghapus jejak plagiatnya atau mengubah file jawabanku," pikir Yuadi puas.

### d. Akses Public Folder

Meski ingin melindungi jawaban praktikumnya, Yuadi tetap ingin berbagi materi kuliah dan referensi dengan Irwandi dan teman-teman lainnya.

Setiap user (termasuk `yuadi`, `irwandi`, atau lainnya) harus dapat **membaca** konten dari file apapun di dalam folder `public`. Misalnya, `cat /mnt/secure_fs/public/materi_kuliah.txt` harus berfungsi untuk `yuadi` dan `irwandi`.

### e. Akses Private Folder yang Terbatas

Inilah bagian paling penting dari rencana Yuadi - memastikan jawaban praktikum benar-benar terlindungi dari plagiat.

1. File di dalam `private_yuadi` **hanya dapat dibaca oleh user `yuadi`**. Jika `irwandi` mencoba membaca file jawaban praktikum di `private_yuadi`, harus gagal (contoh: permission denied).
2. Demikian pula, file di dalam `private_irwandi` **hanya dapat dibaca oleh user `irwandi`**. Jika `yuadi` mencoba membaca file di `private_irwandi`, harus gagal.

"Akhirnya," senyum Yuadi, "Irwandi tidak bisa lagi menyalin jawaban praktikumku, tapi dia tetap bisa mengakses materi kuliah yang memang kubuat untuk dibagi."

## Contoh Skenario

Setelah sistem selesai, beginilah cara kerja FUSecure dalam kehidupan akademik sehari-hari:

- **User `yuadi` login:**

  - `cat /mnt/secure_fs/public/materi_algoritma.txt` (Berhasil - materi kuliah bisa dibaca)
  - `cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.c` (Berhasil - jawaban praktikumnya sendiri)
  - `cat /mnt/secure_fs/private_irwandi/tugas_sisop.c` (Gagal dengan "Permission denied" - tidak bisa baca jawaban praktikum Irwandi)
  - `touch /mnt/secure_fs/public/new_file.txt` (Gagal dengan "Permission denied" - sistem read-only)

- **User `irwandi` login:**
  - `cat /mnt/secure_fs/public/materi_algoritma.txt` (Berhasil - materi kuliah bisa dibaca)
  - `cat /mnt/secure_fs/private_irwandi/tugas_sisop.c` (Berhasil - jawaban praktikumnya sendiri)
  - `cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.c` (Gagal dengan "Permission denied" - tidak bisa baca jawaban praktikum Yuadi)
  - `mkdir /mnt/secure_fs/new_folder` (Gagal dengan "Permission denied" - sistem read-only)

Dengan sistem ini, kedua mahasiswa akhirnya bisa belajar dengan tenang. Yuadi bisa menyimpan jawaban praktikumnya tanpa khawatir diplagiat Irwandi, sementara Irwandi terpaksa harus mengerjakan tugasnya sendiri namun masih bisa mengakses materi kuliah yang dibagikan Yuadi di folder public.

## Catatan

- Anda dapat memilih nama apapun untuk FUSE mount point Anda.
- Ingat untuk menggunakan argument `-o allow_other` saat mounting FUSE file system Anda agar user lain dapat mengaksesnya.
- Fokus pada implementasi operasi FUSE yang berkaitan dengan **membaca** dan **menolak** operasi write/modify. Anda perlu memeriksa **UID** dari user yang mengakses di dalam operasi FUSE Anda untuk menerapkan pembatasan private folder.

---


### Penyelesaian
Kode lengkap :
```c
#define FUSE_USE_VERSION 28
#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <fuse.h>
#include <fuse/fuse.h>
#include <linux/limits.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

static const char *source_path = "/home/shared_files";

void mkdir_mountpoint(const char *path) {
  struct stat st = {0};
  if (stat(path, &st) == -1) {
    mkdir(path, 0755);
  }
}

static int checkAccessPerm(const char *path, uid_t uid, gid_t gid) {
  char fullPath[PATH_MAX];
  struct stat st;

  snprintf(fullPath, sizeof(fullPath), "%s%s", source_path, path);
  if (lstat(fullPath, &st) == -1)
    return -errno;

  if (strncmp(path, "/private_yuadi/", 14) == 0 ||
      strcmp(path, "/private_yuadi") == 0) {
    if (uid != 1001)
      return -EACCES;
  }

  if (strncmp(path, "/private_irwandi/", 16) == 0 ||
      strcmp(path, "/private_irwandi") == 0) {
    if (uid != 1002)
      return -EACCES;
  }

  return 0;
}

static int xmp_getattr(const char *path, struct stat *stbuf) {
  int res;
  char fpath[PATH_MAX];

  sprintf(fpath, "%s%s", source_path, path);

  res = lstat(fpath, stbuf);

  if (res == -1)
    return -errno;

  return 0;
}

static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi) {
  char fpath[PATH_MAX];

  if (strcmp(path, "/") == 0) {
    path = source_path;
    sprintf(fpath, "%s", path);
  } else {
    sprintf(fpath, "%s%s", source_path, path);
  }

  int res = 0;

  DIR *dp;
  struct dirent *de;
  (void)offset;
  (void)fi;

  dp = opendir(fpath);

  if (dp == NULL)
    return -errno;

  while ((de = readdir(dp)) != NULL) {
    char childPath[PATH_MAX];

    snprintf(childPath, sizeof(childPath), "%s%s", path, de->d_name);

    /* if (checkAccessPerm(childPath, fuse_get_context()->uid, */
    /*                     fuse_get_context()->gid) != 0) */
    /*   continue; */

    struct stat st;

    memset(&st, 0, sizeof(st));

    st.st_ino = de->d_ino;
    st.st_mode = de->d_type << 12;
    res = (filler(buf, de->d_name, &st, 0));

    if (res != 0)
      break;
  }

  closedir(dp);

  return 0;
}

static int xmp_read(const char *path, char *buf, size_t size, off_t offset,
                    struct fuse_file_info *fi) {
  char fpath[PATH_MAX];
  if (strcmp(path, "/") == 0) {
    path = source_path;

    sprintf(fpath, "%s", path);
  } else {
    sprintf(fpath, "%s%s", source_path, path);

    int accessGrant =
        checkAccessPerm(path, fuse_get_context()->uid, fuse_get_context()->gid);
    if (accessGrant != 0)
      return accessGrant;
  }

  int res = 0;
  int fd = 0;

  (void)fi;

  fd = open(fpath, O_RDONLY);

  if (fd == -1)
    return -errno;

  res = pread(fd, buf, size, offset);

  if (res == -1)
    res = -errno;

  close(fd);

  return res;
}

static int xmp_open(const char *path, struct fuse_file_info *fi) {
  char fullPath[PATH_MAX];

  snprintf(fullPath, sizeof(fullPath), "%s%s", source_path, path);

  if ((fi->flags & O_ACCMODE) != O_RDONLY)
    return -EACCES;

  int accessGrant =
      checkAccessPerm(path, fuse_get_context()->uid, fuse_get_context()->gid);
  if (accessGrant != 0)
    return accessGrant;

  int res = open(fullPath, fi->flags);
  if (res == -1)
    return -errno;

  close(res);
  return 0;
}

static int xmp_write(const char *path, const char *buf, size_t size,
                     off_t offset, struct fuse_file_info *fi) {
  return -EACCES;
}

static int xmp_create(const char *path, mode_t mode,
                      struct fuse_file_info *fi) {
  return -EACCES;
}

static int xmp_unlink(const char *path) { return -EACCES; }

static int xmp_mkdir(const char *path, mode_t mode) { return -EACCES; }

static int xmp_rmdir(const char *path) { return -EACCES; }

static int xmp_rename(const char *from, const char *to) { return -EACCES; }

static int xmp_truncate(const char *path, off_t size) { return -EACCES; }

static int xmp_utimens(const char *path, const struct timespec ts[2]) {
  return -EACCES;
}

static int xmp_chmod(const char *path, mode_t mode) { return -EACCES; }

static int xmp_chown(const char *path, uid_t uid, gid_t gid) { return -EACCES; }

static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .read = xmp_read,
    .open = xmp_open,
    .write = xmp_write,
    .create = xmp_create,
    .unlink = xmp_unlink,
    .mkdir = xmp_mkdir,
    .rmdir = xmp_rmdir,
    .rename = xmp_rename,
    .truncate = xmp_truncate,
    .utimens = xmp_utimens,
    .chmod = xmp_chmod,
    .chown = xmp_chown,
};

int main(int argc, char *argv[]) {
  umask(0);

  if (argc < 2) {
    fprintf(stderr, "Usage: %s <mountpoint> [options]\n", argv[0]);
    return 1;
}


  return fuse_main(argc, argv, &xmp_oper, NULL);
}

```

### Penjelasan
```c
#define FUSE_USE_VERSION 28
#include <dirent.h>
#include <errno.h>
#include <fcntl.h>
#include <fuse.h>
#include <fuse/fuse.h>
#include <linux/limits.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
```

Bagian ini adalah daftar header files di awal program C. Semua file header tersebut berisi deklarasi fungsi, tipe data, makro, dan konstanta yang dibutuhkan saat menulis program FUSE.
- `#define FUSE_USE_VERSION 28` ini digunakan untuk menentukan API FUSE versi 2.8 (salah satu versi stabil untuk FUSE).
- `#include <dirent.h>` header untuk bekerja dengan direktori (fungsi `opendir()`, `readdir()`, `closedir()`).
- `#include <errno.h>` header untuk menangani error di C (konstanta error seperti `EACCES`, `ENOENT`, dll.).
- `#include <fcntl.h>` header untuk file control (fungsi `open()`, flag file seperti `O_RDONLY`, `O_WRONLY`, `O_CREAT`, dll.).
- `#include <fuse.h>`, `#include <fuse/fuse.h>` header utama untuk FUSE.
- `#include <linux/limits.h>` header untuk konstanta `PATH_MAX` (panjang path file maksimal).
- `#include <stdio.h>` dan `#include <stdlib.h>` header untuk input/output standar dan utility (`printf()`, `sprintf()`, `malloc()`, `exit()`, dll.).
- `#include <string.h>` header untuk manipulasi string (`strcpy()`, `strcmp()`, `strncmp()`, `strlen()`, `memset()`, dll.).
- `#include <sys/stat.h>` header untuk memanipulasi informasi file (struct `stat`, `lstat()`, dll.).
- `#include <sys/time.h>` header untuk waktu (`struct timeval`, `gettimeofday()`), kadang FUSE butuh timestamp.
- `#include <sys/types.h>` header untuk tipe data seperti `uid_t`, `gid_t`, `off_t`, `mode_t`, dll.
- `#include <unistd.h>` header untuk POSIX API (`read()`, `close()`, `unlink()`, `getuid()`, dll.).

```c
static const char *source_path = "/home/shared_files";
```
Baris ini membuat `"base path"` untuk semua file dan direktori di file system FUSE. Dengan begitu, tidak perlu menulis path lengkap setiap kali, cukup gunakan `source_path` sebagai prefix.

```c
void mkdir_mountpoint(const char *path) {
  struct stat st = {0};
  if (stat(path, &st) == -1) {
    mkdir(path, 0755);
  }
}
```

Fungsi ini memastikan mount point ada. Jika belum ada, maka membuatnya. Jika sudah ada, dibiarkan begitu saja. Berikut adalah penjelasan detailnya :
- Mengecek apakah direktori mount point sudah ada
  - `stat(path, &st)` digunakan untuk mengecek status file atau direktori di `path`.
  - Jika `stat()` mengembalikan `-1`, artinya direktori belum ada.
- Membuat direktori jika belum ada
  - Jika `stat()` gagal (mengembalikan `-1`), maka kode memanggil `mkdir(path, 0755)` untuk membuat direktori baru.
  - `0755` berarti izin untuk direktori.
- Mencegah error saat mount
  - Dengan membuat mount point terlebih dahulu, akan menghindari error "No such file or directory" saat FUSE ingin di-mount ke direktori ini.


```c
static int checkAccessPerm(const char *path, uid_t uid, gid_t gid) {
  char fullPath[PATH_MAX];
  struct stat st;

  snprintf(fullPath, sizeof(fullPath), "%s%s", source_path, path);
  if (lstat(fullPath, &st) == -1)
    return -errno;

  if (strncmp(path, "/private_yuadi/", 14) == 0 ||
      strcmp(path, "/private_yuadi") == 0) {
    if (uid != 1001)
      return -EACCES;
  }

  if (strncmp(path, "/private_irwandi/", 16) == 0 ||
      strcmp(path, "/private_irwandi") == 0) {
    if (uid != 1002)
      return -EACCES;
  }

  return 0;
}
```

Pada bagian ini berfungsi untuk mengecek apakah path bisa diakses oleh user tersebut berdasarkan UID. Fungsi ini digunakan untuk:
- Memastikan bahwa hanya pemilik direktori privat (`private_yuadi` dan `private_irwandi`) yang bisa membaca file di dalamnya.
- Mencegah user lain (UID lain) membaca file privat.


```c
static int xmp_getattr(const char *path, struct stat *stbuf) {
  int res;
  char fpath[PATH_MAX];

  sprintf(fpath, "%s%s", source_path, path);

  res = lstat(fpath, stbuf);

  if (res == -1)
    return -errno;

  return 0;
}
```

`getattr` adalah salah satu callback wajib dalam FUSE yang berfuungsi :
- Memberikan informasi atribut file/direktori kepada FUSE dan sistem operasi.
- Informasi atribut meliputi:
  - Ukuran file
  - Izin file (permission bits)
  - Tipe file (reguler file, direktori, symlink, dll.)
  - Timestamp (waktu modifikasi, waktu akses, waktu perubahan)
```c
static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi) {
  char fpath[PATH_MAX];

  if (strcmp(path, "/") == 0) {
    path = source_path;
    sprintf(fpath, "%s", path);
  } else {
    sprintf(fpath, "%s%s", source_path, path);
  }

  int res = 0;

  DIR *dp;
  struct dirent *de;
  (void)offset;
  (void)fi;

  dp = opendir(fpath);

  if (dp == NULL)
    return -errno;

  while ((de = readdir(dp)) != NULL) {
    char childPath[PATH_MAX];

    snprintf(childPath, sizeof(childPath), "%s%s", path, de->d_name);

    struct stat st;

    memset(&st, 0, sizeof(st));

    st.st_ino = de->d_ino;
    st.st_mode = de->d_type << 12;
    res = (filler(buf, de->d_name, &st, 0));

    if (res != 0)
      break;
  }

  closedir(dp);

  return 0;
}
```
Fungsi ini berfungsi membaca isi sebuah direktori dan mengisi daftar entri (file dan subdirektori) agar bisa ditampilkan di mount point FUSE.
Cara kerjanya yaitu sebagai berikut :
- Menentukan path direktori sumber.
- Membuka direktori asli (`opendir()`), lalu membaca semua entri (`readdir()`).
- Untuk setiap entri, membuat struktur `stat` minimal (inode dan mode), lalu memanggil `filler()` agar FUSE bisa menampilkannya di mount point.
- Menutup direktori saat selesai.
Bagian ini dipanggil FUSE saat user membuka direktori (misal perintah `ls /mount/secure_fs`) agar file dan folder di dalamnya bisa tampil.

```c
static int xmp_read(const char *path, char *buf, size_t size, off_t offset,
                    struct fuse_file_info *fi) {
  char fpath[PATH_MAX];
  if (strcmp(path, "/") == 0) {
    path = source_path;

    sprintf(fpath, "%s", path);
  } else {
    sprintf(fpath, "%s%s", source_path, path);

    int accessGrant =
        checkAccessPerm(path, fuse_get_context()->uid, fuse_get_context()->gid);
    if (accessGrant != 0)
      return accessGrant;
  }

  int res = 0;
  int fd = 0;

  (void)fi;

  fd = open(fpath, O_RDONLY);

  if (fd == -1)
    return -errno;

  res = pread(fd, buf, size, offset);

  if (res == -1)
    res = -errno;

  close(fd);

  return res;
}
```
Bagian fungsi ini berfungsi untuk melayani permintaan pembacaan isi file saat ada proses membaca file dari mount point FUSE.
Cara kerjanya yaitu sebagai berikkut :
- Menentukan path file asli di `source_path`.
- Mengecek izin akses ke file privat (menggunakan `checkAccessPerm()` untuk memastikan hanya pemiliknya bisa membaca).
- Membuka file asli dalam mode hanya-baca (`open()`).
- Membaca data dari file ke dalam buffer (`pread()`), sesuai offset dan jumlah byte yang diminta.
- Menutup file dan mengembalikan jumlah byte yang berhasil dibaca.
Hasilnya nanti user hanya bisa membaca file sesuai izin; file di luar izinnya akan memicu error, dan semua file bersifat read-only.

```c
static int xmp_open(const char *path, struct fuse_file_info *fi) {
  char fullPath[PATH_MAX];

  snprintf(fullPath, sizeof(fullPath), "%s%s", source_path, path);

  if ((fi->flags & O_ACCMODE) != O_RDONLY)
    return -EACCES;

  int accessGrant =
      checkAccessPerm(path, fuse_get_context()->uid, fuse_get_context()->gid);
  if (accessGrant != 0)
    return accessGrant;

  int res = open(fullPath, fi->flags);
  if (res == -1)
    return -errno;

  close(res);
  return 0;
}
```

Fungsi `xmp_open` digunakan untuk menangani permintaan membuka file pada file system FUSE. Fungsi ini memastikan bahwa file hanya bisa dibuka dalam mode read-only (tidak boleh write atau modifikasi). Selain itu, fungsi ini memanggil `checkAccessPerm` untuk memeriksa apakah pengguna saat ini memiliki izin akses ke file tersebut (misalnya hanya pemilik folder private yang boleh membuka file di dalamnya). Jika akses diperbolehkan, file asli di path sumber dibuka dan segera ditutup untuk verifikasi, lalu fungsi mengembalikan sukses (0). Jika ada pelanggaran izin atau kegagalan membuka file, fungsi akan mengembalikan kode error yang sesuai.

```c
static int xmp_write(const char *path, const char *buf, size_t size,
                     off_t offset, struct fuse_file_info *fi) {
  return -EACCES;
}
```
Fungsi `xmp_write` ini bertugas menangani permintaan penulisan (write) ke file di filesystem FUSE. Namun, karena sistem ini dibuat read-only, fungsi ini langsung mengembalikan error `-EACCES` (permission denied), sehingga tidak ada file yang bisa dimodifikasi atau ditulis oleh pengguna. Dengan kata lain, semua operasi tulis seperti menambah, mengubah, atau menghapus file ditolak secara otomatis.

```c
static int xmp_create(const char *path, mode_t mode,
                      struct fuse_file_info *fi) {
  return -EACCES;
}
```

Fungsi `xmp_create` menangani permintaan pembuatan file baru di filesystem FUSE. Namun, karena sistem ini bersifat read-only, fungsi ini langsung mengembalikan error `-EACCES` (akses ditolak), sehingga pengguna tidak diperbolehkan membuat file baru di mount point tersebut.


```c
static int xmp_unlink(const char *path) { return -EACCES; }

static int xmp_mkdir(const char *path, mode_t mode) { return -EACCES; }

static int xmp_rmdir(const char *path) { return -EACCES; }

static int xmp_rename(const char *from, const char *to) { return -EACCES; }

static int xmp_truncate(const char *path, off_t size) { return -EACCES; }

static int xmp_utimens(const char *path, const struct timespec ts[2]) {
  return -EACCES;
}

static int xmp_chmod(const char *path, mode_t mode) { return -EACCES; }

static int xmp_chown(const char *path, uid_t uid, gid_t gid) { return -EACCES; }
```
Bagian fungsi-fungsi ini menangani berbagai operasi modifikasi file dan direktori di filesystem FUSE, seperti menghapus file (`unlink`), membuat direktori (`mkdir`), menghapus direktori (`rmdir`), mengganti nama file/direktori (`rename`), memotong ukuran file (`truncate`), mengubah waktu akses/modifikasi (`utimens`), mengubah izin file (`chmod`), dan mengubah kepemilikan file (`chown`). Namun, karena sistem dibuat read-only, semua fungsi ini langsung mengembalikan error `-EACCES` (akses ditolak), sehingga tidak ada perubahan, penghapusan, atau penambahan file/direktori yang diperbolehkan di mount point tersebut.



```c
static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .read = xmp_read,
    .open = xmp_open,
    .write = xmp_write,
    .create = xmp_create,
    .unlink = xmp_unlink,
    .mkdir = xmp_mkdir,
    .rmdir = xmp_rmdir,
    .rename = xmp_rename,
    .truncate = xmp_truncate,
    .utimens = xmp_utimens,
    .chmod = xmp_chmod,
    .chown = xmp_chown,
};
```
Bagian ini mendefinisikan struktur `fuse_operations` yang berisi daftar fungsi callback yang akan dipanggil oleh FUSE untuk menangani berbagai operasi filesystem, seperti membaca atribut file (`getattr`), membaca isi direktori (`readdir`), membuka dan membaca file (`open`, `read`), serta operasi tulis dan modifikasi yang diatur oleh fungsi-fungsi seperti `write`, `create`, `unlink`, dan lain-lain. Dengan menyusun struktur ini, FUSE tahu fungsi mana yang harus dipanggil saat ada permintaan operasi tertentu pada filesystem yang dimount.

```c
int main(int argc, char *argv[]) {
  umask(0);

  if (argc < 2) {
    fprintf(stderr, "Usage: %s <mountpoint> [options]\n", argv[0]);
    return 1;
  }


  return fuse_main(argc, argv, &xmp_oper, NULL);
}
```

Bagian ini adalah fungsi main yang menjadi titik awal program FUSE.
- `umask(0);` mengatur mode pembuatan file agar tidak dibatasi oleh mask permission default, sehingga permission file diatur persis seperti yang didefinisikan oleh filesystem.
- Pemeriksaan argumen memastikan pengguna memasukkan minimal satu argumen, yaitu mount point tempat filesystem akan dipasang. Jika tidak, program menampilkan pesan penggunaan dan keluar.
- Fungsi `fuse_main` dipanggil untuk memulai filesystem FUSE dengan operasi yang sudah didefinisikan di `xmp_oper`, serta menggunakan argumen yang diterima dari command line.



Singkatnya, ini menjalankan filesystem FUSE dan memasangnya di direktori yang ditentukan oleh pengguna.

### Langkah-langkah menjalankan program


### a. Pertama membuat direktori dahulu


Buat source directory dan subfoldernya
```bash
sudo mkdir -p /home/shared_files


sudo mkdir -p /home/shared_files/public
sudo mkdir -p /home/shared_files/private_yuadi
sudo mkdir -p /home/shared_files/private_irwandi
```

Tambahkan user yuadi dan irwandi
```bash
sudo adduser yuadi
sudo adduser irwandi
```

Set permission agar hanya owner bisa baca file private
```bash
sudo chown yuadi:yuadi /home/shared_files/private_yuadi
sudo chmod 700 /home/shared_files/private_yuadi

sudo chown irwandi:irwandi /home/shared_files/private_irwandi
sudo chmod 700 /home/shared_files/private_irwandi
```


Direktori public bebas di akses

```bash
sudo chmod 755 /home/shared_files/public
```

### b. Akses Mount Point
Buat direktori kosong untuk mount point:
```bash
sudo mkdir -p /mnt/secure_fs
```

mount point ini untuk lokasi fuse nanti. Setelah itu aku membuat fusecure.c di dalam direktori sendiri yaitu `fuse_secure_fs`.
dan jangan lupa untuk menginstall libfuse terlebih dahulu.
```bash
sudo apt-get update
sudo apt-get install libfuse-dev
```

setelah itu kompile fusecure yang telah dibuat menggunakan perintah ini.
```bash
gcc -Wall `pkg-config fuse --cflags` fusecure.c -o fusecure `pkg-config fuse --libs`
```

Setelah itu menjalankan FUSE mount agar `source directory` ter-mount di `/mnt/secure_fs`.
```bash
sudo ./fusecure /mnt/secure_fs -o allow_other
```
`-o allow_other` agar semua user (yuadi dan irwandi) bisa melihat kontennya.

Setelah itu uji mount point menggunakan `ls /mnt/secure_fs`, maka akan menampilkan `public/`, `private_yuadi/`, dan `private_irwandi/`.

### c. Read-Only untuk Semua User

Bagian ini adalah bagian memodifikasi file FUSECURE.C sehingga tidak ada user (termasuk root) yang dapat membuat, memodifikasi, atau menghapus file atau folder apapun di dalam `/mnt/secure_fs`. Command seperti `mkdir`, `rmdir`, `touch`, `rm`, `cp`, `mv` harus gagal semua.

### d. Akses Public Folder
Pastikan permission di source direktori sudah benar
```bash
sudo chmod 755 /home/shared_files/public
```

misal jika ingin membuat file di dalam soure direktori contohnya seperti ini
```bash
echo "Ini materi kuliah algoritma" > /home/shared_files/public/materi_kuliah.txt
```

dan jangan lupa untuk menjalankan
```bash
sudo chmod 777 /home/shared_files/public
```

(777 membuat siapa pun bisa membuat file di dalam public.)
maka otomatis di dalam direktori `public` akan terdapat file `materi_kuliah.txt` yang berisi tulisan `Ini materi kuliah algoritma`

Kemudian cobalah melihat isinya misalnya dengan menggunakan `cat /mnt/secure_fs/public/materi_kuliah.txt`

### e. Akses Private Folder yang Terbatas

Coba login pada masing-masing user yang sudah dibuat
```bash
su - yuadi
su - irwandi
```
Kemudian check apakah sudah berhasil
```bash
cat /mnt/secure_fs/public/materi_kuliah.txt
cat /mnt/secure_fs/private_yuadi/jawaban.c
```

### Jika ingin unmount
```bash
sudo fusermount -u /mnt/secure_fs
```
### Jika ingin membuat file di yaudi atau irwandi harus pindah direktori dahulu
```bash
cd /home/shared_files/private_yuadi
```


### Foto Hasil Output
![image](https://github.com/user-attachments/assets/838c744e-f6c9-48e1-ac4d-e96e951ab8ce)

![image](https://github.com/user-attachments/assets/90f01d01-86d6-4fa9-81da-419026fbc2be)

![image](https://github.com/user-attachments/assets/0540f28f-d600-4d42-94c7-a9496e1faa0f)

![image](https://github.com/user-attachments/assets/7c4ef8de-d4b5-4b73-b515-7bd40f4f9f66)

