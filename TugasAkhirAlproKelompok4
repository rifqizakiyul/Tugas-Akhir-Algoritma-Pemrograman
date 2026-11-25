/*
 * Program "Gas Cumlaude"
 * Membangun Konsistensi Belajar Mahasiswa dengan Target Belajar,
 * Tracking Otomatis, dan Streak
 *
 * Tugas Akhir Algoritma dan Pemrograman Komputer - Kelompok 4
 * Target platform : Windows (konsol). Kompatibel C++11 ke atas.
 * Kompilasi       : g++ -std=c++11 gas_cumlaude.cpp -o gas_cumlaude.exe
 *
 * File data (dibuat otomatis di folder yang sama dengan program):
 *   riwayat.txt : satu baris = satu catatan belajar (hanya ditambah, tidak pernah
 *                 ditulis ulang, sehingga baris rusak tidak ikut hilang)
 *   target.txt  : target belajar harian (dalam detik)
 */

#ifdef _WIN32
  #define NOMINMAX
  #include <windows.h>
  #include <conio.h>
#else
  // Hanya agar program bisa diuji di non-Windows (tidak dipakai di Windows)
  #include <termios.h>
  #include <unistd.h>
  #include <sys/select.h>
#endif

#include <iostream>
#include <fstream>
#include <string>
#include <vector>
#include <ctime>
#include <chrono>
#include <iomanip>
#include <cstdlib>

using namespace std;
using namespace std::chrono;

// ======================= KONSTANTA, ENUM, STRUCT =======================

const int MAKS_CATATAN = 2000;
const int MAKS_CEK_STREAK = 3650;          // batas pengecekan streak (10 tahun)
const char* FILE_RIWAYAT = "riwayat.txt";
const char* FILE_TARGET  = "target.txt";

// Untuk pengujian cepat: kompilasi dengan -DMODE_UJI agar 1 "menit" = 1 detik.
#ifdef MODE_UJI
const int DETIK_PER_MENIT = 1;
#else
const int DETIK_PER_MENIT = 60;
#endif

enum Matkul { M_ALGO, M_K3, M_KEAMANAN, M_INGGRIS, M_PERADABAN, M_LAINNYA, JUMLAH_MATKUL };
const char* namaMatkul[JUMLAH_MATKUL] = {
    "Algoritma dan Pemrograman", "K3", "Keamanan Informasi",
    "Bahasa Inggris Profesi", "Peradaban Islam", "Lainnya"
};

enum Mode { MODE_POMODORO, MODE_OPENTIME };
const char* namaMode[2] = { "Pomodoro", "Open Time" };

struct Catatan {
    time_t waktuMulai;                       // dasar filter z hari
    string tanggal, jamMulai, jamSelesai;
    string mode, matkul, materi;
    int durasiDetik;                         // dihitung otomatis
};

Catatan dataBelajar[MAKS_CATATAN];                  // seluruh riwayat belajar
int jumlahData = 0;
int targetHarianDetik = 0;                   // 0 = target belum diatur

Catatan hasilRekap[MAKS_CATATAN];            // penampung catatan hasil filter

// Dilempar saat input ditutup (Ctrl+Z di Windows / Ctrl+D di Linux).
// Dipakai agar sesi yang sedang berjalan tetap sempat dicatat sebelum keluar.
struct InputHabis {};
bool inputHabis = false;

// ======================= TOMBOL & SUARA (khusus Windows) =======================

// Jeda singkat tanpa <thread> (kompatibel dengan MinGW versi lama / Dev-C++)
void tidurMs(int ms) {
#ifdef _WIN32
    Sleep(ms);
#else
    usleep(ms * 1000);
#endif
}

#ifdef _WIN32
struct ModeTombol { };                       // tidak perlu pengaturan khusus
bool tombolDitekan() { return _kbhit() != 0; }
char ambilTombol()   { return (char)_getch(); }
void kosongkanTombol() { while (_kbhit()) _getch(); }
void bunyi() {
    for (int i = 0; i < 3; i++) { Beep(1000, 250); Sleep(100); }
}
#else
struct ModeTombol {                          // mode baca tombol tanpa Enter (uji)
    termios lama; bool aktif;
    ModeTombol() : aktif(false) {
        if (isatty(0) && tcgetattr(0, &lama) == 0) {
            termios baru = lama;
            baru.c_lflag &= ~(ICANON | ECHO);
            tcsetattr(0, TCSANOW, &baru);
            aktif = true;
        }
    }
    ~ModeTombol() { if (aktif) tcsetattr(0, TCSANOW, &lama); }
};
bool tombolDitekan() {
    fd_set fds; FD_ZERO(&fds); FD_SET(0, &fds);
    timeval tv; tv.tv_sec = 0; tv.tv_usec = 0;
    return select(1, &fds, NULL, NULL, &tv) > 0;
}
char ambilTombol() { char c = 0; if (read(0, &c, 1) <= 0) c = 0; return c; }
void kosongkanTombol() { while (tombolDitekan()) { if (ambilTombol() == 0) break; } }
void bunyi() { cout << "\a\a\a" << flush; }
#endif

// ======================= UTILITAS WAKTU & TEKS =======================

string formatTanggal(time_t t) {             // DD-MM-YYYY
    tm* w = localtime(&t);
    char buf[16];
    strftime(buf, sizeof(buf), "%d-%m-%Y", w);
    return string(buf);
}

string formatJam(time_t t) {                 // HH:MM
    tm* w = localtime(&t);
    char buf[8];
    strftime(buf, sizeof(buf), "%H:%M", w);
    return string(buf);
}

string formatDurasi(int detik) {             // HH:MM:SS
    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d", detik / 3600, (detik % 3600) / 60, detik % 60);
    return string(buf);
}

string formatMenitDetik(int detik) {         // MM:SS
    char buf[16];
    snprintf(buf, sizeof(buf), "%02d:%02d", detik / 60, detik % 60);
    return string(buf);
}

string formatJamMenit(int detik) {           // "2 jam 30 menit"
    return to_string(detik / 3600) + " jam " + to_string((detik % 3600) / 60) + " menit";
}

time_t awalHari(time_t t) {                  // pukul 00:00 pada hari yang sama
    tm w = *localtime(&t);
    w.tm_hour = 0; w.tm_min = 0; w.tm_sec = 0; w.tm_isdst = -1;
    return mktime(&w);
}

string tanggalMundur(int k) {                // tanggal k hari sebelum hari ini
    time_t siang = awalHari(time(0)) + 12 * 3600 - (time_t)k * 86400;
    return formatTanggal(siang);
}

string rapikan(string s) {                   // buang spasi di awal dan akhir
    size_t a = s.find_first_not_of(" \t\r\n");
    if (a == string::npos) return "";
    size_t b = s.find_last_not_of(" \t\r\n");
    return s.substr(a, b - a + 1);
}

vector<string> pecah(const string& s, char pemisah) {
    vector<string> hasil;
    string bagian;
    for (size_t i = 0; i < s.size(); i++) {
        if (s[i] == pemisah) { hasil.push_back(bagian); bagian = ""; }
        else bagian += s[i];
    }
    hasil.push_back(bagian);
    return hasil;
}

// ======================= PENYIMPANAN DATA (FILE) =======================

// Menambahkan SATU baris di akhir riwayat.txt. File tidak pernah ditulis ulang,
// jadi baris lama (termasuk yang rusak) tetap utuh.
bool simpanCatatan(const Catatan& c) {
    ofstream f(FILE_RIWAYAT, ios::app);
    if (!f) return false;
    f << (long long)c.waktuMulai << ';' << c.tanggal << ';'
      << c.jamMulai << ';' << c.jamSelesai << ';'
      << c.mode << ';' << c.matkul << ';'
      << c.materi << ';' << c.durasiDetik << '\n';
    f.flush();
    return f.good();
}

void simpanTarget() {
    ofstream f(FILE_TARGET);
    if (!f) { cout << "[!] Gagal menyimpan " << FILE_TARGET << endl; return; }
    f << targetHarianDetik << '\n';
}

void muatData() {
    jumlahData = 0;
    ifstream f(FILE_RIWAYAT);
    if (!f) return;                          // file belum ada: mulai dari kosong
    string baris;
    int rusak = 0, terlewat = 0;
    while (getline(f, baris)) {
        baris = rapikan(baris);
        if (baris.empty()) continue;
        vector<string> p = pecah(baris, ';');
        if (p.size() != 8) { rusak++; continue; }
        try {
            Catatan c;
            c.waktuMulai = (time_t)stoll(p[0]);
            c.tanggal = p[1]; c.jamMulai = p[2]; c.jamSelesai = p[3];
            c.mode = p[4]; c.matkul = p[5]; c.materi = p[6];
            c.durasiDetik = stoi(p[7]);
            if (c.waktuMulai <= 0 || c.durasiDetik <= 0 || c.tanggal.empty()) { rusak++; continue; }
            if (jumlahData >= MAKS_CATATAN) { terlewat++; continue; }
            dataBelajar[jumlahData++] = c;
        } catch (...) { rusak++; }
    }
    if (rusak > 0)
        cout << "[!] " << rusak << " baris di " << FILE_RIWAYAT << " rusak dan dilewati. "
             << "Baris itu tetap utuh di file (tidak dihapus), silakan perbaiki manual." << endl;
    if (terlewat > 0)
        cout << "[!] " << terlewat << " catatan melebihi kapasitas memori (" << MAKS_CATATAN
             << ") dan tidak ikut dihitung di rekap." << endl;
}

void muatTarget() {
    targetHarianDetik = 0;
    ifstream f(FILE_TARGET);
    if (!f) return;
    string baris;
    if (getline(f, baris)) {
        try { targetHarianDetik = stoi(rapikan(baris)); } catch (...) { targetHarianDetik = 0; }
    }
    if (targetHarianDetik < 0) targetHarianDetik = 0;
}

// ======================= INPUT AMAN =======================

string bacaBaris(const string& prompt) {
    cout << prompt << flush;
    string s;
    if (inputHabis || !getline(cin, s)) {    // input ditutup (misal Ctrl+Z)
        inputHabis = true;
        throw InputHabis();
    }
    return rapikan(s);
}

int bacaInt(const string& prompt, int bawah, int atas) {
    while (true) {
        string s = bacaBaris(prompt);
        char* akhir = NULL;
        long v = strtol(s.c_str(), &akhir, 10);
        if (!s.empty() && *akhir == '\0' && v >= bawah && v <= atas) return (int)v;
        cout << "Input tidak valid. Masukkan angka " << bawah << " sampai " << atas << "." << endl;
    }
}

double bacaJam(const string& prompt) {
    while (true) {
        string s = bacaBaris(prompt);
        for (size_t i = 0; i < s.size(); i++) if (s[i] == ',') s[i] = '.';
        char* akhir = NULL;
        double v = strtod(s.c_str(), &akhir);
        if (!s.empty() && *akhir == '\0' && v > 0) return v;
        cout << "Input tidak valid. Masukkan angka lebih dari 0 (contoh: 2 atau 1.5)." << endl;
    }
}

// ======================= TARGET, TOTAL HARI, STREAK =======================

int totalPadaTanggal(const string& tanggal) {
    int total = 0;
    for (int i = 0; i < jumlahData; i++)
        if (dataBelajar[i].tanggal == tanggal) total += dataBelajar[i].durasiDetik;
    return total;
}

bool hariMemenuhi(int k) {                   // k = 0 hari ini, 1 kemarin, dst.
    return targetHarianDetik > 0 && totalPadaTanggal(tanggalMundur(k)) >= targetHarianDetik;
}

// Streak dihitung mundur dari hari ini. Jika hari ini belum memenuhi target,
// streak tetap dihitung dari kemarin (belum dianggap putus sampai hari berganti).
int hitungStreak() {
    int streak = 0;
    int k = hariMemenuhi(0) ? 0 : 1;
    while (k < MAKS_CEK_STREAK && hariMemenuhi(k)) { streak++; k++; }
    return streak;
}

void setTarget() {
    cout << "\n=== SET TARGET BELAJAR ===" << endl;
    cout << "1. Target per hari" << endl;
    cout << "2. Target per minggu" << endl;
    cout << "3. Target per bulan" << endl;
    int jenis = bacaInt("Pilih jenis target (1-3): ", 1, 3);
    double jam = bacaJam("Jumlah jam belajar (contoh 2 atau 1.5): ");

    double jamHarian = jam;                  // 1 minggu = 7 hari, 1 bulan = 30 hari
    if (jenis == 2) jamHarian = jam / 7.0;
    else if (jenis == 3) jamHarian = jam / 30.0;

    if (jamHarian > 24.0) {
        cout << "Target melebihi 24 jam per hari. Silakan atur ulang." << endl;
        return;
    }
    if (jamHarian > 12.0) {
        cout << "Peringatan: target lebih dari 12 jam per hari, kemungkinan sulit dicapai." << endl;
        string y = bacaBaris("Tetap gunakan target ini? (y/n): ");
        if (y != "y" && y != "Y") { cout << "Target tidak diubah." << endl; return; }
    }
    int detik = (int)(jamHarian * 3600.0 + 0.5);
    if (detik < 60) {
        cout << "Target minimal 1 menit per hari. Silakan atur ulang." << endl;
        return;
    }
    targetHarianDetik = detik;
    simpanTarget();
    cout << "\nTarget tersimpan:" << endl;
    cout << "  Per hari   : " << formatJamMenit(targetHarianDetik) << endl;
    cout << "  Per minggu : " << formatJamMenit(targetHarianDetik * 7) << endl;
    cout << "  Per bulan  : " << formatJamMenit(targetHarianDetik * 30) << endl;
}

// ======================= TAMPILAN MENU =======================

void tampilMenu() {
    int hariIni = totalPadaTanggal(tanggalMundur(0));
    int persen = targetHarianDetik > 0 ? (int)((long long)hariIni * 100 / targetHarianDetik) : 0;
    int streak = hitungStreak();

    cout << "\n==============================================" << endl;
    cout << "              GAS CUMLAUDE" << endl;
    cout << "==============================================" << endl;
    cout << " Target harian : " << formatJamMenit(targetHarianDetik) << endl;
    cout << " Hari ini      : " << formatDurasi(hariIni) << " (" << persen << "%)" << endl;
    if (streak > 0)
        cout << " Streak        : " << streak << " hari berturut-turut memenuhi target" << endl;
    else
        cout << " Streak        : 0 hari (penuhi target hari ini untuk memulai streak)" << endl;
    cout << "----------------------------------------------" << endl;
    cout << " 1. Set Target Belajar" << endl;
    cout << " 2. Gas Belajar (mode Pomodoro)" << endl;
    cout << " 3. Gas Belajar (mode Open Time)" << endl;
    cout << " 4. Cek Rekap Belajar" << endl;
    cout << " 0. Keluar" << endl;
    cout << "==============================================" << endl;
}

int pilihMatkul() {
    cout << "\nPilih mata kuliah:" << endl;
    for (int i = 0; i < JUMLAH_MATKUL; i++)
        cout << " " << (i + 1) << ". " << namaMatkul[i] << endl;
    return bacaInt("Pilihan: ", 1, JUMLAH_MATKUL) - 1;
}

bool konfirmasiFokus() {                     // true = Lanjutkan, false = Kembali
    cout << "\n==============================================" << endl;
    cout << " SEBELUM MEMULAI SESI FOKUS" << endl;
    cout << " Pastikan notifikasi handphone Anda dalam mode" << endl;
    cout << " \"Jangan Ganggu\" dan singkirkan hal-hal yang" << endl;
    cout << " dapat mengganggu selama sesi fokus." << endl;
    cout << "==============================================" << endl;
    cout << " 1. Lanjutkan" << endl;
    cout << " 2. Kembali" << endl;
    return bacaInt("Pilihan: ", 1, 2) == 1;
}

// ======================= TIMER =======================

// Hitung mundur 'detik' detik. Tekan 'q' untuk berhenti lebih awal.
// Mengembalikan detik yang benar-benar berjalan; *berhenti = true jika dihentikan.
int countdown(int detik, const string& label, bool* berhenti) {
    *berhenti = false;
    int lewat = 0;
    {
        ModeTombol mode;
        steady_clock::time_point mulai = steady_clock::now();
        int tampilTerakhir = -1;
        while (true) {
            long long ms = duration_cast<milliseconds>(steady_clock::now() - mulai).count();
            lewat = (int)(ms / 1000);
            if (lewat > detik) lewat = detik;
            int sisa = detik - lewat;
            if (lewat != tampilTerakhir) {
                cout << "\r" << label << "  " << formatMenitDetik(sisa)
                     << "   (tekan q untuk berhenti)      " << flush;
                tampilTerakhir = lewat;
            }
            if (sisa <= 0) break;
            if (tombolDitekan()) {
                char c = ambilTombol();
                if (c == 'q' || c == 'Q') { *berhenti = true; break; }
            }
            tidurMs(100);
        }
    }
    cout << endl;
    kosongkanTombol();
    return lewat;
}

// ======================= PENCATATAN & REKAP =======================

void catatSesi(time_t mulai, time_t selesai, Mode mode, int matkul, int durasiDetik) {
    if (durasiDetik <= 0) {
        cout << "\nDurasi belajar 0 detik, sesi tidak dicatat." << endl;
        return;
    }
    string materi = "-";
    try {
        materi = bacaBaris("\nApa yang kamu pelajari? (Enter untuk lewati): ");
    } catch (const InputHabis&) {
        cout << "\n[!] Input ditutup, materi diisi \"-\" agar sesi ini tetap tercatat." << endl;
    }
    if (materi.empty()) materi = "-";
    for (size_t i = 0; i < materi.size(); i++) if (materi[i] == ';') materi[i] = ',';

    Catatan c;
    c.waktuMulai = mulai;
    c.tanggal = formatTanggal(mulai);
    c.jamMulai = formatJam(mulai);
    c.jamSelesai = formatJam(selesai);
    c.mode = namaMode[mode];
    c.matkul = namaMatkul[matkul];
    c.materi = materi;
    c.durasiDetik = durasiDetik;
    if (simpanCatatan(c)) {
        if (jumlahData < MAKS_CATATAN) dataBelajar[jumlahData++] = c;
        else cout << "[!] Memori rekap penuh (" << MAKS_CATATAN << "). Catatan tetap masuk file, tapi belum ikut rekap." << endl;
    } else {
        cout << "[!] GAGAL menyimpan ke " << FILE_RIWAYAT << ". Cek izin tulis folder program. Sesi ini belum tercatat." << endl;
        return;
    }

    int hariIni = totalPadaTanggal(c.tanggal);
    cout << "\n--- Sesi tercatat ---" << endl;
    cout << "Mata kuliah : " << c.matkul << endl;
    cout << "Waktu       : " << c.tanggal << ", " << c.jamMulai << " - " << c.jamSelesai << endl;
    cout << "Durasi      : " << formatDurasi(durasiDetik) << endl;
    cout << "Materi      : " << c.materi << endl;
    cout << "Total hari ini: " << formatDurasi(hariIni) << " / " << formatDurasi(targetHarianDetik) << endl;
    if (hariIni >= targetHarianDetik) {
        cout << "Target hari ini TERCAPAI! Streak: " << hitungStreak() << " hari." << endl;
    } else {
        cout << "Kurang " << formatDurasi(targetHarianDetik - hariIni) << " lagi untuk mencapai target hari ini." << endl;
    }
}

// Dipanggil saat pengguna menekan q di tengah sesi.
// Mengembalikan 1 = akhiri dan simpan, 2 = batalkan semua (tidak dicatat), 3 = lanjutkan.
int tanyaBerhenti(int totalFokusDetik) {
    cout << "\nSesi dihentikan sementara. Total waktu fokus sejak mulai: "
         << formatDurasi(totalFokusDetik) << endl;
    cout << " 1. Akhiri sekarang dan simpan waktu yang sudah berjalan" << endl;
    cout << " 2. Batalkan semua (tidak ada yang dicatat)" << endl;
    cout << " 3. Lanjutkan sesi" << endl;
    try {
        return bacaInt("Pilihan: ", 1, 3);
    } catch (const InputHabis&) {
        return 1;                            // input ditutup: simpan yang sudah berjalan
    }
}

void mulaiPomodoro() {
    cout << "\n=== MODE POMODORO ===" << endl;
    int matkul = pilihMatkul();
    int jumlahSesi = bacaInt("Jumlah sesi pomodoro (1-12): ", 1, 12);
    int menitFokus = bacaInt("Durasi fokus per sesi, menit (1-180): ", 1, 180);
    int menitIstirahat = bacaInt("Durasi istirahat, menit (1-60): ", 1, 60);

    if (!konfirmasiFokus()) {                // dipilih "Kembali"
        cout << "Kembali ke menu utama." << endl;
        return;
    }

    time_t mulai = time(0);
    int totalFokus = 0;
    bool batalSemua = false, akhiriLebihAwal = false;
    for (int i = 1; i <= jumlahSesi && !akhiriLebihAwal; i++) {
        string label = "Sesi " + to_string(i) + "/" + to_string(jumlahSesi) + " - FOKUS    ";
        int sisa = menitFokus * DETIK_PER_MENIT;
        while (true) {                       // berulang jika pengguna memilih "Lanjutkan"
            bool berhenti = false;
            int dur = countdown(sisa, label, &berhenti);
            totalFokus += dur;
            sisa -= dur;
            if (!berhenti) break;            // waktu habis normal
            int aksi = tanyaBerhenti(totalFokus);
            if (aksi == 3) continue;         // lanjutkan sisa waktu sesi ini
            if (aksi == 2) batalSemua = true;
            akhiriLebihAwal = true;
            break;
        }
        if (akhiriLebihAwal) break;

        bunyi();
        cout << "Sesi fokus " << i << " selesai!" << endl;
        if (i < jumlahSesi) {
            cout << " 1. Istirahat" << endl;
            cout << " 2. Lanjut ke sesi fokus berikutnya" << endl;
            int p = 2;
            try {
                p = bacaInt("Pilihan: ", 1, 2);
            } catch (const InputHabis&) {    // input ditutup: akhiri dan simpan yang sudah berjalan
                akhiriLebihAwal = true;
                break;
            }
            if (p == 1) {
                bool lewati = false;
                countdown(menitIstirahat * DETIK_PER_MENIT, "ISTIRAHAT              ", &lewati);
                bunyi();
                cout << "Istirahat selesai. Masuk sesi fokus berikutnya." << endl;
            }
        }
    }
    if (batalSemua) {
        cout << "Sesi dibatalkan. Tidak ada waktu yang dicatat." << endl;
        return;
    }
    catatSesi(mulai, time(0), MODE_POMODORO, matkul, totalFokus);
}

void mulaiOpenTime() {
    cout << "\n=== MODE OPEN TIME ===" << endl;
    int matkul = pilihMatkul();
    if (!konfirmasiFokus()) {
        cout << "Kembali ke menu utama." << endl;
        return;
    }
    cout << "\nTombol: [p] jeda / lanjutkan   [s] selesai dan simpan   [x] batalkan\n" << endl;

    time_t mulai = time(0);
    long long aktifMs = 0;                   // hanya menghitung waktu saat tidak dijeda
    bool jeda = false;
    bool batal = false;
    while (true) {                           // berulang jika pembatalan tidak dikonfirmasi
        bool mintaBatal = false;
        {
        ModeTombol mode;
        steady_clock::time_point terakhir = steady_clock::now();
        int detikTampil = -1;
        bool jedaTampil = false;
        while (true) {
            steady_clock::time_point sekarang = steady_clock::now();
            if (!jeda) aktifMs += duration_cast<milliseconds>(sekarang - terakhir).count();
            terakhir = sekarang;

            int detik = (int)(aktifMs / 1000);
            if (detik != detikTampil || jeda != jedaTampil) {
                cout << "\r" << (jeda ? "[DIJEDA]    " : "[BERJALAN] ")
                     << formatDurasi(detik) << "   (p = jeda/lanjut, s = selesai, x = batal)   " << flush;
                detikTampil = detik;
                jedaTampil = jeda;
            }
            if (tombolDitekan()) {
                char c = ambilTombol();
                if (c == 'p' || c == 'P') jeda = !jeda;
                else if (c == 's' || c == 'S') break;
                else if (c == 'x' || c == 'X') { mintaBatal = true; break; }
            }
            tidurMs(100);
        }
        }
        cout << endl;
        kosongkanTombol();
        if (!mintaBatal) break;              // dihentikan dengan [s]
        string y = "n";
        try {
            y = bacaBaris("Batalkan sesi dan buang waktu yang sudah berjalan? (y/n): ");
        } catch (const InputHabis&) { y = "n"; }   // input ditutup: simpan saja
        if (y == "y" || y == "Y") { batal = true; break; }
        if (inputHabis) break;
        cout << "Sesi dilanjutkan." << endl;
    }
    if (batal) {
        cout << "Sesi dibatalkan. Tidak ada waktu yang dicatat." << endl;
        return;
    }
    catatSesi(mulai, time(0), MODE_OPENTIME, matkul, (int)(aktifMs / 1000));
}

// Menghitung sesi terpanjang dan terpendek; hasil dikembalikan lewat pointer.
void hitungRekap(const Catatan arr[], int n, int* panjang, int* pendek) {
    *panjang = 0;
    *pendek = 0;
    if (n <= 0) return;
    *panjang = arr[0].durasiDetik;
    *pendek = arr[0].durasiDetik;
    for (int i = 1; i < n; i++) {
        if (arr[i].durasiDetik > *panjang) *panjang = arr[i].durasiDetik;
        if (arr[i].durasiDetik < *pendek) *pendek = arr[i].durasiDetik;
    }
}

// Bubble sort. urutan 1 = tanggal terbaru dulu, 2 = durasi terlama dulu,
// 3 = tanggal terlama dulu (urutan kronologis).
void urutkanData(Catatan arr[], int n, int urutan) {
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            bool tukar;
            if (urutan == 1)      tukar = arr[j].waktuMulai < arr[j + 1].waktuMulai;
            else if (urutan == 2) tukar = arr[j].durasiDetik < arr[j + 1].durasiDetik;
            else                  tukar = arr[j].waktuMulai > arr[j + 1].waktuMulai;
            if (tukar) { Catatan tmp = arr[j]; arr[j] = arr[j + 1]; arr[j + 1] = tmp; }
        }
    }
}

string potong(const string& s, size_t maks) {
    if (s.size() <= maks) return s;
    return s.substr(0, maks - 2) + "..";
}

void cetakTabelCatatan(const Catatan arr[], int n) {
    cout << "\n" << left << setw(4) << "No" << setw(12) << "Tanggal" << setw(7) << "Mulai"
         << setw(8) << "Selesai" << setw(11) << "Mode" << setw(26) << "Mata Kuliah"
         << setw(10) << "Durasi" << "Materi" << endl;
    cout << string(110, '-') << endl;
    for (int i = 0; i < n; i++) {
        cout << left << setw(4) << (i + 1) << setw(12) << arr[i].tanggal
             << setw(7) << arr[i].jamMulai << setw(8) << arr[i].jamSelesai
             << setw(11) << arr[i].mode << setw(26) << potong(arr[i].matkul, 25)
             << setw(10) << formatDurasi(arr[i].durasiDetik)
             << potong(arr[i].materi, 40) << endl;
    }
}

// Mencari catatan terbaru (seluruh riwayat) dari satu mata kuliah yang materinya terisi.
// Mengembalikan true jika ada; hasilnya lewat parameter referensi.
bool materiTerakhir(const string& matkul, string& tanggal, string& materi) {
    int terbaik = -1;
    for (int i = 0; i < jumlahData; i++) {
        if (dataBelajar[i].matkul != matkul || dataBelajar[i].materi == "-") continue;
        if (terbaik < 0 || dataBelajar[i].waktuMulai > dataBelajar[terbaik].waktuMulai) terbaik = i;
    }
    if (terbaik < 0) return false;
    tanggal = dataBelajar[terbaik].tanggal;
    materi = dataBelajar[terbaik].materi;
    return true;
}

Catatan progresMatkul[MAKS_CATATAN];         // penampung catatan satu mata kuliah

void tampilProgresMatkul() {
    cout << "\n=== PROGRES SATU MATA KULIAH ===" << endl;
    int m = pilihMatkul();
    int n = 0, total = 0;
    for (int i = 0; i < jumlahData; i++) {
        if (dataBelajar[i].matkul == namaMatkul[m]) {
            progresMatkul[n++] = dataBelajar[i];
            total += dataBelajar[i].durasiDetik;
        }
    }
    if (n == 0) {
        cout << "Belum ada catatan belajar untuk " << namaMatkul[m] << "." << endl;
        return;
    }
    urutkanData(progresMatkul, n, 3);        // dari yang paling awal ke terbaru
    long selisih = (long)((awalHari(time(0)) - awalHari(progresMatkul[n - 1].waktuMulai) + 43200) / 86400);

    cout << "\nMata kuliah       : " << namaMatkul[m] << endl;
    cout << "Jumlah sesi       : " << n << endl;
    cout << "Total waktu       : " << formatDurasi(total) << endl;
    cout << "Pertama belajar   : " << progresMatkul[0].tanggal << endl;
    cout << "Terakhir belajar  : " << progresMatkul[n - 1].tanggal
         << (selisih == 0 ? " (hari ini)" : " (" + to_string(selisih) + " hari lalu)") << endl;

    cout << "\n--- Yang sudah dipelajari (dari awal sampai terbaru) ---" << endl;
    cout << left << setw(4) << "No" << setw(12) << "Tanggal" << setw(10) << "Durasi" << "Materi" << endl;
    cout << string(80, '-') << endl;
    for (int i = 0; i < n; i++)
        cout << left << setw(4) << (i + 1) << setw(12) << progresMatkul[i].tanggal
             << setw(10) << formatDurasi(progresMatkul[i].durasiDetik)
             << potong(progresMatkul[i].materi, 55) << endl;
}

// Menu lanjutan di akhir rekap. n = jumlah catatan pada rentang rekap (hasilRekap).
void menuTindakanRekap(int n) {
    while (true) {
        cout << "\nPilih tindakan:" << endl;
        cout << " 1. Daftar catatan (tanggal terbaru)" << endl;
        cout << " 2. Daftar catatan (durasi terlama)" << endl;
        cout << " 3. Progres satu mata kuliah" << endl;
        cout << " 0. Selesai" << endl;
        int u = bacaInt("Pilihan: ", 0, 3);
        if (u == 0) return;
        if (u == 3) { tampilProgresMatkul(); continue; }
        if (n == 0) { cout << "Tidak ada catatan pada rentang ini." << endl; continue; }
        urutkanData(hasilRekap, n, u);
        cetakTabelCatatan(hasilRekap, n);
    }
}

void tampilRekap() {
    cout << "\n=== REKAP BELAJAR ===" << endl;
    int z = bacaInt("Rekap berapa hari ke belakang (1-365)? ", 1, 365);

    // Hari ini dihitung sebagai hari ke-1, jadi rentang = z hari kalender.
    time_t batas = awalHari(time(0)) - (time_t)(z - 1) * 86400;
    int n = 0;
    for (int i = 0; i < jumlahData; i++)
        if (dataBelajar[i].waktuMulai >= batas) hasilRekap[n++] = dataBelajar[i];

    if (n == 0) {
        cout << "Belum ada data belajar pada " << z << " hari terakhir." << endl;
        menuTindakanRekap(0);
        return;
    }

    int total = 0;
    for (int i = 0; i < n; i++) total += hasilRekap[i].durasiDetik;
    int rata = total / z;                    // dibagi z: hari tanpa belajar ikut dihitung
    int panjang, pendek;
    hitungRekap(hasilRekap, n, &panjang, &pendek);

    int hariMemenuhiTarget = 0;
    for (int k = 0; k < z; k++) if (hariMemenuhi(k)) hariMemenuhiTarget++;

    cout << "\nRentang        : " << formatTanggal(batas) << " s.d. " << tanggalMundur(0)
         << " (" << z << " hari)" << endl;
    cout << "Jumlah sesi    : " << n << endl;
    cout << "Total belajar  : " << formatDurasi(total) << endl;
    cout << "Rata-rata/hari : " << formatDurasi(rata) << endl;
    cout << "Target harian  : " << formatDurasi(targetHarianDetik) << endl;
    cout << "Status target  : "
         << (rata >= targetHarianDetik ? "TERPENUHI (rata-rata mencapai target)"
                                       : "BELUM TERPENUHI (rata-rata di bawah target)") << endl;
    cout << "Hari memenuhi  : " << hariMemenuhiTarget << " dari " << z << " hari" << endl;
    cout << "Sesi terpanjang: " << formatDurasi(panjang) << endl;
    cout << "Sesi terpendek : " << formatDurasi(pendek) << endl;
    cout << "Streak saat ini: " << hitungStreak() << " hari" << endl;

    // Rekap per hari (hanya hari yang ada data)
    cout << "\n--- Rekap per hari ---" << endl;
    for (int k = 0; k < z; k++) {
        string tgl = tanggalMundur(k);
        int tot = totalPadaTanggal(tgl);
        if (tot > 0)
            cout << tgl << "  " << formatDurasi(tot) << "  "
                 << (tot >= targetHarianDetik ? "[target tercapai]" : "[belum tercapai]") << endl;
    }

    // Total per mata kuliah, diurutkan dari yang paling lama dipelajari
    int totalMatkul[JUMLAH_MATKUL];
    int urutan[JUMLAH_MATKUL];
    for (int m = 0; m < JUMLAH_MATKUL; m++) { totalMatkul[m] = 0; urutan[m] = m; }
    for (int i = 0; i < n; i++)
        for (int m = 0; m < JUMLAH_MATKUL; m++)
            if (hasilRekap[i].matkul == namaMatkul[m]) totalMatkul[m] += hasilRekap[i].durasiDetik;
    for (int a = 0; a < JUMLAH_MATKUL - 1; a++) {      // selection sort menurun
        int maks = a;
        for (int b = a + 1; b < JUMLAH_MATKUL; b++)
            if (totalMatkul[urutan[b]] > totalMatkul[urutan[maks]]) maks = b;
        int tmp = urutan[a]; urutan[a] = urutan[maks]; urutan[maks] = tmp;
    }
    cout << "\n--- Total per mata kuliah dalam " << z << " hari (terlama ke tersingkat) ---" << endl;
    for (int a = 0; a < JUMLAH_MATKUL; a++) {
        int m = urutan[a];
        cout << left << setw(28) << namaMatkul[m] << formatDurasi(totalMatkul[m])
             << (totalMatkul[m] == 0 ? "  (tidak ada sesi dalam rentang ini)" : "") << endl;
        string tgl, materi;
        if (materiTerakhir(namaMatkul[m], tgl, materi))
            cout << "    Terakhir dipelajari (" << tgl << "): " << potong(materi, 60) << endl;
    }

    menuTindakanRekap(n);
}

// ======================= PROGRAM UTAMA =======================

int main() {
    muatData();
    muatTarget();

    cout << "==============================================" << endl;
    cout << "   Program \"Gas Cumlaude\" - Kelompok 4" << endl;
    cout << "==============================================" << endl;

    try {
        if (targetHarianDetik <= 0) {        // pertama kali dipakai: wajib atur target
            cout << "\nSelamat datang! Atur target belajarmu terlebih dahulu." << endl;
            while (targetHarianDetik <= 0) setTarget();
        }

        int pilihan = -1;
        while (pilihan != 0 && !inputHabis) {
            tampilMenu();
            pilihan = bacaInt("Pilih menu (0-4): ", 0, 4);
            switch (pilihan) {
                case 1: setTarget(); break;
                case 2: mulaiPomodoro(); break;
                case 3: mulaiOpenTime(); break;
                case 4: tampilRekap(); break;
                case 0: break;
            }
        }
    } catch (const InputHabis&) {
        // Input ditutup (Ctrl+Z / Ctrl+D). Tidak ada data yang tertunda karena setiap
        // sesi langsung ditambahkan ke riwayat.txt saat selesai.
    }
    if (inputHabis)
        cout << "\n\nInput ditutup, program berhenti. Semua sesi yang sudah selesai sudah tersimpan." << endl;
    else
        cout << "\nTerima kasih, tetap semangat dan jaga streak-mu!" << endl;
    return 0;
}