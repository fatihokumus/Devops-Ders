# Bölüm 7 — Webhook Mekanizması ve Sürekli Entegrasyonun Tetiklenmesi: Jenkins, GitHub ve Ngrok ile Uygulamalı Kurulum

## 1. CI Sürecinde Tetikleme Problemi

Yazılım geliştirme süreçlerinde, kodda yapılan her değişiklik otomatik olarak derlenmeli, test edilmeli ve hazırlanmalıdır. Bu, sürekli entegrasyon yapısının temelini oluşturur. Ancak bu otomasyonun sağlıklı bir şekilde çalışabilmesi için, değişikliklerin ne zaman derlenmesi gerektiğini sistemin bilmesi gerekir.

### Manuel Tetikleme Yaklaşımının Sınırlılıkları

Geliştiriciler, önceki bölümlerde gördüğümüz gibi, Jenkins arayüzünde "Build Now" düğmesine tıklayarak işlemleri başlatabilirler. Fakat bu yaklaşım birkaç ciddi sorunu beraberinde getirir:

İlk olarak, bir geliştirici kodu depoya gönderdikten sonra, ilgili kişi veya ekip bu işlemi fark etmeli ve Jenkins'e gitmeli, düğmeyi tıklamalıdır. Bu, otomasyon olmayan, insana bağımlı bir süreçtir. İkinci olarak, eğer geliştirici işlemi başlatmayı unutursa, kod derlenmeyen, test edilmeyen hâlde kalabilir. Üçüncü olarak, bu yaklaşım ölçeklenebilir değildir; çok sayıda geliştirici ve çok sayıda proje varsa, her biri için ayrı ayrı tetikleme yapmak imkânsız hâle gelir.

### Polling Yaklaşması ve Zaman Maliyeti

Bu sorunu çözmek için, birçok sistem "polling" (yoklama) yöntemi kullanır. Jenkins'in bu yöntemi desteklemesi vardır. Polling yaklaşımında, Jenkins periyodik olarak (örneğin her 5 dakikada bir) depo sunucusuna bağlanır ve şu soruyu sorar: "Kodda bir değişiklik oldu mu?" Eğer cevap "evet" ise, Jenkins otomatik olarak build işlemini başlatır.

Bu yöntem manuel tetiklemeden daha iyidir; en azından otomatiktir. Ancak önemli dezavantajları vardır. Jenkins'in her 5 dakikada bir depo sunucusuna bağlanması, merkezi sunucular için ağ trafiği oluşturur. Eğer hiç kod değişikliği olmadığı hâlde Jenkins yoklama yapıyorsa, bu gereksiz kaynak tüketimidir. Ek olarak, geliştirici kodu gönderdikten sonra Jenkins'in bu değişikliği fark etmesi maksimum 5 dakika sürebilir. Hızlı geri bildirim gereken modern yazılım geliştirme ortamlarında, bu gecikme kabul edilebilir değildir.

### Anlık Tetikleme İhtiyacı

Modern DevOps uygulamalarında, kod depoya gönderildikten hemen sonra (saniyeler içinde) build işleminin başlaması beklenir. Geliştirici, kodu gönderer göndermez, geri bildirimi almak ister. Hata varsa, hemen düzeltebilir. Bu hızlı geri bildirim döngüsü, yazılım kalitesini artırır ve geliştirme sürecini hızlandırır.

Bu ihtiyacı karşılamak için webhook mekanizması ortaya çıkmıştır. Webhook, olayların gerçek zamanlı olarak (real-time) iletildiği bir yöntemdir.

---

## 2. Webhook Kavramı

### Webhook Nedir?

Webhook, bir olayın meydana geldiğinde, belirli bir HTTP POST isteği aracılığıyla önceden belirlenmiş bir URL'ye veri gönderme mekanizmasıdır. Webhook, "web kancası" anlamına gelir; yani, belirli bir olay tetiklendiğinde, bu "kanca" sistem X'ten sistem Y'ye otomatik olarak bilgi taşır.

Webhook, bir olaydaki mesajın (örneğin "yeni commit var") pasif bir şekilde beklenilmesi yerine, etkin bir şekilde (proaktif) hedef sisteme iletilmesi demektir. Yazılım mimarisi perspektifinden bakıldığında, webhook event-driven (olay tabanlı) sistemlerin temel yapı taşıdır.

### HTTP POST Mantığı

Webhook'u daha iyi anlamak için, HTTP protokolünün temellerine geri dönmek gerekir. HTTP protokolü, internet üzerinde veri transferi için kullanılan standar bir protokoldür. GET, POST, PUT, DELETE gibi farklı metotlar vardır.

GET metodu, sunucudan veri talep eder; sunucuya gönderilen parametreler URL'nin içine yerleştirilir. POST metodu ise, sunucuya veri gönderir; veriler HTTP isteğinin gövdesinde (body) yer alır.

Webhook, POST metodu kullanarak çalışır. Örneğin, GitHub'da yeni bir commit yapıldığında, GitHub sunucusu şöyle bir POST isteği hazırlar:

```
POST https://jenkins.example.com/github-webhook

Content-Type: application/json

{... veri ...}
```

Bu POST isteğini Jenkins sunucusunun alması ve işlemesi gerekir. Jenkins, isteği aldığında, gövdedeki verileri inceleyerek "Bu kodu derlemem gerekiyor" kararını verir ve build başlatır.

### Event-Driven Sistem Yaklaşımı

Webhook kavramı, event-driven (olay tabanlı) mimarilerin mantığını yansıtır. Geleneksel programlamada, bir işlem sırasıyla yapılır:

```
1. Adım 1'i yap
2. Adım 2'yi yapana kadar bekle
3. Adım 3'ü yap
```

Event-driven mimaride ise:

```
1. "Kod depoya gönderildi" olayının gerçekleşmesini bekle
2. Bu olay gerçekleşirse, belirlenmiş olan işlemi yap
```

Webhook, bu event-driven mimarininin gerçek dünyada uygulanmasıdır. GitHub "kod gönderildi" olayını algılar ve Jenkins'e bildirir; Jenkins bu olay üzerine harekete geçer.

### Polling vs Webhook Karşılaştırması

Bu iki yöntemi karşılaştırarak aralarındaki temel farkları ortaya koyabiliriz:

| Özellik | Polling | Webhook |
|---------|---------|---------|
| **İnisiyatif** | Jenkins depo sunucusunu sorgular | GitHub Jenkins'e haber gönderir |
| **Zaman Gecikmesi** | Maksimum poll aralığı kadar (örneğin 5 dakika) | Milisaniyeler (hemen hemen anlık) |
| **Ağ Trafiği** | Sürekli, gereksiz de olsa sorgulamalar yapılır | Yalnızca olay gerçekleştiğinde |
| **Kaynak Kullanımı** | Yüksek (gereksiz bağlantılar) | Düşük (yalnızca gerekli bildirimler) |
| **Ölçeklenebilirlik** | Problematik (çok sorgu = yüksek yük) | İyi (sadece gereken verileri gönder) |
| **Gerçek Zamanlılık** | Kısıtlı | Tam |

Webhook, modern CI/CD sistemlerinde polling yöntemi yerine tercih edilmesinin nedeni, yukarıdaki avantajlarıdır. GitHub, GitLab, Bitbucket gibi tüm önemli bir Git hizmeti sağlayıcıları webhook desteği sunar ve bu, endüstri standardı hâline gelmiştir.

---

## 3. GitHub Webhook Yapısı

### Olay Türleri

GitHub, farklı türde olaylar için webhook gönderebilir. En yaygın olanlar şunlardır:

- **push**: Belirli bir branch'e kod gönderildiğinde tetiklenir. Geliştirici `git push` komutunu çalıştırır ve bu olay oluşur.
- **pull_request**: Bir çekme isteği (PR) açıldığında, kapatıldığında veya güncellendiğinde tetiklenir.
- **pull_request_review**: Bir PR'ye kod incelemesi yapıldığında tetiklenir.
- **issues**: Bir issue açıldığında veya kapatıldığında tetiklenir.
- **release**: Bir release oluşturulduğunda tetiklenir.

CI/CD bağlamında, en önemli olan "push" olayıdır. Bu bölümde ağırlıklı olarak push webhook'u üzerinde duracağız.

### Payload Kavramı

Webhook'un gönderdiği verilerin tamamına payload denir. Payload, olay hakkındaki bilgileri JSON formatında içerir. Bu bilgiler, commit kim tarafından yapıldığını, hangi branch'e yapıldığını, hangi dosyaların değiştiğini ve daha birçok detayı içerir.

GitHub webhook payload'ı, oldukça kapsamlı bir JSON yapısıdır. Tam dokumentasyonu yüzlerce satırı kapsayan alanlar içerir. Bununla birlikte, CI/CD için gerekli olan değerlerin nispeten az olduğunu anlamamız önemlidir.

### JSON Veri Yapısı ve Örnek Payload

GitHub'ın push olayı için gönderdiği webhook payload'ının basitleştirilmiş bir örneği aşağıda gösterilmiştir:

```json
{
  "ref": "refs/heads/main",
  "before": "9049503e7c6a5e11c21b16a67a4e35bc9e3edda3",
  "after": "0d1a26e67d8eb5a926f28d8e6c1b9f8a2a8d5c1a",
  "repository": {
    "id": 574532167,
    "name": "user-service",
    "full_name": "mycompany/user-service",
    "private": false,
    "url": "https://github.com/mycompany/user-service"
  },
  "pusher": {
    "name": "developer",
    "email": "developer@mycompany.com"
  },
  "sender": {
    "login": "developer",
    "id": 12345678
  },
  "commits": [
    {
      "id": "0d1a26e67d8eb5a926f28d8e6c1b9f8a2a8d5c1a",
      "message": "Add user authentication endpoint",
      "timestamp": "2026-04-16T10:30:45Z",
      "author": {
        "name": "developer",
        "email": "developer@mycompany.com"
      },
      "modified": [
        "src/main/java/com/example/userservice/controller/AuthController.java",
        "src/main/java/com/example/userservice/security/JwtTokenProvider.java"
      ]
    }
  ],
  "head_commit": {
    "id": "0d1a26e67d8eb5a926f28d8e6c1b9f8a2a8d5c1a",
    "message": "Add user authentication endpoint",
    "timestamp": "2026-04-16T10:30:45Z",
    "author": {
      "name": "developer",
      "email": "developer@mycompany.com"
    },
    "url": "https://github.com/mycompany/user-service/commit/0d1a26e67d8eb5a926f28d8e6c1b9f8a2a8d5c1a"
  }
}
```

Bu payload'ın içerdiği başlıca alanları açıklamak gerekirse:

- **ref**: Gönderilen branch'in tam adı. `refs/heads/main` formatında gelir. Main branch demektir.
- **before**: Gönderilmeden önceki commit'in hash'i.
- **after**: Gönderilen yeni commit'in hash'i.
- **repository.name**: Repositorinin adı. Burada "user-service"dir.
- **pusher.name**: Kim tarafından gönderildiği.
- **commits**: Bu push'ta yer alan tüm commit'ler.
- **head_commit**: En son gönderilen (en yeni) commit.

Jenkins, bu payload'ı aldığında, aşağıdaki kararları verebilir:

1. Branch main mi? Evet. O zaman build başlat.
2. Repository "user-service" mi? Evet. O zaman user-service için build başlat.
3. Commit'te hangi dosyalar değişti? Eğer yalnızca dokümantasyon değişmişse, build başlatma.

Bu kararlar, Jenkins'in webhook payload'ını nasıl işlediğini gösterir.

---

## 4. GitHub ve Jenkins Entegrasyonu

### GitHub'da Webhook Konfigürasyonu

GitHub'da bir webhook oluşturmak oldukça basittir. Repository'nin ayarlarından "Webhooks" bölümüne gidilir. Buraya şu bilgiler girilir:

- **Payload URL**: Jenkins'in webhook'u alacağı URL. Örneğin: `https://jenkins.example.com/github-webhook/`
- **Content type**: Verinin hangi formatta geleceği. Webhook payload'ı JSON olarak gönderildiği için, "application/json" seçilir.
- **Events**: Hangi olaylar için webhook gönderilecek. "Just the push event" seçilirse, yalnızca push olayında webhook gönderilir.
- **Active**: Webhook'un aktif olup olmadığı. Açık olmalıdır.

GitHub, webhook oluşturduktan sonra, otomatik olarak test etme imkânı sunar. "Send test" düğmesine tıklanırsa, GitHub örnek bir payload'ı Jenkins'e gönderir. Bu sayede bağlantı kontrol edilebilir.

### Jenkins'de Webhook Alıcısı Yapılandırması

Jenkins tarafında, webhook'u almak için iki şey gerekir:

Birinci olarak, uygun bir Jenkins plugin'i yüklü olmalıdır. "GitHub" plugin'i (GitHub Integration Plugin veya GitHub Branch Source Plugin) webhook'ları otomatik olarak işler. Bu plugin varsayılan olarak birçok Jenkins kurulumunda zaten vardır.

İkinci olarak, Jenkins'de webhook yapılandırması yapılmalıdır. Klasik Jenkins arayüzünde, "Manage Jenkins" → "Configure System" bölümüne gidilir. Buraya "GitHub" başlığı altında GitHub sunucusu bilgileri girilir.

Modern Jenkins kurulumlarında (özellikle Jenkins Configuration as Code (JCasC) kullananlar için), bu konfigürasyon YAML dosyaları ile yapılabilir.

### Jenkins Pipeline'da GitHub Hook Tetiklemesi

Bir Jenkins Pipeline'ı, webhook tarafından tetiklenmek üzere yapılandırmak için, Pipeline tanımında (Jenkinsfile) şu blok kullanılır:

```groovy
pipeline {
    triggers {
        githubPush()
    }
    ...
}
```

Bu basit satır, Pipeline'ı GitHub webhook'una duyarlı hâle getirir. Webhook geldikten sonra, Jenkins Pipeline'ı otomatik olarak başlatır.

Daha gelişmiş yapılandırmalarda, sadece belirli branch'ler için webhook tetiklemesi yapılabilir:

```groovy
pipeline {
    triggers {
        githubPush()
    }
    stages {
        stage('Build') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Building main branch..."
                    // build steps
                }
            }
        }
    }
}
```

Burada, webhook gelse bile, Pipeline yalnızca `main` branch'ine yapılan push'lar için build başlatır.

---

## 5. HTTP Request Mantığı ve Webhook İletişimi

### HTTP İstek-Yanıt Döngüsü

Webhook'u anlamak için, HTTP protokolünün nasıl çalıştığını bilmek gerekir. HTTP, istemci-sunucu mimarisine dayanır. Istemci bir istek (request) gönderir, sunucu bu isteği işler ve bir yanıt (response) gönderir.

POST isteğinde, veri HTTP isteğinin gövdesinde (body) yer alır. Istek başlığında (header) meta bilgiler vardır. Örneğin, `Content-Type` başlığı, gönderlerin veri türünü belirtir.

Webhook, GitHub'ın istemci, Jenkins'in sunucu olduğu bir HTTP POST isteğidir. GitHub, Jenkins'in URL'sine erişmeye çalışır. Jenkins, bu isteği alırsa, HTTP 200 OK yanıtı göndermelidir. Bu yanıt, GitHub'a "İsteği başarıyla aldık" mesajını iletir.

### GitHub'dan Jenkins'e Veri Akışı

Gerçek dünyada, bu akış şu şekilde işler:

1. Geliştirici, lokal makinesinde `git push origin main` komutu çalıştırır.
2. Kod, GitHub'a yüklenilir.
3. GitHub, "Kod main branch'ine gönderildi" olayını algılar.
4. GitHub, webhook payload'ını hazırlar (JSON formatında, yukarıda gördüğümüz değerler).
5. GitHub, bir POST isteğini Jenkins'in webhook URL'sine gönderir. Örneğin: `https://jenkins.example.com/github-webhook/`
6. Jenkins, bu isteği alır ve payload'ı işler.
7. Jenkins, payload içindeki bilgileri inceleyip (branch nedir, repository adı nedir), kararını verir.
8. Eğer koşullar sağlanıyorsa, Jenkins alakalı Pipeline veya Job'ı başlatır.
9. Jenkins, GitHub'a HTTP 200 OK yanıtı gönderir ve böylelikle "isteği aldım" mesajını iletir.

Bu tüm süreç, genellikle saniyenin onda birinden az bir zamanda gerçekleşir.

### Jenkins'in Webhook Payload'ını İşlemesi

Jenkins, webhook payload'ı aldığında, içerindeki bilgileri parçalara ayırır ve analiz eder. GitHub Plugin, payload'ı otomatik olarak işler. Geliştirici tarafında, çoğunlukla Plugin'in otomatik işlemesi yeterlidir.

Ancak, ileri düzey senaryolarda, Jenkins scripti yazılarak payload'ın belirli alanları kontrol edilebilir. Örneğin:

```groovy
pipeline {
    triggers {
        githubPush()
    }
    stages {
        stage('Check Branch') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'main'
                    echo "Webhook triggered for branch: ${branch}"
                    
                    if (branch == 'main') {
                        echo "Building from main branch"
                    } else if (branch == 'develop') {
                        echo "Building from develop branch"
                    }
                }
            }
        }
    }
}
```

Jenkins, ortam değişkenlerini (environment variables) kullanarak webhook verisine erişir. `BRANCH_NAME`, `GIT_COMMIT`, `GIT_AUTHOR` gibi değişkenler, Jenkins tarafından otomatik olarak ayarlanır.

---

## 6. Lokal Ortamda Webhook Test Etmenin Zorlukları

### İnternet Üzerinden Erişilebilirlik Sorunu

Jenkins'in webhook alması için, GitHub'ın Jenkins'in URL'sine erişebilmesi gerekir. Bu, lokal makine veya özel veri merkezi ortamlarında büyük bir sorundur.

Geliştirme aşamasında, bir yazılımcı genellikle kendi bilgisayarında (localhost) Jenkins kurabilir. Örneğin, Jenkins `http://localhost:8080` adresinde çalışıyor olabilir. GitHub, webhook'u gönderirken, bu URL'ye erişmeye çalışırsa, başarısız olur. Çünkü GitHub, internete açık olan halka açık bir URL bekliyorken, `localhost`, GitHub sunucularından erişilemez.

Benzer şekilde, eğer Jenkins şirket ağının içinde (intranet) bulunan bir sunucuda çalışıyorsa ve GitHub bu ağa (public internete) erişemiyorsa, aynı sorun ortaya çıkar.

### Firewall ve NAT Sorunları

Çoğu işletme güvenliği sağlamak için firewall kullanır. Firewall, dışından gelen istekleri engeller. GitHub, Jenkins'e erişmeye çalışırsa, firewall bunu potansiyel bir tehdit olarak görebilir ve isteği bloklar.

Ek olarak, NAT (Network Address Translation) teknolojisi, lokal ağ içindeki makineleri internete çıksalar bile gizler. Dış dünyadan biri, NAT arkasındaki bir makineye doğrudan erişemez.

### Geliştirme Aşamasında Pratik Sorunlar

Bir geliştirici, webhook'u test etmek istiyorsa, birkaç yöntem deneyebilir:

Birinci yaklaşımda, curl gibi araçlar kullanarak manuel olarak istemci tarafında istek gönderilebilir. Bu, Plugin'in temel işlevini test eder ancak GitHub'ın tam payload formatını taklit etmek zahmetli olabilir. İkinci yaklaşımda, Jenkins'i internete açık bir sunucuya yayınlamak seçeneği vardır. Ancak bu, geçici çözüm olarak zahmetli ve güvenlik riskleri bulunmaktadır.

Üçüncü yöntemde, webhook test araçları kullanılabilir. Örneğin, curl komutu ile manuel olarak Jenkins'in webhook URL'sine POST isteği gönderilebilir. Ancak bu yöntemde GitHub'ın kesin payload yapısını taklit etmek zorunludur ve bu da zahmetli bir işlemdir.

Bu zorlukları etkili bir şekilde çözmek için, ngrok gibi özel araçlar geliştirilmiştir.

---

## 7. Ngrok ile Lokal Webhook Test Ortamı Kurulumu

### Ngrok Nedir?

Ngrok, lokal makinelerde çalışan sunucuları internete açan bir araçtır. Ngrok, lokal makine (örneğin localhost:8080) ile internete açık bir URL arasında bir tünel (tunnel) oluşturur. Bu sayede, internet üzerindeki hizmetler (GitHub gibi), lokal makinedeki sunucuya erişebilir.

Ngrok, "buradan buraya tünel aç" demek gibidir. Geliştirici, lokal makinesinde Jenkins çalıştırabilir, ngrok aracılığıyla internete açık bir URL elde edebilir ve GitHub webhook'unu bu URL'ye yönlendirebilir. GitHub webhook'u, internet üzerinden ngrok'un sunucusuna gidecek, ngrok bu isteği lokal Jenkins'e iletecektir.

### Ngrok Kurulumu ve Konfigürasyonu

Ngrok'u kullanmak oldukça basittir. Ngrok'un resmi web sitesinden indirip (https://ngrok.com), kurulum yapılır. Kurulumdan sonra, aşağıdaki komut, lokal Jenkins'i internete açar:

```bash
ngrok http 8080
```

Bu komut, lokal makinede 8080 portunda çalışan her şeyi internete açar. Ngrok, şöyle bir çıktı verir:

```
Session Status                online
Account                       Limited
Version                       3.3.5
Region                        eu
Forwarding                    https://1a2b3c4d5e6f.eu.ngrok.io -> http://localhost:8080
Web Interface                 http://127.0.0.1:4040
```

Bu çıktıda, `https://1a2b3c4d5e6f.eu.ngrok.io` internete açık olan URL'dir. GitHub, webhook'unu bu URL'ye gönderebilir. Ngrok, gelen istekleri lokal `localhost:8080` adresine iletecektir.

### Ngrok URL'sini GitHub'da Yapılandırma

GitHub'da webhook oluştururken, Payload URL alanına ngrok URL'si yazılır. Örneğin:

```
https://1a2b3c4d5e6f.eu.ngrok.io/github-webhook/
```

Bu URL, GitHub webhook'unun giteceği adrestir. Ngrok bu isteği lokal Jenkins'e iletecektir.

Önemli bir not: Ngrok'un ücretsiz hesabı kullanırken, her yeniden başlandığında URL değişir. Bu, otomasyonu zorlaştırabilir. Ancak geliştirme ve test aşamasında, bu kabul edilebilir bir sınırlamadır.

### Ngrok Web İnterface'i İle İzleme

Ngrok, `http://127.0.0.1:4040` adresinde web tabanlı bir arayüz sunar. Bu arayüzde, ngrok'a gelen tüm istekler real-time olarak izlenebilir. Geliştirici, GitHub'ın gönderdiği webhook'u bu arayüzde görebilir, istekle ilgili detaylar (header, body) inceleyebilir.

Bu, webhook hataları ayıklamak için çok kullanışlı bir araçtır. Eğer Jenkins webhook'u almadığını ifade ediyorsa, ngrok web arayüzüne gidip GitHub'ın isteği gönderip göndermediğini, alınan HTTP status kodunu kontrol edebilirsiniz.

---

## 8. Uçtan Uca Webhook Tetikleme Senaryosu: user-service Projesi

### Senaryo Tanımı

Bu bölümde, önceki bölümlerde kurduğumuz user-service projesi ile webhook'un tam bir çalışma akışını anlatılacaktır. Senaryo şu şekildedir:

- Bir geliştirici, user-service repository'sine yeni bir özellik ekler.
- Kodu GitHub'a gönderir (git push).
- GitHub webhook'u tetiklenir.
- Jenkins, webhook aracılığıyla bilgi alır.
- Jenkins, user-service'i derler, test eder ve paketler.
- Derleme tamamlanınca, sonuç geliştiriye bildirilir.

### Önceki Bölümlerdeki Kurulumun Gözden Geçirilmesi

Bölüm 5 ve 6'da, user-service projesi için bir Jenkins Pipeline oluşturmuş olduk. Önceki kurulumda oluşturulan Declarative Pipeline Jenkinsfile aşağıda basit olarak gösterilmiştir:

```groovy
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
    }
    
    post {
        success {
            echo 'Build successful!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
```

Bu Pipeline, manuel olarak tetiklendiğinde çalışır. Webhook mekanizması üzerinden tetiklenmesi için aşağıdaki adımları uygulamalıyız.

### Jenkinsfile'ı Webhook İçin Güncellemek

Pipeline'ı webhook ile tetiklemek için, `triggers` bloğu eklenir:

```groovy
pipeline {
    agent any
    
    triggers {
        githubPush()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
    }
    
    post {
        success {
            echo 'Build successful!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
```

`githubPush()` trigger'ı, Pipeline'ı GitHub webhook'una bağlar.

### GitHub Webhook Oluşturma ve Yapılandırma

user-service repository'sine gidilir. Settings → Webhooks bölümünde, "Add webhook" düğmesine tıklanır.

Aşağıdaki bilgiler girilir:

- **Payload URL**: Production ortamında `https://jenkins.example.com/github-webhook/`, lokal test ortamında ngrok URL'si. Örneğin: `https://abc123def456.eu.ngrok.io/github-webhook/`
- **Content type**: `application/json`
- **Secret** (opsiyonel): Webhook'un güvenliği için, GitHub ve Jenkins arasında bir sır paylaşılabilir. GitHub bu sırrı kullanarak isteği imzalar; Jenkins imzayı kontrol eder. Bu, güvenliği arttırır. Ancak basit test ortamında, bu opsiyonel bırakılabilir.
- **Events**: "Just the push event" seçilir (test için), veya "Send me everything" seçilir (öğrenme amaçlı).
- **Active**: Checkbox işaretlenir.

Webhook oluşturduktan sonra, GitHub'ın "Send test" seçeneği kullanılarak test yapılabilir.

### Geliştirme Akışı ve Webhook Tetiklemesi

Senaryo başlıyor. Geliştirici, lokal makinesinde user-service repository'sini klonlamış ve güncellemiş:

```bash
cd user-service
echo "New API endpoint for user authentication" >> README.md
git add README.md
git commit -m "Update README with authentication endpoint documentation"
git push origin main
```

Bu komutlar çalıştırıldıktan sonra:

1. Kod, GitHub'a (main branch'ine) gönderilir.
2. GitHub, push olayını algılar.
3. GitHub webhook'u tetiklenir.
4. GitHub, webhook payload'ını Jenkins'e (veya ngrok'a, lokal test durumunda) gönderir.
5. Jenkins, isteği alır ve payload'ı işler.
6. Jenkins githubPush() trigger'ı aktif olduğu için, Pipeline'ı başlatır.
7. Pipeline, Checkout başlayarak kod deposundan en son kodu çeker.
8. Build stage'de, `mvn clean install` komutu çalıştırılır.
9. Test stage'de, `mvn test` komutu çalıştırılır.
10. Package stage'de, `mvn package` komutu çalıştırılır.
11. Build başarılı olursa, post section'da success mesajı (echo) çalıştırılır.

Geliştirici, tüm bu süreci Jenkins web arayüzündeki Pipeline'ın konsolunda izler. Build bittikten sonra, logu inceleyerek sorunları görebilir, hataları çözüp yeniden push edebilir. Tamamı otomatiktir.

### Koşullu Tetikleme

Bazen, yalnızca belirli branch'ler webhook'u tetiklemelidir. Örneğin, sadece main branch'ine yapılan push'lar build başlatsın, geliştirici branch'lerine yapılan push'lar başlatmasın:

```groovy
pipeline {
    agent any
    
    triggers {
        githubPush()
    }
    
    stages {
        stage('Check Branch') {
            steps {
                script {
                    if (env.BRANCH_NAME != null && env.BRANCH_NAME != 'main') {
                        echo "Skipping build for branch: ${env.BRANCH_NAME}"
                        currentBuild.result = 'SKIPPED'
                        return
                    }
                }
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }
    }
    
    post {
        success {
            echo 'Build successful!'
        }
        failure {
            echo 'Build failed!'
        }
    }
}
```

Bu yapılandırmada, webhook gelse bile, yalnızca main branch'ine yapılan push'lar Pipeline'ı başlatır.

### Hata Durumunda İşlemler

Webhook tetiklemesi sırasında sorunlarla karşılaşılabilir. Daha önceki kısımlarda gösterildiği gibi ngrok web arayüzü kullanılarak bu sorunlar teşhis edilebilir. Ayrıca Jenkins logu da sorunun nereden kaynaklandığını aydınlatır.

Örneğin, Jenkinsfile söz dizim hatası varsa, Jenkins logu şöyle görünür:

```
org.codehaus.groovy.runtime.InvokerHelper.groovy.ScriptException: groovy.lang.MissingPropertyException: No such property: env.BRANCH_NAME
```

Bu durumda, Jenkinsfile'da uygun değişkeni kontrol etmeli, kullanmadan önce null kontrolü yapılmalı.

Başka bir örnek: GitHub webhook'u Jenkins'e ulaşamıyorsa (firewall nedeniyle), GitHub arayüzünde webhook'un durumu "red" olur. GitHub, "Delivery failed" mesajı verir ve kaç defa denediğini, hangi HTTP status kodu aldığını gösterir.

---

## 9. Webhook Güvenliği ve İyi Uygulamalar

### Webhook Payload'ı İmzalama

Webhook'u güvenli bir şekilde uygulamak için, GitHub webhook'unun meşru taraf tarafından gönderilip gönderilmediğini kontrol etmek önemlidir. GitHub, webhook oluştururken bir "Secret" (gizli) alan sağlar.

GitHub, webhook payload'ını gönderirken, bu sırrı kullanarak HMAC imzası oluşturur ve HTTP başlığının `X-Hub-Signature-256` alanına koyar. Jenkins, bu imzayı kontrol ederek, webhook'un GitHub'ın kendi tarafından gönderildiğini doğrular.

Jenkins'de, "GitHub" plugin'i varsayılan olarak bu doğrulamayı yapar, eğer Secret yapılandırıldıysa. Doğrulama başarısız olursa, webhook gözardı edilir.

### Webhook URL'sinin Gizlenmesi

Webhook URL'nin bilinmesi, bir saldırganın Jenkins'e yetkisiz istekler göndermesini mümkün kılabilir. Örneğin, `https://jenkins.example.com/github-webhook/` açıksa, saldırgan bu URL'ye POST isteği göndererek, yanlış build işlemleri başlatabilir.

Bu riski azaltmak için:

1. Webhook URL'sinde rasgele bir token eklenebilir. Örneğin: `https://jenkins.example.com/github-webhook/abc12345xyz789`
2. Webhook URL'si, GitHub API kullanılarak otomatik olarak oluşturulabilir. Jenkins Plugin'i, Jenkins'in konfigürasyonu yapıldığında, GitHub'da webhook'u otomatik olarak kaydedebilir.

### Webhook Frequency Limiting

Saldırganın webhook'u çok sık ​​tetiklemesi, Jenkins'in kaynaklarını tüketebilir. Bu durumda, Jenkins'de "throttle" (kısıtla) mekanizmazı kullanılabilir. Bu sayede, belirli bir süre içinde yalnızca belirli sayıda Pipeline çalıştırılır.

```groovy
pipeline {
    triggers {
        githubPush()
    }
    
    options {
        throttle(
            categories: ['user-service-builds'],
            throttleMatches: 1,
            throttleTimeoutInMinutes: 5
        )
    }
    
    stages {
        // ...
    }
}
```

Bu yapılandırmada, her 5 dakikada maksimum 1 build çalıştırılabilir.

---


