# tefas_search_robot

TEFAS fon tarama ve model portföy sistemi. GitHub Pages üzerinden yayınlanan
statik bir site üretir. Sadece Python standart kütüphanesiyle yazılmış
(urllib, json, csv).

**13.09.2026'da ayrıldı:** Bu repo eskiden `~/bist_model_portfoy` adıyla BIST
Supertrend/sinyal sistemini de içeriyordu. O taraf ayrı, bağımsız bir repoya
(`~/hersey_guzel_olacak/bist_stocks_signal_robot`) taşındı. Bu repo eski git
geçmişini ve GitHub remote'unu aynen devraldı — GitHub Pages sitesi kesintisiz
yayında kaldı.

**13.09.2026'da yeniden adlandırıldı:** GitHub reposu `Bist-model-portfoy` →
`tefas-search-robot`. **Neden:** klasör TEFAS'tı ama repo kimliği BIST kalmıştı;
Claude Code sol barda projeyi remote'taki repo adından etiketlediği için her
oturumda yanlış proje görünüyordu. **Bedeli kabul edildi:** Pages adresi
`alprgl.github.io/Bist-model-portfoy/` → `alprgl.github.io/tefas-search-robot/`
oldu. **Eski Pages adresi yönlendirilmiyor — 404 veriyor** (doğrulandı).
GitHub yeniden adlandırmada sadece *repo* URL'ini yönlendiriyor, *Pages*
sitesini değil; git remote eski URL'le çalışmaya devam eder ama tarayıcıdaki
eski site linki ölür. `docs/` içeriğinde ve scriptlerde mutlak URL yok, o
yüzden site içi linkler etkilenmedi.

Yeni yayın adresi: `https://alprgl.github.io/tefas-search-robot/fon-model-portfoy.html`

Bu dosyanın amacı: aylar sonra geri döndüğünde "bu eşik neden bu, bu satır
neden böyle" sorularına cevap vermek. Sadece "ne" değil "neden" yazılıyor.

## ⚠️ Bu dosyayı güncel tutma kuralı (her oturum için geçerli)

**Her işin sonunda, commit etmeden önce bu dosyayı güncelle.** Konuşma bitince
orada konuşulanlar kaybolur; kalıcı olan tek şey kod ve bu dosyadır.

Şunlar olduğunda buraya yaz:
- Bir eşik/sabit değiştiyse → yeni değeri ve **neden** değiştiğini
- Bir komut/servis eklendi, kaldırıldı veya yeniden adlandırıldıysa
- Bir tasarım kararı verildiyse → özellikle "şöyle de yapabilirdik ama şu
  yüzden yapmadık" türünden olanları
- Bir tuzak/hata keşfedildiyse → belirtisi, sebebi, çözümü
- Bir sınır öğrenildiyse (veri kaynağı, API, platform) → "Bilinen sınırlar"a

Yazma: tek seferlik düzeltmeler, yazım hataları, geçici denemeler. Dosya
şişerse okunmaz olur, o zaman işe yaramaz hale gelir.

## Parçalar

| Dosya | Görev |
|---|---|
| `fon_model_portfoy.py` | TEFAS fon tarayıcı ve model portföy üretici. `docs/` klasörünü (GitHub Pages) otomatik commit+push eder (`GIT_AUTO_PUBLISH`). |
| `backtest_fon.py` | Fon model portföy kuralını TEFAS'ın geçmiş verisiyle yeniden kurup test eder. |
| `test_fon_model_portfoy.py` | Birim testleri. |

## Servis durumu (launchd)

launchd ile `~/Library/LaunchAgents/com.alpergul.fon-tarama.plist` üzerinden
çalışır. **Şu an AÇIK.**

`fon-tarama` hafta içi 09:30 ve 20:00'de `fon_model_portfoy.py`'yi çalıştırır;
script kendi içinde `docs/`'u güncelleyip GitHub'a push eder, Pages birkaç
dakikada tazelenir. **Neden kuruldu:** fon taraması uzun süre elle
çalıştırılıyordu ve site fark edilmeden 4 gün bayatladı. İki kez
çalıştırılmasının sebebi TEFAS'ın gün verisini akşam yayınlaması — akşamki
koşu o günü yakalar, sabahki koşu akşam kaçırılmışsa telafi eder.
**Dikkat:** bu servis `/opt/homebrew/bin/python3` kullanır (sistem
python'unda `openpyxl` yok, Excel çıktısı orada patlar).

Durumu kontrol et: `launchctl list | grep alpergul`

Yeniden başlat: `launchctl kickstart -k gui/501/com.alpergul.fon-tarama`
Durdur: `launchctl bootout gui/501/com.alpergul.fon-tarama`
Başlat: `launchctl bootstrap gui/501 ~/Library/LaunchAgents/com.alpergul.fon-tarama.plist`

Mac uyku moduna geçerse servis durur — geri döndüğünde koşu kaçmış olabilir,
kontrol et (aynı gün içindeki ikinci koşu genelde telafi eder).

**Servisin sağlığını nasıl anlarsın:** `launchctl list | grep alpergul`
çıktısındaki ikinci sütun son koşunun çıkış kodudur. `0` = başarılı,
`1` = koşu patlamış. Site bayatsa ilk bakılacak yer burası, ikincisi
`fon_tarama.log`'un sonu.

### Tuzak: geçici ağ hatası taramayı komple çökertiyordu (13.09.2026)

**Belirti:** `launchctl list` çıkış kodu `1`, log'un sonunda
`http.client.RemoteDisconnected`, site sessizce bayat kalıyor.

**Sebep:** `tefas_post` tekrar denemeyi sadece HTTP 429 için yapıyordu.
Bağlantı seviyesindeki hatalar (`URLError`, `RemoteDisconnected`,
`IncompleteRead`, `ConnectionError`, `socket.timeout`) hiçbir `except`
bloğuna düşmediği için tarama ilk istekte ölüyordu. TEFAS el sıkışmayı
yanıtsız kapattığında script fon türlerini bile çekemeden patlıyordu.

**Çözüm:** Bu hata türleri `GECICI_AG_HATALARI` altında toplandı ve artan
beklemeyle tekrar deneniyor. HTTP 5xx de tekrar denenebilir sayıldı. 4xx
(429 hariç) hâlâ anında patlıyor — istek bozuk demektir, tekrar denemek aynı
sonucu verir.

**Not:** Ağır istekteki asıl sebep ayrıca düzeltildi (aşağıya bak) — retry
semptomu hafifletiyordu, timeout sebebi kapattı.

### Eşik: `AGIR_ISTEK_TIMEOUT_SEC = 180` (19.09.2026)

`REQUEST_TIMEOUT_SEC = 30` bütün isteklere aynı uygulanıyordu. Tüm fonların
zaman serisi **tek istekte ~45 bin satır** dönüyor ve 30 saniye bu yanıt için
dardı: 17.09 sabah, 17.09 akşam ve 18.09 sabah koşularının üçü de tam bu
istekte (`fonGnlBlgSiraliGetir`) timeout'a girip düştü. Küçük isteklerde
(`fonTurGetir`) hiç sorun yoktu. Bu yüzden timeout istek başına ayrıldı:
ağır uç nokta 180sn, geri kalan 30sn.

### Tuzak: rapor takvim gününe bağlıydı, veri gününe değil (19.09.2026)

**Belirti:** İşlem olmayan bir günde (hafta sonu / resmî tatil / akşam yayını
çıkmadan koşan sabah taraması) skor ve akış geçmişine o günün tarihiyle satır
yazılıyordu. Değerler bir önceki günden **farklı** çıkıyordu — yeni veri
geldiği için değil, metrik penceresi bir gün kaydığı için. Sayfa da o güne ait
veri varmış gibi görünüyordu.

**Sebep:** `main()` içinde `run_date = date.today()`.

**Daha sinsi hâli:** Hafta içi 09:30 koşusu TEFAS'ın akşam yayınından önce
çalışıyor. Bugünün tarihiyle **dünün verisini** yazıyordu; akşam 20:00'de
gerçek veri geldiğinde `append_*_gecmis` "bu tarih zaten var" deyip atlıyordu.
Yani günün doğru verisi hiç kaydedilmiyordu.

**Çözüm:** Veri günü artık serinin kendisinden türetiliyor
(`veri_tarihi = max(...)`). Veri çekme penceresi bugüne kadar gitmeye devam
ediyor — orası doğruydu — ama rapor, geçmiş CSV'leri ve sayfa gerçek veri
gününe bağlanıyor. Takvim günüyle veri günü ayrıştığında koşu bunu log'a
yazıyor ("TEFAS'ın son veri günü … — rapor … üzerinden yazılıyor").

### Sayfadaki tazelik rozeti (19.09.2026)

**Neden eklendi:** "Veri tarihi: 18.09" ekranda iki ayrı durumda birebir aynı
görünüyordu — TEFAS'ta daha yenisi yok (normal) ve tarama günlerdir çöküyor
(kötü). Ayırt etmenin tek yolu `fon_tarama.log`'a bakmaktı, bu da sayfaya
bakan kişinin işi değil.

Artık "Veri tarihi"nin yanında **güncel** / **N iş günü geride** rozeti, altında
da **Son kontrol** zaman damgası var. Beklenen son veri günü tarayıcıda
hesaplanıyor: saat 20'den önceyse dünkü iş günü, sonraysa bugün; hafta sonu
geriye sarılıyor. **Sınır:** resmî tatiller bilinmiyor, o günlerde rozet
yanlışlıkla "geride" diyebilir — ipucu metni bunu söylüyor.

**Hâlâ açık:** Koşu patlarsa aktif bir uyarı gitmiyor. Rozet siteye *bakınca*
fark etmeni sağlıyor ama bakmıyorsan yine sessiz. Karar verilmedi.

**Hâlâ açık:** 17.09.2026 verisi geçmiş CSV'lerde eksik (o gün iki koşu da
ağ hatasından düştü). TEFAS'ta veri duruyor, sonradan doldurulabilir —
karar verilmedi.

**`docs/` klasörü hem bu proje hem eski `bist_model_portfoy.py`'nin çıktısını
barındırıyordu** (`index.html` + `fon.html`/`fon-model-portfoy.html`).
Bölünmede `docs/` bu repoda kaldığı için `index.html`/`metodoloji.pdf` hâlâ
burada duruyor ama artık kimse tarafından güncellenmiyor (üreten script
`bist_model_portfoy.py`, `bist_stocks_signal_robot` reposuna taşındı ve orada
`docs/`'a erişimi yok). Bu dosyalar istenirse silinebilir ya da elle
güncellenmeye devam edilebilir — karar verilmedi.

## Fon sistemi — backtest bulguları (12.09.2026)

`backtest_fon.py` kuralı TEFAS'ın geçmiş verisiyle 10 dönem yeniden kurdu
(2025-11 … 2026-08). Ortaya çıkanlar:

- **`fon_portfoy.csv`'deki `entry_price` tarama gününün fiyatıdır**, ama o
  fiyattan alım mümkün değil — emir o akşam verilir, ertesi işlem günü
  fiyatından gerçekleşir (valör). Gerçek getiri CSV'nin gösterdiğinden bir
  miktar düşük.
- **Getiriyi üreten şey momentum, üç katmanlı skor değil.** Ablasyon testinde
  "sadece 1 aylık getirisi en yüksek 6 fon" daha yüksek ortalama verdi
  (+%14,6 vs +%11,0). Skorun asıl işlevi **kuyruğu kesmek**: tek başına skor
  ortalamada zayıf (+%5,1) ama hiç eksi ay vermiyor; saf momentumun en kötü
  ayı −%39, tam kuralınki −%3,2.
- **Kapasite tuzağı (düzeltilmedi, dikkat):** Mevcut risk filtresi (≥10 Mn TL,
  ≥10 kişi) karar günü ölçtüğü için küçülmekte olan fonları geçiriyor. Bir
  dönemde seçilen STM, ertesi gün 102,9 Mn TL'den 402 bin TL'ye düşmüş ve
  kalan minik bakiyede NAV %51 sıçramıştı — o aya yazılan +%25'in tamamı bu
  artefakttan geliyordu. Kapasite şartı (≥100 yatırımcı, ≥50 Mn TL) eklenince
  sonuç kötüleşmiyor, **iyileşiyor** (+%11,9, en kötü ay +%5,8).
- **Çeşitlendirme illüzyonu:** Kural varlık sınıfına bakmıyor. Bir dönemde
  seçilen 6 fonun altısı da gümüş fonuydu — "6 fonluk sepet" tek bir emtia
  bahsiydi.
- **Sepet çoğu ay dolmuyor:** aday havuzu 2-12 fon arası; bazı aylar 2-3 fon
  seçiliyor. Boş kontenjan nakitte mi tutulmalı, kalanlara mı bölünmeli —
  karar verilmedi, 10 ayda aradaki fark ~500 bin TL.
- **TEFAS API tuzağı:** tarih aralığı sınırı gün değil **takvim ayı** bazlı.
  30 günlük adım Şubat'ı aşınca API hata vermeden **boş liste** dönüyor.
  `backtest_fon.py` bu yüzden 28 günlük adım kullanıyor ve boş cevabı asla
  önbelleğe almıyor — yoksa sessizce eksik veriyle çalışılır.

## Kurulum / bakım

**Telegram:** Bu sistem Telegram kullanmıyor (fon tarafı hiç Telegram
entegrasyonu içermiyor, doğrulandı). Tüm çıktı `docs/` üzerinden statik site
olarak yayınlanıyor.

## Kod alışkanlıkları

- Sadece stdlib (urllib, json, csv). Yeni bağımlılık eklemeden önce sor.
- Kod içi yorumlar Türkçe + ASCII (ı/ğ/ş yerine i/g/s); docstring'lerde
  Türkçe karakter serbest. Bu dosya (CLAUDE.md) istisna, tam Türkçe
  karakterle yazıldı.
- Yorumlar "ne" değil "neden" açıklar — kod zaten ne yaptığını anlatıyor.
- Ölü kod bırakılmaz: bir özellik kaldırılınca ona ait sabit/yardım metni de gider.
- Değişiklik sonrası `python3 -m py_compile <dosya>` ile derle; servisi
  ilgilendiriyorsa `launchctl kickstart -k gui/501/com.alpergul.fon-tarama`.
