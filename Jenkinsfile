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
    return rawOutput.substring(firstBrace, lastBrace + 1)
}

// ================= PIPELINE =================
pipeline {
    agent any

    environment {
        ANDROID_HOME = "C:\\Users\\Nisrina\\AppData\\Local\\Android\\Sdk"
        JAVA_HOME    = "C:\\Program Files\\Eclipse Adoptium\\jdk-17.0.17.10-hotspot"

        PATH = "${JAVA_HOME}\\bin;${ANDROID_HOME}\\platform-tools;${ANDROID_HOME}\\emulator;${env.PATH}"

        AVD_NAME     = "Pixel_4_XL"
        APK_PATH     = "test-apk\\AndroGoat.apk"
        APP_PACKAGE  = "owasp.sat.agoat"

        MOBSF_URL   = "http://localhost:8000"
        MOBSF_TOKEN = "67f8dcdbaf63751750653685407053c3e1762a3394c5833de1d00379ca06c0fe"
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
        // ================= PREPARE WORKSPACE =================
        stage('Prepare Workspace') {
            steps {
                bat 'if not exist apk-outputs mkdir apk-outputs'
                echo "✅ Workspace ready"
            }
        }
        // ================= DEBUG FILES =================
        stage('List Workspace Files') {
            steps {
                bat 'dir /s /b'
            }
        }
        // ================= SAST =================
        stage('SAST - MobSF') {
            steps {
                script {
                    echo "Uploading APK to MobSF..."

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
                        writeFile file: 'sast_report.json', text: json
                        archiveArtifacts artifacts: 'sast_report.json'
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

                sleep 60
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
                    def destPath   = "apk-outputs\\androgoat-${timestamp}.apk"

                    // pastikan file ada sebelum copy
                    if (!fileExists(sourcePath)) {
                        error "❌ Source APK not found: ${sourcePath}"
                    }

                    // copy APK ke folder outputs
                    bat "copy \"${sourcePath}\" \"${destPath}\""

                    // uninstall (jangan fail pipeline)
                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)

                    // install APK
                    bat "adb install -r \"${destPath}\""
                    echo "✅ APK installed"
                }
            }
        }


        // ================= DAST =================
        stage('DAST - MobSF + Frida') {
            steps {
                script {

                    bat "adb shell input keyevent 82"
                    sleep 2

                    // start dynamic
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 20

                    // frida hooks
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}&default_hooks=api_monitor,ssl_pinning_bypass,root_bypass,debugger_check_bypass" ^
                    ${env.MOBSF_URL}/api/v1/frida/instrument
                    """

                    // trigger app behavior
                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 1000 -v 300"

                    // TLS test
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

                    // stop
                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/stop_analysis
                    """

                    // get report
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
                        echo "✅ DAST done"
                    }
                }
            }
        }

        // ================= METRICS (FOR PAPER) =================
        stage('Extract Metrics') {
            steps {
                script {
                    writeFile file: 'dast_results.csv', text: """
hook,description
api_monitor,intercepts API calls
ssl_pinning_bypass,bypass SSL pinning
root_bypass,detect root checks
debugger_check_bypass,detect debugger
"""
                    archiveArtifacts artifacts: 'dast_results.csv'
                    echo "✅ Metrics ready for paper"
                }
            }
        }

        // ================= CLEANUP =================
        stage('Cleanup') {
            steps {
                bat 'taskkill /F /IM qemu-system-x86_64.exe /T || echo already stopped'
            }
        }
    }
}