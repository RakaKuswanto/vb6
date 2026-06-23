# Dokumentasi Patching: Mengatasi "Not Responding" pada Instalasi Visual Basic 6.0 (acmsetup.exe)

## Latar Belakang Masalah
Saat melakukan instalasi Visual Studio 6.0 atau Visual Basic 6.0 di sistem operasi modern (Windows 10/11), proses instalasi sering kali terhenti secara tiba-tiba dengan status **"Not Responding"**. 

Masalah ini bukan disebabkan oleh kegagalan kompilator atau core IDE, melainkan karena `acmsetup.exe` (Microsoft Acme Setup Engine) mencoba mengeksekusi komponen prerequisite lawas seperti:
* **Microsoft Java Virtual Machine (MSJavx86.exe)**
* **Microsoft Data Access Components (MDAC_TYP.EXE)**
* **DCOM98 / IE4 Extensions**

Karena sistem keamanan dan kompatibilitas OS modern (seperti UAC atau mitigasi perlindungan file) memblokir installer usang ini secara diam-diam (silent block), proses anak tidak pernah mengembalikan status exit. Akibatnya, thread utama `acmsetup.exe` mengalami deadlock saat menunggunya (via `WaitForSingleObject`), menghentikan antrean pesan UI, dan menyebabkan hang.

---

## Analisis Teknis (Root Cause)
Melalui reverse engineering statis menggunakan **IDA Pro**, dalang dari masalah ini ditemukan di dalam sub-routine eksekusi utama: `sub_4039A0`.

Di dalam routine tersebut, setup engine membaca antrean instalasi dan memanggil Virtual Function (VTable) secara dinamis menggunakan calling convention `__stdcall` atau `__thiscall`:

    .text:00403CB7    push    1                    ; Argumen 3
    .text:00403CB9    mov     eax, [ecx]
    .text:00403CBB    push    ebx                  ; Argumen 2
    .text:00403CBC    push    esi                  ; Argumen 1
    .text:00403CBD    call    dword ptr [eax+48h]  <-- TITIK DEADLOCK
    .text:00403CC0    mov     ebx, eax             ; Return code disimpan di EBX

Fungsi di offset `+48h` (atau `+72` dalam desimal) adalah pemicu eksekusi proses eksternal. Jika proses eksternal hang, baris `call` ini tidak akan pernah selesai dieksekusi. 

Program mengharapkan nilai return sukses berupa angka `3` (atau `1`) yang nantinya disimpan di register `EBX` untuk dievaluasi oleh state machine instalasi.

---

## Solusi Patching (Bypass Komponen Eksternal)
Solusi paling efektif dan permanen adalah melakukan binary patching untuk memotong pemanggilan fungsi tersebut dan secara manual menyuntikkan return value yang menyatakan "Sukses".

### Kriteria Patch yang Aman:
1. **Bypass eksekusi:** Hilangkan instruksi `call`.
2. **Inject Success Code:** Masukkan nilai `3` langsung ke register `EBX`.
3. **Stack Alignment:** Hilangkan instruksi `push` agar nilai Stack Pointer (`ESP`) tidak bergeser, mencegah Access Violation / Crash pada instruksi di bawahnya.
4. **Memory Alignment:** Ganti semua sisa byte (hantu dari instruksi lama) dengan `NOP` (No Operation / `0x90`).

### Perbandingan Assembly (Sebelum vs Sesudah)

**Sebelum Patching (Original):**
    .text:00403CB7    push    1
    .text:00403CB9    mov     eax, [ecx]
    .text:00403CBB    push    ebx
    .text:00403CBC    push    esi
    .text:00403CBD    call    dword ptr [eax+48h]
    .text:00403CC0    mov     ebx, eax
    .text:00403CC2    lea     ecx, [esp+34Ch+var_310]

**Sesudah Patching (Patched):**
    .text:00403CB7    mov     ebx, 3               ; Inject success code
    .text:00403CBC    nop                          ; Alignment padding
    .text:00403CBD    nop
    .text:00403CBE    nop
    .text:00403CBF    nop
    .text:00403CC0    nop
    .text:00403CC1    nop
    .text:00403CC2    lea     ecx, [esp+34Ch+var_310] ; Resume eksekusi normal

---

## Hasil & Implikasi
Dengan menerapkan patch di atas dan menyimpan ulang executable (`Edit -> Patch program -> Apply patches to input file`), hasil yang didapatkan adalah:

1. **Bypass Instan:** Instalasi MS Java, MDAC, dan komponen lawas lainnya akan dilompati seolah-olah berhasil diinstal.
2. **Bebas Hang:** Proses instalasi tidak akan lagi tersangkut di pertengahan jalan.
3. **Core IDE Aman:** Visual Basic 6.0 IDE (`VB6.exe`), kompiler C++, dan seluruh runtime intinya tetap disalin dan diregistrasi ke sistem operasi tanpa masalah.

Metode ini jauh lebih bersih dan teknis dibandingkan membuat dummy file MSJAVA.DLL, karena langsung mematikan logic pemicunya di level engine.