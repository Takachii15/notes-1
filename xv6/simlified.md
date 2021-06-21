# H3H3H3H3H3H3H3

## Logging Layer

    Salah satu masalah yang menarik di design file system adalah crash recovery. masalah itu terjadi karena banyak operasi sistem file yang terlibat dalam banyak penulisan di dalam disk dan crash terjadi setelah beberapa penulisan akan menyebabkan sistem file didalam disk dalam keadaan yang tidak konsisten. Sebagai contoh, crash seharusnya terjadi pada saat penyederhanaan file (mengatur panjang file menjadi nol dan membersihkan blok konten). Bergantung pada urutan dari penulisan disk, crash tersebut bisa meninggalkan inode dengan mereferensikan blok konten yang bebas atau meninggalkan blok konten yang dialokasikan tetapi tidak tereferensi.

    Bagian terakhir relatif tidak berbahaya, tetapi inode tersebut mereferensikan blok bebas yang bisa menyebabkan masalah serius setelah reboot. Setelah reboot, kernel mungkin mengalokasikan blok ke file lain, dan kemudian memiliki dua file berbeda yang menunjukkan blok yang sama. Jika xv6 mendukung banyak pengguna, situasi ini bisa menjadi masalah keamanan, dikarenakan pemilik file yang lama bisa merubah blok di file yang baru yang dimiliki oleh pengguna yang berbeda.

    Xv6 mengatasi masalah crash pada saat operasi file system berlangsung dengan bentuk sederhana dari logging. Sebuah system call xv6 tidak langsung merubah struktur data file system on-disk, melainkan menempatkan deskripsi dari semua penulisan disk yang ingin dibuat di dalam **log** pada disk. Ketika system call sudah mengolah semua data yang telah ditulis, itu akan menuliskan catatan commit yang spesial ke dalam disk yang mengindikasikan log tersebut mengandung operasi yang sudah selesai. Disitu juga system call menyalin penulisan data ke struktur data on-disk file system. Setelah penulisan tersebut selesai, system call akan menghapus log yang ada di dalam disk tersebut.

    Jika system crash dan reboot, code file system akan memulihkan dari crash tersebut, sebelum melanjutkan proses yang lainnya. Jika log bertanda sebagaimana mengandung sebuah operasi yang telah selesai, maka recovery code akan menyalin penulisan data ke tempat dimana data tersebut seharusnya berada di dalam on-disk file system. Tetapi jika log tidak bertanda sebagaimana mengandung operasi yang telah selesai, maka recovery code akan mengabaikan log. Setelah recovery code selesai, maka itu akan membersihkan log.

    - Mengapa log dari xv6 menyelesaikan masalah dari crash pada saat operasi file system berjalan?

    > Jika crash terjadi sebelum melakukan suatu operasi, maka log di dalam disk tidak akan ditandai selesai, recovery code juga akan mengabaikan log tersebut, dan kondisi dari disk tersebut juga akan seperti operasi yang belum dimulai (kondisi awal). 
    
    > Jika crash terjadi setelah melakukan suatu operasi, maka recovery code akan menjalankan ulang semua operasi penulisan data, bahkan memungkinkan untuk mengulangi penulisan data jika operasi sudah lebih dulu dimulai untuk menuliskan data tersebut kedalam struktur data on-disk.

    - Dalam kasus lain, log akan membuat operasi kecil yang berhubungan dengan crash. 
    
    Setelah dipulihkan, kemungkinannya adalah diantara semua operasi penulisan data muncul di dalam disk atau tidak ada  sama sekali operasi penulisan data yang muncul.

## Log Design

    Sebuah log diketahui berada di lokasi yang sudah ditetapkan dan dispesifikasikan di dalam superblock yang terdiri dari header block yang diikuti dengan sebuah urutan salinan block yang sudah diupdate ("logged blocks"). Header block memiliki sekumpulan array nomor sektor yang masing-masing terdiri dari logged block. Header block juga memiliki jumlah dari logged block. Xv6 menulis header block ketika proses terjadi dan header block akan tersetel ulang setelah menyalin logged block menuju file system. Jika terjadi crash dipertengahan jalannya proses proses akan mengosongkan log dari header block. Jika crash terjadi setelah proses, maka tidak akan mengosongkan log.

    Setiap code system call mengindikasikan permulaan dan akhir dari urutan setiap penulisan data yang diharuskan atomik (sederhana) dan berhubungan dengan crash. Untuk menjalankan eksekusi secara bersamaan dari operasi file system yang didasarkan pada proses yang berbeda, logging system bisa mengakumulasikan penulisan data dari banyak system call ke dalam satu proses. Setiap masing-masing commit akan melibatkan penulisan data dari beberapa system call yang selesai. Untuk menghindari terpecahnya system call pada saat proses berlangsung, logging system hanya akan menjalankan proses ketika tidak ada file system yang sedang terpanggil.

    Ide dari melakukan berbagai transaksi bersama diketahui sebagai group commit. Group commit menurunkan jumlah dari operasi pada disk karena memecah kebutuhan tetap dari proses selama beberapa operasi berlangsung. Group commit juga dapat menangani system dalam disk lebih banyak secara bersamaan didalam satu waktu dan mungkin juga bisa mengizinkan disk tersebut untuk menulis semua data bersamaan dengan rotasi disk tunggal. Driver IDE dari xv6 tidak mendukung jenis dari bashing yang sedang dilakukan ini, tetapi desain dari file system xv6 nya sendiri mendukung untuk melakukan hal tersebut.

    Xv6 menyediakan sejumlah ukuran tetap didalam disk untuk menahan log. Jumlah dari blok yang dicatat oleh system call dalam suatu proses harus pas dimuat dalam ukuran tetap tersebut. Hal ini menyebabkan dua konsekuensi.
    
    > Tidak ada system call yang diperbolehkan untuk mencatat blok lebih banyak dibandingkan dengan ukuran yang tersedia didalam log. Ini bukanlah masalah untuk sebagian besar system call, tetapi system call write dan unlink berpotensi untuk mencatat banyak blok. Pencatatan file yang besar memungkinkan untuk mencatat banyak blok data dan juga banyak blok bitmap sebagai blok inode; memutuskan koneksi file besar mungkin akan menyebabkan banyak penulisan dari blok bitmap dan juga sebuah inode. xv6 mencatat system call yang memutuskan pencatatan besar menjadi bagian kecil yang muat didalam log dan system call unlink tidak menyebabkan masalah karena dalam prakteknya, file system xv6 hanya menggunakan satu blok bitmap. 

    > Konsekuensi lainnya adalah logging system tidak dapat mengizinkan system call untuk memulai proses kecuali pencacatan data oleh system call-nya bisa muat didalam ukuran yang sudah tersedia didalam log.

## Code: Logging

    Berikut adalah contoh dari penggunaan dari log pada sebuah system call :

    ```
    begin_op();
    ...
    bp = bread(...);
    bp->data[...] = ...;
    log_write(bp);
    ...
    end_op();
    ```

    > begin_op akan menunggu (idle) sampai logging system-nya sudah tidak menjalankan proses dan juga sampai terdapat ruang yang cukup didalam log untuk menyimpan pencatatan data dari system call ini dan juga sekaligus semua system call yang sedang berjalan. log.outstanding menghitung total dari panggilan yang terjadi; kenaikan yang terjadi dapat mencadangkan ruang dan mencegah terjadinya proses lain pada saat system call ini berlangsung. Kode ini mengasumsikan bahwa setiap system call dapat mencatat data sampai MAXOPBLOCKS blok yang berbeda.

    > log_write akan berperan sbeagai proxy untuk bwrite. log_write akan merekam nomor sektor dari setiap blok didalam memori, disajikan dalam sebuah slot didalam log yang ada pada disk, dan ditandai dengan buffer B_DIRTY untuk mencegah blok tersebut diganggu oleh cache block. Blok tersebut harus menetap didalam cache sampai proses selesai. Sampai proses selesai,  salinan dari cache tersebut adalah satu-satunya catatan modifikasi; catatan tersebut tidak dapat dicatat pada tempatnya yang terletak di dalam disk sampai proses selesai dan pembacaan data lain didalam proses yang sama harus melihat modifikasi tersebut. log_write menyadari ketika sebuah blok ditulis berulang kali pada saat proses tunggal dan blok tersebut dialokasikan pada slot yang sama didalam log. Optimasi ini bisa disebut sebagai absorption. Sebagai contoh,  blok disk mengandung inode dari beberapa file yang ditulis beberapa kali didalam sebuah proses. Dengan menyerap beberapa penulisan disk menjadi satu, file system bisa menyimpan ruang log dan bisa mendapatkan performa yang lebih baik karena hanya satu salinan dari blok disk yang harus ditulis didalam disk.

    > end_op mengurangi jumlah dari system system call. Jika jumlah tersebut menjadi 0, maka akan menjalankan proses dengan memanggil fungsi commit(). Didalam proses ini, terdapat 4 tahapan :

        - write_log() menyalin setiap blok yang dimodifikasi didalam proses dari buffer cache menuju ke sebuah slot yang ada di log didalam disk.

        - write_head() mencatat header blok menuju disk. ini adalah poin dari melakukan suatu proses dan crash terjadi setelah proses pencatatan data akan menghasilkan didalam pemulihan yang mengulangi proses penulisan dari log.

        - install_trans membaca setiap blok dari log dan mencatat data menuju ke tempat yang sesuai di dalam file system.

    Dan akhirnya end_op akan mencatat sebuah log header dengan jumlah 0; hal ini harus terjadi sebelum proses selanjutnya mulai mencatat logged block, jadi crash tersebut tidak menghasilkan didalam pemulihan yang menggunakan satu proses header dengan blok log proses berikutnya.

    recover_from_log dipanggil dari initlog yang dimana sudah dipanggil pada saat boot berlangsung sebelum proses user pertama berjalan. recover_from_log membaca log header dan meniru tindakan dari end_op jika header tersebut mengindikasikan kalau log tersebut memiliki proses yang telah dilakukan.

    Sebagai contoh dari penggunaan log yang terjadi didalam filewrite. Proses tersebut adalah sebagai berikut:

    ```
    begin_op();
    ilock(f->ip);
    r = writei(f->ip, ...);
    iunlock(f->ip);
    end_op();
    ```

    Kode ini dibungkus didalam loop yang memecah penulisan data yang besar menjadi proses individual hanya dalam beberapa sektor dalam satu waktu, untuk menghindari meluapnya log. Pemanggilan writei mencatat banyak blok sebagai bagian dari proses ini (file inode, satu atau lebih blok bitmap, dan beberapa blok data).
    
## Code: Block Allocator

    Isi dari file dan direktori disimpan didalam disk block yang dimana harus dialokasikan dari memori yang kosong. Block allocator dari xv6 sendiri mengatur bitmap kosong yang ada didalam disk dengan satu bit per block. Bit kosong mengindikasikan bahwa blok tersebut tersedia; bit bernilai satu menunjukkan kalau blok tersebut sedang terpakai. Program mkfs mengatur bit yang sesuai menuju ke boot sector, superblock, log blocks, inode blocks, dan bitmap blocks.
    
    - Block allocator memiliki dua fungsi :
    
    > balloc => mengalokasikan disk block baru

    > bfree => membersihkan suatu block

    - Balloc

        Pengulangan yang terjadi di balloc memertimbangkan setiap block yang ada, dimulai dari block 0 sampai sb.size, yaitu jumlah block yang ada di file system. Jika ada block yang memiliki bitmap bit nya bernilai 0, maka itu mengindikasikan bahwa itu kosong (free). Jika balloc menemukan sejumlah block, balloc akan mengupdate bitmap dan akan mengembalikan block tersebut. Supaya lebih efektif, pengulangannya dibagi menjadi dua bagian:

        > Pengulangan yang terjadi diluar balloc akan membaca setiap block dari setiap bit yang ada didalam bitmap.

        > Pengulangan yang terjadi didalam balloc akan memeriksa semua bits BPB didalam satu block dalam bitmap.

        Hal yang mungkin terjadi jika dua proses mencoba untuk mengalokasikan block di waktu yang sama akan dicegah dengan fakta bahwa buffer cache hanya menjalankan satu proses yang menggunakan satu block bitmap apapun dalam satu waktu.
    
    - Bfree

        Bfree menemukan block bitmap yang tepat dan membersihkan bit yang sesuai. Penggunaan dari bread dan belse menghindari penguncian yang eksplisit. 
        
    Untuk menjalankan fungsinya, balloc dan bfree harus dipanggil didalam sebuah proses.