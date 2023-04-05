pipeline {
  agent any
  environment {
        ANDROID_HOME = "/usr/local/android-sdk"
        GRADLE_HOME = "/usr/local/gradle"
        APPCENTER_TOKEN = credentials('appcenter-token')
        NEXUS_USERNAME = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
  }
  tools { 
        maven 'Maven_3_5_2'  
    }
   stages{
    stage('CompileandRunSonarAnalysis') {
            steps {	
		sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=devsecopsproject -Dsonar.organization=devsecops2023 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=445f45ae0d23d026c8060aaa27955baa9f06bc1c'
			}
    }
	stage('Build') {
            // Compila el proyecto de Android
            steps {
                sh "${GRADLE_HOME}/bin/gradle build"
            }
    }
	stage('Test') {
            // Ejecuta las pruebas unitarias de Android
            steps {
                sh "${GRADLE_HOME}/bin/gradle test"
            }
        }
	stage('SonarQube Scan') {
		// Escanea el proyecto con SonarQube
		environment {
			scannerHome = tool 'SonarScanner'
		}
		steps {
			sh "${scannerHome}/bin/sonar-scanner"
		}
	}
	stage('Sign APK') {
		// Firma el archivo APK para su uso en producción
		steps {
			sh "echo 'Signing APK...'"
			sh "./gradlew assembleRelease --stacktrace --no-daemon"
		}
	}
	stage('Quality Gate') {
		// Evalúa el resultado del análisis de SonarQube para el Quality Gate
		steps {
			script {
				def qg = waitForQualityGate()
				if (qg.status != 'OK') {
					error "Pipeline aborted due to quality gate failure: ${qg.status}"
				}
			}
		}
	}
	stage('Upload to App Center') {
		// Sube el archivo APK a App Center si la calidad del código cumple con el Quality Gate
		when {
			expression { currentBuild.result == 'SUCCESS' }
		}
		steps {
			withCredentials([string(credentialsId: 'appcenter-token', variable: 'APPCENTER_TOKEN')]) {
				sh "echo 'Uploading APK to App Center...'"
				sh "curl -X POST -H 'Content-Type: application/octet-stream' -H 'Accept: application/json' -H 'X-API-Token: $APPCENTER_TOKEN' https://api.appcenter.ms/v0.1/apps/your-username/your-appname/releases/upload -T app/build/outputs/apk/release/app-release.apk"
			}
		}
	}
	stage('Upload to Nexus') {
		// Sube el archivo APK a Nexus si la calidad del código cumple con el Quality Gate
		when {
			expression { currentBuild.result == 'SUCCESS' }
		}
		steps {
			withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
				sh "echo 'Uploading APK to Nexus...'"
				sh "curl -v -u $NEXUS_USERNAME:$NEXUS_PASSWORD --upload-file app/build/outputs/apk/release/app-release.apk https://nexus.example.com/repository/releases/your-appname/app-release.apk"
			}
		}
    }
  }
	post {
		always {
			junit 'app/build/test-results/**/*.xml'
			archiveArtifacts 'app/build/outputs/**/*.apk'
			publishHTML target: [
				allowMissing: false,
				alwaysLinkToLastBuild: true,
				keepAll: true,
				reportDir: 'app/build/reports',
				reportFiles: 'index.html',
				reportName: 'Test Report'
			]
		}
		success {
			script {
				def qualityGate = waitForQualityGate()
				if (qualityGate.status != 'OK') {
					error "Pipeline failed due to quality gate status: ${qualityGate.status}"
				}
			}
		}
	}
}