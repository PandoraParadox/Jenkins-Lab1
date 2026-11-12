pipeline {
    agent { label 'win-agent' }
    triggers { cron('*/2 * * * *') }
    options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }
    parameters {
        booleanParam(name: 'FORCE_FAIL', defaultValue: false, description: 'Test: ep Stage 02 loi')
    }
    environment { MAIL_TO = 'security-team@example.test' }

    stages {
        stage('01 Check Security Logs') {
            steps {
                powershell '''
                $ErrorActionPreference = 'Stop'
                Get-WinEvent -LogName Security -MaxEvents 1 | Out-Null
                $ev = @(Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-1)} -ErrorAction SilentlyContinue)
                $rows = $ev | ForEach-Object {
                    $x = [xml]$_.ToXml(); $d = @{}
                    $x.Event.EventData.Data | ForEach-Object { $d[$_.Name] = $_.'#text' }
                    [pscustomobject]@{ Time=$_.TimeCreated; User=$d['TargetUserName']; IP=$d['IpAddress'] }
                }
                if ($rows) { $rows | Export-Csv failed_logins.csv -NoTypeInformation } else { 'Time,User,IP' | Set-Content failed_logins.csv }
                Write-Output "Found $($ev.Count) failed logins in last 1 hour"
                '''
            }
        }
        stage('02 Analyze Failed Logins') {
            steps {
                powershell '''
                $ErrorActionPreference = 'Stop'
                if ($env:FORCE_FAIL -eq 'true') { throw 'FORCE_FAIL: simulated error in Stage 02' }
                $rows = @(Import-Csv failed_logins.csv)
                $rows | Group-Object User | Sort-Object Count -Descending | Select-Object Count, Name | Export-Csv summary.csv -NoTypeInformation
                Write-Output "Total failed logins: $($rows.Count)"
                '''
            }
        }
        stage('03 Generate Report') {
            steps {
                powershell '''
                $rows = @(Import-Csv failed_logins.csv)
                $top = 'none'
                if (Test-Path summary.csv) { $top = (Import-Csv summary.csv | Select-Object -First 5 | ForEach-Object { "$($_.Name): $($_.Count)" }) -join '; ' }
                "Security log report $(Get-Date -Format s)`nTotal failed logins (last 1h): $($rows.Count)`nTop users: $top" | Set-Content report.txt
                Get-Content report.txt
                '''
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: 'report.txt,failed_logins.csv', allowEmptyArchive: true
            script {
                def report = readFile('report.txt')
                mail to: env.MAIL_TO,
                     subject: "[SUCCESS] Hourly Security Log Check #${env.BUILD_NUMBER}",
                     body: "${report}\nBuild: ${env.BUILD_URL}"
            }
        }
        failure {
            mail to: env.MAIL_TO,
                 subject: "[FAILED] Hourly Security Log Check #${env.BUILD_NUMBER}",
                 body: "Pipeline failed. Check console: ${env.BUILD_URL}console"
        }
    }
}
