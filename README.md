# Yol Takip

Birkaç ortağın her gün ortak para koyup bir "yol" açtığı, yolların sonucunu ve bakiyesini takip eden mobil web uygulaması (PWA). Arayüz Türkçedir. Tek dosyadır (`index.html`, vanilla JS + CSS), framework ve build adımı yoktur.

## Ne yapar

- Her gün bir yol açılır. Varsayılan: 3 ortak x 25 TL = 75 TL. Ortaklar ve paylar **Ayarlar**'dan değişir.
- Yolun sonucu girilir:
  - **Kazandı**: oran ya da yeni bakiye girilir.
  - **Kaybetti**: yol kapanır, bakiye 0 olur.
  - **Çek**: istenen tutar çekilir.
- Bir dönem 30 gündür (31 yapılabilir). Bakiye 500 TL'ye ulaşınca 50 TL oyunda bırakılıp kalanı çekmek önerilir. Tutar değiştirilebilir.
- **Kasayı böl**: son açılan yol ve kaybeden yollar hariç, aktif yolların toplam bakiyesi eşit paylara bölünür. Kuruş farkı ilk yollara gider. Önizleme çıkar, elle tetiklenir ve geri alınabilir.
- Yanlış girişler **Son işlemi sil**, **Yolu sil** ve **Geri al** ile düzeltilir.
- **WhatsApp özeti**: `4.Gün 877 - 127` biçiminde satırlar, kapanan yollar üstü çizili, altında yatırılan / çekilen / oyunda / net toplamları.
- Açık ve koyu tema, çevrimdışı çalışma, telefona eklenebilir (ana ekran).

## Kullanım

1. Sitenin adresini telefonda açıp "Ana ekrana ekle" ile kurun.
2. **Ayarlar**'dan ortakları, payları ve dönem kurallarını girin.
3. Alttaki **Yol aç** ile günün yolunu başlatın, sonucu kart üzerindeki düğmelerle girin.
4. **Özet** sekmesinde kişi bazında yatırılan, çekilen, oyundaki ve net tutarlar görünür.

Yerel olarak denemek için `index.html` dosyasını tarayıcıda açmak yeterlidir. Service worker için bir yerel sunucu gerekir, örneğin `python3 -m http.server`.

## Düzenleme şifresi

**Ayarlar > Düzenleme şifresi**'nden şifre belirlenirse uygulama yalnızca sonuçları gösterir (net kart, yollar, bakiyeler, özet, WhatsApp özeti). **Düzenle** düğmesi şifre ister. 5 dakika işlem yapılmazsa ya da sayfa yenilenirse tekrar kilitlenir. Şifre belirlenmezse düzenleme eskisi gibi herkese açıktır.

Şifrenin yalnızca tuzlu özeti (PBKDF2-SHA256) `S.lock` içinde veriyle birlikte eşitlenir, böylece bütün ortaklarda aynı şifre geçerli olur. Şifre repoda tutulmaz.

Bu kilit yanlışlıkla değişikliği ve sonuçlara bakanların düzenlemesini engeller. Güvenlik sınırı değildir: kontrol tarayıcıda yapılır ve Firebase kuralı grup kodunu bilen herkese yazma izni verir. Grup kodunu paylaşmayın.

## Bulut eşitleme (Firebase)

Eşitleme Firebase Realtime Database üzerinden REST + SSE ile yapılır, SDK yoktur.

1. [Firebase konsolunda](https://console.firebase.google.com) bir proje ve **Realtime Database** oluşturun.
2. `index.html` içindeki `DB` sabitine veritabanı adresini yazın.
3. Kurallar sekmesine şunu yazın:

   ```json
   {
     "rules": {
       "gruplar": {
         "$kod": { ".read": true, ".write": true }
       }
     }
   }
   ```

   Erişim, uzun ve rastgele grup koduyla korunur. Kodu bilmeyen kimse veriye ulaşamaz. Kodu herkese açık yerlerde paylaşmayın.
4. Uygulamada **Ayarlar > Yeni grup başlat** ile grup kodu oluşturun. Diğer ortak kodu girerek ya da davet linkiyle (`#k=kod`) bağlanır.

Veri `/gruplar/<grupKodu>.json` altında `{u: güncellemeZamanı, by: cihazId, d: JSON.stringify(S)}` olarak durur. Çevrimdışıyken girilen işlemler bağlantı gelince gönderilir.

Repoda API anahtarı, token ya da şifre yoktur ve olmamalıdır.

## Yayınlama (Netlify)

1. Netlify'da **Add new site > Import an existing project** ile bu GitHub reposunu seçin.
2. **Build command**: boş bırakın. **Publish directory**: `.` (kök).
3. Deploy edin. `main` dalına yapılan her push otomatik yayınlanır.
4. Yayına çıkan değişikliğin telefonlarda görünmesi için `sw.js` içindeki `CACHE` adını artırın (örn. `yol-takip-v4`).

## Veri modeli

Veri tarayıcıda `localStorage` içinde `yol-takip-v1` anahtarında tutulur.

```
S = {
  rules:    { target, keep, length },
  partners: [ { id, name, amt, on } ],
  cycles:   [ {
    id, start, length, rules,
    splits: {},
    paths: [ {
      id, n, date, stake,
      contrib, own, wdBy,
      balance, status,   // "active" | "lost" | "cashed"
      withdrawn,
      log: []            // win | loss | cek | split kayıtları
    } ]
  } ],
  lock,      // { salt, hash } düzenleme şifresi özeti, isteğe bağlı
  updated
}
```

Veri modeli değişirse `migrate()` eski veriyi yeni alanlara taşımalıdır. Geriye dönük uyumluluk bozulmamalıdır.

## Dosyalar

- `index.html`: uygulamanın tamamı
- `manifest.webmanifest`: PWA tanımı
- `sw.js`: ağ öncelikli önbellek (service worker)
- `icons/`: uygulama simgeleri
