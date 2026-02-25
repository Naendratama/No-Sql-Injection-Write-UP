# No-Sql-Injection-Write-UP
Category: Web Exploitation
Level: Medium
Author: NGIRIMANA Schadrack

Description:
Can you try to get access to this website to get the flag?
You can download the source here.
Additional details will be available after launching your challenge instance.

Hints: 1. Not only SQL injection exist but also NonSQL injection exists.
       2. Make sure you look at everything the server is sending back.

Recoon:
Pertama cek source codenya dulu, dan analisis menggunakan chatgpt
Ditemukan endpoint penting:
POST /login
GET /admin
/ admin bisa diakses tanpa autentikasi tambahan. Ini indikasi tidak ada middleware proteksi seperti:
app.use(authMiddleware)

Analisis Response Login
Clue: “Look at everything the server is sending back.”
Saat login gagal dan berhasil, server mengembalikan response JSON berisi seluruh object user:
if (user) {
  res.json(user);
}
Artinya seluruh field database dikirim, termasuk token.

Recon 3: Analisis Cara Input Diproses
Bagian krusial:
if (email.startsWith("{") && email.endsWith("}")) {
  email = JSON.parse(email);
}

if (password.startsWith("{") && password.endsWith("}")) {
  password = JSON.parse(password);
}
Dari sini muncul dua insight penting saat recon:
1. Server memperbolehkan input berbentuk JSON.
2. Hasil parsing langsung digunakan di query.

Recon 4: Analisis Query ke Database
Query autentikasi:
const user = await User.findOne({
  email: email,
  password: password
});
Tidak ada:
Validasi tipe data
Casting ke string
Sanitasi operator

Recon 5: Identifikasi Permukaan Serangan
MongoDB mendukung operator query dalam bentuk object, misalnya:
{ $ne: null }
{ $gt: "" }
{ $regex: ".*" }

Karena server mengizinkan email dan password menjadi object,
maka attacker bisa mengirim operator Mongo sebagai input.

Recon 6: Perubahan Logika Query
Login normal:
{
email: "user@example.com
",
password: "secret"
}
Maknanya:
(user.email = E) AND (user.password = P)
Jika dikirim:
{
"email": {"$ne": null},
"password": {"$ne": null}
}
Query berubah menjadi:
{
email: { $ne: null },
password: { $ne: null }
}
Maknanya:
(user.email ≠ null) AND (user.password ≠ null)
Itu kondisi umum, bukan validasi kredensial.

Recon Conclusion
Dari tahap recon terlihat bahwa:
Input dapat menjadi object
Object langsung dimasukkan ke Mongo query
Mongo menginterpretasikan object sebagai operator logika

Exploitation:
Menggunakan burp suite, kita coba memasukkan payload
email:
{"$ne": null}
password:
{"$ne": null}






