
from flask import Flask, request, render_template_string
from datetime import datetime
from collections import deque
from threading import Lock
import os
import hmac

app = Flask(__name__)

# Les visites restent en mémoire jusqu'à l'arrêt du serveur.
visites = deque(maxlen=1000)
verrou = Lock()

# Change ce mot de passe avant utilisation.
DASHBOARD_PASSWORD = os.environ.get(
    "S404_DASHBOARD_PASSWORD", ""
)

PAGE = """
<!doctype html>
<html lang="fr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>SYSTÈME 404 — Visites</title>
<style>
body {
  background:#090711; color:#eee;
  font-family:Arial,sans-serif; padding:18px;
}
h1 {color:#b66cff}
.notice {color:#ffb4c7}
.card {
  background:#171322; border:1px solid #6e35a1;
  border-radius:12px; padding:14px; margin:12px 0;
  overflow-wrap:anywhere;
}
small {color:#c5b4d9}
</style>
</head>
<body>
<h1>SYSTÈME 404</h1>
<h2>Tableau de bord des visites</h2>
<p class="notice">
Les données affichées sont des informations techniques
de visite. Informez les visiteurs de cette collecte.
</p>
<p>Visites conservées en mémoire : {{ visites|length }}</p>
{% for v in visites %}
<div class="card">
  <b>{{ v.date }}</b>
  <p>IP vue par le serveur : {{ v.ip }}</p>
  <p>Navigateur : {{ v.navigateur }} {{ v.version }}</p>
  <p>Système déclaré : {{ v.systeme }}</p>
  <p>Langue : {{ v.langue }}</p>
  <p>Page : {{ v.page }}</p>
  <p>Référent : {{ v.referent }}</p>
  <p>Méthode / statut HTTP : {{ v.methode }} / {{ v.statut }}</p>
  <details>
    <summary>Autres détails techniques</summary>
    <p>Agent utilisateur : {{ v.agent }}</p>
    <p>Hôte : {{ v.hote }}</p>
  </details>
</div>
{% else %}
<p>Aucune visite enregistrée pour le moment.</p>
{% endfor %}
</body>
</html>
"""

@app.after_request
def enregistrer_visite(response):
    if request.path not in ("/dashboard", "/favicon.ico"):
        ua = request.user_agent
        visite = {
            "date": datetime.now().astimezone().isoformat(
                timespec="seconds"
            ),
            "ip": request.remote_addr or "Inconnue",
            "navigateur": ua.browser or "Inconnu",
            "version": ua.version or "Inconnue",
            "systeme": ua.platform or "Inconnu",
            "langue": request.headers.get(
                "Accept-Language", "Non fournie"
            ),
            "referent": request.referrer or "Non transmis",
            "page": request.path,
            "methode": request.method,
            "statut": response.status_code,
            "agent": request.headers.get(
                "User-Agent", "Non fourni"
            ),
            "hote": request.host,
        }
        with verrou:
            visites.appendleft(visite)
    return response

@app.route("/")
def accueil():
    return """
    <!doctype html><html lang="fr"><meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <title>SYSTÈME 404</title>
    <body style="background:#090711;color:#eee;font-family:Arial;
                 text-align:center;padding:40px">
    <h1 style="color:#b66cff">SYSTÈME 404</h1>
    <p>Bienvenue sur le site de démonstration.</p>
    <p>Des données techniques de visite sont enregistrées.</p>
    </body></html>
    """

@app.route("/dashboard")
def dashboard():
    if not DASHBOARD_PASSWORD:
        return (
            "Configure d'abord S404_DASHBOARD_PASSWORD "
            "dans Termux, puis redémarre le serveur.",
            503
        )

    auth = request.authorization
    if not auth or not hmac.compare_digest(
        auth.password or "", DASHBOARD_PASSWORD
    ) or auth.username != "admin":
        return (
            "Accès réservé.",
            401,
            {"WWW-Authenticate": 'Basic realm="SYSTÈME 404"'}
        )

    with verrou:
        copie = list(visites)
    return render_template_string(PAGE, visites=copie)

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8084, debug=False
