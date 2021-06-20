# MATERI RANGKUMAN PRO

## Logging Layer

    Salah satu masalah yang menarik di design file system adalah crash recovery. masalah itu terjadi karena banyak operasi sistem file yang terlibat dalam banyak penulisan di dalam disk dan crash terjadi setelah beberapa penulisan akan menyebabkan sistem file didalam disk dalam keadaan yang tidak konsisten. Sebagai contoh, crash seharusnya terjadi pada saat penyederhanaan file (mengatur panjang file menjadi nol dan membersihkan blok konten). Bergantung pada urutan dari penulisan disk, crash tersebut bisa meninggalkan inode dengan mereferensikan blok konten yang bebas atau meninggalkan blok konten yang dialokasikan tetapi tidak tereferensi.

    Bagian terakhir relatif tidak berbahaya, tetapi inode tersebut mereferensikan blok bebas yang bisa menyebabkan masalah serius setelah reboot. Setelah reboot, kernel mungkin mengalokasikan blok ke file lain, dan kemudian memiliki dua file berbeda yang menunjukkan blok yang sama. Jika xv6 mendukung banyak pengguna, situasi ini bisa menjadi masalah keamanan, dikarenakan pemilik file yang lama bisa merubah blok di file yang baru yang dimiliki oleh pengguna yang berbeda.

    Xv6 mengatasi masalah crash pada saat operasi file system berlangsung dengan bentuk sederhana dari logging. Sebuah system call xv6 tidak langsung merubah struktur data file system on-disk, melainkan menempatkan deskripsi dari semua penulisan disk yang ingin dibuat di dalam **log** pada disk. Ketika system call sudah mengolah semua data yang telah ditulis, itu akan menuliskan catatan commit yang spesial ke dalam disk yang mengindikasikan log tersebut mengandung operasi yang sudah selesai. Disitu juga system call menyalin penulisan data ke struktur data on-disk file system. Setelah penulisan tersebut selesai, system call akan menghapus log yang ada di dalam disk tersebut.

    Jika system crash dan reboot, code file system akan memulihkan dari crash tersebut, sebelum melanjutkan proses yang lainnya. Jika log bertanda sebagaimana mengandung sebuah operasi yang telah selesai, maka recovery code akan menyalin penulisan data ke tempat dimana data tersebut seharusnya berada di dalam on-disk file system. Tetapi jika log tidak bertanda sebagaimana mengandung operasi yang telah selesai, maka recovery code akan mengabaikan log. Setelah recovery code selesai, maka itu akan membersihkan log.

    - Mengapa log dari xv6 menyelesaikan masalah dari crash pada saat operasi file system berjalan?

    > Jika crash terjadi sebelum melakukan suatu operasi, maka log di dalam disk tidak akan ditandai selesai, recovery code juga akan mengabaikan log tersebut, dan kondisi dari disk tersebut juga akan seperti operasi yang belum dimulai (kondisi awal). 
    
    > Jika crash terjadi setelah melakukan suatu operasi, maka recovery code akan menjalankan ulang semua operasi penulisan data, bahkan memungkinkan untuk mengulangi penulisan data jika operasi sudah lebih dulu dimulai untuk menuliskan data tersebut kedalam struktur data on-disk.

    - Dalam kasus lain, log akan membuat operasi kecil yang sehubungan dengan crash. 
    
    Setelah dipulihkan, kemungkinannya adalah diantara semua operasi penulisan data muncul di dalam disk atau tidak ada  sama sekali operasi penulisan data yang muncul.

## Log Design

    