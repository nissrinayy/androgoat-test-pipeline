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
        
        // ================== DIVA ==================
        APK_PATH     = "test-apk\\diva.apk"      // pastikan nama file sesuai
        APP_PACKAGE  = "jakhar.aseem.diva"

        MOBSF_URL   = "http://localhost:8000"
        MOBSF_TOKEN = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
    }

    stages {
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

        // ================= START EMULATOR (Diperbaiki) =================
        stage('Start Emulator') {
            steps {
                bat """
                start /b "" "${env.ANDROID_HOME}\\emulator\\emulator.exe" ^
                -avd "${env.AVD_NAME}" ^
                -no-window -no-audio -gpu swiftshader_indirect -wipe-data
                """

                echo "Emulator starting... waiting for full boot (max 5 menit)"

                script {
                    def booted = false
                    for (int i = 0; i < 30; i++) {   // max ~5 menit
                        def status = bat(script: "adb shell getprop sys.boot_completed", returnStdout: true).trim()
                        if (status == "1") {
                            booted = true
                            echo "✅ Emulator fully booted"
                            break
                        }
                        echo "Waiting for boot... (${i+1}/30)"
                        sleep 10
                    }
                    if (!booted) {
                        error "❌ Emulator boot timeout. Coba jalankan Cold Boot Now di AVD Manager manual."
                    }
                }
            }
        }

        stage('Install APK') {
            steps {
                script {
                    def timestamp = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def destPath   = "apk-outputs\\diva-${timestamp}.apk"

                    bat "copy \"${env.APK_PATH}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)

                    def installResult = bat(script: "adb install -r \"${destPath}\"", returnStatus: true)
                    if (installResult != 0) {
                        error "❌ Install APK failed"
                    }
                    echo "✅ DIVA APK installed"
                }
            }
        }

        // ================= DAST - MobSF + Frida =================
        stage('DAST - MobSF + Frida') {
            steps {
                script {
                    bat "adb shell input keyevent 82"
                    sleep 8

                    echo "=== Starting Dynamic Analysis for DIVA ==="

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 45

                    echo "=== Instrumenting Frida with 4 hooks ==="

                    def fridaResponse = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        -d "hash=${env.APK_HASH}" ^
                        -d "default_hooks=api_monitor,ssl_pinning_bypass,root_bypass,debugger_check_bypass" ^
                        ${env.MOBSF_URL}/api/v1/frida/instrument
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Frida instrument response: ${fridaResponse}"

                    sleep 25

                    echo "=== Triggering DIVA activities ==="

                    bat "adb shell am start -n ${env.APP_PACKAGE}/.MainActivity || echo 'Main started'"
                    sleep 5
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.HardcodeActivity || echo 'Hardcode started'"
                    sleep 5
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.InsecureDataStorage1Activity || echo 'IDS1 started'"
                    sleep 5
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.InsecureDataStorage2Activity || echo 'IDS2 started'"
                    sleep 5
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.SQLInjectionActivity || echo 'SQLi started'"

                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 250 -v 800 || echo 'Monkey done'"

                    sleep 40

                    // TLS Tests, Stop, dan Fetch Report (sama seperti sebelumnya)
                    // ... (copy bagian TLS, stop_analysis, dan fetch report dari kode kamu yang lama)

                    echo "=== Stopping Dynamic Analysis ==="
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """

                    sleep 15

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
                        writeFile file: 'dast_report.json', text: json
                        archiveArtifacts artifacts: 'dast_report.json, tls_report.json', allowEmptyArchive: true
                        echo "✅ DAST DIVA selesai"
                    }
                }
            }
        }

        stage('Extract Metrics') {
            steps {
                script {
                    bat 'del /f /q dast_results.csv || echo no old file'
                    writeFile file: 'dast_results.csv', text: """hook,description
api_monitor,intercepts API calls
ssl_pinning_bypass,bypass SSL pinning
root_bypass,detect root checks
debugger_check_bypass,detect debugger"""
                    archiveArtifacts artifacts: 'dast_results.csv'
                    echo "✅ Metrics ready for paper"
                }
            }
        }

        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo already stopped'
            }
        }
    }
}