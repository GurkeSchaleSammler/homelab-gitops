# Argo CD unter https://argocd.gurkeschale.ch

Dieser Host ist ausdrücklich öffentlich freigegeben. Der Public Traefik auf
`192.168.0.240` leitet HTTPS und gRPC anhand des TLS-SNI zu `argocd-server:443`
im Namespace `argocd` weiter. Der private Traefik `.241` bleibt unverändert.
Argo CD terminiert TLS selbst und behält seine normale Anmeldung bei.

Die Root-Application übernimmt `applications/argocd-access.yaml` automatisch.
Diese Application synchronisiert Route und Certificate aus
`infrastructure/argocd-access`. Der vorhandene ClusterIssuer
`letsencrypt-cloudflare` stellt über DNS-01 ein Zertifikat für den Host aus.
Das dazu benötigte Cloudflare-Token muss bereits als Secret im Cluster vorhanden
sein. Argo CD liest `argocd-server-tls` automatisch, auch bei Erneuerungen.

## DNS und Router

In Cloudflare in der Zone `gurkeschale.ch` anlegen:

| Typ | Name | Ziel | Proxy |
| --- | --- | --- | --- |
| CNAME | `argocd` | `gurkeschale.ch` | DNS only |

Der A-Eintrag der Hauptdomain muss die aktuelle öffentliche Router-IPv4 enthalten.
Im Router TCP 443 auf `192.168.0.240:443` weiterleiten. Die allgemeine Weiterleitung
von TCP 80 auf `.240:80` kann bestehen bleiben; diese Argo-CD-Route bedient nur
HTTPS, also ausdrücklich `https://` öffnen. Keine Freigabe für 6443 oder 8080.
IPv6/AAAA erst nach dem dokumentierten Dual-Stack-Neuaufbau aktivieren.

Für Zugriff aus dem LAN muss der Router NAT-Loopback unterstützen. Alternativ
lokal `argocd.gurkeschale.ch` auf `192.168.0.240` auflösen lassen. Ein direkter Test
ohne DNS-Änderung ist nach der Synchronisierung möglich:

```sh
curl --resolve argocd.gurkeschale.ch:443:192.168.0.240 https://argocd.gurkeschale.ch/
```

## Voraussetzungen und Prüfung auf master512

```sh
sudo k3s kubectl -n argocd get application argocd-access
sudo k3s kubectl -n argocd get ingressroutetcp argocd-public
sudo k3s kubectl get clusterissuer letsencrypt-cloudflare
sudo k3s kubectl -n argocd get certificate argocd-server
sudo k3s kubectl -n argocd get svc argocd-server
```

Die Application soll `Synced`, das Certificate `Ready=True` sein. Bei fehlendem
Zertifikat zunächst Certificate/CertificateRequest/Challenge-Status prüfen.
Der Backend-Service muss `argocd-server` heißen und HTTPS auf Port 443 anbieten;
`server.insecure` darf nicht aktiviert sein. Bis zur Zertifikatsausstellung kann
Argo CD noch sein selbstsigniertes Zertifikat ausliefern.

`bootstrap/argocd-values.yaml` enthält passende Einstellungen für spätere
Bootstrap-Helm-Aufrufe (öffentliche URL, TLS aktiviert, chart-eigener Ingress
deaktiviert). Diese Datei wird nicht automatisch auf eine bestehende
Argo-CD-Installation angewandt. Die neue Route benötigt beim bisherigen
Standard-Setup keine Änderung des Deployments. Falls bereits ein zusätzlicher
Ingress manuell über Helm eingerichtet wurde, dessen Helm-Konfiguration mit
diesen Bootstrap-Werten abgleichen, damit nur eine Route den Host verwaltet.

Quellen: [Argo CD TLS](https://argo-cd.readthedocs.io/en/stable/operator-manual/tls/),
[Traefik TLS Passthrough](https://doc.traefik.io/traefik/reference/routing-configuration/tcp/tls/).
