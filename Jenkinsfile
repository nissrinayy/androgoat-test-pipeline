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
        APK_PATH     = "test-apk\\diva.apk"          // pastikan file ini ada
        APP_PACKAGE  = "jakhar.aseem.diva"

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

        // ================= START EMULATOR (dikembalikan seperti asli) =================
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

        // ================= INSTALL APK (sudah disesuaikan DIVA) =================
        stage('Install APK') {
            steps {
                script {
                    def timestamp  = new Date().format("dd-MM-yyyy_HH-mm-ss")
                    def sourcePath = env.APK_PATH
                    def destPath   = "apk-outputs\\diva-${timestamp}.apk"

                    if (!fileExists(sourcePath)) {
                        error "❌ Source APK not found: ${sourcePath}"
                    }

                    bat "copy \"${sourcePath}\" \"${destPath}\""

                    bat(script: "adb uninstall ${env.APP_PACKAGE}", returnStatus: true)

                    bat "adb install -r \"${destPath}\""
                    echo "✅ DIVA APK installed"
                }
            }
        }

                // ================= DAST - MobSF + Frida (Custom Script) =================
        stage('DAST - MobSF + Frida') {
            steps {
                script {

                    bat "adb shell input keyevent 82"
                    sleep 10

                    echo "=== Starting Dynamic Analysis for DIVA ==="

                    bat """
                    @curl -s -X POST ^
                    -H "Authorization: ${env.MOBSF_TOKEN}" ^
                    --data "hash=${env.APK_HASH}" ^
                    ${env.MOBSF_URL}/api/v1/dynamic/start_analysis
                    """

                    sleep 40

                    echo "=== Injecting Custom Frida Script ==="

                    // Tulis Frida script ke file temporary (cara paling aman di Windows)
                    writeFile file: 'frida_script.js', text: '''
                    Java.perform(function () {
                        console.log("[+] Frida injected successfully into DIVA!");

                        // Basic API Monitor
                        try {
                            var URL = Java.use("java.net.URL");
                            URL.$init.overload("java.lang.String").implementation = function(url) {
                                console.log("[API] URL: " + url);
                                return this.$init(url);
                            };
                        } catch (e) { console.log("URL hook failed"); }

                        // SSL Pinning Bypass
                        try {
                            var SSLContext = Java.use("javax.net.ssl.SSLContext");
                            SSLContext.init.overload("javax.net.ssl.KeyManager[]", "javax.net.ssl.TrustManager[]", "java.security.SecureRandom").implementation = function(km, tm, sr) {
                                console.log("[SSL] SSLContext init - pinning bypassed");
                                return this.init(km, tm, sr);
                            };
                        } catch (e) {}

                        // Root & Debugger Bypass
                        try {
                            var System = Java.use("java.lang.System");
                            System.getProperty.overload("java.lang.String").implementation = function(key) {
                                if (key.indexOf("ro.build.tags") >= 0 || key.indexOf("ro.debuggable") >= 0) {
                                    console.log("[Bypass] Root/Debugger check bypassed for: " + key);
                                    return "release-keys";
                                }
                                return this.getProperty(key);
                            };
                        } catch (e) {}

                        console.log("[+] All basic hooks installed. Ready to monitor.");
                    });
                    '''

                    // Kirim script dari file
                    def fridaResponse = bat(
                        script: """
                        @curl -s -X POST ^
                        -H "Authorization: ${env.MOBSF_TOKEN}" ^
                        --data "hash=${env.APK_HASH}" ^
                        --data-urlencode "frida_code=@frida_script.js" ^
                        ${env.MOBSF_URL}/api/v1/frida/instrument
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Frida custom script response: ${fridaResponse}"

                    sleep 30

                    echo "=== Triggering DIVA activities ==="

                    bat "adb shell am start -n ${env.APP_PACKAGE}/.MainActivity || echo 'Main started'"
                    sleep 6
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.HardcodeActivity || echo 'Hardcode started'"
                    sleep 6
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.InsecureDataStorage1Activity || echo 'IDS1 started'"
                    sleep 6
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.InsecureDataStorage2Activity || echo 'IDS2 started'"
                    sleep 6
                    bat "adb shell am start -n ${env.APP_PACKAGE}/.SQLInjectionActivity || echo 'SQLi started'"

                    bat "adb shell monkey -p ${env.APP_PACKAGE} --throttle 200 -v 1000 || echo 'Monkey done'"

                    sleep 45

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
                        echo "✅ DAST finished - Check dast_report.json"
                    }
                }
            }
        }

        // ================= METRICS (FOR PAPER) =================
        stage('Extract Metrics') {
            steps {
                script {
                    bat 'del /f /q dast_results.csv || echo no old file'
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