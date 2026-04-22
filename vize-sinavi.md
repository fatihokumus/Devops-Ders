### 2025-2026 Bahar Dönemi Vize Sınavı Soru-Cevap 
---

### Soru 1 

Bir ekipte iki geliştirici aynı anda farklı özellikler geliştiriyor. Özellik dalını `main` ile güncellemek için `merge` yerine `rebase` kullandığınızda ne kazanırsınız, ne kaybedersiniz? Hangi durumda `rebase` tercih edilmemeli, neden?

**Cevap:**

`rebase`, commit geçmişini doğrusal tutar ve ekstra bir merge commit oluşturmaz; geçmişi takip etmek kolaylaşır. Ancak commit hash'lerini değiştirdiği için **paylaşılan (ortak) dallarda kullanılmamalıdır.** `main` veya `develop` gibi herkesin kullandığı bir dal rebase edilirse diğer geliştiricilerin yerel kopyalarıyla geçmiş uyuşmazlığı oluşur.

---

### Soru 2 

Jenkins'te build adımlarını arayüzden (freestyle job) yapılandırmak yerine `Jenkinsfile` yazmanın, özellikle bir DevOps ekibi için ne gibi somut avantajları vardır? En az iki farklı açıdan düşününüz.

**Cevap:**

**1. Versiyon Kontrolü:** `Jenkinsfile` depoda saklandığı için pipeline tanımı da kod gibi izlenebilir, geri alınabilir ve dal bazlı farklılaştırılabilir.

**2. Taşınabilirlik:** Arayüzden yapılan yapılandırmalar Jenkins sunucusuna özgüdür; sunucu değiştiğinde kaybolur. `Jenkinsfile` varsa yeni bir ortamda projeyi bağlamak yeterlidir.

---

### Soru 3 

Aşağıdaki iki Dockerfile sıralamasını inceleyin:

**A versiyonu:**
```dockerfile
COPY . /app
RUN mvn package
```

**B versiyonu:**
```dockerfile
COPY pom.xml /app
RUN mvn dependency:go-offline
COPY src /app/src
RUN mvn package
```

B versiyonu neden build süresi açısından daha avantajlıdır? Docker'ın katman sistemi bu farka nasıl katkı sağlar?

**Cevap:**

Docker her komutu ayrı bir katman olarak cache'ler. A versiyonunda herhangi bir kaynak dosya değiştiğinde `COPY . /app` katmanı geçersiz hale gelir ve bağımlılıklar her seferinde yeniden indirilir. B versiyonunda `pom.xml` değişmediği sürece bağımlılık katmanı cache'den gelir; yalnızca `src/` değiştiğinde son derleme adımı yeniden çalışır.

---

### Soru 4 

Bir uygulamanın Docker imajı başarıyla oluşturulmuş (`build` tamamlanmış) olması, o uygulamanın production ortamına alınmaya hazır olduğu anlamına gelir mi? Bu üç kavramı (build, release, deploy) birbirinden ayıran temel farkı açıklayınız.

**Cevap:**

Hayır. Bu üç kavram birbirinin devamıdır ama birbirinin yerini tutmaz:

- **Build:** Kaynak koddan çalıştırılabilir çıktı (JAR, imaj) üretmektir.
- **Release:** O çıktının belirli bir ortama gönderilmeye hazır olarak işaretlenmesidir; versiyon etiketleme ve onay süreçlerini içerir.
- **Deploy:** Release edilmiş sürümün hedef ortamda ortam değişkenleri, sağlık kontrolü ve rollback planıyla birlikte fiilen çalıştırılmasıdır.

---

### Soru 5

Jenkins'i her 5 dakikada bir GitHub'ı kontrol edecek şekilde (polling) yapılandırdığınızı düşünün. 50 geliştirici ve 20 proje deposu olan bir organizasyonda bu yapılandırmanın yol açacağı iki somut sorunu açıklayınız. Webhook bu sorunları nasıl çözer?

**Cevap:**

**Sorun 1 — Gereksiz Ağ Yükü:** 20 depo için günde ~5.760 sorgu gönderilir; büyük çoğunluğu "değişiklik yok" yanıtıyla döner ve GitHub API rate limit'e takılabilir.

**Sorun 2 — Gecikmiş Geri Bildirim:** Kod gönderildikten sonra Jenkins'in fark etmesi için en fazla 5 dakika beklemek gerekir; geliştirici bu sürede başka işe geçmiş olabilir.

Webhook'ta GitHub olayın gerçekleştiği anda Jenkins'e HTTP POST gönderir; gecikme milisaniyeler düzeyindedir ve gereksiz sorgu trafiği ortadan kalkar.

---

### Soru 6

Aşağıda özellikleri verilen bir Java Spring Boot projesini ele alınız. Bu proje için eksiksiz bir `Jenkinsfile` yazınız.

**Proje Özellikleri:**
- Build aracı: Maven
- Test altyapısı: JUnit (Maven test aşamasıyla çalışır)
- Docker Hub'a imaj gönderilecek
- `main` dalına push yapıldığında `production` ortamına, `develop` dalına push yapıldığında `staging` ortamına deploy edilecek
- Başarısız bir aşamada pipeline durmalı

---

**Beklenen Cevap:**

```groovy
pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = "dockerhub-kullanici-adi/user-service"
        DOCKER_TAG      = "${GIT_COMMIT[0..6]}"
        DOCKERHUB_CRED  = credentials('dockerhub-credentials-id')
        STAGING_HOST    = 'deploy@staging.example.com'
        PRODUCTION_HOST = 'deploy@production.example.com'
    }

    triggers {
        githubPush()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} ."
            }
        }

        stage('Docker Push') {
            steps {
                sh """
                    echo ${DOCKERHUB_CRED_PSW} | docker login -u ${DOCKERHUB_CRED_USR} --password-stdin
                    docker push ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                    docker logout
                """
            }
        }

        stage('Deploy — Staging') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(credentials: ['staging-ssh-credentials-id']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${env.STAGING_HOST} '
                            docker pull ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} &&
                            docker stop user-service || true &&
                            docker rm   user-service || true &&
                            docker run -d --name user-service -p 8080:8080 \
                                --env-file /etc/user-service/staging.env \
                                ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                        '
                    """
                }
            }
        }

        stage('Deploy — Production') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['production-ssh-credentials-id']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${env.PRODUCTION_HOST} '
                            docker pull ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} &&
                            docker stop user-service || true &&
                            docker rm   user-service || true &&
                            docker run -d --name user-service -p 8080:8080 \
                                --env-file /etc/user-service/production.env \
                                ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                        '
                    """
                }
            }
        }
    }
}
```
