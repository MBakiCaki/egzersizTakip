# egzersizTakip
Fitness egzersizlerinin ağırlık ve tekrar bilgilerini kaydederek hatırlamaya yardımcı olan web uygulaması. Hedeflerinizi, son egzersiz verilerini ve sonraki antrenmanda yoğunluğu değiştirmek isteyip istemediğinizi takip etmenize yarar.
Zamanla gelişimi takip eden bir özellik eklenmedi.

_Uygulama ve teknik dokümantasyon: Claude Sonnet 4.6_

# Egzersiz Takip — Teknik Dokümantasyon

## Kullanılan Teknolojiler

| Teknoloji | Kullanım Amacı |
|---|---|
| **HTML5** | Yapı ve semantik işaretleme |
| **CSS3** | Layout (Flexbox, Grid), CSS değişkenleri, geçişler ve mikro-etkileşimler |
| **Vanilla JavaScript (ES6+)** | Tüm uygulama mantığı — kütüphane veya framework yok |
| **Web Storage API** (`localStorage`) | Kalıcı veri katmanı, tamamen istemci tarafında |
| **Google Fonts** (CDN) | Bebas Neue, DM Sans, DM Mono |

---

## Genel Mimari
Bu bir **Single-Page Application (SPA)** — ama kasıtlı olarak sade tutulmuş bir tane. Sunucu yok, derleme adımı yok, framework yok. Tamamen tarayıcıda çalışan, bağımsız bir `.html` dosyası. "Tek sayfa" burada gerçek anlamda tek sayfa demek: bir `.html` dosyası, bir URL, hiçbir zaman başka bir sayfaya yönlendirme yok. İki "sayfa" (ana ekran ve Hedefler) aslında JavaScript ile `display: none / block` arasında geçiş yapılan iki `<div>`den ibaret.

Uygulama üç temel sorumluluğa sahip: **veri**, **durum (state)** ve **arayüz (UI)**. Bunların hepsi tek bir `<script>` bloğu içinde yer alıyor, ancak sorumluluk açısından birbirinden net biçimde ayrılmış.

---

### 1. Veri Katmanı — localStorage'ı Veritabanı Olarak Kullanmak

Her veri parçası, `localStorage`'da isimlendirilmiş anahtarlar altında JSON string olarak tutulur:

```
liftlog_p1          →  1. Gün hareket dizisi
liftlog_p2          →  2. Gün hareket dizisi
liftlog_note_p1     →  1. Gün not alanı metni
liftlog_note_p2     →  2. Gün not alanı metni
liftlog_goals       →  Hedefler dizisi
liftlog_created     →  ISO format tarih (versiyon gösterimi için)
```

Her okuma işlemi `loadData()` → `JSON.parse()` üzerinden geçer; her yazma işlemi ise `JSON.stringify()` → `localStorage.setItem()` ile gerçekleşir. Senkrondan çıkabilecek bir bellek içi önbellek tutulmaz — her işlem depolamadan taze okur ve hemen geri yazar.

---

### 2. Durum Katmanı — Şu An Ne Oluyor?

Bellek içi state (durum), yalnızca localStorage'ın ifade edemeyeceği geçici UI durumlarını tutar:

```js
let currentProgram = 1          // hangi gün sekmesinin aktif olduğu
const expandedIds  = new Set()  // hangi hareket kartlarının açık olduğu
const editingIds   = new Set()  // hangi kartların düzenleme modunda olduğu
let openPickerId   = null       // hangi trend picker açık
let confirmCb      = null       // onay dialogu için bekleyen callback
let drag           = null       // aktif sürükleme işlemi nesnesi
```

Bunlar sıradan değişkenler ve Set yapıları; reaktif bir store değil. State değiştiğinde ilgili fonksiyon DOM'u manuel olarak günceller — ya `render()` çağrısıyla listenin tamamını yeniden boyar, ya da `updateEditState(id)` gibi bir fonksiyonla `data-card` attribute'unu kullanarak ilgili kartı bulur ve yalnızca o kartın alt elemanlarına dokunur.

---

### 3. Arayüz Katmanı — İki Farklı Render Stratejisi

Uygulama, işlemin maliyetine göre iki farklı yaklaşım kullanır:

**Tam yeniden render** — `render()` ve `renderGoals()`, tüm listeyi `.innerHTML = data.map(...).join('')` ile yeniden inşa eder. Listeler küçük olduğu için bu yöntem basit ve doğru çalışır. Her öğe, güncel durumu (açık mı? düzenleniyor mu? tamamlandı mı?) doğrudan CSS sınıfları ve attribute olarak içine gömmüş bir template literal'dır.

**Cerrahi DOM güncellemesi** — Trend rozeti değiştirme, hedef tamamlama veya düzenleme modunu açıp kapama gibi işlemlerde, kod ilgili öğeyi `data-*` attribute'u ile bulur ve yalnızca değişen kısmı günceller. Bu yöntem, küçük bir state değişikliği için listenin tamamının yanıp sönmesini önler.

---

### 4. Olay Yönetimi — Delegasyon ve Inline Yaklaşımı

Render edilen kartların içindeki butonlar, öğenin `id`'si kodun içine gömülmüş inline `onclick` handler'ları kullanır:

```js
onclick="askDelete(${m.id}, event)"
```

Bu kasıtlı bir tercihtir — liste tamamen yeniden render edildiği için, render sonrasında event listener eklemek temizleme mantığı gerektirir. Inline handler'lar ise yeniden render'lardan sağlıklı biçimde kurtulur.

Picker'ları dışarı tıklamayla kapatma, ayarları kapatma ve klavye kısayolları gibi genel olaylar ise bir kez `document`'e `addEventListener` ile bağlanır.

---

### 5. Sayfa Navigasyonu

Router yok. `showGoals()` fonksiyonu `main-view`'i `display:none` yapar, `goals-view`'i `display:block` yapar. `showMain()` bunu tersine çevirir. Tarayıcının URL'i hiç değişmez.

---

### 6. Onay Dialogu — Command Pattern

Hareketler ve hedefler için iki ayrı dialog yerine tek bir genel dialog kullanılır. Saklanan bir callback aracılığıyla çalışır:

```js
openConfirm('Hareketi sil', m.name, () => {
  saveData(getData().filter(x => x.id !== id));
  render();
});
```

`doConfirm()` yalnızca en son atanan `confirmCb`'yi çağırır. Bu yöntem (callback saklayıp daha sonra çağırma), **Command Pattern**'in basit bir biçimidir.

---

### 7. Sürükle-Bırak ile Yeniden Sıralama

Hedefler listesindeki sürükle-bırak yeniden sıralama, ham pointer olayları kullanır — mobil için `touchstart` / `touchmove` / `touchend`, masaüstü için `mousedown` / `mousemove` / `mouseup`.

Sürükleme başladığında, görsel bir klon (`.drag-ghost`) `<body>`'e eklenir ve parmak hareket ettikçe `position: fixed` ile taşınır. Ekleme noktası, her öğenin orta Y koordinatının anlık imleç konumuyla karşılaştırılmasıyla bulunur. Bırakıldığında hedefler dizisi `splice` ile yeniden düzenlenerek kaydedilir, klon kaldırılır ve `renderGoals()` yeni sırayı çizer.
