# Bölüm 6  
# `CICDOrnek` Repository ile Sürekli Teslimat ve Deployment Uygulama Örneği

Bu bölümde uygulama örneği doğrudan `https://github.com/fatihokumus/CICDOrnek` repository'si üzerinden yürütülmektedir. Proje, yemek sipariş senaryosuna dayalı iki katmanlı bir yapıya sahiptir:

- `backend/`: Java Spring Boot REST API
- `frontend/`: React + Vite kullanıcı arayüzü

Örnek uygulama aşağıdaki işlevleri içerir:

- login ekranı
- menü listeleme
- sepete ürün ekleme ve adet güncelleme
- sipariş oluşturma
- son siparişleri görüntüleme

Demo kullanıcı bilgileri:

- kullanıcı adı: `demo`
- şifre: `demo123`

Bu bölümün amacı, önceki bölümde açıklanan kavramsal çerçevenin aynı proje üzerinde adım adım nasıl işletildiğini göstermektir.

## 6.1 Uygulama Mimarisi ve Çalışma Davranışı

Repository kökünde üç temel çalışma birimi vardır:

- `frontend` servisi: Nginx üzerinde statik React çıktısını sunar ve `/api` çağrılarını backend'e iletir
- `backend` servisi: Spring Boot API, login/menü/sipariş uç noktalarını sağlar
- `postgres` servisi: Docker Compose içinde bağımlı veritabanı servisi olarak tanımlanır

Kritik yapılandırmalar:

- frontend container portu: `4242`
- backend container portu: `4343`
- PostgreSQL host portu: `5544`

Yerel geliştirme kipinde frontend, Vite üzerinden `5173` portunda çalışır ve `/api` çağrılarını `http://localhost:8080` adresine proxy'ler. Compose kipinde ise frontend Nginx üzerinden `http://backend:4343` adresine yönlenir.

## 6.2 Aşama 1: Yerel Çalıştırma ve Başlangıç Durumu

İlk aşamada sistem deployment hattına girmeden önce geliştirici makinesinde doğrulanır.

### 6.2.1 Backend'in Yerelde Çalıştırılması

Başlangıç durumu:  
Sadece kaynak kod bulunmaktadır.

Yapılan değişiklik:  
Spring Boot servis yerelde ayağa kaldırılır.

İlgili komut:

```bash
cd backend
mvn spring-boot:run
```

Varsayılan erişim:

- backend API: `http://localhost:8080`

Bu adımın süreç içindeki anlamı:  
API geliştirme ve hızlı geri bildirim döngüsü sağlanır.

### 6.2.2 Frontend'in Yerelde Çalıştırılması

Başlangıç durumu:  
Arayüz kaynak kodu statik durumdadır.

Yapılan değişiklik:  
React uygulaması Vite geliştirme sunucusunda çalıştırılır.

İlgili komut:

```bash
cd frontend
npm install
npm run dev
```

Varsayılan erişim:

- frontend: `http://localhost:5173`

Bu adımın süreç içindeki anlamı:  
UI ve API entegrasyonu geliştirme modunda hızlı biçimde test edilir.

## 6.3 Spring Boot Yapılandırmasının Deployment Açısından Okunması

Bu repository'de backend yapılandırması environment variable odaklıdır ve deployment için uygundur.

Başlangıç durumu:  
Port ve veritabanı bilgileri dışarıdan verilmezse varsayılan değerler kullanılır.

Yapılan değişiklik:  
Ortam bazlı değerler deployment komutlarında dışarıdan geçirilir.

İlgili dosya:

```properties
spring.application.name=yemek-siparis
server.port=${SERVER_PORT:8080}
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5544/yemek_siparis}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:yemek_user}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:yemek_pass}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.hikari.initialization-fail-timeout=0
```

Bu adımın süreç içindeki anlamı:  
Aynı backend imajı staging ve production ortamlarında farklı bağlantı bilgileriyle yeniden kullanılabilir.

## 6.4 Docker Run ile Tek Tek Servis Çalıştırma

Bu adımda compose öncesi düşük seviyeli deployment davranışı görünür kılınır.

Başlangıç durumu:  
Servisler yalnızca geliştirme komutlarıyla açılmaktadır.

Yapılan değişiklik:  
Servisler Docker imajı olarak build edilip ayrı ayrı container şeklinde çalıştırılır.

İlgili komut:

```bash
docker network create yemek-net

docker run -d \
  --name postgres \
  --network yemek-net \
  -e POSTGRES_DB=yemek_siparis \
  -e POSTGRES_USER=yemek_user \
  -e POSTGRES_PASSWORD=yemek_pass \
  -p 5544:5432 \
  postgres:16-alpine

docker build -t yemek-backend:local ./backend
docker run -d \
  --name backend \
  --network yemek-net \
  -p 4343:4343 \
  -e SERVER_PORT=4343 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/yemek_siparis \
  -e SPRING_DATASOURCE_USERNAME=yemek_user \
  -e SPRING_DATASOURCE_PASSWORD=yemek_pass \
  yemek-backend:local

docker build -t yemek-frontend:local ./frontend
docker run -d \
  --name frontend \
  --network yemek-net \
  -p 4242:4242 \
  yemek-frontend:local
```

Bu adımın süreç içindeki anlamı:  
Deployment sürecinin container, ağ ve environment variable düzeyindeki mekanikleri açık biçimde gözlenir.

## 6.5 Aşama 2: Docker Compose ile Bütünleşik Çalıştırma

Repository'nin önerdiği standart çalışma şekli compose tabanlıdır.

Başlangıç durumu:  
Servisler tek tek komutlarla yönetilmektedir.

Yapılan değişiklik:  
Üç servis tek dosyadan orkestre edilir.

İlgili dosya (`docker-compose.yml`):

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: yemek-postgres
    environment:
      POSTGRES_DB: yemek_siparis
      POSTGRES_USER: yemek_user
      POSTGRES_PASSWORD: yemek_pass
    ports:
      - "5544:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U yemek_user -d yemek_siparis"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
    container_name: yemek-backend
    environment:
      SERVER_PORT: 4343
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/yemek_siparis
      SPRING_DATASOURCE_USERNAME: yemek_user
      SPRING_DATASOURCE_PASSWORD: yemek_pass
    ports:
      - "4343:4343"
    depends_on:
      postgres:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
    container_name: yemek-frontend
    ports:
      - "4242:4242"
    depends_on:
      - backend

volumes:
  postgres_data:
```

İlgili komut:

```bash
docker compose up --build
```

Yayınlanan portlar:

- frontend: `http://localhost:4242`
- backend: `http://localhost:4343`
- PostgreSQL: `localhost:5544`

Bu adımın süreç içindeki anlamı:  
Geliştirme ve test ortamlarında tekrar üretilebilir bir çok servisli çalışma modeli sağlanır.

## 6.6 Aşama 3: Jenkins ile Image Üretimi, Registry Push ve Uzak Sunucuya Deployment

Bu aşamada compose tabanlı yerel akış, CI/CD hattına taşınır.

### 6.6.1 Monorepo İçin Image Sürümleme Yaklaşımı

Bu repository iki ayrı deploy edilebilir birim üretir:

- backend image
- frontend image

Her ikisi için aynı build numarasını ve commit SHA bilgisini kullanan etiket yapısı önerilir:

- `registry.example.com/yemek-backend:<BUILD>-<SHA>`
- `registry.example.com/yemek-frontend:<BUILD>-<SHA>`

### 6.6.2 Registry Push Komutları

Başlangıç durumu:  
İmajlar yalnızca Jenkins ajanında veya yerelde bulunur.

Yapılan değişiklik:  
İmajlar merkezi registry'ye gönderilir.

İlgili komut:

```bash
docker build -t registry.example.com/yemek-backend:58-a1b2c3d ./backend
docker build -t registry.example.com/yemek-frontend:58-a1b2c3d ./frontend

docker push registry.example.com/yemek-backend:58-a1b2c3d
docker push registry.example.com/yemek-frontend:58-a1b2c3d
```

Bu adımın süreç içindeki anlamı:  
Deployment sunucusu koddan build almak yerine doğrulanmış imajı çeker.

### 6.6.3 Jenkinsfile Örneği

Başlangıç durumu:  
Pipeline sadece tek servis veya yalnızca build odaklı kurgulanmıştır.

Yapılan değişiklik:  
Backend ve frontend için paralel build/push, staging deploy, production onayı ve production deploy adımları eklenir.

İlgili dosya örneği:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    tools {
        maven 'M3'
    }

    environment {
        STAGING_HOST = 'deploy@staging-server'
        PRODUCTION_HOST = 'deploy@prod-server'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/fatihokumus/CICDOrnek.git'

                script {
                    env.GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.VERSION_TAG = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
                    env.BACKEND_IMAGE = "yemek-backend:${env.VERSION_TAG}"
                    env.FRONTEND_IMAGE = "yemek-frontend:${env.VERSION_TAG}"
                }
            }
        }

        stage('Test Backend') {
            steps {
                dir('backend') {
                    sh 'mvn -B test'
                }
            }
        }

        stage('Build Images') {
            parallel {
                stage('Build Backend Image') {
                    steps {
                        sh 'docker build -t $BACKEND_IMAGE ./backend'
                    }
                }
                stage('Build Frontend Image') {
                    steps {
                        sh 'docker build -t $FRONTEND_IMAGE ./frontend'
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                    ssh $STAGING_HOST "
                      export BACKEND_IMAGE=$BACKEND_IMAGE &&
                      export FRONTEND_IMAGE=$FRONTEND_IMAGE &&
                      cd /opt/yemek-siparis &&
                      docker compose -f docker-compose.deploy.yml up -d --remove-orphans
                    "
                '''
            }
        }

        stage('Staging Validation') {
            steps {
                sh '''
                    ssh $STAGING_HOST "
                      curl -fsS http://localhost:4343/api/menu > /dev/null &&
                      curl -fsS http://localhost:4242 > /dev/null
                    "
                '''
            }
        }

        stage('Production Approval') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Production deployment onayı verilsin mi?', ok: 'Deploy'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    ssh $PRODUCTION_HOST "
                      export BACKEND_IMAGE=$BACKEND_IMAGE &&
                      export FRONTEND_IMAGE=$FRONTEND_IMAGE &&
                      cd /opt/yemek-siparis &&
                      docker compose -f docker-compose.deploy.yml up -d --remove-orphans
                    "
                '''
            }
        }

        stage('Production Validation') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    ssh $PRODUCTION_HOST "
                      curl -fsS http://localhost:4343/api/menu > /dev/null &&
                      curl -fsS -X POST http://localhost:4343/api/auth/login \
                        -H 'Content-Type: application/json' \
                        -d '{\"username\":\"demo\",\"password\":\"demo123\"}' > /dev/null &&
                      curl -fsS http://localhost:4242 > /dev/null
                    "
                '''
            }
        }
    }
}
```

Bu adımın süreç içindeki anlamı:  
Monorepo yapısındaki iki servis tek bir pipeline içinde kontrollü olarak yayımlanır.

### 6.6.4 Uzak Sunucu İçin Deploy Compose Dosyası

Başlangıç durumu:  
Repository kökündeki `docker-compose.yml` dosyası `build` odaklıdır.

Yapılan değişiklik:  
Uzak sunucuda build yerine registry'den image çekmek için ayrı deployment dosyası kullanılır.

İlgili dosya (`docker-compose.deploy.yml`) örneği:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: yemek-postgres
    environment:
      POSTGRES_DB: yemek_siparis
      POSTGRES_USER: yemek_user
      POSTGRES_PASSWORD: yemek_pass
    ports:
      - "5544:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  backend:
    image: ${BACKEND_IMAGE}
    container_name: yemek-backend
    environment:
      SERVER_PORT: 4343
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/yemek_siparis
      SPRING_DATASOURCE_USERNAME: yemek_user
      SPRING_DATASOURCE_PASSWORD: yemek_pass
    ports:
      - "4343:4343"
    depends_on:
      - postgres

  frontend:
    image: ${FRONTEND_IMAGE}
    container_name: yemek-frontend
    ports:
      - "4242:4242"
    depends_on:
      - backend

volumes:
  postgres_data:
```

Bu adımın süreç içindeki anlamı:  
Sunucu tarafında kaynak koddan build alma zorunluluğu ortadan kalkar; doğrudan versiyonlu image ile deployment yapılır.

### 6.6.5 Uzak Sunucu Deployment Komutları

Başlangıç durumu:  
Sunucuda güncelleme elle yapılmaktadır.

Yapılan değişiklik:  
Jenkins SSH üzerinden versiyonlu imajları hedef sunucuda etkinleştirir.

İlgili komut:

```bash
ssh deploy@staging-server "
  export BACKEND_IMAGE=registry.example.com/yemek-backend:58-a1b2c3d &&
  export FRONTEND_IMAGE=registry.example.com/yemek-frontend:58-a1b2c3d &&
  cd /opt/yemek-siparis &&
  docker compose -f docker-compose.deploy.yml pull &&
  docker compose -f docker-compose.deploy.yml up -d --remove-orphans
"
```

Bu adımın süreç içindeki anlamı:  
Deployment süreci tekrarlanabilir hâle gelir ve manuel sunucu işlemleri azalır.

## 6.7 Deployment Sonrası Doğrulama

Bu repository'de varsayılan durumda Actuator health endpoint'i yoktur. Bu nedenle doğrulama uygulama uç noktaları üzerinden yapılır.

### 6.7.1 Backend Doğrulaması

```bash
curl -fsS http://localhost:4343/api/menu
curl -fsS http://localhost:4343/api/orders
curl -fsS -X POST http://localhost:4343/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo123"}'
```

### 6.7.2 Frontend Doğrulaması

```bash
curl -I http://localhost:4242
```

### 6.7.3 Log Doğrulaması

```bash
docker logs --tail 100 yemek-backend
docker logs --tail 100 yemek-frontend
```

Bu adımın süreç içindeki anlamı:  
Container'ın ayakta olması ile uygulamanın işlevsel olması arasındaki fark doğrulama katmanıyla kapatılır.

## 6.8 Başarısız Deployment Senaryoları

Bu projede olası hata sınıfları:

- frontend ayakta, backend erişilemez
- backend ayakta, veritabanı bağlantı parametresi yanlış
- Nginx `/api` yönlendirmesi backend servis adına çözümlenemiyor
- yanlış image tag kullanımı nedeniyle eski/yeni sürüm karışıklığı

İlk tanı komutları:

```bash
docker ps -a
docker compose -f docker-compose.deploy.yml ps
docker logs yemek-backend
docker logs yemek-frontend
```

## 6.9 Aşama 4: Rollback Uygulaması

Rollback, backend ve frontend için bir önceki kararlı image etiketine birlikte dönmeyi gerektirir.

Başlangıç durumu:  
`58-a1b2c3d` sürümü sonrası hatalar gözlenmektedir.

Yapılan değişiklik:  
Her iki servis bir önceki stabil sürüme çekilir.

İlgili komut:

```bash
ssh deploy@prod-server "
  export BACKEND_IMAGE=registry.example.com/yemek-backend:57-f9e8d7c &&
  export FRONTEND_IMAGE=registry.example.com/yemek-frontend:57-f9e8d7c &&
  cd /opt/yemek-siparis &&
  docker compose -f docker-compose.deploy.yml pull &&
  docker compose -f docker-compose.deploy.yml up -d --remove-orphans
"
```

Bu adımın süreç içindeki anlamı:  
Versioned deployment yaklaşımı sayesinde geri dönüş hızlı ve denetlenebilir olur.

## 6.10 Baştan Sona Yayın Senaryosu

Bu bölümdeki adımlar tek akışta aşağıdaki sırayı izler:

1. Geliştirici `CICDOrnek` deposuna commit gönderir.
2. Jenkins pipeline tetiklenir.
3. Backend testleri çalışır.
4. Backend ve frontend imajları build edilir.
5. İmajlar registry'ye push edilir.
6. Staging sunucusunda compose tabanlı deployment güncellenir.
7. Backend (`/api/menu`, `/api/auth/login`) ve frontend erişimi doğrulanır.
8. Production onayı verilir.
9. Aynı image etiketleri production ortamına geçirilir.
10. Hata oluşursa backend ve frontend birlikte bir önceki sürüme rollback edilir.

Bu akışta temel ilke, staging'de doğrulanan aynı backend ve frontend imajlarının production'a taşınmasıdır.

## 6.11 Değerlendirme

`CICDOrnek` örneği, tek servisli bir anlatımdan farklı olarak iki uygulama katmanının birlikte teslim edilmesi gereğini ortaya koymaktadır. Backend ve frontend'in aynı sürüm bağlamında yayımlanması, deployment doğruluğu açısından kritik bir zorunluluktur. Compose, bu birlikte çalışma modelini yerel ortamda ve sunucuda tekrarlanabilir hâle getirir.

Jenkins + Docker + registry + SSH tabanlı deployment zinciri kurulduğunda, süreç yalnızca build otomasyonundan çıkar ve uçtan uca yayın hattına dönüşür. Bu hattın güvenilirliği, doğrulama adımları ve rollback kabiliyetiyle tamamlanır. Bu nedenle CI/CD olgunluğu, yalnızca yeni sürüm çıkarma hızına değil; hatalı sürümden kontrollü şekilde geri dönebilme kapasitesine de bağlıdır.
