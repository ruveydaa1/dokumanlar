# Veritabanı Tasarımı

SPM uygulamasında kullanıcı bilgileri, yetkilendirme işlemleri, yönetilen projeler, güvenlik taramaları ve tarama sonuçlarının saklanması için PostgreSQL veritabanı kullanılmaktadır. Veritabanı işlemleri Entity Framework Core aracılığıyla gerçekleştirilmektedir. Uygulama içerisinde veritabanı tablolarını ve bu tablolar arasındaki ilişkileri tanımlamak için `AppDbContext` sınıfı kullanılmaktadır.

Veritabanında uygulamanın temel işleyişini sağlayan yedi tablo bulunmaktadır: `users`, `roles`, `user_roles`, `assets`, `scan_runs`, `findings` ve `scheduled_scans`.

### 7.1. Users Tablosu

`users` tablosu, sisteme kayıtlı kullanıcıların bilgilerini saklamaktadır. Kullanıcının adı, soyadı, e-posta adresi, şifre özeti ve hesap durumu bu tabloda tutulmaktadır. Ayrıca kullanıcı oluşturulma ve güncellenme tarihleri de kaydedilmektedir.

Şifre sıfırlama işlemlerinde kullanılmak üzere `password_reset_token` ve `password_reset_token_expires_at` alanları bulunmaktadır. Bu alanlar, şifre yenileme bağlantısının doğrulanması ve geçerlilik süresinin kontrol edilmesi için kullanılmaktadır.

### 7.2. Roles ve User\_Roles Tabloları

`roles` tablosunda sistemde tanımlı kullanıcı rolleri tutulmaktadır. `user_roles` tablosu ise kullanıcılar ile roller arasındaki ilişkiyi sağlamaktadır.

Bu iki tablo birlikte kullanılarak bir kullanıcının birden fazla role sahip olması ve aynı rolün birden fazla kullanıcıya atanabilmesi desteklenmektedir. `user_roles` tablosundaki `user_id` alanı `users` tablosuna, `role_id` alanı ise `roles` tablosuna bağlıdır.

### 7.3. Assets Tablosu

`assets` tablosu, sistem üzerinden güvenlik taraması gerçekleştirilebilen projelerin ve kaynakların bilgilerini saklamaktadır.

Tabloda asset adı, türü, kaynak adresi, açıklaması ve aktiflik durumu gibi bilgiler bulunmaktadır. Ayrıca kayıt oluşturulma ve güncellenme tarihleri de tutulmaktadır.

Tarama işlemleri bu tabloda kayıtlı asset'ler üzerinden gerçekleştirilmektedir.

### 7.4. Scan\_Runs Tablosu

`scan_runs` tablosu, sistemde gerçekleştirilen tarama işlemlerinin kayıtlarını tutmaktadır.

Her tarama kaydında ilgili asset, kullanılan tarayıcı, taramanın kaynak adresi, işlem durumu, başlangıç ve tamamlanma tarihleri ile varsa hata bilgisi saklanmaktadır. `started_by_user_id` alanı sayesinde taramayı başlatan kullanıcı da ilişkilendirilebilmektedir.

Bu tablo, gerçekleştirilen taramaların geçmişinin takip edilmesini sağlamaktadır.

### 7.5. Findings Tablosu

`findings` tablosu, güvenlik taramaları sonucunda tespit edilen bulguların saklandığı tablodur.

Bulguların hangi asset ve tarama çalıştırmasına ait olduğu `asset_id` ve `scan_run_id` alanlarıyla belirlenmektedir.

Tabloda bulgunun kimliği, dosya yolu, satır ve sütun bilgileri, açıklaması, kategorisi, önem seviyesi, güven seviyesi, etkisi ve güvenlik standartlarıyla ilişkili bilgileri tutulmaktadır. Ayrıca CWE, OWASP, zafiyet sınıfı ve referans bilgileri de bulunmaktadır.

Bağımlılık taramalarında kullanılabilecek paket adı, kurulu sürüm, düzeltilmiş sürüm, zafiyet kimliği ve kimlik türü gibi alanlar da bu tabloda yer almaktadır. Böylece farklı tarayıcılardan gelen bulgular ortak bir veritabanı yapısında saklanabilmektedir.

### 7.6. Scheduled\_Scans Tablosu

`scheduled_scans` tablosu, ileri bir tarih ve saate planlanan tarama işlemlerinin kayıtlarını tutmaktadır.

Tabloda taramanın hangi asset için oluşturulduğu, hangi tarayıcının kullanılacağı, planlanan tarih ve saat, işlem durumu ve işlemi oluşturan kullanıcı bilgileri bulunmaktadır. Ayrıca taramanın başlama ve tamamlanma tarihleri ile varsa hata bilgisi de saklanmaktadır.

`scan_run_id` alanı, zamanlanmış tarama gerçekleştirildiğinde oluşan `scan_runs` kaydıyla ilişki kurulmasını sağlar. Bu alanın boş bırakılabilmesi, tarama henüz çalıştırılmadan önce zamanlanmış tarama kaydının oluşturulabilmesini mümkün kılmaktadır.

### 7.7. Tablolar Arasındaki İlişkiler

Veritabanındaki tablolar birbirleriyle foreign key (yabancı anahtar) ilişkileri üzerinden bağlantılıdır.

* `findings` tablosu hem `assets` hem de `scan_runs` tablolarına bağlıdır.
* `scan_runs` tablosu `assets` tablosuna ve taramayı başlatan kullanıcıyı belirtmek için `users` tablosuna bağlıdır.
* `scheduled_scans` tablosu `assets`, `scan_runs` ve `users` tablolarıyla ilişkilidir.
* `user_roles` tablosu, `users` ve `roles` tabloları arasındaki bağlantıyı sağlar.

Bu ilişkiler sayesinde her güvenlik bulgusunun hangi projeye ve hangi tarama işlemine ait olduğu takip edilebilmekte, tarama geçmişi ve kullanıcı işlemleri birbiriyle ilişkilendirilebilmektedir.

### 7.8. Entity Framework Core ve AppDbContext

Entity Framework Core, uygulamanın C# sınıfları üzerinden PostgreSQL veritabanıyla iletişim kurmasını sağlamaktadır.

`AppDbContext` sınıfı, uygulamadaki entity sınıflarını veritabanı tablolarıyla eşleştirmekte ve veritabanı işlemlerinin yönetilmesinde kullanılmaktadır.

Entity Framework Core migration yapısı sayesinde veritabanı şemasındaki değişiklikler takip edilmekte ve gerekli tablo güncellemeleri gerçekleştirilmektedir. `__EFMigrationsHistory` tablosu da uygulanan migration kayıtlarını ve kullanılan Entity Framework Core sürümünü saklamaktadır.
