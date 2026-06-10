pipeline {
    agent { label 'Agent-1' } 

    triggers {
        githubPush()
    }

    environment {
        RECEIVER_EMAIL = 'astoudieng941@gmail.com'
        PROJECT_NAME   = 'ArgoCD'
        SEMGREP_RULES  = 'p/owasp-top-ten p/secrets p/security-audit'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '=== Recuperation automatique du code source (ArgoCD) ==='
                checkout scm
            }
        }

        stage('SAST - Semgrep OWASP') {
            steps {
                echo '=== Lancement analyse Semgrep sur ArgoCD ==='
                sh '''
                    semgrep scan \
                        --config "p/owasp-top-ten" \
                        --config "p/secrets" \
                        --config "p/security-audit" \
                        --json \
                        --output semgrep-results.json \
                        . || true

                    echo "=== Scan Semgrep termine ==="
                    echo "Nombre de resultats :"
                    cat semgrep-results.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d.get('results',[])))"
                '''
            }
        }

        stage('Export - Rapport HTML') {
            steps {
                echo '=== Generation du rapport HTML ==='
                writeFile file: 'gen_rapport.py', text: '''import json
from datetime import datetime

try:
    with open("semgrep-results.json") as f:
        data = json.load(f)
    results = data.get("results", [])
except Exception as e:
    results = []
    print("Erreur lecture JSON: " + str(e))

total   = len(results)
errors  = sum(1 for r in results if r.get("extra", {}).get("severity") == "ERROR")
warns   = sum(1 for r in results if r.get("extra", {}).get("severity") == "WARNING")
infos   = sum(1 for r in results if r.get("extra", {}).get("severity") == "INFO")

rows = ""
for r in results:
    extra = r.get("extra", {})
    sev   = extra.get("severity", "INFO")
    msg   = extra.get("message", "").replace("<", "&lt;").replace(">", "&gt;")[:200]
    path  = r.get("path", "")
    line  = str(r.get("start", {}).get("line", ""))
    rule  = r.get("check_id", "").split(".")[-1]
    color = {"ERROR": "#dc3545", "WARNING": "#fd7e14", "INFO": "#0d6efd"}.get(sev, "#6c757d")
    rows += "<tr><td><span style=\\"background:" + color + ";color:white;padding:3px 8px;border-radius:4px;font-size:11px\\">" + sev + "</span></td><td style=\\"font-size:11px;color:#555\\">" + rule + "</td><td style=\\"font-size:12px\\">" + msg + "</td><td style=\\"font-size:11px;color:#777\\">" + path + "</td><td style=\\"text-align:center;font-size:11px\\">" + line + "</td></tr>"

with open("rapport_semgrep.html", "w") as f:
    f.write("""<!DOCTYPE html>
<html lang=\\"fr\\">
<head>
<meta charset=\\"UTF-8\\">
<title>Rapport Semgrep - ArgoCD</title>
<style>
body{font-family:Arial,sans-serif;margin:30px;background:#f8f9fa}
h1{color:#1E4D8C;border-bottom:3px solid #2E75B6;padding-bottom:10px}
.summary{display:flex;gap:15px;margin:20px 0;flex-wrap:wrap}
.card{background:white;border-radius:8px;padding:15px 25px;text-align:center;box-shadow:0 2px 5px rgba(0,0,0,.1);min-width:100px}
.num{font-size:32px;font-weight:bold}
.label{font-size:13px;color:#666}
table{width:100%;border-collapse:collapse;background:white;border-radius:8px;overflow:hidden;box-shadow:0 2px 5px rgba(0,0,0,.1);margin-top:20px}
th{background:#1E4D8C;color:white;padding:12px;text-align:left;font-size:13px}
td{padding:10px 12px;border-bottom:1px solid #eee;vertical-align:top}
tr:hover{background:#f0f4ff}
.date{color:#888;font-size:13px;margin-bottom:20px}
</style>
</head>
<body>
<h1>Rapport SAST Semgrep - ArgoCD (OWASP)</h1>
<div class=\\"date\\">Genere le : """ + datetime.now().strftime("%d/%m/%Y a %H:%M:%S") + """</div>
<div class=\\"summary\\">
<div class=\\"card\\"><div class=\\"num\\">\"\"\"+str(total)+\"\"\"</div><div class=\\"label\\">Total Issues</div></div>
<div class=\\"card\\"><div class=\\"num\\" style=\\"color:#dc3545\\">\"\"\"+str(errors)+\"\"\"</div><div class=\\"label\\">Critical</div></div>
<div class=\\"card\\"><div class=\\"num\\" style=\\"color:#fd7e14\\">\"\"\"+str(warns)+\"\"\"</div><div class=\\"label\\">Warning</div></div>
<div class=\\"card\\"><div class=\\"num\\" style=\\"color:#0d6efd\\">\"\"\"+str(infos)+\"\"\"</div><div class=\\"label\\">Info</div></div>
</div>
<table>
<thead><tr><th>Severite</th><th>Regle</th><th>Message</th><th>Fichier</th><th>Ligne</th></tr></thead>
<tbody>""" + rows + """</tbody>
</table>
</body>
</html>""")

print("Rapport genere : " + str(total) + " issues")
'''
                sh 'python3 gen_rapport.py'
            }
        }
    }

    post {
        always {
            echo '=== Envoi du rapport par mail ==='
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                emailext(
                    to: "${env.RECEIVER_EMAIL}",
                    subject: 'Rapport Semgrep ArgoCD - Build #' + env.BUILD_NUMBER + ' - ' + currentBuild.currentResult,
                    body: '''Bonjour,

Le pipeline de securite s est declenche automatiquement suite au commit sur GitHub.

L analyse statique OWASP du code ArgoCD est terminee.
Vous trouverez en piece jointe le rapport HTML complet.

Statut : ''' + currentBuild.currentResult + '''
Logs   : ''' + env.BUILD_URL + '''console

Cordialement,
Votre Automation Agent-1.''',
                    attachmentsPattern: 'rapport_semgrep.html'
                )
            }
            archiveArtifacts artifacts: 'rapport_semgrep.html,semgrep-results.json', fingerprint: true
            cleanWs()
        }
    }
}
