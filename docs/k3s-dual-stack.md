# Public und privater Ingress mit K3s

## Ziel und aktueller Zustand

| Zugriff | Ingress-Klasse | LoadBalancer | DNS |
| --- | --- | --- | --- |
| Öffentlich | `traefik-public` | `192.168.0.240`, später zusätzlich `2a02:21b4:4a85:b800::240` | Cloudflare: A auf öffentliche IPv4, AAAA auf IPv6-VIP |
| Nur LAN | `traefik-internal` | `192.168.0.241`, ausschließlich IPv4 | Lokaler DNS: A auf `.241` |

Alle Web-Anwendungen teilen sich den jeweiligen Ingress. Traefik wählt anhand
des vollständigen Hostnamens die Anwendung. Eigene LAN-IPs pro Anwendung und
ein Pool `.200–.239` sind dafür nicht erforderlich. Anwendungs-Services bleiben
`ClusterIP`; Pods erhalten keine öffentliche IPv6 aus dem Wingo-Präfix.

Die Anwendungen unter `applications/` bleiben zunächst IPv4-only. Die optionalen
IPv6-Felder im MetalLB-Chart und die explizite Trennung beider Ingress-Provider
können bereits mit dem bestehenden Cluster verwendet werden. Der interne
Traefik bleibt auch nach dem Neuaufbau explizit `SingleStack`/`IPv4`.

Die Dateien unter `docs/dual-stack/` sind vorbereitete Helm-Werte, keine aktiven
ArgoCD-Anwendungen. Die Root-Application beobachtet nur `applications/`.
`PreferDualStack` allein repariert keinen IPv4-only-Cluster: Zwei explizite
IP-Familien und zwei angeforderte VIPs dürfen erst nach dem Neuaufbau aktiviert
werden.

## Vollständige Subdomain im Helm-Chart

Für künftige Anwendungen bietet `homelab-helm/charts/app-ingress` einen Ingress
für einen bereits vorhandenen Service im gleichen Namespace. Das Chart erstellt
keine Anwendung, keinen Service, keinen DNS-Eintrag und kein Zertifikat von selbst.
Beispiel für eine eigene ArgoCD-Application im GitOps-Repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-internal-ingress
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/GurkeSchaleSammler/homelab-helm.git
    targetRevision: main
    path: charts/app-ingress
    helm:
      valuesObject:
        enabled: true
        host: app.lab.stoasis.ch
        className: traefik-internal
        service:
          name: my-app
          port: 80
        # Für HTTPS: vorhandenes TLS-Secret oder cert-manager verwenden.
        # annotations:
        #   cert-manager.io/cluster-issuer: <Name des vorhandenen ClusterIssuers>
        # tls:
        #   enabled: true
        #   secretName: my-app-tls
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Für eine öffentliche Anwendung stattdessen `host: app.stoasis.ch` und
`className: traefik-public` setzen. `service.name`, `service.port` und Namespace
müssen zum tatsächlichen Backend passen. Die IP-Adressen stehen nicht im
Anwendungs-Chart. Das Chart ist standardmäßig deaktiviert und verlangt bei
Aktivierung einen Hostnamen, einen Service und eine explizite Ingress-Klasse.

Hat ein Fremd-Chart bereits eigene Ingress-Einstellungen, diese verwenden und
keinen zweiten Ingress für denselben Host anlegen. Die genaue Values-Struktur
ist chartabhängig. Im gerenderten Ingress müssen `spec.ingressClassName` und eine
eventuelle Annotation `kubernetes.io/ingress.class` dieselbe Klasse angeben.
Bei Traefik-`IngressRoute`-Ressourcen ist die Annotation
`kubernetes.io/ingress.class: traefik-internal` beziehungsweise `traefik-public`
erforderlich. Beide Provider sind jetzt ausdrücklich getrennt. Bereits vorhandene
unmarkierte IngressRoutes müssen vor einer Synchronisierung klassifiziert werden;
sie werden mit diesen Einstellungen nicht mehr automatisch übernommen.

## DNS im LAN und bei Cloudflare

LAN-Geräte verwenden aktuell den Router als DNS-Server. Dort, sofern unterstützt,
lokale A-Einträge für `lab.stoasis.ch`, `lab.gurkeschale.ch` und die tatsächlich
verwendeten Hosts wie `app.lab.stoasis.ch` auf `192.168.0.241` anlegen. Ein A-Eintrag
für `lab.stoasis.ch` deckt seine Subdomains nicht automatisch ab. Falls der Router
lokale Wildcards unterstützt, können `*.lab.stoasis.ch` und
`*.lab.gurkeschale.ch` ebenfalls auf `.241` zeigen. Andernfalls einzelne Einträge
anlegen; ohne lokale DNS-Funktion braucht es einen LAN-DNS wie Pi-hole/AdGuard
und dessen Verteilung an Clients über DHCP. Die konkreten Menüs hängen vom
Routermodell ab. Browser mit eigenem öffentlichem DNS müssen den LAN-DNS nutzen.

Interne Hosts erhalten keine öffentlichen A-/AAAA-/CNAME-Einträge auf den Public
Traefik. Der lokale Resolver soll für ihre AAAA-Abfragen keine öffentliche
IPv6 zurückliefern. Öffentliche DNS-Wildcards dürfen interne `lab`-Namen nicht
versehentlich auf den Public Traefik auflösen. DNS ist keine Zugriffssperre:
Interne Anwendungen dürfen ausschließlich eine interne Ingress-Route besitzen.

Für öffentliche Hosts beider Zonen (`stoasis.ch`, `gurkeschale.ch`):

- Bestehende A-Einträge auf die echte öffentliche IPv4 behalten, nicht auf `.240`.
- Nach erfolgreicher IPv6-Prüfung AAAA auf `2a02:21b4:4a85:b800::240` hinzufügen.
- Alle öffentlichen Web-Subdomains können dieselbe Ingress-IP verwenden.
  Alternativ einen öffentlichen Zielhost mit A/AAAA und CNAMEs für öffentliche
  Subdomains verwenden. Die ursprüngliche Subdomain bleibt der HTTP-Hostname.
- Während direkter Verbindungstests Cloudflare auf **DNS only** stellen.
- DNS-Einträge hier bewusst manuell verwalten. Helm konfiguriert die Route;
  Cloudflare liest keine Helm-Werte. Es wird kein ExternalDNS installiert.

Ohne echte erreichbare öffentliche IPv4 funktioniert der IPv4-Zugriff über NAT
nicht automatisch, etwa bei CGNAT. Bestehende IPv4-Konfiguration trotzdem erhalten.

## Geplanter K3s-Neuaufbau

K3s muss laut [K3s-Netzwerkdokumentation](https://docs.k3s.io/networking/basic-network-options#dual-stack-ipv4--ipv6-networking)
bereits bei der Erstellung für Dual Stack konfiguriert werden. Ein als IPv4-only
gestarteter Cluster lässt sich nicht durch Helm-Änderungen in-place umstellen.

Vor einem separat geplanten Neuaufbau Daten/PVs, Secrets, Clusterkonfiguration und
ArgoCD-Bootstrap sichern und die Wiederherstellung prüfen. Eine alte IPv4-only
Datastore-Sicherung ist kein Verfahren zur Dual-Stack-Konvertierung. Hier werden
keine Deinstallation, Neuinstallation oder Live-Änderungen ausgeführt.

Beispiel **nur für einen neuen Cluster**, auf dem ersten Server:

```sh
curl -sfL https://get.k3s.io | sh -s - server \
  --disable=traefik \
  --disable=servicelb \
  --cluster-cidr=10.42.0.0/16,fd42:42::/56 \
  --service-cidr=10.43.0.0/16,fd43:43::/112
```

Alle Server benötigen übereinstimmende Netzwerk-/CIDR-/Flannel-Einstellungen.
Agents verbinden sich weiterhin über `K3S_URL` und `K3S_TOKEN`; Tokens niemals
ins Repository schreiben. Auf allen Nodes IPv6, Routing und die LAN-Verbindung
prüfen. Bei Flannel und gewünschtem IPv6-Internetzugriff aus ULA-Pods zusätzlich
`--flannel-ipv6-masq` auf allen Servern einplanen. Wenn der IPv6-Standardweg aus
Router Advertisements stammt, muss die RA-Annahme auch bei aktiviertem Forwarding
funktionieren (gegebenenfalls `accept_ra=2` auf der LAN-Schnittstelle).

Pods verwenden `10.42.0.0/16` und `fd42:42::/56`; Services `10.43.0.0/16` und
`fd43:43::/112`. Niemals das öffentliche Wingo-/64 als Pod-CIDR einsetzen.

## IPv6 nach dem Neuaufbau aktivieren

1. Dual-Stack-PodCIDRs und IPv6-Konnektivität aller Nodes prüfen.
2. Aktuelles Wingo-LAN-Präfix bestätigen und sicherstellen, dass die gewählte
   VIP frei ist. Das Präfix ist nicht als dauerhaft statisch anzusehen.
3. Werte aus `docs/dual-stack/metallb-config.values.yaml` in
   `applications/metallb-config.yaml` unter `spec.source.helm.valuesObject`
   übernehmen. Der Public-Pool enthält genau `.240/32` und die einzelne IPv6
   mit `/128`, niemals das gesamte öffentliche `/64`. ArgoCD synchronisieren lassen.
4. Werte aus `docs/dual-stack/traefik-public.values.yaml` in das `valuesObject`
   von `applications/traefik-public.yaml` übernehmen; vorhandene Ingress-Provider-
   Einstellungen behalten. ArgoCD synchronisiert `PreferDualStack`, beide
   IP-Familien und die kommagetrennte VIP-Anforderung. Intern bleibt unverändert.
5. Router-Firewall und externe Erreichbarkeit prüfen, erst dann AAAA veröffentlichen.

Bei Präfixwechsel sowohl den Public-Pool als auch die Traefik-Annotation im
GitOps-Repository, Router-Freigaben und Cloudflare-AAAA aktualisieren. Auch die
Vorlagen unter `docs/dual-stack/` aktuell halten. Das generische Helm-Repository
enthält keine konkreten Cluster-IP-Adressen.

## Router und Zugriffstrennung

IPv4 bleibt: Internet → öffentliche IPv4 → NAT-Portweiterleitung TCP 80/443
→ `192.168.0.240` → Public Traefik.

IPv6 verwendet keine IPv4-NAT-Portweiterleitung: Internet → öffentliche IPv6-VIP
→ Wingo-IPv6-Firewall → MetalLB L2/NDP → Public Traefik. TCP 80/443 zur VIP über
den Clusterpfad erlauben. NDP sowie notwendiges ICMPv6 dürfen nicht pauschal
blockiert werden. Layer 2 genügt; BGP ist nicht erforderlich.

Keine WAN-Weiterleitung auf `.241`, Anwendungs-Services oder NodePorts einrichten.
Kubernetes API 6443, ArgoCD und Node-Verwaltungsports nicht öffentlich freigeben.
Die IPv6-Firewall muss auch die globalen Node-Adressen schützen. Die Trennung
der Ingress-Klassen ersetzt keine Router-Firewall oder Mandantenisolation.

Die Reconciliation bleibt: GitHub → ArgoCD → Helm → Kubernetes-CRDs → MetalLB.
MetalLB beobachtet die API; kein Git-Zugriff, kein periodisches `git pull` oder
`kubectl apply`. `prune: true` und `selfHeal: true` bleiben erhalten.

## Prüfung

Im Helm-Repository:

```sh
helm lint charts/metallb-config
helm lint charts/app-ingress
helm template metallb-config charts/metallb-config --namespace metallb-system \
  --set public.address=192.168.0.240/32 --set internal.address=192.168.0.241/32
helm template metallb-config charts/metallb-config --namespace metallb-system \
  -f ../homelab-gitops/docs/dual-stack/metallb-config.values.yaml
```

IPv4-only darf keine leeren Adressen erzeugen. Dual Stack muss zwei Adressen
im Public-Pool und nur `.241/32` im internen Pool erzeugen. Ohne konfigurierte
Adressen erzeugt das Chart weder Pools noch zugehörige Advertisements.

Nach dem tatsächlichen Deployment lesend prüfen:

```sh
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDRS:.spec.podCIDRs
kubectl -n traefik-public get service traefik-public -o yaml
kubectl -n traefik-internal get service traefik-internal -o yaml
kubectl -n metallb-system get ipaddresspools,l2advertisements
```

Den tatsächlichen Service-Namen bei abweichendem Release-Namen anpassen. Für
eine vorhandene Anwendung `curl -4`/`curl -6` von außerhalb des LAN testen.
Intern muss DNS auf `.241` zeigen; ein Zugriff auf die öffentliche VIP mit dem
internen Hostnamen darf die interne Anwendung nicht liefern. Auch IPv4 weiter
testen. Render-Prüfungen allein belegen keine Router- oder Live-Erreichbarkeit.

Weitere Referenzen: [MetalLB-Service-Nutzung](https://metallb.io/usage/),
[PreferDualStack-Pools](https://metallb.io/configuration/_advanced_ipaddresspool_configuration/),
[Traefik-Chart 41.5.0](https://github.com/traefik/traefik-helm-chart/tree/v41.5.0/traefik).
