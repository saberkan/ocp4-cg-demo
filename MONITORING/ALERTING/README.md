# Alerting
## Deployer un service manager	

## Ajouter des alerts, Crée un prometheus rule
<pre>
# utiliser l'alerte par default pour demo
</pre>

## Ajouter une alert manager configuration
<pre>
spec:
  alerting:
    alertmanagers:
      - name: alertmanager-operated # nom du service d'alert manager
        namespace: saberkan-monitoring
        port: web
</pre>