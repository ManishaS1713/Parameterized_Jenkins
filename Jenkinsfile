pipeline {

    agent any

    parameters {
      //  string(name: 'THREADS', defaultValue: '5', description: 'Number of Users')
        string(name: 'RAMPUP', defaultValue: '1', description: 'Ramp-up Time')
        string(name: 'DURATION', defaultValue: '60', description: 'Test Duration (seconds)')

        choice(
            name: 'ENV',
            choices: ['DEV', 'QA', 'PROD'],
            description: 'Select Environment'
        )
    }

    environment {
        PT_REPO = 'https://github.com/ManishaS1713/Parameterized_Jenkins.git'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: "${PT_REPO}",
                    credentialsId: 'PT_PipelineToken'
            }
        }

        stage('Set Environment URL') {
            steps {
                script {
                    if (params.ENV == 'DEV') {
                       env.HOST = "prefscale-frontend-9icy.onrender.com"
                    } else if (params.ENV == 'QA') {
                        env.HOST = "prefscale-frontend-9icy.onrender.com"
                    } else {
                        env.HOST = "prefscale-frontend-9icy.onrender.com"
                    }

                    echo "Environment: ${params.ENV}"
                    echo "Host: ${HOST}"
                }
            }
        }

        stage('Run Performance Test') {
            steps {
                bat """
                IF EXIST Parameterized_Jenkins-result.jtl del Parameterized_Jenkins-result.jtl
                IF EXIST Parameterized_Jenkins-report rmdir /s /q Parameterized_Jenkins-report

                C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
                -t Parameterized_Jenkins.jmx ^
                -Jthreads=%THREADS% ^
                -Jhost=${HOST} ^
                -l Parameterized_Jenkins-result.jtl ^
                -e -o Parameterized_Jenkins-report
                """
            }
        }

        stage('Generate Summary') {
    steps {
        script {

            def raw = bat(
                script: '''
                powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                "$data = Import-Csv 'Parameterized_Jenkins-result.jtl'; ^
                $total = $data.Count; ^
                $success = ($data | Where-Object {$_.success -eq 'true'}).Count; ^
                $fail = $total - $success; ^
                $avg = [math]::Round(($data | Measure-Object -Property elapsed -Average).Average,2); ^
                $start = $data[0].timeStamp; ^
                $end = $data[-1].timeStamp; ^
                $duration = ($end - $start)/1000; ^
                $tps = [math]::Round($total / $duration,2); ^
                $errorPct = [math]::Round(($fail/$total)*100,2); ^
                Write-Output ('TOTAL=' + $total); ^
                Write-Output ('SUCCESS=' + $success); ^
                Write-Output ('FAIL=' + $fail); ^
                Write-Output ('AVG=' + $avg); ^
                Write-Output ('TPS=' + $tps); ^
                Write-Output ('ERRORPCT=' + $errorPct)"
                ''',
                returnStdout: true
            ).trim()

            echo raw

            def lines = raw.split("\\r?\\n")

            TOTAL = lines.find { it.startsWith("TOTAL=") }?.split("=")[1]
            SUCCESS = lines.find { it.startsWith("SUCCESS=") }?.split("=")[1]
            FAIL = lines.find { it.startsWith("FAIL=") }?.split("=")[1]
            AVG = lines.find { it.startsWith("AVG=") }?.split("=")[1]
            TPS = lines.find { it.startsWith("TPS=") }?.split("=")[1]
            ERRORPCT = lines.find { it.startsWith("ERRORPCT=") }?.split("=")[1]

            SLA_STATUS = "PASS"
            if ((AVG ?: "0").toFloat() > 1000 || (ERRORPCT ?: "0").toFloat() > 1) {
                SLA_STATUS = "FAIL"
            }
        }
    }
}
        stage('Publish Report') {
            steps {
                publishHTML([
                    allowMissing: true,
                    reportDir: 'Parameterized_Jenkins-report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true
                ])
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: "Performance Test Result - ${SLA_STATUS}",
                        body: """
<h2>Performance Test Report (${params.ENV})</h2>

<b>Total Requests:</b> ${TOTAL} <br>
<b>Success:</b> ${SUCCESS} <br>
<b>Failures:</b> ${FAIL} <br>
<b>Avg Response Time:</b> ${AVG} ms <br>
<b>Throughput:</b> ${TPS} req/sec <br>
<b>Error %:</b> ${ERRORPCT} % <br>

<br><b>SLA Status:</b> <span style="color:${SLA_STATUS == 'PASS' ? 'green' : 'red'}">${SLA_STATUS}</span>

<br><br>
<b>Environment:</b> ${params.ENV} <br>
<b>Host:</b> ${HOST} <br>

<br>
<a href="${BUILD_URL}">👉 View Full Report</a>
""",
                        to: 'manishas@ivavsys.com',
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
