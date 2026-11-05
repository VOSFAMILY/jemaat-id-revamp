# API_MAP — JemaatID Revamp (frontend statis → Apps Script API JSON)

Frontend: `web/index.html` (statis, GitHub Pages). Backend: Google Apps Script
Web App `Code.gs` (doPost, routeApi_). Semua panggilan server lewat SATU fungsi
`run(fn, arg, ok)` di `index.html` (~baris 575).

## Cara kerja run()

```js
var API_URL = 'https://script.google.com/macros/s/AKfycbxF6O1BxbDe0G1_nFOx751ANml-x2iktPjP2AMstUHsR07OTs4hKuMVbGeFSP7fYRIP/exec';
function run(fn, arg, ok){
  var body = Object.assign({action:fn}, (typeof arg==='string' ? {token:arg} : (arg||{})));
  fetch(API_URL, {
    method:'POST',
    headers:{'Content-Type':'text/plain;charset=utf-8'},  // tanpa preflight CORS
    redirect:'follow',
    body:JSON.stringify(body)
  }).then(function(res){ return res.text(); }).then(function(txt){
    var r;
    try { r = JSON.parse(txt); } catch(e){ toast('Respons server tidak valid'); return; }
    if(r && r.ok===false){
      toast(r.err||'Gagal');
      if(r.err && /login lagi|Belum login/i.test(String(r.err))){
        TOKEN=''; sessionStorage.removeItem('jemaat_token'); location.reload();
      }
      return;
    }
    ok(r);
  }).catch(function(e){ toast('Gagal: '+(e&&e.message?e.message:e)); });
}
```

- `arg` string mentah → dikirim sebagai `{action, token: <string>}` (dipakai
  `apiLogout` dan `apiMe`). `arg` objek → digabung ke `{action, ...arg}`.
- Respons dibaca sebagai text lalu `JSON.parse`. Bila `r.ok===false` →
  `toast(r.err)`; bila `err` mengandung "login lagi" atau "Belum login" →
  token dihapus dari `sessionStorage` (key `jemaat_token`), `TOKEN` dikosongkan,
  lalu `location.reload()` (kembali ke halaman login).
- Gagal jaringan / respons bukan JSON → toast "Gagal: …" / "Respons server
  tidak valid". TIDAK dipakai `mode:'no-cors'` (respons opaque menyembunyikan
  error).
- Token sesi tetap di `sessionStorage` key `jemaat_token`; perilaku
  login/logout sama seperti versi Apps Script. Sisi server (`routeApi_` di
  Code.gs) menerima token mentah untuk `apiLogout`/`apiMe` (`p.token`) dan
  objek `p` untuk 13 action lainnya.

## Pemetaan action

| action | Bentuk argumen yang dikirim (body JSON) | Penanganan token / error |
|---|---|---|
| `apiLogin` | `{action, username, password, perangkat}` | Tanpa token. `r.ok===false` → toast(r.err), tetap di login. Sukses: token disimpan ke `sessionStorage['jemaat_token']` + `TOKEN` (logika lama di `doLogin`). |
| `apiLogout` | `{action, token: "<token string mentah>"}` — arg string mentah | Token dikirim apa adanya. Setelah panggil, frontend selalu menghapus token + `location.reload()` (logika lama di `doLogout`). |
| `apiMe` | `{action, token: "<token string mentah>"}` — arg string mentah | Token mentah. `r.ok===false` (token kedaluwarsa / "Belum login") → toast + hapus token + reload ke login (aturan umum run()). |
| `apiGantiPassword` | `{action, token: TOKEN, lama, baru}` | Token di body. Error → toast, form tetap terbuka. |
| `apiList` | `{action, token: TOKEN, tab}` | Token di body. Dipakai via `apiList(tab, cb)`; bila respons tanpa `rows` → `cb([])`. |
| `apiSave` | `{action, token: TOKEN, tab, row}` | Token di body. Error → toast (atau `toast(r.err)` di pemanggil). |
| `apiDelete` | `{action, token: TOKEN, tab, id}` | Token di body. Error → toast. |
| `apiDashboard` | `{action, token: TOKEN}` | Token di body. |
| `apiLaporanKeuangan` | `{action, token: TOKEN, bulan, tahun}` | Token di body. |
| `apiGenerateJadwal` | `{action, token: TOKEN, master_id, mulai, sampai}` | Token di body. |
| `apiKirimKeIdCard` | `{action, token: TOKEN, jemaat_id}` | Token di body. |
| `apiPengaturan` | `{action, token: TOKEN}` | Token di body. |
| `apiSimpanPengaturan` | `{action, token: TOKEN, kunci, nilai}` | Token di body. Sukses → `PENGATURAN[kunci]=nilai` + toast (logika lama). |
| `apiListUsers` | `{action, token: TOKEN}` | Token di body. Bila tanpa `users` → `USERS=[]`. |
| `apiSimpanUser` | `{action, token: TOKEN, user}` | Token di body. Sukses → toast + `muatUsers()` (logika lama). |

## Catatan

- Tidak ada lagi `google.script.run` / `google.script` di frontend ini.
- Tidak ada sintaks template Apps Script `<?= ... ?>` di frontend ini.
- UI, kolom, dropdown, teks, dan aturan role identik dengan `gas/Index.html`
  (hanya `run()` dan `<head>` yang berubah).
