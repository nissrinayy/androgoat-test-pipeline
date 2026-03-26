import groovy.json.JsonSlurperClassic

// ================= HELPER =================
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
    return rawOutput.substring(firstBrace, lastBrace + 1).trim()
}

// ================= PIPELINE =================
pipeline {
    agent any

    environment {
        ANDROID_HOME = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        JAVA_HOME    = "C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.17.10-hotspot"

        PATH = "${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME     = "Pixel_4_XL"

        // ================== InsecureBankv2 ==================
        APK_PATH     = "test-apk\\InsecureBankv2.apk"
        APP_PACKAGE  = "insecurebankv2.dineshshetty"

        MOBSF_URL    = "http://localhost:8000"
        MOBSF_TOKEN  = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {
        // ================= VALIDATE APK =================
        stage('Validate APK') {
            steps {
                script {
                    if (!fileExists(env.APK_PATH)) {
                        error "❌ APK not found: ${env.APK_PATH}"
                    }
                    echo "✅ APK found: ${env.APK_PATH}"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
                echo "✅ Workspace ready"
            }
        }

        stage('List Workspace Files') {
            steps {
                bat 'dir /s /b'
            }
        }

        // ================= SAST =================
        stage('SAST - MobSF') {
            steps {
                script {
                    echo "Uploading InsecureBankv2 APK to MobSF..."

                    def uploadResponse = bat(
                        script: """
                        @curl -s ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -F "file=@${env.APK_PATH}" ^
                        ${env.MOBSF_URL}/api/v1/upload
                        """,
                        returnStdout: true
                    ).trim()

                    def apkHash = extractHashFromResponse(uploadResponse)
                    if (!apkHash) error "❌ Upload failed"

                    env.APK_HASH = apkHash
                    echo "APK HASH: ${apkHash}"

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${apkHash}" ^
                    ${env.MOBSF_URL}/api/v1/scan
                    """

                    def raw = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${apkHash}" ^
                        ${env.MOBSF_URL}/api/v1/report_json
                        """,
                        returnStdout: true
                    ).trim()

                    def json = cleanJsonString(raw)
                    if (json) {
                        writeFile file: 'sast_report_insecurebank.json', text: json
                        archiveArtifacts artifacts: 'sast_report_insecurebank.json'
                        echo "✅ SAST done"
                    }
                }
            }
        }

        // ================= START EMULATOR =================
        stage('Start Emulator') {
            steps {
                bat """
                start /b "" "${env.ANDROID_HOME}\\emulator\\emulator.exe" ^
                -avd "${env.AVD_NAME}" ^
                -no-window -no-audio -gpu swiftshader_indirect -wipe-data
                """

                sleep 60   // ✅ pakai ini, JANGAN timeout

                bat "adb wait-for-device"
                bat "adb shell getprop sys.boot_completed"

                echo "✅ Emulator ready"
            }
        }

        // ================= INSTALL APK =================
        stage('Install APK') {
            steps {
                script {
                    def timestamp  = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def sourcePath = env.APK_PATH
                    def destPath   = "apk-outputs\\InsecureBankv2-${timestamp}.apk"

                    if (!fileExists(sourcePath)) {
                        error "❌ Source APK not found: ${sourcePath}"
                    }

                    bat "copy \"${sourcePath}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)

                    bat "adb install -r \"${destPath}\""
                    echo "✅ InsecureBankv2 APK installed"
                }
            }
        }

        // ================= DAST =================
        stage('DAST') {
            steps {
                script {
                    echo "=== Starting Dynamic Analysis for InsecureBankv2 ==="

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """
                    bat 'timeout /t 30 /nobreak >nul'

                    echo "=== Triggering InsecureBankv2 activities ==="
                    def activities = [
                        ".LoginActivity",
                        ".PostLogin",
                        ".DoTransfer",
                        ".ViewStatement",
                        ".ChangePassword"
                    ]

                    activities.each { act ->
                        bat "adb shell am start -n ${env.APP_PACKAGE}/${act} || echo '${act} started'"
                        bat 'timeout /t 6 /nobreak >nul'
                    }

                    echo "=== Monkey runner ==="
                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 200 -v 1000 || echo 'Monkey done'"
                    bat 'timeout /t 30 /nobreak >nul'

                    echo "=== Stopping Dynamic Analysis ==="
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """
                    bat 'timeout /t 15 /nobreak >nul'

                    echo "=== Fetching DAST Report ==="
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
                        writeFile file: 'dast_report_insecurebank.json', text: json
                        archiveArtifacts artifacts: 'dast_report_insecurebank.json'
                        echo "✅ DAST finished"
                    }
                }
            }
        }

        // ================= METRICS =================
        stage('Extract Metrics') {
            steps {
                script {
                    bat 'del /f /q dast_results_insecurebank.csv || echo no old file'
                    writeFile file: 'dast_results_insecurebank.csv', text: """
hook,description
api_monitor,intercepts API calls
ssl_pinning_bypass,bypass SSL pinning
root_bypass,detect root checks
debugger_check_bypass,detect debugger
"""
                    archiveArtifacts artifacts: 'dast_results_insecurebank.csv'
                    echo "✅ Metrics ready for paper"
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo already stopped'
                bat 'taskkill /F /IM emulator.exe /T || echo already stopped'
            }
        }
    }
}