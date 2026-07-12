# QueryGSC 🚀

QueryGSC, Google Search Console (GSC) verilerinizi yerel bilgisayarınızda işleyerek en kritik SEO fırsatlarını, anahtar kelime yamyamlıklarını (cannibalization), içerik eskimesini (content decay) tespit eden ve **Gemini API** desteğiyle otomatik SEO eylem planları hazırlayan, **Tauri** tabanlı premium bir masaüstü uygulamasıdır.

Uygulama, verilerinizin gizliliğini en üst düzeyde tutmak amacıyla **%100 yerel (local-first)** çalışacak şekilde tasarlanmıştır. Hiçbir veriniz veya API anahtarınız dış bir sunucuya yüklenmez, tamamen tarayıcınızın/bilgisayarınızın yerel depolama alanında (`localStorage`) saklanır.

---

## 🌟 Öne Çıkan Özellikler

### 1. Panel & Genel Bakış (Dashboard)
*   Google Search Console performans verilerini grafik ve trend çizgileriyle izleme.
*   Toplam Tıklama, Gösterim, Ortalama Tıklama Oranı (CTR) ve Ortalama Pozisyon değerlerinin takibi.
*   Yerel zip dosyası (GSC dışa aktarım dosyası) veya canlı API bağlantısı ile anında analiz.

### 2. Fırsat Kaşifi (Opportunity Explorer)
*   **Striking Distance Fırsatları:** Arama sonuçlarında 4. ile 20. sıra arasında yer alan, gösterimi yüksek fakat tıklaması düşük "hemen kazanılabilir" anahtar kelimeleri tespit eder.
*   **Düşük CTR (Low CTR) Fırsatları:** Pozisyonuna göre beklenenin altında tıklama alan sayfaları ve sorguları listeler.
*   **Tahmini Aylık Tıklama Kazancı (+Clicks):** Yapılan optimizasyonların getireceği potansiyel trafik artışını matematiksel olarak tahmin eder.
*   **Detaylı Analiz Paneli:** Seçilen her fırsat için CTR eğrisi, detaylı metrikler ve **Yapay Zeka SEO Raporu** oluşturma arayüzü.

### 3. Yamyamlık Denetçisi (Keyword Cannibalization)
*   Aynı anahtar kelime/sorgu için Google'da birbiriyle rekabet eden birden fazla sayfanızı listeler.
*   İçerik çakışmalarını çözebilmeniz için 301 yönlendirmesi, Canonical etiketi veya içerik birleştirme (consolidation) önerileri sunar.

### 4. İçerik Eskimesi Analizi (Content Decay)
*   Güncel performans dönemi ile geçmiş dönemi karşılaştırarak sıralama ve trafik kaybeden sorguları listeler.
*   Düşüş hızına göre (Click drop velocity) sayfalarınızı önceliklendirir.

### 5. Gemini Destekli Yapay Zeka SEO Asistanı
*   **Gemini 2.5 Flash / Pro** modelleriyle entegre chat arayüzü.
*   Fırsat Kaşifi'nden seçilen kelimelere tek tıkla otomatik SEO raporu hazırlama.
*   Rapor çıktılarında **Markdown** formatlama desteği ile başlık, kod blokları ve tabloları şık ve okunabilir şekilde listeleme.
*   Konuşma geçmişinin yerel bilgisayarda otomatik olarak saklanması (Persistence) ve sohbet esnasında otomatik aşağı kaydırma (Auto-Scroll).

### 6. Kolay ve Güvenli GSC API Bağlantısı
*   Gömülü webview'leri engelleyen Google güvenlik protokolünü aşmak için yerel tarayıcı üzerinden yetkilendirme (OAuth) yönlendirmesi.
*   **Google OAuth Playground** üzerinden 30 saniyede hızlı API entegrasyon kılavuzu ve Token doğrulaması.

---

## 🛠️ Teknoloji Yığını (Tech Stack)

*   **Çekirdek/Çalışma Zamanı (Desktop Shell):** [Tauri v2](https://tauri.app/) (Rust tabanlı yüksek performanslı ve güvenli masaüstü uygulama çalışma ortamı)
*   **Arayüz (Frontend):** Vue 3 (Composition API, `<script setup>`), TypeScript, Vite
*   **Tasarım/Arayüz:** Özel koyu tema (#2A2927), neon kırmızı vurgular (#E1251A), modern Glassmorphism bileşenleri
*   **Markdown Derleyici:** [Marked](https://marked.js.org/)

---

## 🚀 Başlangıç ve Kurulum

Projeyi yerel bilgisayarınızda çalıştırmak ve derlemek oldukça basittir.

### Gereksinimler
*   Node.js (v18+)
*   Rust / Cargo (Tauri için)

### Geliştirme Ortamını Başlatma
1.  Bağımlılıkları yükleyin:
    ```bash
    npm install
    ```
2.  Uygulamayı geliştirme modunda (Tauri dev server) başlatın:
    ```bash
    npx tauri dev
    ```

### Üretim Derlemesi (Build)
Uygulamayı kendi işletim sisteminiz için paketlenmiş (macOS `.dmg`, Windows `.msi`, Linux `.deb`) hale getirmek için:
```bash
npx tauri build
```
Bu komut, uygulamanın tüm görsel ikon paketleri dahil olmak üzere son kullanıcıya sunulabilir sürümünü derler.

---

## 🛡️ Güvenlik ve Gizlilik Beyanı

QueryGSC bir **Local-First** yazılımdır.
*   GSC'den çektiğiniz veya CSV ile yüklediğiniz veriler sadece tarayıcınızın/webview'in hafızasında işlenir.
*   Gemini API Anahtarınız ve Google Access Token verileriniz, işletim sisteminizin yerel depolamasında saklanır ve doğrudan ilgili Google uç noktalarına (endpoint) gönderilir.
*   Araya giren hiçbir üçüncü taraf sunucu, veri tabanı veya bulut servisi bulunmamaktadır.

## 📄 Lisans
Bu proje MIT Lisansı altında lisanslanmıştır.
