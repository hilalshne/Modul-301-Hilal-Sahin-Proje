# **ZipSplit v1.0 (MVP) \- Kapsamlı Teknik Ürün Gereksinim Dokümanı (PRD)**

**Hazırlayan:** Technical Product Owner (TPO)  
**Vizyon:** Kurumsal gezginler için sıfır sürtünmeli harcama yönetimi.  
**Versiyon:** 1.0 (MVP)

## **1\. Proje Özeti ve Stratejik Hedef**

ZipSplit, iş seyahati gerçekleştiren profesyonellerin harcama takibi, fiş arşivleme ve ekip içi borç dengesi süreçlerini dijitalleştiren, offline-first (önce çevrimdışı) prensibiyle çalışan bir iOS uygulamasıdır. Temel amaç, bir harcama girişinin 1.5 saniyeden kısa sürede tamamlanmasını ve seyahat sonunda saniyeler içinde muhasebeye uygun PDF raporu oluşturulmasını sağlamaktır.

## **2\. Teknik Mimari ve Teknoloji Yığını**

| Bileşen | Teknoloji Seçimi | Teknik Gerekçe   |
| :---- | :---- | :---- |
| **Mobil Frontend** | Swift (iOS Native) | Native performans, akıcı animasyonlar ve derin sistem entegrasyonu (Kamera, Share Sheet). |
| **Lokal Veritabanı** | SwiftData | Çevrimdışı modda veri tutarlılığı ve internet geldiğinde asenkron senkronizasyon. |
| **Backend API** | FastAPI (Python) | Yüksek asenkron performans, otomatik Swagger dokümantasyonu ve kolay dağıtım. |
| **Veritabanı (Cloud)** | Supabase (PostgreSQL) | İlişkisel veri güvenliği ve Vendor Lock-in riskini minimize eden açık kaynak SQL yapısı. |
| **Dosya Depolama** | Supabase Storage | Fiş fotoğrafları için güvenli ve ölçeklenebilir S3 uyumlu depolama. |

## **3\. Detaylı Veritabanı Şeması (ERD)**

Veri tutarlılığı için PostgreSQL üzerinde kurulacak temel tablo yapıları:

### **3.1. Tablo: users**

* id (UUID, PK): Kullanıcının benzersiz kimliği (Auth ID).  
* email (VARCHAR, Unique): İletişim e-postası.  
* full\_name (VARCHAR): Kullanıcı adı ve soyadı.

### **3.2. Tablo: groups (Seyahat Grupları)**

* id (UUID, PK): Grup kimliği.  
* title (VARCHAR): Seyahat adı (Örn: "Barselona Konferansı").  
* base\_currency (VARCHAR): Gruba ait ana para birimi (USD, EUR, TRY).  
* invite\_code (VARCHAR, Unique): Katılım için üretilen 8 haneli kod.

### **3.3. Tablo: transactions (Harcamalar)**

* id (UUID, PK): Harcama kimliği.  
* group\_id (UUID, FK): Ait olduğu grup.  
* payer\_id (UUID, FK): Parayı ödeyen kullanıcı.  
* amount (DECIMAL 10,2): Harcama tutarı.  
* category (VARCHAR): Harcama kategorisi (Yemek, Ulaşım, Konaklama vb.).  
* receipt\_path (VARCHAR, Nullable): Storage üzerindeki görsel yolu.  
* created\_at (TIMESTAMP): Kayıt zamanı.

### **3.4. Tablo: transaction\_splits (Borç Dağılımı)**

* id (UUID, PK): Dağılım kimliği.  
* transaction\_id (UUID, FK): İlgili harcama.  
* user\_id (UUID, FK): Borçlu olan kullanıcı.  
* owed\_amount (DECIMAL 10,2): Bu işlemden düşen borç payı.

## **4\. Ekran Tasarımları ve Kullanıcı Akışları**

### **4.1. Dashboard (Ana Ekran)**

* **Fonksiyon:** Aktif seyahat gruplarının listelenmesi.  
* **Kritik Bileşen:** Sağ alt köşede yüzen "+" butonu (Yeni seyahat başlatma).  
* **Teknik Aksiyon:** SwiftData üzerinden lokaldeki grupları anında render eder, arka planda API ile senkronize olur.

### **4.2. Harcama Giriş Ekranı**

* **Fonksiyon:** Harcama miktarının ve detaylarının girilmesi.  
* **Kritik Bileşen:** Sayfa açılır açılmaz odağı alan sayısal klavye ve kamera ikonu.  
* **Split Mantığı:** Varsayılan olarak gruptaki herkes seçili gelir (Eşit bölüşüm).

### **4.3. Raporlama ve Dışa Aktarma**

* **Fonksiyon:** PDF raporu oluşturma.  
* **Kritik Bileşen:** "Hemen Paylaş" (Share Sheet) butonu.  
* **Teknik Aksiyon:** PDF, Swift tarafında PDFKit kullanılarak cihaz içinde üretilir. Fiş fotoğrafları thumbnail olarak eklenir ve bulut linkleri gömülür.

## **5\. User Stories ve Kabul Kriterleri (Acceptance Criteria)**

### **User Story 1: Çevrimdışı Harcama Kaydı**

**Gereksinim:** Bir kurumsal gezgin olarak, internetimin olmadığı anlarda harcama girebilmek istiyorum.

* **AC1:** Uygulama internet yokken "Kaydet" denildiğinde takılmamalı ve işlemi lokale yazmalıdır.  
* **AC2:** Harcama, listede "Senkronizasyon Bekliyor" ikonu ile gösterilmelidir.  
* **AC3:** İnternet geldiğinde veri kaybı olmadan otomatik olarak buluta aktarılmalıdır.

### **User Story 2: Fiş Fotoğrafı Ekleme**

**Gereksinim:** Muhasebe onayı için harcamaya anında fiş fotoğrafı ekleyebilmek istiyorum.

* **AC1:** Kamera uygulaması içinden saniyeler içinde açılmalıdır.  
* **AC2:** Görsel, istemci tarafında 1MB altına düşürülecek şekilde sıkıştırılmalıdır.  
* **AC3:** Yükleme işlemi arka planda (Background Task) tamamlanmalı, kullanıcıyı engellememelidir.

## **6\. API Kontratı ve Senkronizasyon Stratejisi**

| Endpoint | Method | Açıklama   |
| :---- | :---- | :---- |
| /api/v1/sync/transactions | POST | Lokaldeki harcamaları toplu olarak buluta aktarır. |
| /api/v1/groups/{id}/balances | GET | Net borç dengesini (Settlement) hesaplar. |
| /api/v1/storage/upload-url | GET | Güvenli görsel yükleme için geçici URL (Presigned URL) döndürür. |

## **7\. Non-Functional Requirements (NFR)**

* **Güvenlik:** Apple Sign-In zorunludur. Tüm API trafiği HTTPS/TLS 1.3 ile korunacaktır.  
* **Performans:** Uygulama soğuk açılış (Cold Launch) süresi \< 1.5 saniye olmalıdır.  
* **Maliyet:** PDF üretimi tamamen istemci tarafında yapılarak sunucu maliyetleri minimize edilecektir.