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

