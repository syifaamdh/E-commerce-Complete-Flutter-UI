import groovy.json.JsonSlurperClassic

// ================= HELPER METHODS =================
@NonCPS
def extractHashFromResponse(String response) {
    def matcher = response =~ /"hash"\s*:\s*"([^"]+)"/
    return matcher.find() ? matcher.group(1) : ""
}

@NonCPS
def cleanJsonString(String rawOutput) {
    int firstBrace = rawOutput.indexOf('{')
    int lastBrace  = rawOutput.lastIndexOf('}')
    if (firstBrace == -1 || lastBrace == -1) return null
    return rawOutput.substring(firstBrace, lastBrace + 1)
}

// ================= PIPELINE =================
pipeline {
    agent any

    parameters {
        choice(
            name: 'BUILD_TYPE',
            choices: ['debug', 'release'],
            description: 'Pilih tipe build APK (Release direkomendasikan untuk audit)'
        )
    }

    environment {
        ANDROID_HOME     = "C:\\Users\\Administrator\\AppData\\Local\\Android\\Sdk"
        ANDROID_SDK_ROOT = "${ANDROID_HOME}"
        FLUTTER_HOME     = "D:\\flutter"
        JAVA_HOME = "C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.19.10-hotspot"

        PATH = "${FLUTTER_HOME}\\bin;${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${ANDROID_HOME}\\cmdline-tools\\latest\\bin;${env.PATH}"
        AVD_NAME    = "Pixel_4_XL"
        APP_PACKAGE = "com.example.shop"

        MOBSF_URL   = "http://localhost:8000"
        MOBSF_TOKEN = "68f94303e8c795a5a7680850a034e78ccdb27cbabe84bd7af34e029c09be9057"
    }

    stages {

        // ================= CHECKOUT =================
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/syifaamdh/E-commerce-Complete-Flutter-UI.git'
            }
        }

        // ================= PREPARE =================
        stage('Prepare Workspace') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
            }
        }

        // ================= FLUTTER ENV =================
        stage('Flutter Environment Check') {
            steps {
                bat """
                git config --global --add safe.directory "%WORKSPACE%"

                set ANDROID_HOME=${env.ANDROID_HOME}
                set ANDROID_SDK_ROOT=${env.ANDROID_HOME}
                set JAVA_HOME=${env.JAVA_HOME}

                set PATH=%JAVA_HOME%\\bin;%PATH%
                set PATH=%PATH%;${env.ANDROID_HOME}\\platform-tools
                set PATH=%PATH%;${env.ANDROID_HOME}\\emulator
                set PATH=%PATH%;${env.ANDROID_HOME}\\cmdline-tools\\latest\\bin

                java -version
                flutter config --android-sdk "${env.ANDROID_HOME}"
                flutter config --jdk-dir "${env.JAVA_HOME}"

                sdkmanager --licenses
                flutter doctor -v
                """
            }
        }

        // ================= BUILD APK =================
        stage('Build APK') {
            steps {
                bat '''
                set JAVA_HOME=C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.19.10-hotspot
                set PATH=%JAVA_HOME%\\bin;%PATH%

                echo ================= JAVA =================
                echo JAVA_HOME=%JAVA_HOME%
                java -version
                where java

                echo ================= FLUTTER =================
                flutter doctor -v

                echo ================= CLEAN =================
                flutter clean

                echo ================= PUB GET =================
                flutter pub get

                echo ================= BUILD APK =================
                flutter build apk --%BUILD_TYPE% --verbose

                echo ================= APK OUTPUT =================
                dir build\\app\\outputs /s
                '''
            }
        }

        // ================= SAST =================
        stage('SAST - Static Analysis ') {
            steps {
                script {
                    def apkPath = "build/app/outputs/flutter-apk/app-${params.BUILD_TYPE}.apk"
                    if (!fileExists(apkPath)) {
                        error "APK not found: ${apkPath}"
                    }

                    echo "Uploading APK to MobSF (SAST)..."

                    def uploadResponse = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${apkPath}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    def apkHash = extractHashFromResponse(uploadResponse)
                    if (!apkHash) error "MobSF upload failed"

                    env.APK_HASH = apkHash
                    echo "SAST Hash: ${apkHash}"

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${apkHash}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """

                    def rawReport = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${apkHash}" ^
                        ${env.MOBSF_URL}/api/v1/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(rawReport)
                    if (json) {
                        writeFile file: 'sast_report.json', text: json
                        archiveArtifacts artifacts: 'sast_report.json'
                        echo "✅ SAST Report URL: ${env.MOBSF_URL}/static_analyzer/${apkHash}/"
                    }
                }
            }
        }

        // ================= EMULATOR =================
        stage('Start Emulator') {
            steps {
                bat """
                start /b "" "${env.ANDROID_HOME}\\emulator\\emulator.exe" ^
                -avd "${env.AVD_NAME}" ^
                -no-window -no-audio ^
                -gpu swiftshader_indirect -wipe-data
                """
                sleep 60
                bat "adb wait-for-device"
                bat "adb shell getprop sys.boot_completed"
            }
        }

        // ================= INSTALL APK =================
        stage('Install APK') {
            steps {
                script {
                    def timestamp  = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def sourcePath = "build\\app\\outputs\\flutter-apk\\app-${params.BUILD_TYPE}.apk"
                    def destPath = "apk-outputs\\ecommerce-${params.BUILD_TYPE}-${timestamp}.apk"

                    bat "copy \"${sourcePath}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)
                    bat "adb install -r \"${destPath}\""
                }
            }
        }

        // ================= DAST =================
        stage('DAST - Dynamic Analysis (MobSF)') {
            steps {
                script {
                    bat "adb shell input keyevent 82"
                    sleep 2

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 25

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}&default_hooks=api_monitor,ssl_pinning_bypass,root_bypass,debugger_check_bypass" ^
                    ${env.MOBSF_URL}/api/v1/frida/instrument
                    """

                    try {
                        bat "adb shell monkey -p ${env.APP_PACKAGE} --pct-syskeys 0 --throttle 1500 -v 200"
                    } catch (Exception e) {
                        echo "Monkey finished."
                    }

                    def tlsRaw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/android/tls_tests
                        """,
                        returnStdout: true
                    ).trim()

                    def tlsJson = cleanJsonString(tlsRaw)
                    if (tlsJson) {
                        writeFile file: 'tls_report.json', text: tlsJson
                    }

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """

                    def raw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        ${env.MOBSF_URL}/api/v1/dynamic/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(raw)
                    if (json) {
                        writeFile file: 'dast_report.json', text: json
                        archiveArtifacts artifacts: 'dast_report.json, tls_report.json', allowEmptyArchive: true
                        echo "✅ DAST Report URL: ${env.MOBSF_URL}/dynamic_report/${env.APK_HASH}/ "
                    }
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo Emulator already stopped'
            }
        }
stage('Download PDF Report') {
    steps {
        script {
            if (env.APK_HASH) {

                bat """
                @curl -s -X POST ^
                -H "Authorization: ${env.MOBSF_TOKEN}" ^
                --data "hash=${env.APK_HASH}" ^
                ${env.MOBSF_URL}/api/v1/download_pdf ^
                -o sast_report.pdf
                """
                echo "Downloading DAST PDF (Custom)..."
                bat """
                @curl -L -f ^
                ${env.MOBSF_URL}/dynamic_pdf/${env.APK_HASH}/ ^
                --output dast_report.pdf --silent --show-error
                """

                sleep 2
 

                bat "dir dast_report.pdf"
                bat "for %%I in (dast_report.pdf) do @echo Size: %%~zI bytes"

            } else {
                echo "APK_HASH not found."
            }
            def size = bat(
                script: '@for %%I in (dast_report.pdf) do @echo %%~zI',
                returnStdout: true
            ).trim().split("\\r?\\n")[-1]

            if (size.toInteger() < 50000) {
                error "DAST PDF INVALID: ${size} bytes"
            }
        }
    }
}

stage('Send Email Manual') {
    steps {
        writeFile file: 'send_email.ps1', text: '''
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12

$smtpServer = "smtp.gmail.com"
$smtpPort = 587
$from = "syifaamaraqueen@gmail.com"
$to = "syifaamdh@gmail.com"
$password = "lyftygsebuyndied"

$mail = New-Object System.Net.Mail.MailMessage
$mail.From = $from
$mail.To.Add($to)
$mail.Subject = "Security Report Jenkins"
$mail.Body = "Laporan PDF terlampir"

$att1 = New-Object System.Net.Mail.Attachment("sast_report.pdf")
$att2 = New-Object System.Net.Mail.Attachment("dast_report.pdf")

$mail.Attachments.Add($att1)
$mail.Attachments.Add($att2)

$smtp = New-Object System.Net.Mail.SmtpClient($smtpServer, $smtpPort)
$smtp.EnableSsl = $true
$smtp.UseDefaultCredentials = $false
$smtp.Credentials = New-Object System.Net.NetworkCredential($from, $password)
$smtp.DeliveryMethod = [System.Net.Mail.SmtpDeliveryMethod]::Network
$smtp.Timeout = 30000

$smtp.Send($mail)
'''
        bat 'powershell -ExecutionPolicy Bypass -File send_email.ps1'
    }
}
    }

    post {
        always {
            archiveArtifacts artifacts: 'apk-outputs/*.apk', allowEmptyArchive: true
            archiveArtifacts artifacts: '*.pdf', allowEmptyArchive: true
        }
    }
}
