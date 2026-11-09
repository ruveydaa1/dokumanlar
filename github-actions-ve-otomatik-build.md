# GitHub Actions ve Otomatik Build

### 1. GitHub Actions Nedir?

GitHub Actions, yazılım projelerinde belirli işlemlerin otomatik olarak çalıştırılmasını sağlayan bir otomasyon platformudur. Kodun derlenmesi, testlerin çalıştırılması ve yazılım geliştirme süreçlerindeki farklı görevlerin otomatikleştirilmesi için kullanılabilir.

SPM projesinde GitHub Actions, backend kodunun otomatik olarak derlenmesini kontrol etmek amacıyla yapılandırılmıştır. Böylece belirlenen koşullarda GitHub üzerinde bir Pull Request açıldığında kodun build işlemi otomatik olarak çalıştırılır.

### 2. Otomatik Build Nedir?

Build, kaynak kodun derlenerek uygulamanın oluşturulabilir durumda olup olmadığının kontrol edilmesini sağlayan işlemdir.

.NET projelerinde build işlemi, kodun derlenmesi sırasında oluşabilecek hataların tespit edilmesine yardımcı olur. Örneğin, hatalı bir C# ifadesi veya çözümlenemeyen bir bağımlılık build işleminin başarısız olmasına neden olabilir.

Otomatik build sayesinde bu kontrolün her seferinde geliştirici tarafından elle başlatılması gerekmez. GitHub Actions, workflow içerisinde tanımlanan koşullar gerçekleştiğinde build işlemini çalıştırır ve sonucu GitHub üzerinde gösterir.

**Önemli:** Build işleminin başarılı olması, uygulamadaki bütün işlevlerin doğru çalıştığını veya bütün güvenlik açıklarının giderildiğini kanıtlamaz. Build, derleme ve ilgili yapılandırma adımlarının başarılı olup olmadığını kontrol eder.

### 3. Workflow Dosyası

GitHub Actions işlemleri, depodaki `.github/workflows` dizininde bulunan YAML dosyaları üzerinden tanımlanır.

SPM projesinde kullanılan workflow dosyası:

```
.github/workflows/dotnet.yml
```

Bu dosyada workflow'un hangi olaylarda çalışacağı ve hangi adımları uygulayacağı belirtilir.

Projede kullanılan yapılandırmada .NET 10 ortamı hazırlanır ve backend için restore ile build adımları çalıştırılır.

Workflow'un temel görevleri şunlardır:

1. Kod deposunun GitHub Actions ortamına alınması.
2. Gerekli .NET SDK sürümünün kurulması.
3. Projenin bağımlılıklarının geri yüklenmesi.
4. Backend projesinin derlenmesi.
5. İşlem sonucunun GitHub üzerinde raporlanması.

#### .NET SDK Kurulumu

Workflow içerisinde `actions/setup-dotnet@v4` kullanılarak .NET SDK ortamı hazırlanır ve `10.0.x` sürüm ailesi hedeflenir.

#### Restore İşlemi

Restore, projenin ihtiyaç duyduğu NuGet paketlerinin geri yüklenmesini sağlar. Böylece build sırasında gerekli bağımlılıklar kullanılabilir hâle gelir.

#### Build İşlemi

Build adımı, projeyi derler ve derleme hatalarının olup olmadığını kontrol eder. İşlem başarısız olursa GitHub Actions kontrolü başarısız olarak işaretlenir.

Projede ayrı bir test projesi bulunmadığı için workflow'dan `dotnet test` adımı çıkarılmıştır. Bu nedenle mevcut otomasyon, otomatik test çalıştırma değil, restore ve build kontrolü gerçekleştirmektedir.

### 4. Pull Request Nedir?

Pull Request (PR), bir branch üzerinde yapılan değişikliklerin başka bir branch'e alınması için açılan inceleme talebidir.

Örneğin, geliştirici değişikliklerini `github-actions` isimli branch üzerinde hazırlayabilir. Değişikliklerin doğrudan `master` branch'ine aktarılması yerine bir Pull Request açılır. Bu PR üzerinden değişiklikler incelenebilir ve gerekli kontrollerin sonuçları değerlendirilebilir.

Pull Request kullanımı şu avantajları sağlar:

* Değişikliklerin ana branch'e alınmadan önce incelenmesine yardımcı olur.
* Build kontrolünün sonuçlarının değerlendirilmesini sağlar.
* Ekip içindeki kod inceleme sürecini destekler.
* Hatalı değişikliklerin ana branch'e doğrudan aktarılmasını sınırlandırabilir.

Pull Request açılması, değişikliklerin otomatik olarak birleştirildiği anlamına gelmez. Birleştirme işlemi, depo kurallarına ve kullanılan merge ayarlarına bağlıdır.

### 5. Branch Koruması ve Ruleset

Branch, Git deposundaki değişikliklerin birbirinden ayrı geliştirme çizgileri üzerinde yürütülmesini sağlar.

SPM projesinde `master` branch'inin kontrolsüz değişikliklerden korunması için GitHub üzerinde bir ruleset yapılandırılmıştır.

Bu kurallar kapsamında:

* `master` branch'ine değişikliklerin Pull Request üzerinden aktarılması istenir.
* `.NET / build (pull_request)` kontrolünün başarılı olması zorunlu tutulur.
* Gerekli kontroller başarılı olmadan Pull Request'in birleştirilmesi engellenir.

Bu yapı, ana branch'e aktarılacak değişikliklerin belirlenen kontrol sürecinden geçmesine yardımcı olur.

### 6. Otomatik Build ve Otomatik Merge Arasındaki Fark

Otomatik build ile otomatik merge birbirinden farklı işlemlerdir.

**Otomatik build**, kodun derlenmesini ve yapılandırılmış build adımlarının başarılı olup olmadığının kontrol edilmesini sağlar.

**Otomatik merge**, belirlenen koşullar karşılandığında Pull Request'in hedef branch ile birleştirilmesini otomatikleştirebilir.

SPM projesinde Pull Request için gerekli build kontrolü yapılandırılmış, ayrıca uygun koşullar sağlandığında kullanılmak üzere squash yöntemiyle auto-merge etkinleştirilmiştir.

Burada önemli olan, auto-merge özelliğinin tek başına her Pull Request'in anında birleştirileceği anlamına gelmemesidir. Gerekli kontrollerin başarılı olması ve depo kurallarının karşılanması gerekir.

#### Squash Merge Nedir?

Squash merge, Pull Request içerisindeki birden fazla commit'in hedef branch'e tek bir commit olarak aktarılmasını sağlar.

Bu yöntem, ana branch üzerindeki commit geçmişinin daha düzenli tutulmasına yardımcı olabilir.

### 7. Kod GitHub'a Gönderildiğinde Ne Olur?

Projede otomatik build sürecinin genel akışı şu şekildedir:

1. Geliştirici kod değişikliklerini bir branch üzerinde hazırlar.
2. Değişiklikler GitHub'a push edilir.
3. Değişiklikler için hedef branch'e yönelik bir Pull Request açılır.
4. Pull Request olayı, workflow'un çalışmasını tetikler.
5. GitHub Actions .NET ortamını hazırlar.
6. Restore ve build adımları çalıştırılır.
7. Build başarılı veya başarısız olarak raporlanır.
8. Gerekli kontroller başarılıysa ve diğer koşullar da karşılanıyorsa Pull Request birleştirilebilir.

Bu süreçte kodun yerel bilgisayarda daha önce build alınmış olması, GitHub Actions üzerindeki kontrolü gereksiz hâle getirmez. GitHub Actions, kodun GitHub'daki sürümünü kendi çalışma ortamında yeniden kontrol eder.

### 8. GitHub Actions Kullanmanın Avantajları

SPM projesinde GitHub Actions kullanımının başlıca faydaları şunlardır:

* Derleme hatalarının Pull Request aşamasında fark edilmesini sağlar.
* Manuel build kontrolü ihtiyacını azaltır.
* Ana branch'e aktarılacak kod için ortak bir kontrol standardı oluşturur.
* Kod değişikliklerinin daha kontrollü biçimde birleştirilmesine yardımcı olur.
* Geliştirme sürecinin izlenebilirliğini artırır.

Bununla birlikte, mevcut workflow'un yalnızca restore ve build adımlarını içerdiği unutulmamalıdır. Otomatik test, kod kalitesi analizi veya güvenlik taraması gibi işlemler için ayrıca adımlar eklenmesi gerekir.

### 9. Sonuç

SPM projesinde GitHub Actions, backend kodunun Pull Request aşamasında otomatik olarak derlenmesini sağlamak amacıyla kullanılmıştır. .NET SDK kurulumu, restore ve build adımları workflow üzerinden yürütülmektedir.

`master` branch'i için uygulanan Pull Request ve başarılı build koşulları, kod değişikliklerinin ana branch'e aktarılmadan önce kontrol edilmesine yardımcı olur. Auto-merge ve squash merge ayarları da bu sürecin uygun koşullar altında daha düzenli yürütülmesini destekler.
