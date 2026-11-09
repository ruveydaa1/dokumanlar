# Kullanılan Teknolojiler

SPM uygulamasının geliştirilmesinde backend, frontend, veritabanı yönetimi, mesajlaşma, zamanlanmış görev yönetimi ve güvenlik analizi alanlarında farklı teknolojiler kullanılmıştır. Her teknoloji sistemin belirli bir sorumluluğunu üstlenerek uygulamanın modüler ve yönetilebilir bir yapıda çalışmasına katkı sağlamaktadır.

### 1. C# ve .NET 10

Uygulamanın backend tarafı C# programlama dili ve .NET 10 platformu kullanılarak geliştirilmiştir.

.NET altyapısı; HTTP API'lerinin oluşturulması, servisler arası iletişimin sağlanması, iş mantığının yürütülmesi, bağımlılıkların yönetilmesi ve veritabanı işlemlerinin gerçekleştirilmesi amacıyla kullanılmaktadır.

Backend tarafında kullanıcı yönetimi, asset işlemleri, tarama süreçleri, tarama sonuçlarının işlenmesi ve zamanlanmış görevlerin yönetimi bu altyapı üzerinde gerçekleştirilmektedir.

### 2. Next.js

Uygulamanın frontend tarafında Next.js kullanılmıştır. Next.js, React tabanlı bir web geliştirme framework'üdür.

Kullanıcıların sistemle etkileşim kurduğu arayüz bu teknoloji kullanılarak geliştirilmiştir. Kullanıcı girişi, dashboard, kullanıcı yönetimi, profil işlemleri, asset listesi, asset detayları, tarama başlatma ve güvenlik bulgularının görüntülenmesi frontend tarafında gerçekleştirilmektedir.

Frontend, backend tarafından sunulan API endpoint'leri üzerinden veri alışverişi yapmaktadır.

### 3. PostgreSQL

Uygulamanın verilerinin saklanması için PostgreSQL ilişkisel veritabanı yönetim sistemi kullanılmıştır.

Kullanıcı bilgileri, roller, asset kayıtları, tarama geçmişi, zamanlanmış taramalar ve güvenlik bulguları PostgreSQL üzerinde tutulmaktadır.

İlişkisel veritabanı yapısı sayesinde kullanıcılar, asset'ler, ScanRun kayıtları, ScheduledScan kayıtları ve Finding kayıtları arasında gerekli ilişkiler kurulmaktadır.

### 4. Entity Framework Core

Entity Framework Core, backend tarafında veritabanı işlemlerini yönetmek amacıyla kullanılan ORM (Object Relational Mapper) teknolojisidir.

C# sınıflarının veritabanı tablolarıyla eşleştirilmesini ve veritabanı işlemlerinin C# üzerinden gerçekleştirilmesini sağlar.

Uygulamada Entity Framework Core aracılığıyla:

* Entity sınıfları veritabanı tablolarıyla eşleştirilir.
* Kayıt ekleme, güncelleme, silme ve sorgulama işlemleri gerçekleştirilir.
* Tablolar arasındaki ilişkiler tanımlanır.
* Migration işlemleriyle veritabanı şeması yönetilir.

### 5. Apache Kafka

Apache Kafka, uygulamanın tarama isteklerinin asenkron olarak iletilmesi amacıyla kullanılmaktadır.

Kullanıcı bir tarama başlattığında backend tarafından oluşturulan tarama isteği Kafka üzerindeki ilgili topic'e gönderilir. ScannerService bu mesajı consumer aracılığıyla tüketerek tarama sürecini başlatır.

Kafka sayesinde tarama işlemlerinin backend içerisindeki HTTP isteğine bağlı olarak doğrudan ve senkron şekilde yürütülmesi yerine mesajlaşma tabanlı bir yapı kullanılır.

Uygulamada kullanılan temel topic'ler şunlardır:

* `asset-scan`: Tarama isteklerinin iletilmesi.
* `asset-scan-progress`: Tarama ilerleme bilgilerinin iletilmesi.
* `asset-scan-result`: Tarama sonuç mesajları için tanımlanan topic.

Tarama sonuçlarının UserManagement Backend'e iletilmesi ise mevcut yapıda ScannerService içerisindeki HTTP tabanlı sonuç gönderim mekanizması üzerinden gerçekleştirilmektedir.

### 6. Hangfire

Hangfire, uygulamada zamanlanmış tarama işlemlerinin yönetilmesi amacıyla kullanılmaktadır.

Kullanıcı tarafından belirlenen tarih ve saat bilgisine göre tarama işleminin otomatik olarak başlatılmasını sağlar.

Zamanlanmış taramalara ait bilgiler PostgreSQL üzerinde saklanırken Hangfire, zamanı gelen görevleri arka planda çalıştırır.

Uygulamada ScheduledScanJob sınıfı üzerinden ilgili tarama işlemi başlatılır. Böylece kullanıcının taramayı manuel olarak başlatmasına gerek kalmadan belirlenen zamanda tarama gerçekleştirilebilir.

### 7. Docker

Docker, uygulamada özellikle Kafka ve Trivy gibi bileşenlerin çalıştırılması amacıyla kullanılmaktadır.

Kafka, Docker container içerisinde çalıştırılarak mesajlaşma altyapısı sağlanmaktadır.

Trivy tarama işlemi ise Docker üzerinden `aquasec/trivy` imajı kullanılarak gerçekleştirilir. ScannerService tarafından TrivyService'e gönderilen istek üzerine TrivyService, Docker komutunu çalıştırarak tarama işlemini container içerisinde gerçekleştirir.

Bu yaklaşım sayesinde Trivy'nin doğrudan işletim sistemine kurulmasına gerek kalmadan Docker imajı üzerinden çalıştırılması sağlanır.

### 8. Semgrep

Semgrep, kaynak kod içerisindeki güvenlik problemlerini tespit etmek amacıyla kullanılan statik analiz aracıdır.

SPM uygulamasında bağımsız bir SemgrepService üzerinden kullanılmaktadır.

Tarama isteği ilgili servise iletildiğinde Semgrep çalıştırılarak kaynak kod analiz edilir ve tespit edilen güvenlik bulguları sonuç formatına dönüştürülür.

### 9. Trivy

Trivy, yazılım projelerindeki bağımlılıkları ve bilinen güvenlik açıklarını tespit etmek amacıyla kullanılan güvenlik tarama aracıdır.

SPM uygulamasında bağımsız bir TrivyService üzerinden çalıştırılmaktadır.

GitHub deposu üzerinde gerçekleştirilen tarama sonucunda tespit edilen zafiyetler JSON formatında elde edilir. Bu sonuçlar uygulamanın ortak güvenlik bulgusu yapısına dönüştürülerek diğer tarama aracı olan Semgrep ile aynı platform üzerinden takip edilebilir.

### 10. JWT (JSON Web Token)

JWT, kullanıcı kimlik doğrulama ve yetkilendirme süreçlerinde kullanılmaktadır.

Kullanıcı başarılı şekilde giriş yaptığında backend tarafından oluşturulan token frontend tarafına iletilir. Frontend, korumalı API isteklerinde bu token'ı kullanır.

Backend tarafında token doğrulaması gerçekleştirilerek kullanıcının kimliği ve rol bilgisi kontrol edilir. Böylece kullanıcıların rollerine göre erişim yetkileri belirlenir.

### 11. SMTP

SMTP, uygulamada e-posta gönderim işlemlerinde kullanılmaktadır.

Özellikle şifremi unuttum sürecinde kullanıcının e-posta adresine parola sıfırlama bağlantısı gönderilmesini sağlar.

Bu sayede kullanıcılar parola sıfırlama işlemlerini e-posta üzerinden gerçekleştirebilir.
