# Bölüm 5  
# Sürekli Teslimat, Sürekli Dağıtım ve Deployment: Kavramsal Çerçeve

Önceki bölümlerde Git ile sürümlenen, Jenkins ile sürekli entegrasyon hattına alınan ve Docker ile konteynerleştirilen `user-service` adlı Java Spring Boot REST API projesi temel alınmıştır. Bu noktada kaynak kodun derlenmesi, test edilmesi ve imaj üretilmesi mümkün hâle gelmiştir. Ancak yazılımın gerçek değer üretmeye başladığı yer, build çıktısının bir sunucuda çalıştırılmasıdır. Bu nedenle deployment konusu, CI hattının doğal devamı olarak ele alınmalıdır.

Sürekli entegrasyon hattı kurulduğunda yazılımın belirli kalite eşiklerinden geçtiği doğrulanır; fakat bu doğrulama, uygulamanın staging veya production ortamında güvenilir biçimde çalışacağı anlamına gelmez. Farklı ortamlar, farklı yapılandırmalar, farklı risk düzeyleri ve farklı operasyonel beklentiler doğurur. Sürekli teslimat ve sürekli dağıtım kavramları da bu ayrımın üzerine kuruludur.

## 5.1 CI Sonrasında Ortaya Çıkan Yeni Problem

Sürekli entegrasyonun temel işlevi, ortak koda eklenen değişikliklerin hızlı biçimde doğrulanmasıdır. `user-service` gibi bir projede bu doğrulama genellikle şu çıktıları üretir:

- kaynak kodun derlenebilir olduğu bilgisi
- testlerin başarılı geçtiği bilgisi
- paketlenmiş artifact’in üretildiği bilgisi
- Docker imajının oluşturulabildiği bilgisi

Bu aşamadan sonra yeni bir problem ortaya çıkar. Uygulamanın yalnızca üretilebilmesi yeterli değildir; güvenilir, izlenebilir ve tekrarlanabilir biçimde hedef ortama aktarılması gerekir. Bu gereksinim ortaya çıkmadığında CI hattı eksik kalır. Çünkü üretim sistemi yalnızca kaynak koddan veya imajdan oluşmaz; aynı zamanda yapılandırma, ağ erişimi, veritabanı bağlantısı, sağlık doğrulaması, log yönetimi ve geri dönüş planı da gerektirir.

`user-service` için aşağıdaki soru kümesi deployment disiplininin neden gerekli olduğunu açık biçimde gösterir:

- Hangi imaj staging ortamına alınacaktır?
- Production ortamına geçiş otomatik mi olacak, onay mı gerektirecektir?
- Hedef sunucu imajı nereden çekecektir?
- Aynı imaj farklı ortamlarda nasıl farklı yapılandırmalarla çalıştırılacaktır?
- Hatalı sürümün geri alınması hangi mekanizma ile yapılacaktır?

Bu noktadan sonra mesele yalnızca yazılım üretimi değil, yazılımın işletime alınmasıdır.

## 5.2 Deployment Kavramı

Deployment, yazılımın belirli bir hedef ortamda aktif hizmet üreten çalışan sisteme dönüştürülmesi sürecidir. Bu süreç yalnızca dosya kopyalama veya container başlatma işlemi değildir. Deployment; sürüm seçimi, yapılandırmanın yüklenmesi, süreçlerin başlatılması, doğrulama yapılması, izleme ve gerektiğinde geri dönüş mekanizmalarının işletilmesini birlikte kapsar.

### 5.2.1 Build, Release ve Deploy Ayrımı

Teslimat hattının sağlıklı kurulabilmesi için `build`, `release` ve `deploy` kavramlarının ayrılması gerekir.

- `Build`, kaynak koddan çalıştırılabilir çıktı üretme aşamasıdır.
- `Release`, belirli bir sürümün dağıtıma aday veya dağıtıma hazır sürüm olarak işaretlenmesidir.
- `Deploy`, release edilmiş sürümün belirli bir ortamda fiilen çalıştırılmasıdır.

Bu ayrım yapılmadığında sık karşılaşılan hata, build alınmış olmasını doğrudan yayına hazır olmakla eşitlemektir. Oysa Docker imajının üretilmiş olması, tek başına staging veya production ortamında doğru biçimde çalışacağı anlamına gelmez.

### 5.2.2 Artifact ile Çalışan Sistem Arasındaki Fark

Artifact, build sürecinin ürettiği paketlenmiş teslimat birimidir. `user-service` için bu bir `jar` dosyası veya Docker imajı olabilir. Çalışan sistem ise bu artifact’in belirli bir ortamda, belirli yapılandırmalarla ve belirli bağımlılıklarla etkinleştirilmiş hâlidir.

Bu iki düzey arasındaki fark uygulama yaşam döngüsü açısından kritiktir:

- artifact değişmez bir birimdir
- çalışan sistem çevresel koşullara bağlıdır
- deployment süreci bu iki dünya arasındaki geçişi yönetir

Dolayısıyla deployment, artifact üretiminin son satırı değil, ayrı bir mühendislik katmanıdır.

## 5.3 Continuous Delivery ve Continuous Deployment

Sürekli teslimat ve sürekli dağıtım, CI sonrasında başlayan otomasyonun kapsamını tanımlar. Her iki yaklaşım da yazılımın hızlı ve güvenilir biçimde yayımlanmasını amaçlar; ancak karar noktaları farklıdır.

### 5.3.1 Continuous Delivery

Continuous Delivery modelinde sistem her başarılı build sonrasında dağıtıma hazır durumda tutulur. Buna rağmen production geçişi genellikle insan onayı ile yapılır. Teknik olarak süreç uçtan uca hazırlanmıştır; fakat son karar manuel bir kontrol noktasına bırakılır.

Bu yaklaşım aşağıdaki koşullarda tercih edilir:

- üretim geçişi için yönetsel veya operasyonel onay gerekiyorsa
- düzenleyici çerçeveler nedeniyle kayıtlı onay mekanizması aranıyorsa
- uygulamanın hata maliyeti yüksekse
- otomasyon güçlü olsa bile iş birimi değerlendirmesi korunmak isteniyorsa

### 5.3.2 Continuous Deployment

Continuous Deployment modelinde doğrulama hattını başarıyla geçen sürüm, ek bir insan müdahalesi olmadan production ortamına alınır. Bu modelin çalışabilmesi için test kapsamı, gözlemlenebilirlik, sağlık kontrolü ve rollback yeteneği yeterli olgunluğa ulaşmış olmalıdır.

Bu yaklaşım şu durumlarda anlamlıdır:

- değişiklikler küçük ve sık ise
- otomatik test katmanları yüksek güven veriyorsa
- hata durumunda süratli geri dönüş mümkündür
- operasyonel izleme altyapısı yeterince güçlüdür

### 5.3.3 İnsan Onayı ve Otomatik Geçiş Mantığı

İki model arasındaki temel ayrım, teknik olanaktan çok karar mekanizmasının konumudur.

- Continuous Delivery, production öncesinde onay barındırır.
- Continuous Deployment, production geçişini de otomasyon kapsamına alır.

Bu nedenle bir kurumun CI/CD kullandığını söylemesi, tek başına production’a otomatik dağıtım yaptığı anlamına gelmez.

## 5.4 Ortam Kavramı

Deployment süreci farklı ortamlar arasındaki geçişe dayanır. Aynı kod tabanı her ortamda aynı amaca hizmet etmez. Ortamlar; veri niteliği, yapılandırma ayrıntısı, erişim düzeyi ve hata toleransı bakımından birbirinden ayrılır.

### 5.4.1 Local

Yerel geliştirme ortamıdır. Geliştirici hızlı geri bildirim almak için uygulamayı burada çalıştırır. Hata maliyeti düşüktür, yapılandırmalar basitleştirilebilir ve deneysel değişiklikler daha serbest yapılabilir.

### 5.4.2 Development

Takım içi ortak geliştirme alanıdır. Her kurumda ayrı bir development ortamı bulunmayabilir; ancak bulunduğu durumlarda entegrasyonun ilk paylaşılmış yüzeyi olarak görev yapar.

### 5.4.3 Test

Teknik doğrulama için ayrılmış ortamdır. Entegrasyon testleri, uçtan uca kontroller ve bazı otomatik kalite denetimleri burada çalıştırılır.

### 5.4.4 Staging

Production ortamına olabildiğince benzeyen ön yayın alanıdır. Yapısal benzerlik, burada yapılan doğrulamanın production davranışını öngörmesini sağlar.

### 5.4.5 Production

Gerçek kullanıcı trafiğinin geçtiği canlı ortamdır. Değişikliklerin etkisi doğrudan iş sonuçlarına yansır. Bu nedenle izleme, geri dönüş ve sürüm takibi burada kritik önemdedir.

### 5.4.6 Ortamlar Arası Yapılandırma Farkları

Aynı imaj, farklı ortamlarda farklı davranışlar gösterebilir. Bunun nedeni imajın değişmesi değil, yapılandırmanın değişmesidir. Ortamlar arasında genellikle şu alanlar farklılaşır:

- aktif profil
- veritabanı bağlantı bilgisi
- port veya ağ eşlemeleri
- log düzeyi
- dış servis erişimleri
- gizli bilgiler ve erişim anahtarları

Sağlıklı bir deployment yaklaşımı, uygulama imajını sabit tutarken yapılandırmayı ortam dışına taşır.

## 5.5 Deployment Hattının Temel Bileşenleri

Uçtan uca deployment akışı birbiriyle ilişkili birkaç bileşenden oluşur. `user-service` gibi bir servis için bu bileşenler aşağıdaki mantıkla açıklanabilir:

- Git deposu kaynak kodu taşır.
- Jenkins veya benzeri bir otomasyon sunucusu build ve doğrulama işlemlerini yürütür.
- Docker, teslimat birimini imaj olarak paketler.
- Registry, bu imajın merkezi ve güvenilir saklama alanını oluşturur.
- Hedef sunucu, ilgili sürümü registry’den çekerek çalıştırır.

Bu yapı sayesinde build ortamı ile çalışma ortamı birbirinden ayrılır. Jenkins imaj üretir; hedef sunucu ise aynı imajı güvenilir kaynaktan tüketir.

## 5.6 Docker Image Sürümleme ve İzlenebilirlik

Deployment süreçlerinde en kritik gereksinimlerden biri, çalışan sürümün açık biçimde tanımlanabilmesidir. Yalnızca `latest` etiketi ile çalışmak, hangi kodun hangi anda hangi ortamda çalıştığını belirsizleştirir.

Sağlıklı sürümleme yaklaşımı genellikle şu bileşenleri birlikte kullanır:

- uygulama sürümü
- build numarası
- commit SHA bilgisi

Bu yaklaşımın sağladığı başlıca yararlar şunlardır:

- sürümün kaynağı geriye doğru izlenebilir
- staging ve production arasında aynı imajın kullanıldığı doğrulanabilir
- rollback için açık referans oluşur
- denetim ve raporlama kolaylaşır

Hareketli etiketler kolaylık sağlayabilir; ancak esas dağıtım referansı sabit ve versiyonlu etiket olmalıdır.

## 5.7 Registry Kullanımının Gerekçesi

Registry, Docker imajlarının merkezi biçimde saklandığı ve dağıtıldığı yapıdır. Deployment hattında registry bulunmadığında build çıktısının sunuculara manuel yöntemlerle taşınması gerekir. Bu yaklaşım izlenebilirliği azaltır ve hata riskini artırır.

Registry kullanımı şu nedenlerle temel bir ihtiyaçtır:

- tek bir doğrulanmış imajın birden fazla ortama dağıtılmasını sağlar
- hedef sunucunun build yapmasını gereksiz kılar
- sürüm etiketlerinin merkezi olarak yönetilmesine imkân verir
- yetkilendirme ve erişim kontrolü sağlar

Bu nedenle modern deployment sürecinde registry, yalnızca bir depolama alanı değil, teslimat zincirinin güvenilir referans noktasıdır.

## 5.8 Yapılandırma Yönetimi ve Environment Variable Yaklaşımı

Bir uygulamanın aynı imaj ile farklı ortamlarda çalışabilmesi için yapılandırma ile ikili çıktı birbirinden ayrılmalıdır. Bu ayrım özellikle konteyner tabanlı dağıtımlarda temel ilkelerden biridir.

Yapılandırma yönetiminde öne çıkan unsurlar şunlardır:

- aktif Spring profili
- veritabanı bağlantı bilgileri
- log düzeyi ve log hedefi
- servis portu
- dış servis adresleri
- gizli bilgiler

Bu bilgilerin imaj içine gömülmesi, taşınabilirliği ve güvenliği zedeler. Yapılandırmanın environment variable veya benzeri dış kaynaklardan verilmesi ise aynı artifact’in farklı ortamlarda yeniden kullanılmasını mümkün kılar.

## 5.9 Doğrulama, Hata Yönetimi ve Geri Dönüş

Deployment işlemi yeni sürümün başlatılmasıyla tamamlanmış sayılmaz. Sürecin güvenilir olabilmesi için dağıtım sonrasında doğrulama yapılmalı, hata durumları sınıflandırılmalı ve geri dönüş yolu önceden tanımlanmalıdır.

Başlıca doğrulama katmanları şunlardır:

- health check
- log inceleme
- temel endpoint doğrulaması
- bağımlı servis bağlantılarının kontrolü

Başarısız deployment senaryoları ise çoğu zaman şu başlıklarda toplanır:

- container’ın hiç başlamaması
- port çakışması
- eksik veya hatalı environment variable
- hatalı build edilmiş imaj
- dış bağımlılıklara erişim problemi

Bu koşullarda rollback yalnızca operasyonel kolaylık değil, teslimat hattının zorunlu bir bileşenidir. Hangi sürümün geri alınacağı bilinmiyorsa deployment hattı eksik tasarlanmış demektir.

## 5.10 Yayın Stratejileri

Deployment yalnızca teknik komut dizisi değil, aynı zamanda yayın stratejisidir. Yeni sürümün hangi yöntemle devreye alınacağı; kesinti süresini, risk düzeyini ve geri dönüş hızını doğrudan etkiler.

### 5.10.1 Recreate

En basit yöntemdir. Eski sürüm kapatılır ve yeni sürüm onun yerine başlatılır. Tek sunuculu ve giriş düzeyi yapılarda öğretim amacıyla en anlaşılır modeldir; ancak kısa süreli kesinti üretir.

### 5.10.2 Rolling Update

Birden fazla uygulama örneği bulunan yapılarda eski örnekler kademeli biçimde yeni sürümle değiştirilir. Kesintiyi azaltır; fakat orkestrasyon ve trafik yönetimi gerektirir.

### 5.10.3 Blue-Green

Production’a eşdeğer iki ayrı ortam tutulur. Yeni sürüm pasif ortamda hazırlanır, ardından trafik yönü değiştirilir. Hızlı rollback avantajı sunar; ancak maliyetlidir.

### 5.10.4 Canary

Yeni sürüm önce sınırlı trafik üzerinde denenir. Gözlem sonuçları olumluysa dağıtım genişletilir. İleri düzey izleme ve yönlendirme kabiliyeti gerektirir.

Giriş düzeyinde kurulan teslimat hatlarında recreate yaklaşımı çoğu zaman başlangıç noktası olur; daha ileri stratejiler ise altyapı olgunluğu arttıkça anlam kazanır.

## 5.11 Yaygın Hatalı Uygulamalar

Deployment sürecinde teknik olarak çalışıyor görünen fakat uzun vadede kırılganlık üreten bazı uygulamalar vardır. Bunlar süreç kalitesini düşürür ve geri dönüş maliyetini yükseltir.

Yaygın hatalı uygulamalar şu başlıklarda toplanabilir:

- artifact’in sunucuya manuel olarak taşınması
- yalnızca `latest` etiketi ile sürüm yönetimi yapılması
- deployment sonrası health check uygulanmaması
- rollback planının tanımlanmamış olması
- production yapılandırmasının imaj içine gömülmesi
- doğrulanmamış imajın canlı ortama alınması

Bu hatalar, çoğu zaman otomasyon eksikliğinden değil, teslimat hattının kavramsal olarak eksik tasarlanmasından kaynaklanır.

## 5.12 Değerlendirme

Deployment süreci, DevOps pratiğinde build ile işletim arasında kurulan disiplinli köprüdür. Kodun derlenmesi, test edilmesi ve imaj üretimi bu köprünün gerekli parçalarıdır; fakat tek başına yeterli değildir. Sürekli teslimat ve sürekli dağıtım kavramları, bu çıktının kontrollü biçimde çalışan sisteme dönüştürülmesini hedefler.

Bu bölümde ele alınan kavramsal çerçeve; build, release ve deploy ayrımını, ortam mantığını, sürümleme disiplinini, registry kullanımını, yapılandırma yönetimini, doğrulama adımlarını ve rollback gereksinimini açıklamaktadır. Bir sonraki bölümde aynı çerçeve, `user-service` uygulaması üzerinden adım adım bir deployment örneğine dönüştürülecektir.
