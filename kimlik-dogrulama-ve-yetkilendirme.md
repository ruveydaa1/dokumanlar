# Kimlik Doğrulama ve Yetkilendirme

### 1. Kimlik Doğrulama ve Yetkilendirme Nedir?

Kimlik doğrulama (Authentication), sisteme erişmek isteyen kullanıcının kim olduğunu doğrulama işlemidir.

Yetkilendirme (Authorization) ise kimliği doğrulanan kullanıcının hangi işlemleri gerçekleştirebileceğini ve hangi kaynaklara erişebileceğini belirler.

Bu iki kavram birbiriyle ilişkili olsa da farklı görevleri yerine getirir. Kullanıcının sisteme giriş yapabilmesi, uygulamadaki bütün işlemleri gerçekleştirebileceği anlamına gelmez.

SPM projesinde kimlik doğrulama ve yetkilendirme mekanizmaları, kullanıcı erişimini kontrol etmek ve uygulamadaki işlemlerin belirlenen yetki sınırları içerisinde gerçekleştirilmesini sağlamak amacıyla kullanılmaktadır.

### 2. Login İşlemi

Login, kullanıcının sisteme giriş yaptığı işlemdir.

SPM projesinde kullanıcı, frontend üzerindeki giriş ekranına e-posta ve parola bilgilerini girerek backend'e bir istek gönderir. Backend, gönderilen bilgileri veritabanındaki kullanıcı bilgileriyle karşılaştırarak doğrular. Bilgiler geçerliyse kullanıcıya ait kimlik doğrulama bilgileri oluşturularak yanıt döndürülür.

Başarılı giriş sonrasında frontend, oturum durumunu yönetmek ve korunan API uç noktalarına erişmek için aldığı JWT token'ı kullanır.

Genel işlem sırası şöyledir:

1. Kullanıcı giriş bilgilerini frontend üzerindeki giriş ekranına girer.
2. Frontend, `/api/auth/login` uç noktasına HTTP isteği gönderir.
3. Backend, kullanıcı bilgilerini doğrular.
4. Doğrulama başarılıysa kullanıcıya ait JWT oluşturulur ve istemciye döndürülür.
5. Frontend, alınan token ve kullanıcı bilgileriyle oturum durumunu yönetir.
6. Kullanıcı korunan bir API'ye istek gönderdiğinde token kullanılır.

### 3. JWT Nedir?

JWT (JSON Web Token), taraflar arasında belirli bilgilerin taşınmasını sağlayan bir token formatıdır. Web uygulamalarında kullanıcının kimliğine ve erişimle ilgili bazı bilgilere ilişkin verileri taşımak için kullanılabilir.

JWT genellikle üç bölümden oluşur:

* **Header:** Token'ın türü ve kullanılan imza algoritması gibi bilgileri içerir.
* **Payload:** Kullanıcı kimliği, rol veya token'ın geçerlilik süresi gibi claim'leri içerebilir.
* **Signature:** Token'ın imzasını doğrulamak için kullanılır.

JWT'nin içeriği, imzalı olması nedeniyle otomatik olarak gizli hâle gelmez. Bu nedenle token içerisine parola gibi gizli bilgiler konulmamalıdır.

SPM projesinde JWT, kullanıcının başarılı girişinden sonra oturum ve API erişim süreçlerinde kullanılmaktadır. Backend, gelen token'ın geçerliliğini kontrol ederek kullanıcının kimliğini doğrular.

### 4. Token Nasıl Kullanılır?

Kullanıcı giriş yaptıktan sonra frontend, aldığı token'ı sonraki korunan API isteklerinde kullanır.

HTTP isteklerinde token genellikle `Authorization` başlığı üzerinden gönderilir:

```
Authorization: Bearer <token>
```

Buradaki `Bearer`, istekte token kullanıldığını belirten şemadır. Backend, gelen token'ın geçerliliğini kontrol eder ve doğrulama başarılı olduğunda isteği ilgili yetkilendirme kurallarına göre değerlendirir.

Token geçersizse veya süresi dolmuşsa istek reddedilebilir ve kullanıcının yeniden giriş yapması gerekebilir.

### 5. AuthProvider Yapısı

SPM projesinin frontend tarafında `AuthProvider` yapısı, kimlik doğrulama durumunun uygulama genelinde yönetilmesine yardımcı olur.

Bu yapı kapsamında token, kullanıcı bilgileri ve token'ın sona erme zamanı gibi oturum bilgileri tutulur.

Bu bilgilerin ortak bir yapı üzerinden yönetilmesi, farklı sayfaların kullanıcının oturum durumunu kontrol edebilmesini sağlar. Böylece uygulama içerisinde oturum açmış kullanıcının bilgilerine ihtiyaç duyan bileşenler bu yapıdan yararlanabilir.

Frontend'de oturum durumunun tutulması tek başına güvenlik için yeterli değildir. Backend'in de korunan API isteklerinde gerekli kimlik doğrulama ve yetkilendirme kontrollerini gerçekleştirmesi gerekir.

### 6. LocalStorage ve Oturum Bilgileri

Projede token ve bazı oturum bilgileri tarayıcının `localStorage` alanında saklanmaktadır.

`localStorage`, tarayıcıda anahtar-değer biçiminde veri saklamaya yarayan bir mekanizmadır. Sayfa yenilendiğinde saklanan bilgiler korunabilir.

Bununla birlikte, `localStorage` içinde saklanan token'lar JavaScript tarafından erişilebilir olduğundan XSS gibi saldırılar karşısında risk oluşturabilir. Bu nedenle token saklama yöntemi seçilirken uygulamanın güvenlik gereksinimleri dikkate alınmalıdır.

Token'ın tarayıcıda saklanması, sunucu tarafındaki doğrulama ve yetkilendirme kontrollerinin yerini tutmaz.

### 7. Kullanıcı Rolleri

SPM projesinde kullanıcıların erişim seviyelerini birbirinden ayırmak amacıyla aşağıdaki roller tanımlanmıştır:

**SUPERADMIN**

Sistemdeki en yüksek yetki seviyesini temsil eden roldür. Sistem genelindeki yönetim işlemleri için kullanılır.

**ADMIN**

Yönetim işlemlerini gerçekleştiren kullanıcıları temsil eder. Erişebileceği işlemler uygulamada tanımlanan yetki kurallarına bağlıdır.

**DEVELOPER**

Yazılım geliştirme süreçleriyle ilişkili kullanıcı rolüdür. Erişebileceği özellikler backend ve uygulama içerisinde tanımlanan yetki kurallarına göre belirlenir.

**USER**

Standart kullanıcı rolüdür. Erişimi, kendisine tanımlanan yetkilerle sınırlandırılır.

Rollerin tanımlanmış olması, her rolün otomatik olarak belirli işlemlere erişebileceği anlamına gelmez. Gerçek erişim kuralları backend tarafında uygulanmalıdır.

### 8. Kimlik Doğrulama ve Yetkilendirme Arasındaki Fark

Bir kullanıcının sisteme giriş yapması kimlik doğrulama sürecidir. Giriş yapan kullanıcının belirli bir işlemi yapmaya yetkili olup olmadığının kontrol edilmesi ise yetkilendirmedir.

Örneğin, bir kullanıcının geçerli bir JWT token'a sahip olması onun bütün kullanıcı kayıtlarını değiştirebileceği anlamına gelmez. Backend, ilgili işlem için kullanıcının rolünü ve erişim kurallarını ayrıca değerlendirmelidir.

Bu ayrım, uygulamada kullanıcı erişiminin kontrollü biçimde yönetilmesini sağlar.

### 9. Güvenlik Açısından Dikkat Edilmesi Gerekenler

Kimlik doğrulama ve yetkilendirme mekanizmalarının güvenli biçimde uygulanması için bazı noktalara dikkat edilmelidir:

* Parolalar düz metin olarak saklanmamalı, uygun parola hash mekanizmaları kullanılmalıdır.
* JWT imzası, geçerlilik süresi ve gerekli claim'ler backend tarafından doğrulanmalıdır.
* Kullanıcı yetkileri yalnızca frontend'de kontrol edilmemeli, backend tarafında da uygulanmalıdır.
* Token'lar ve gizli anahtarlar kaynak koduna açık şekilde yazılmamalıdır.
* Token saklama yöntemi, XSS gibi saldırı riskleri dikkate alınarak seçilmelidir.
* Korunan API uç noktalarına erişim kuralları açık ve tutarlı olmalıdır.

### 10. Sonuç

SPM projesinde kimlik doğrulama, kullanıcıların sisteme güvenli biçimde giriş yapmasını; yetkilendirme ise kullanıcıların erişebileceği işlemlerin kontrol edilmesini sağlar.

Login işlemi sonrasında kullanılan JWT, frontend tarafındaki `AuthProvider` yapısı ve backend tarafındaki erişim kontrolleri birlikte çalışarak oturum yönetimini destekler. Kullanıcı rolleri de uygulamadaki farklı erişim seviyelerinin düzenlenmesine yardımcı olur.
