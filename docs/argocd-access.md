# Argo CD unter https://argocd.gurkeschale.ch

Dieser Host soll ausdrücklich über die öffentliche IPv6 des Public Traefik
erreichbar sein, ohne IPv4-Portweiterleitung. Vorgesehene VIP:
`2a02:21b4:4a85:b800::240`. Der Public Traefik leitet HTTPS und gRPC anhand des TLS-SNI zu `argocd-server:443`
im Namespace `argocd` weiter. Der private Traefik `.241` bleibt unverändert.
Argo CD terminiert TLS selbst und behält seine normale Anmeldung bei.

Die Root-Application übernimmt `applications/argocd-access.yaml` automatisch.
Diese Application synchronisiert Route und Certificate aus
`infrastructure/argocd-access`. Der vorhandene ClusterIssuer
`letsencrypt-cloudflare` stellt über DNS-01 ein Zertifikat für den Host aus.
Das dazu benötigte Cloudflare-Token muss bereits als Secret im Cluster vorhanden
sein. Argo CD liest `argocd-server-tls` automatisch, auch bei Erneuerungen.

## DNS und Router

Voraussetzung ist der Dual-Stack-Neuaufbau aus [k3s-dual-stack.md](k3s-dual-stack.md)
und die anschließende Aktivierung der vorbereiteten MetalLB-/Traefik-Werte.
Eine globale IPv6 auf `master512` allein aktiviert keine IPv6-Kubernetes-Services.
Die IPv4 `.240` bleibt für den LAN-Zugriff bestehen, intern bleibt ausschließlich `.241`.

Erst nach erfolgreichem IPv6-Test in Cloudflare in der Zone `gurkeschale.ch` anlegen:

| Typ | Name | Ziel | Proxy |
| --- | --- | --- | --- |
| AAAA | `argocd` | `2a02:21b4:4a85:b800::240` | DNS only |

Ein vorhandener CNAME für genau `argocd` muss durch diesen AAAA-Eintrag ersetzt
werden. Für diesen Host ohne IPv4-Erreichbarkeit keinen A-Eintrag veröffentlichen.
Bestehende A-Einträge anderer Hosts und der Hauptdomain werden nicht verändert.
Direkter Zugriff mit DNS only benötigt beim Client eine funktionierende IPv6-Verbindung.

Im Router die IPv6-Firewall für TCP 443 zur Ingress-VIP freigeben. Für andere
öffentliche HTTP-Anwendungen kann zusätzlich TCP 80 erforderlich sein. Diese
Argo-CD-Route bedient HTTPS; ausdrücklich `https://` öffnen. Keine IPv4-NAT-
Portweiterleitung erforderlich, keine Freigabe für 6443 oder 8080. Die Firewall
nicht insgesamt abschalten; notwendiges ICMPv6/NDP zulassen.

Das beobachtete LAN-Präfix ist `2a02:21b4:4a85:b800::/64`.
`2a02:21b4:4a85:b800:a6ce:daff:feb5:7750` ist eine einzelne Router-Adresse,
kein Adressbereich. Weder diese Adresse noch die bestehende IPv6 von master512
als MetalLB-VIP verwenden. Vor Aktivierung bestätigen, dass `::240` frei ist
und zum weiterhin aktuellen LAN-Präfix gehört. Bei Präfixwechsel Pool, Service,
DNS und Firewall gemeinsam aktualisieren.

Im LAN mit IPv6 denselben AAAA-Eintrag verwenden; IPv4-NAT-Loopback ist dafür
nicht nötig. Alternativ lokal `argocd.gurkeschale.ch` auf `192.168.0.240` auflösen
lassen. Direkte Tests ohne DNS-Änderung nach Synchronisierung und Zertifikatsausstellung:

```sh
curl --resolve argocd.gurkeschale.ch:443:192.168.0.240 https://argocd.gurkeschale.ch/
# Erst nach aktivierter IPv6-VIP, zusätzlich von einem externen IPv6-Anschluss testen:
curl -6 --resolve 'argocd.gurkeschale.ch:443:[2a02:21b4:4a85:b800::240]' https://argocd.gurkeschale.ch/
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
