pipeline {
    agent any

    environment {
        DOCKER_IMAGE    = 'techstore-app'
        DOCKER_HUB_USER = 'berbatadam'          // ← Buraya Docker Hub kullanıcı adını yaz
        SONAR_HOST      = 'http://localhost:9000'
        SONAR_TOKEN     = credentials('sonar-token')
        SLACK_CHANNEL   = '#devops-techstore'
    }

    stages {

        // ── 1. KAYNAK KOD ───────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "✅ Kod GitHub'dan alındı: ${env.GIT_COMMIT?.take(7)}"
            }
        }

        // ── 2. ORTAM KURULUMU ───────────────────────────────────
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
                echo "✅ Python sanal ortamı hazır"
            }
        }

        // ── 3. BİRİM TESTLERİ ──────────────────────────────────
        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    mkdir -p test-results
                    PYTHONPATH=. pytest tests/test_app.py \
                        -v \
                        --tb=short \
                        --junit-xml=test-results/unit-tests.xml \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                '''
            }
            post {
                always {
                    junit 'test-results/unit-tests.xml'
                    publishCoverage adapters: [coberturaAdapter('coverage.xml')]
                }
            }
        }

       stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            withEnv(["PATH+SONAR=${tool 'sonarqube scanner'}/bin"]) {
                sh '''
                    sonar-scanner \
                        -Dsonar.projectKey=techstore \
                        -Dsonar.projectName="TechStore E-Commerce" \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=venv/**,tests/**,**/__pycache__/** \
                        -Dsonar.python.coverage.reportPaths=coverage.xml \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_TOKEN
                '''
            }
        }
    }
}

        // ── 5. KALİTE KAPISI ───────────────────────────────────
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
                echo "✅ SonarQube kalite kapısı geçildi"
            }
        }

        // ── 6. DOCKER İMAJI ─────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                        -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} \
                        -t ${DOCKER_IMAGE}:latest \
                        --build-arg BUILD_DATE=\$(date -u +%Y-%m-%dT%H:%M:%SZ) \
                        --build-arg GIT_COMMIT=${env.GIT_COMMIT?.take(7)} \
                        .
                """
                echo "✅ Docker imajı oluşturuldu: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}"
            }
        }

        // ── 7. DOCKER HUB'A GÖNDER ──────────────────────────────
        // ── 7. DOCKER HUB'A GÖNDER ──────────────────────────────
stage('Push to Docker Hub') {
    steps {
        script {
            try {
                sh 'docker --version'
                
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    // Çift tırnağa ( """) geçtik, böylece Jenkins değişkenleri doğrudan okuyacak
                    sh """
                        echo "Logging in to Docker Hub..."
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                        
                        echo "Tagging images..."
                        docker tag techstore-app:latest ${DOCKER_USER}/techstore-app:${BUILD_NUMBER}
                        docker tag techstore-app:latest ${DOCKER_USER}/techstore-app:latest
                        
                        echo "Pushing images..."
                        docker push ${DOCKER_USER}/techstore-app:${BUILD_NUMBER}
                        docker push ${DOCKER_USER}/techstore-app:latest
                    """
                }
                echo "✅ İmaj Docker Hub'a yüklendi"
            } catch (Exception e) {
                echo "❌ Docker Hub aşamasında hata yakalandı: ${e.getMessage()}"
                currentBuild.result = 'FAILURE'
                error("Docker Hub push adımı başarısız oldu.")
            }
        }
    }
}

        stage('Deploy') {
    steps {
        sh '''
            echo "Eski konteynerler temizleniyor..."
            docker stop techstore-app || true
            docker rm techstore-app || true
            
            echo "Yeni konteyner ortak ağda (techstore-network) başlatılıyor..."
            docker run -d \
              --name techstore-app \
              --network techstore-network \
              --restart unless-stopped \
              -p 5000:5000 \
              berbatadam/techstore-app:latest
        '''
    }
}

stage('Smoke Test') {
    steps {
        sh '''
            echo "Gunicorn'un ayağa kalkması için 5 saniye bekleniyor..."
            sleep 5
            
            echo "Uygulama sağlık durumu kontrol ediliyor..."
            # KRİTİK DEĞİŞİKLİK: localhost yerine techstore-app yazıyoruz!
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://techstore-app:5000/health || echo "000")
            
            echo "Gelen HTTP Kodu: $STATUS"
            if [ "$STATUS" -eq 200 ]; then
                echo "✅ Harika! Smoke test başarıyla geçildi, uygulama aktif."
            else
                echo "❌ Smoke test başarısız! Konteyner logları:"
                docker logs techstore-app
                exit 1
            fi
        '''
    }
}

        // ── 10. UI TESTLERİ ─────────────────────────────────────
        stage('UI Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    PYTHONPATH=. pytest tests/test_ui.py -v --tb=short || true
                '''
            }
        }
    }

    // ── POST ACTIONS ────────────────────────────────────────────
    post {
        success {
            echo "🎉 Pipeline başarıyla tamamlandı!"
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: """
✅ *TechStore Deploy Başarılı*
• Branch: `${env.BRANCH_NAME}`
• Build: `#${env.BUILD_NUMBER}`
• Commit: `${env.GIT_COMMIT?.take(7)}`
• URL: ${env.BUILD_URL}
                """
            )
        }
        failure {
            echo "❌ Pipeline başarısız!"
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: """
❌ *TechStore Deploy Başarısız*
• Branch: `${env.BRANCH_NAME}`
• Build: `#${env.BUILD_NUMBER}`
• Aşama: ${env.STAGE_NAME}
• Detay: ${env.BUILD_URL}console
                """
            )
        }
        always {
            sh "docker image prune -f --filter 'until=72h' || true"
            cleanWs()
        }
    }
}
