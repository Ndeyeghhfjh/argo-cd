pipeline {
agent { label 'Agent-1' }

```
triggers {
    githubPush()
}

environment {
    RECEIVER_EMAIL = 'astoudieng941@gmail.com'
    PROJECT_NAME   = 'ArgoCD'
}

stages {

    stage('Checkout') {
        steps {
            echo '=== Recuperation automatique du code source ==='
            checkout scm
        }
    }

    stage('SAST - Semgrep OWASP') {
        steps {
            echo '=== Lancement analyse Semgrep ==='

            sh '''
                semgrep scan \
                  --config p/owasp-top-ten \
                  --config p/secrets \
                  --config p/security-audit \
                  --json \
                  --output semgrep-results.json \
                  . || true

                echo "=== Scan termine ==="

                python3 - <<EOF
```

import json

try:
with open("semgrep-results.json") as f:
data = json.load(f)

```
print("Issues :", len(data.get("results", [])))
```

except Exception as e:
print("Erreur :", e)
EOF
'''
}
}

```
    stage('Export - Rapport HTML') {
        steps {

            echo '=== Generation rapport HTML ==='

            writeFile(
                file: 'gen_rapport.py',
                text: '''
```

import json
from datetime import datetime

try:
with open("semgrep-results.json") as f:
data = json.load(f)
except:
data = {"results":[]}

results = data["results"]

total = len(results)

rows = ""

for r in results[:300]:

```
extra = r.get("extra", {})

severity = extra.get("severity","INFO")

rule = r.get("check_id","")

message = (
    extra.get("message","")
    .replace("<","&lt;")
    .replace(">","&gt;")
)

file_path = r.get("path","")

line = r.get("start",{}).get("line","")

rows += f"""
```

<tr>
<td>{severity}</td>
<td>{rule}</td>
<td>{message}</td>
<td>{file_path}</td>
<td>{line}</td>
</tr>
"""

html = f"""

<html>
<head>
<meta charset="utf-8">
<title>Rapport Semgrep</title>

<style>
body {{
font-family: Arial;
padding:30px;
}}

table {{
width:100%;
border-collapse: collapse;
}}

th,td {{
border:1px solid #ddd;
padding:10px;
}}

th {{
background:#1E4D8C;
color:white;
}}
</style>

</head>

<body>

<h1>Rapport SAST Semgrep - ArgoCD</h1>

<p>Date : {datetime.now()}</p>

<p>Total findings : {total}</p>

<table>

<tr>
<th>Severity</th>
<th>Rule</th>
<th>Message</th>
<th>File</th>
<th>Line</th>
</tr>

{rows}

</table>

</body>
</html>
"""

with open(
"rapport_semgrep.html",
"w",
encoding="utf-8"
) as f:
f.write(html)

print("Rapport HTML genere")
'''.stripIndent()
)

```
            sh 'python3 gen_rapport.py'
        }
    }
}

post {

    always {

        echo '=== Envoi rapport ==='

        catchError(
            buildResult: 'SUCCESS',
            stageResult: 'FAILURE'
        ) {

            emailext(
                to: "${env.RECEIVER_EMAIL}",

                subject:
                "Rapport Semgrep - Build #${BUILD_NUMBER}",

                body:
```

"""
Bonjour,

Le pipeline SAST ArgoCD a termine.

Statut :
${currentBuild.currentResult}

Logs :
${BUILD_URL}

Cordialement
Agent-1
""",

```
                attachmentsPattern:
                'rapport_semgrep.html'
            )
        }

        archiveArtifacts(
            artifacts:
            'rapport_semgrep.html,semgrep-results.json',
            fingerprint: true
        )

        cleanWs()
    }
}
```

}
