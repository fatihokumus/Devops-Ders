# Bölüm 8 — Kubernetes Temelleri: Container Orchestration, Cluster Mimarisi ve Çekirdek Kavramlar

## 1. Tek Konteynerli ve Tek Sunuculu Yaklaşımın Sınırları

Önceki bölümlerde, user-service uygulamasını Docker konteynerinde çalıştırdık. Makine üzerinde aşağıdaki komutu çalıştırarak, uygulamayı izole bir ortamda başlattık:

```bash
docker run -d -p 8080:8080 registry.example.com/user-service:1.0.0
```

Bu komut, bir Docker konteynerini başlatır ve uygulamayı çalışır hâle getirir. Lokal geliştirme ortamında veya küçük ölçekli sistemlerde bu yaklaşım işe yarar. Ancak gerçek üretim ortamlarında birkaç ciddi sorunla karşılaşırız.

### Konteyner Yeniden Başlatma Sorunu

Üretim ortamında çalışan konteyner, çeşitli nedenlerle kapanabilir. Belki uygulama içindeki bir hata meydana geldi, belki sunucu kilitlendi, belki sistem yöneticisi makinayı yeniden başlattı. Konteyner kapandığında, uygulama artık erişilebilir değildir.

Eğer biz bunu manuel olarak yönetiyorsak ne olur? Birisi kontrol paneline bağlanarak konteynerinin durumunu kontrol etmeli, konteyner kapanmışsa yeniden başlatmalıdır. Bu, otomasyonun tersidir. Ayrıca, konteyner üçüncü kali başarısız olduysa ne yapılacak? Sistem devam mı etmeli, yoksa hata bildirmeli mi? Bu kararlar kime ait?

Üretim sistemleri, belirli bir sayıda başarısızlık sonrası otomatik olarak harekete geçmelidir. Eğer konteyner art arda beş kez başlatıldıktan sonra kapanıyorsa, bu bir sorun işareti olabilir. Sistem yapılandırıcısı bu durumu ele almalıdır.

### Ölçekleme Sorunu

user-service uygulaması başarılı olmuş, gelen talep arttığını varsayalım. Bir makine üzerinde çalışan tek konteyner, tüm istekleri işlemeye etkisiz hâle geldi. Yanıt süreleri arttı, hatta bazı istekler timeout'a uğradı.

Çözüm basit görünüyor: daha fazla konteyner çalıştırmak. Ancak nereye çalıştıracağız? Mevcut makine RAM ve CPU sınırlarına ulaştı. Başka bir makine satın almamız gerekir.

Eğer manuel yönetim yapıyorsak, yeni makineyi kurmalı, konteynerı o makineye yayınlamalı, trafiği yeni makineye dağıtmak için bir load balancer yapılandırmalıyız. Birkaç saat iş. İsteği arttığında bu işi tekrar tekrar mı yapacağız?

Üretim ortamlarında ölçekleme dinamik olmalıdır. Talep yükseldiğinde otomatik olarak konteyner sayısı artmalı, talep düştüğünde konteyner sayısı azalmalıdır. Bunu yapılandırmak günlük operasyon olmaktan ziyade, bir defa ayarlanması gereken politika olmalıdır.

### Servis Keşfi Sorunu

Şimdi, user-service'in yanında başka bir mikro hizmet, order-service oluşturduğumuzu varsayalım. order-service, user-service'i çağırması gerekir, örneğin siparişi oluşturan kullanıcının bilgilerini almak için.

Eğer manuel yönetim yapıyorsak, order-service'de user-service'in sabit adresini yazarız. Örneğin: `http://192.168.1.100:8080`. Ancak sorun, user-service makinesi başarısız olursa ne olacağı. IP adresi değişecek. user-service'in başka bir makineye taşınması gerekebilir. O zaman order-service'deki konfigürasyonu güncellemeliyiz.

Ayrıca, user-service'i ölçeklemesi durumunda ne olacak? Birkaç makineye yayılmış olacak. Order-service, bunlardan hangisine bağlanacak? Load balancer'ı elle yapılandıracak mıyız? Bu çok hata açısından risklidir.

Üretim ortamınız dinamik olduğunda (konteynerler dinamik olarak başlıyor ve kapanıyor), servis keşfi otomatik olmalıdır. Bir hizmet başladığında, diğer hizmetler otomatik olarak onu bulabilmelidir. Hizmetin adresi değişirse, diğerleri otomatik olarak yeni adresi bilmelidir.

### Dağıtık Yönetim İhtiyacı

Bu sorunları çözmek için, sistemi yönetmek için merkezi bir yönetim katmanı gerekir. Bu katman:

- Konteynerları nereye yerleştireceğini bilmeli
- Konteyner başarısız olursa otomatik olarak başlamalı
- Talep yükseldiğinde otomatik olarak ölçeklemelidir
- Ağ trafiğini konteynerler arasında dağıtmalı
- Hizmetleri otomatik olarak keşfetmeli
- Konfigürasyonu merkezi olarak yönetmeli

Basitçe, konteynerleri tek tek yönetmek değil, tümünü bir sistem olarak yönetmek gerekir.

---

## 2. Container Orchestration Problemi

### Konteyner Çalıştırmak ile Sistem Yönetmek Arasındaki Fark

"Konteyner çalıştırmak" basit bir görevdir. `docker run` komutu çalıştırırsınız, konteyner başlar. Bu, dişçiye gitmek gibi: belirli, sınırlı bir görev.

"Sistemi yönetmek" ise, tümüyle farklı bir görevdir. Sistemi yönetmek, konteynerlerin yaşam döngüsünü izlemek, başarısız olanları iyileştirmek, talebin yükselişi ve düşüşü ile uyum sağlamak, ağ bağlantılarını yönetmek anlamına gelir. Bu, bir şehri yönetmeğe benzer.

İlk görev (konteyner çalıştırmak) Docker'ın işidir. Docker size "bu konteynerı başlat" kapabilirliğini verir.

İkinci görev (sistemi yönetmek) Docker'ın yapamadığı şeydir. Docker, yalnızca tek makine üstünde çalışır. Konteyner başarısız olursa Docker onu yeniden başlatabilir. Ancak makine tamamen kapanırsa, Docker da kapanır. Makine arızalanırsa, konteyner başka bir makineye otomatik olarak taşınmaz.

### Otomatik Planlama İhtiyacı

"Planlama" (scheduling), bir konteynerı çalışmak için uygun bir makineyi seçmek anlamına gelir. Biz, `docker run` komutunu çalıştırdığımız makineye konteyner gider. Başka seçeneğimiz yoktur.

Ancak 100 makineniz varsa ne yaparsınız? Hangi makineye hangi konteynerı yerleştireceğinize kim karar verir? Elle yapılamaz. Sistem, otomatik olarak karar çıkarmak için bir mekanizmaya ihtiyaç duyar.

Planlama mekanizması:
- Her makinenin ne kadar kaynağı (CPU, RAM) olduğunu bilmeli
- Her konteynerın ne kadar kaynağa ihtiyacı olduğunu bilmeli
- Konteyneri, gerekli kaynaklara sahip bir makineye yerleştirmeli
- Makinelerin dengeli şekilde yüklenmesini sağlamalı

Bunun insan tarafından yapılması, ölçeklenebilir değildir.

### Sürekli Gözlem İhtiyacı

Sistem başladıktan sonra, sürüp gitmesi gerekir. Ancak az sonra neler olabilir:

- Bir konteyner hata verdi ve kapandı. Otomatik olarak yeniden başlamalı.
- Bir makine çöktü. O makinada çalışan konteynerler başka makinelere taşınmalı.
- Bir makine kaynakları yetersiz kaldı. İçindeki konteynerların bazıları başka makinelere taşınmalı.
- Bir konteyner aşırı kaynak tüketmeye başladı. Sınırlandırılmalı veya işlenmeli.

Bu olayları biri izlemeli ve harekete geçmeli. Eğer insan izlemiyorsa, sistem otomatik olarak izlemeli.

Container orchestration, bu problemleri çözen bir sistemi yönetim platformudur.

---

## 3. Kubernetes'in Genel Tanımı

### Kubernetes Nedir?

Kubernetes, konteyner uygulamalarını ölçülü bir şekilde çalıştırmak, yönetmek ve dağıtmak için açık kaynaklı bir platformdur. Google tarafından başlangıçta dahili olarak geliştirilen Borg adlı sistem'in fikirlerinden esinlenilerek oluşturulmuştur ve 2014'te açık kaynak hâline getirilmiştir.

Kubernetes, container orchestration sorunlarını doğrudan ele alır. Konteyner çalıştırmaktan ziyade, konteyner tabanlı uygulamaları yönetmeye odaklanır.

### Neyi Çözdüğü

Kubernetes aşağıdaki sorunları çözer:

1. **Otomatik yerleştirme (scheduling)**: Konteynerler, kaynaklara göre otomatik olarak uygun makinelere yerleştirilir.

2. **Kendi kendine iyileştirme (self-healing)**: Konteyner başarısız olursa otomatik olarak yeniden başlatılır. Node başarısız olursa, o node'daki konteynerler başka makinelere taşınır.

3. **Yatay ölçekleme (horizontal scaling)**: Talep yükseldiğinde konteyner sayısı otomatik olarak artırılabilir. Talep düştüğünde azaltılabilir. Bunu yapılandırma olarak tanımlayabilirsiniz.

4. **Geri alınabilir güncellemeler (rolling updates)**: Uygulamayı yeni sürüme güncellerken, hizmet kesmeden yapabilirsiniz. Kubernetes, eski konteynerler kapanırken yeni konteynerler başlatır.

5. **Otomatik servis keşfi**: Konteynerler otomatik olarak ağda keşfedilir. DNS adları otomatik olarak atanır.

6. **Depolama düzenlemesi (storage orchestration)**: Konteynerlar kalıcı depolama (persistent storage) bağlayabilir ve başka yere taşınabilir.

7. **Düşük kaynak tüketimi**: Kubernetes, sistem kaynaklarını verimli bir şekilde kullanarak konteynerleri tercih edilen dağıtır.

### Neden Modern Deployment Sistemlerinin Merkezinde Yer Aldığı

Üretim ortamlarında yazılım dağıtmak, konteyner teknolojisinin ortaya çıkması ile radikal şekilde değişti. Sanal makinelerden farklı olarak, konteynerler hafif ve hızlıdır. Ancak bu hafiflik, yönetimi de karmaşık hale getiriyor. Binlerce konteyner çalıştırmak, bunu manuel olarak yapmak imkânsızdır.

Kubernetes, bu ölçekte yönetimi mümkün kılmıştır. Kubernetes'in sayesinde, bir konfigürasyon dosyasıyla yüzlerce makine ve binlerce konteyner yönetilebilir. Bu, DevOps'un temelini oluşturur. Eğer Kubernetes'iniz varsa, manuel müdahale ihtiyacı minimum düzeye iner. Sistem kendi kendine çalışır ve gereken durumlarda iyileştirme yapar.

Günümüzde, cloud ortamlarında konteyner tabanlı uygulamalar çalıştırmak neredeyse Kubernetes ile eşanlamlıdır. AWS'de Elastic Kubernetes Service (EKS), Azure'da Azure Kubernetes Service (AKS), Google Cloud'da Google Kubernetes Engine (GKE) gibi yönetilen Kubernetes hizmetleri mevcuttur. Bu, Kubernetes'in endüstri standardı hâline geldiğini gösterir.

---

## 4. Cluster Kavramı

### Cluster Nedir?

Kubernetes'de bir cluster, Kubernetes tarafından yönetilen makinelerin (nodes) bir grubudur. Cluster, bir işletme olarak düşünülebilir: üretim hattı (worker nodes), yönetim merkezi (control plane), iş sahipleri (konteynerler) ve kurallar (policies) vardır.

Bir cluster'ın minimum yapısı:

- **Control plane**: Küme kararlarını alan ve uygulayan merkezi yönetim birimi
- **En az bir worker node**: Gerçek işleri (konteynerler) çalıştıran makine

Üretim ortamında, cluster çoğunlukla daha büyük yapıdadır:

- **1 veya 3 control plane node**: Yüksek kullanılabilirlik için 3 tavsiye edilir
- **Çok sayıda worker node**: Talep ve yüklenmeye göre değişir

### Tek Makine ile Cluster Mantığı Arasındaki Fark

Tek bir makineyi inceleyelim. Makinde birkaç konteyner çalıştırıyoruz:

```
Makine (192.168.1.100)
├── user-service konteyner
├── order-service konteyner
└── payment-service konteyner
```

Konteynerler birbirine IP adresi ile bağlanır. Örneğin, order-service, `localhost:8080` adresinde user-service'i çağırabilir. Basit ve anlaşılırdır.

Şimdi iki makine olduğunu varsayalım:

```
Makine 1 (192.168.1.100)
├── user-service konteyner 1
└── user-service konteyner 2

Makine 2 (192.168.1.101)
├── order-service konteyner
└── payment-service konteyner
```

Order-service, user-service'i çağırmak istiyor. Ancak user-service iki makineye yayılmış. Hangisini çağıracak? Eğer 192.168.1.100 makinesini çağırırsa, load balancer olmadığı için talep dengeli dağıtılmaz. Eğer 192.168.1.101'e Makine 1'den bir linking yaparsa, ağ trafiği makineler arasında seksiyon yapıp verimsiz olur.

Cluster mantığında, Kubernetes bu detaylı yönetimi sizden almır. Siz, "5 tane user-service kopya çalıştır, 2 tane order-service çalıştır" dersiniz. Kubernetes:

- Bu konteynerları küme içindeki uygun makinelere dağıtır
- Ağ bağlantısını otomatik olarak kurar
- Servis keşfi yönetir (DNS adları oluşturur)
- Load balancing gerçekleştirir

Sizin yapmanız gereken, makineleri ve Kubernetes'i yönetmek. Konteyner yerleştirmesinin detayları Kubernetes'in sorumluluğundadır.

---

## 5. Kubernetes Mimarisi: Genel Görünüm

Kubernetes, iki ana bileşen grubundan oluşur:

### Control Plane

Cluster'ı yönetmek ve karar vermek için gereken bileşenleri içerir. "Beyin" olarak düşünülebilir. Control plane, şunları yapar:

- Kullanıcıdan gelen komutları alır
- Küme durumunu izler
- Konteynerleri yerleştirmek için karar verir
- Küme ayarlarını saklır

### Worker Node (İşçi Düğüm)

Gerçek konteynerler burada çalışır. Birçok işçi olabilir. Her işçü, birçok konteyner çalıştırabilir.

### Aralarındaki İletişim

Control plane'den worker node'lara şu tür mesajlar gider:

"Şu konteyner'ı başlat"
"Şu pod'ı kaldır"
"Sistem sağlığını kontrol et"

Worker node'lardan control plane'e geri:

"Konteyner başladı"
"CPU kullanımı %80"
"Pod kapandı"

Bu, control plane'ye, cluster'ın mevcut durumunu bildir. Kubernetes, bu bilgi ile "desired state" (istenilen durum) ile karşılaştırır.

---

## 6. Control Plane Bileşenleri

Control plane, dört ana bileşenden oluşur. user-service'i Kubernetes'e dağıtırken, bu bileşenlerin her biri geri planda çalışır.

### kube-apiserver (API Sunucusu)

API sunucusu, Kubernetes'e "kapı" gibi davranır. Tüm istekler API sunucusu vasıtasıyla yapılır.

Örneğin, siz şu komutu çalıştırıyoruz:

```bash
kubectl apply -f user-service-pod.yaml
```

Bu komut, pod bilgilerini API sunucusuna gönderir. API sunucusu:

1. İsteği doğrular (yaml'nin geçerli olup olmadığını kontrol eder)
2. İsteği etcd'ye kaydeder
3. Diğer bileşenlere haberini verir

Siz de pod durumunu sormak istediğinizde:

```bash
kubectl get pods
```

Bu istek de API sunucusuna gider. API sunucusu, etcd'den bilgiler alır, size cevap döner.

### etcd (Distributed Key-Value Store)

etcd, Kubernetes'in veritabanıdır. Küme'nin bütün sayı ("desired state") burada saklanır.

Örneğin:
- "user-service-pod adında bir pod olmalı", etcd'de saklanır
- Pod'a ait etiketler, etcd'de saklanır
- Konfigürasyon, etcd'de saklanır

etcd, dağıtık bir sistemdir. Birden fazla control plane node varsa, etcd replikasyon yaparak verileri synchronized tutuo. Bu, yüksek kullanılabilirlik sağlar.

### kube-scheduler (Planlayıcı)

Scheduler, konteynerların nereye yerleştirileceğini karar verir.

Siz "yeni bir pod çalıştır" dediğinizde, scheduler:

1. Pod'ın kaynak gereksinimlerini okur (ne kadar CPU ve RAM istediği)
2. Kullanılabilir worker node'ları inceler (hangisinin yeterli kaynağı olduğunu)
3. Optimal node'u seçer
4. Pod'u o node'a yerleştirmek üzere işareti koyar

Bu, otomatik bir işlemdir. Siz "bu makinaya yerleştirilsin" diye söylemezsiniz.

### kube-controller-manager (Denetleyici Yöneticisi)

Controller manager, Kubernetes'in "kalbi" gibi çalışır. Sürekli olarak cluster'ı izler ve istenilen durumu (desired state) mevcut duruma yaklaştırmaya çalışır.

Örneğin, siz "3 tane user-service replikası çalıştır" dediğinizdi. Controller manager:

1. "3 replika olmalı" gereksinimini yapılandırda bulur
2. Şu anda kaç replika çalışıyor kontrol eder
3. Eğer 2 çalışıyorsa, 1 tane daha başlatır
4. Eğer 4 çalışıyorsa, 1 tanesini kapatır

Bu, başlangıcında bir kez yapılmaz. Sistem sürekli olarak bu kontrol eder. Bir pod'un kapanması durumunda, controller manager bunu fark eder ve otomatik olarak yeni bir pod başlatır.

---

## 7. Worker Node Bileşenleri

Worker node'u, Kubernetes'in "üretim hattı" olarak düşünebilirsiniz. Bunsunda konteynerler gerçekten çalışır.

### kubelet (İşçi Ajanı)

Kubelet, worker node üzerinde çalışan aracıdır. Control plane'den gelen talmatları alır ve gerçekten uygular.

Örneğin, scheduler "user-service pod'u node-5'e yerleştirilsin" kararı verirse, kubelet:

1. Bu talimatlı API sunucusundan alır
2. Container runtime'a "bu image'dan konteyner oluştur" der
3. Konteyner başlatıldıktan sonra, bu bilgiyi API sunucusuna rapor eder

Kubelet aynı zamanda, konteyner sağlığını izler. Konteyner başarısız olursa bunu farkeder ve yeniden başlatılması gerektiğini rapor eder.

### kube-proxy (Ağ Vekili)

Kube-proxy, cluster içindeki ağ trafiğini yönetir.

user-service'i çalışan bir pod, order-service'i çağırmak istiyor. Order-service birden fazla pod'da çalışıyor olabilir. Kube-proxy:

1. Gelen trafiği algılar
2. Order-service'in tüm pod'larına yönlendirir
3. Zaman içinde dengeli bir dağıtım sağlar (load balancing)

Bu, ağ seviyesinde gerçekleşir. Uygulama kodu, hangi pod'a gideceğini düşünmez. Kube-proxy, bunu otomatik olarak yönetir.

### Container Runtime

Container runtime, gerçekten konteyner çalıştıran yazılımdır. Kubernetes bağımsızdır: Docker kullanılabilir, ancak containerd veya CRI-O de kullanılabilir.

Kubelet, container runtime'a "git, bu image'dan konteyner oluştur" der. Container runtime bunu yapar.

---

## 8. Desired State ve Reconciliation Mantığı

Kubernetes'i anlamak için, "desired state" (istenilen durum) ve "actual state" (mevcut durum) arasındaki farkı anlamak gerekir.

### Kullanıcı Ne Tanımlar (Desired State)

Siz bir YAML dosyası yazarsınız:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
```

Bu dosya, "benim şu duruma ihtiyacım var" demektir:

- `user-service-pod` adında bir pod
- İçinde `user-service` konteyner
- Image: `registry.example.com/user-service:1.0.0`

Bu, istenilen durumdur. Siz "bunu çalıştır" demiş olursunuz ama "nasıl" demezsiniz.

### Sistem Neyi Korumaya Çalışır (Reconciliation)

Kubernetes üst çizgiyi alır ve harekete geçer:

1. "Kullanıcı ne istiyor" sorusunun cevabını elde eder
2. "Şu anda ne var" sorusunun cevabını elde eder
3. Aradaki farkı kapatmaya çalışır

Örneğin:
- İstenilen: 1 pod çalışıyor
- Mevcut: İlk başta hiç pod yok
- Fark: 1 pod eksik
- Harekete: Scheduler pod'u bir node'a yerleştirip başlatır

Daha sonra:
- İstenilen: 1 pod çalışıyor
- Mevcut: 1 pod çalışıyor
- Fark: Yok
- Harekete: Hiçbir şey yapılır

Eğer daha sonra pod kapanırsa:
- İstenilen: 1 pod çalışıyor
- Mevcut: 0 pod çalışıyor (çünkü pod kapandı)
- Fark: 1 pod eksik
- Harekete: Yeni pod başlatılır

Bu döngü süreklidir. Kubernetes, saniyede birçok kez "şu anda durum ne" kontrol eder. İstenilen durumdan sapış varsa, otomatik olarak iyileştirme yapılır.

Bu mekanizmaya "reconciliation loop" denir. Kubernetes'i güçlü kılan, bu basit ama etkili mekanizmadır.

---

## 9. Pod Kavramı

### Pod Nedir?

Pod, Kubernetes'deki en küçük çalıştırılabilir birimdir. Pod, bir veya birden fazla konteyner içerir.

Önemli bir nokta: Pod, container DEĞILDIR. Pod, container'ları TUTUŞ bir yapıdır. Kubernetes, container'ları doğrudan yönetmez. Pod'ları yönetir.

### Neden Container Değil Pod Yönetilir?

Docker'da, konteyner birbirinden tümüyle izole edilmiştir. Her konteyner'ın kendi ağ interface'i, kendi dosya sistemi var.

Kubernetes'de ise, genellikle bir pod'un içindeki konteynerlerin işbirliği gerekmektedir.

Örneğin:
- Ana uygulama konteyner: user-service
- Yardımcı konteyner: logging ajanı

Logging ajanı, user-service'in log dosyasını okumalı ve merkezi bir sunucuya göndermeli. Bu, paylaşılmış bir dosya sistemine ihtiyaç duyar. Pod, bu paylaşımı sağlar.

### Tek Konteyner Pod

Basit durumda, bir pod bir konteyner içerir:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
```

Bu, user-service uygulaması için bir pod. Pod başladığında, içindeki konteyner başlar. Pod kapandığında, konteyner kapanır.

### Çok Konteyner Pod

Daha karmaşık durumlarda, bir pod birkaç konteyner içerir:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
    - name: logging-sidecar
      image: fluentd:latest
```

Bu pod, iki konteyner içerir. İkisi aynı anda başlar, aynı anda kapanır.

### Paylaşılmış Ağ Bağlamı

Bir pod içindeki tüm konteynerler, aynı ağ interface'i paylaşır. Bu, onların:

- Aynı IP adresine sahip olması
- Localhost üzerinden birbirine bağlanabilmesi
- Aynı port numaralarını kullanamaması anlamına gelir

Örneğin, bir pod'da user-service 8080 portunda çalışıyorsa, aynı pod'da başka bir uygulama 8080 portunda çalışamaz. Ancak 8081'de çalışabilir.

### Paylaşılmış Depolama

Pod içindeki konteynerler, volume (depolama) paylaşabilir. Örneğin, logging sidecar'ı user-service'in log dosyalarını okuması için.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
      volumeMounts:
        - name: log-volume
          mountPath: /var/log
    - name: logging-sidecar
      image: fluentd:latest
      volumeMounts:
        - name: log-volume
          mountPath: /var/log
  volumes:
    - name: log-volume
      emptyDir: {}
```

Her iki konteyner de aynı `/var/log` dizinine erişebilir.

---

## 10. User-Service Örneği ile İlk Pod

### Mevcut Durumdan Başlangıç

Önceki bölümlerde, user-service'i Docker image'ı olarak hazırlamıştık:

```bash
docker build -t registry.example.com/user-service:1.0.0 .
docker push registry.example.com/user-service:1.0.0
```

Image, bir container registry'de saklanmıştır. Artık, bu image'dan Kubernetes pod'unda çalıştırıyoruz.

### Docker'dan Kubernetes'e Geçiş Mantığı

Docker'da şu komut çalıştırıyorduk:

```bash
docker run -d -p 8080:8080 registry.example.com/user-service:1.0.0
```

Bu komut:
- Konteyner başlatmak için
- 8080 portunda erişilebilir kılmak için

Kubernetes'de ise, biz bu detaylara sahip olmuyoruz. Yerine, deklaratif bir tanım veriyoruz.

### Pod Manifest Tanımı

user-service için bir pod manifest'i oluşturalım:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
  labels:
    app: user-service
    version: "1.0.0"
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
      ports:
        - containerPort: 8080
          protocol: TCP
      resources:
        requests:
          cpu: "100m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

Bu manifest'i açıklamak gerekir:

**apiVersion: v1**
- Kubernetes API sürümü

**kind: Pod**
- Ne oluşturduğumüz (pod)

**metadata**
- Pod hakkında bilgiler
- name: Pod adı (benzersiz olmalı)
- labels: Etiketler (pod'u sorgulamak için kullanılır)

**spec**
- Pod'un spesifikasyonu
- containers: İçindeki konteynerler
- image: Hangi Docker image'ı kullanılacak
- ports: Konteyner'in açığa çıkardığı portlar (belirtici, yönetim amaçlı)
- resources: CPU ve bellek talep ve sınırları

### Manifest'i Kubernetes'e Uygulamak

Manifest'i `user-service-pod.yaml` dosyasına kaydettikten sonra:

```bash
kubectl apply -f user-service-pod.yaml
```

Bu komut, manifest'i Kubernetes'e gönderir. Kubernetes:

1. Manifest'i doğrular
2. etcd'ye kaydeder
3. Scheduler, pod'u uygun node'a yerleştirir
4. İlgili node'da kubelet, konteyner başlatır

### Pod'u Sorgulamak

Pod'un durumunu kontrol etmek:

```bash
kubectl get pods
```

Çıktı:

```
NAME                 READY   STATUS    RESTARTS   AGE
user-service-pod     1/1     Running   0          2m
```

Pod hakkında detaylı bilgi:

```bash
kubectl describe pod user-service-pod
```

Pod'un log'larını görmek:

```bash
kubectl logs user-service-pod
```

Eğer pod birden fazla konteyner içerse, hangi konteyner'ın log'unu istediğinizi belirtmelisiniz:

```bash
kubectl logs user-service-pod -c user-service
```

---

## 11. Declarative Yaklaşım

### Imperative Komut ile Declarative YAML Farkı

**Imperative (Emrediş) Yaklaşım:**

```bash
kubectl run user-service-pod --image=registry.example.com/user-service:1.0.0 --port=8080
```

Bu komut, Kubernetes'e "git, bu image'dan pod oluştur, 8080 portunda çal" der. Anında çalışır.

Sorun: Eğer bir daha aynı pod'u beden ortamında oluşturmak isterse, komutu hatırlamanız gerekir. Eğer komutu unutursanız veya yanlış yazarsanız, farklı bir sonuç alırsınız.

**Declarative (Bildirimsel) Yaklaşım:**

```bash
kubectl apply -f user-service-pod.yaml
```

`user-service-pod.yaml` dosyası, "pod hangi durumda olması gerektiği" tanımlar. Kubernetes, bu durumu korumaya çalışır.

Avantaj: YAML dosyası kaynak kontrolü altında tutulebilir (Git'e kaydedilebilir). Aynı konfigürasyonu tekrar tekrar uygulanabilir. Değişiklikler takip edilebilir.

### Neden YAML Kullanıldığı

YAML, insan tarafından okunabilir bir formatdır. JSON da kullanılabilir, fakat YAML daha basittir.

YAML kurtarım:

```yaml
- İsim: user-service-pod
  Konteyner:
    - İsim: user-service
      Image: registry.example.com/user-service:1.0.0
```

JSON:

```json
{
  "name": "user-service-pod",
  "container": {
    "name": "user-service",
    "image": "registry.example.com/user-service:1.0.0"
  }
}
```

YAML daha okunaklı. Ayrıca, Kubernetes ekosistemi YAML'ye standardlaştığı için bütün örnekler YAML'de bulunur.

### Declarative Yönetim Avantajları

1. **Tekrar Edilebilirlik**: Aynı YAML dosyasını kaç kez çalıştırırsanız çalıştırın, sonuç aynıdır.

2. **Versiyon Kontrol**: YAML dosyasını Git'e saklayabilirsiniz. Değişiklik geçmişi takip edilir.

3. **Otomatimeleme**: CI/CD pipelineları, pod'u dağıtırken YAML dosyasını referans alabilir.

4. **Denetlenebilirlik**: Kim ne dağıttığını ve neden dağıttığını görebilirsiniz.

İşletmeler, üretim ortamında Kubernetes'i declarative yaklaşımla kullanır. İmperatif komutlar yalnızca öğrenme veya hızlı test amaçlı kullanılır.

---

## 12. Kubectl Mantığı

kubectl, Kubernetes API sunucusu ile iletişim kuran komut satırı aracıdır. Tüm Kubernetes işlemleri kubectl aracılığıyla gerçekleştirilir.

### kubectl apply

Manifest dosyalarını Kubernetes'e göndermek için:

```bash
kubectl apply -f user-service-pod.yaml
```

`apply` komutu:
- Dosyayı okur
- API sunucusuna gönderir
- Sunucu, ilgili durumu kontrol eder ve gerekli işlemleri yapar

Aynı dosya tekrar çalıştırırsanız:

```bash
kubectl apply -f user-service-pod.yaml
```

Kubernetes, "bu sonuçta aynı" der ve hiçbir şey yapmaz. Bu, idempotent davranıştır - aynı işlemi kaç kez yaparsanız yapın, nihai sonuç aynıdır.

### kubectl get

Kaynakları sorgulamak için:

```bash
kubectl get pods
```

Çıktı:

```
NAME               READY   STATUS    RESTARTS   AGE
user-service-pod   1/1     Running   0          5m
```

Daha detaylı bilgi:

```bash
kubectl get pods -o wide
```

Ayrıca spesifik pod:

```bash
kubectl get pod user-service-pod
```

### kubectl describe

Bir kaynağın detaylı bilgisini görmek için:

```bash
kubectl describe pod user-service-pod
```

Çıktı:

```
Name:         user-service-pod
Namespace:    default
Priority:     0
Node:         worker-node-1
Status:       Running
IP:           10.244.0.5
Containers:
  user-service:
    Container ID:   docker://abc123...
    Image:          registry.example.com/user-service:1.0.0
    Image ID:       docker-pullable://registry.example.com/user-service@sha256:...
    Port:           8080/TCP
    State:          Running
      Started:      Mon, 16 Apr 2026 10:30:00 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  512Mi
```

Bu, pod'un tam durumunu gösterir: nerede çalışıyor, kaynakları nasıl kullanıyor, hangi durumdadır.

### kubectl logs

Konteyner'in çıktısını görmek için:

```bash
kubectl logs user-service-pod
```

Eğer pod birden fazla konteyner içerse:

```bash
kubectl logs user-service-pod -c user-service
```

Son satırlardan başlayarak görmek:

```bash
kubectl logs user-service-pod --tail=50
```

### kubectl delete

Pod'u silmek için:

```bash
kubectl delete pod user-service-pod
```

YAML dosyasından silmek:

```bash
kubectl delete -f user-service-pod.yaml
```

Pod silindiğinde, içindeki konteyner de kapatılır.

---

## 13. Pod Yaşam Döngüsüne Giriş

Pod'un, yaşamboyu çeşitli durumlardan geçer.

### Pending (Beklemede)

Pod oluşturulduktan sonra, scheduler bir node seçmektedir. Bu sürede pod Pending durumundadır.

```
NAME               READY   STATUS    RESTARTS   AGE
user-service-pod   0/1     Pending   0          10s
```

Pending durumu kısacık sürmeli. Eğer uzun sürerse, bir sorun vardır (örneğin yeterli kaynak olmayabilir).

### Running (Çalışmada)

Pod'un konteyner'i başladı ve çalışmaktadır.

```
NAME               READY   STATUS    RESTARTS   AGE
user-service-pod   1/1     Running   0          1m
```

### Succeeded (Başarıyla Tamamlandı)

Pod, başarıyla tamamlandı ve kapandı. Genellikle batch işleri (bir kez çalışan işler) için kullanılır, web uygulamaları için değil.

### Failed (Başarısız)

Pod'un konteyner'i hata verdi ve kapandı.

```
NAME               READY   STATUS   RESTARTS   AGE
user-service-pod   0/1     Failed   0          1m
```

### CrashLoopBackOff

Pod'un konteyner'i sürekli olarak kapanıyor ve yeniden başlatılıyor. Kubernetes, başlatmayı bırakmadan önce exponential backoff kullanarak bekler. Bu, genellikle uygulama kodunda bir sorun olduğu anlamına gelir.

```
NAME               READY   STATUS             RESTARTS   AGE
user-service-pod   0/1     CrashLoopBackOff   5          2m
```

Sebepleri:
- Uygulama açılışta çöküyor
- Bağımlılıklar eksik
- Konfigürasyon yanlış

Bu durumda, `kubectl logs` kullanarak hata mesajlarını incelemek gerekir.

---

## 14. Namespace Kavramına Giriş

### Namespace Nedir?

Namespace, Kubernetes kümesi içinde mantıksal yol bölümü oluşturmak için kullanılır.

Basit örnek:

```
Cluster
├── default
│   ├── user-service-pod
│   └── order-service-pod
├── development
│   ├── test-user-service
│   └── test-order-service
└── production
    ├── user-service-pod
    └── order-service-pod
```

Her namespace, özel bir kuruluş alanıdır. Kaynaklar, namespace içinde yalıtılmıştır.

### Neden Gerekli?

1. **İzolasyon**: Farklı ekipler, farklı namespaces'de çalışabilir. Birer namespace başka namespace'deki kaynakları tamamen etkilemez.

2. **Kaynak Yönetimi**: Her namespace'e kaynak kotları atanabilir. Örneğin, development namespace'i maksimum 10 CPU, production maksimum 50 CPU kullanabilir.

3. **Erişim Kontrolü**: RBAC (Role Based Access Control) policies, namespace seviyesinde uygulanabilir.

### Namespace İçinde Pod

Pod oluşturduğunuzda, varsayılan `default` namespace'ine gider.

Farklı namespace'de pod oluşturmak:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: user-service-pod
  namespace: production
spec:
  containers:
    - name: user-service
      image: registry.example.com/user-service:1.0.0
```

veya komut satırı ile:

```bash
kubectl apply -f user-service-pod.yaml -n production
```

Bir namespace'deki pod'ları görmek:

```bash
kubectl get pods -n production
```

---

## 15. Anti-Pattern: Yapmaması Gerekenler

### Tekil Docker Düşüncesini Kubernetes'e Taşımak

**Yanlış yaklaşım:**

```bash
kubectl run user-service-pod-1 --image=registry.example.com/user-service:1.0.0
kubectl run user-service-pod-2 --image=registry.example.com/user-service:1.0.0
kubectl run user-service-pod-3 --image=registry.example.com/user-service:1.0.0
```

Bunu, üç ayrı pod oluşturmak için yapıyorsunuz. Ancak, Docker düzeyinde düşünüyorsunuz.

**Doğru yaklaşım:**

Deployment kullanmak (bu sonraki bölümde anlatılacak) veya DeploymentSet. Kubernetes, replikaları otomatik olarak yönetir.

### YAML'ı Anlamadan Ezberlemek

**Yanlış yaklaşım:**

Internetten bulup bir YAML dosyasını kopyala-yapıştır olarak kullanmak ve ne yaptığı anlamadan çalıştırmak.

**Doğru yaklaşım:**

Her satırın ne yaptığını anlamak. Kendi manifest'lerinizi yazabilmek.

### Pod ile Deployment'ı Karıştırmak

Pod, tek bir konteyner instance'ıdır. Eğer pod kapanırsa Kubernetes otomatik olarak yeniden başlatmaz.

Bu bölümde biz pod kullanıyoruz, ama üretim ortamlarında pod doğrudan kullanılmaz. Deployment kullanılır. Deployment, pod'ları yönetir ve otomatik olarak yeniden başlatmalarını sağlar.

**Yanlış:**

```bash
kubectl run user-service-pod --image=registry.example.com/user-service:1.0.0
```

Bu komut, pod oluşturur, ama eğer pod kapanırsa, Kubernetes yeniden başlatmaz (yeterli ki bende hata var, kubelet pod'u yeniden başlatır ama başarısız kapanış ile ilgilenmiyor).

**Doğru (sonraki bölümde öğrenilecek):**

Deployment manifest'i kullanmak.

### Node Üzerinde Elle Müdahale Etmek

**Yanlış yaklaşım:**

Bir pod başarısız olursa, node'a SSH bağlantı yaparak manuel olarak container'ı yeniden başlatmak.

```bash
ssh node-1
docker restart container-id
```

**Doğru yaklaşım:**

Pod'u siliniz, Kubernetes otomatik olarak yeniden başlatmasını sağlayabilir. Veya, sorunu çözün (örneğin image'ı güncelle) ve pod'u yeniden boyun.

Elle müdahale, sistem tutarsızlığına ve hataların tekrarlanmastna neden olur.

---

