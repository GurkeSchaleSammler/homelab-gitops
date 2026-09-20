# Geplanter Dual-Stack-Neuaufbau

Stand der Bestandsaufnahme: 2026-09-20. Nur Vorbereitung und Sicherung durchgeführt;
kein Dienst gestoppt, keine Installation entfernt und keine CIDRs live geändert.

## Verifizierter Bestand

- K3s `v1.36.4+k3s1`, ein SQLite-Server `master512` (`192.168.0.77`).
- Vier Agents: `gurke32` `.73`, `gurke128` `.74`, `stoasis32` `.75`, `stoasis128` `.76`.
- Alle Nodes Ready; ausschließlich IPv4-PodCIDRs `10.42.0.0/24` bis `10.42.4.0/24`.
- Argo CD Helm-Chart `10.9.0`, Argo CD `v3.5.2`.
- Acht ArgoCD-Applications Synced/Healthy: homelab, argocd-access, cert-manager,
  cert-manager-config, metallb, metallb-config, traefik-public, traefik-internal.
- Keine PVCs/PVs und keine HostPath-Mounts in laufenden Pods. Es laufen derzeit
  nur Infrastruktur-Komponenten. Vor dem Wartungsfenster erneut prüfen.
- Public Traefik `.240`, Private Traefik `.241`, beide aktuell IPv4-only.
- Certificate `argocd/argocd-server` meldet Ready=True. Das ersetzt keinen
  Ende-zu-Ende-Zugriffstest; zuletzt schlug die lokale TLS-Vertrauensprüfung fehl.
- Alle fünf Nodes haben globale IPv6 im LAN-Präfix `2a02:21b4:4a85:b800::/64`
  und einen Standardweg über den Router `fe80::a6ce:daff:feb5:7750`.
- IPv6-Internetzugriff von master512 erfolgreich. Kernel-Werte aktuell
  `eth0.accept_ra=0`, `all.forwarding=0`; auf master512 verwaltet NetworkManager
  das Netz. Die aktuelle RA-Route wird daher nicht allein vom Kernel-Wert erklärt.
  Beim Neuaufbau IPv6-Forwarding und Erhalt der RA-Route prüfen, einschließlich
  nach Neustart und RA-Erneuerung; NetworkManager-Einstellungen berücksichtigen.

## Bereits gesichert

Master-Archiv: `/var/backups/k3s-dual-stack/20260920T190411Z.tar.gz`.
Geschützte Kopie außerhalb des Repositorys:
`C:\Users\joelw\.codex\backups\k3s-dual-stack\20260920T190411Z.tar.gz`.

SHA256: `5117240f54be4163b6e280ce611355552f026b3ee8780950c152478632c8e302`.

Enthalten sind ein konsistentes SQLite-Online-Backup, Server-Token,
K3s-/Systemd-Konfiguration, Manifeste sowie API-Exporte von Secrets,
ConfigMaps, Applications, Workloads, Services, Zertifikaten und Netzressourcen.
SQLite `integrity_check=ok`, Archiv lesbar, erforderliche Secrets vorhanden,
Prüfsumme der zweiten Kopie identisch. Keine Secrets im Git-Repository.
Die API-Exporte wurden nacheinander erstellt und sind kein atomarer Cluster-Snapshot.
Ein vollständiger Restore-Test wurde noch nicht ausgeführt.

## Vor der Freigabe des Wartungsfensters noch erledigen

1. Agent-Konfigurationen sichern. SSH funktioniert, nichtinteraktives sudo
   ist auf den vier Agents derzeit nicht verfügbar. Auf jedem Agent als dessen
   normaler Benutzer ausführen (sudo fragt lokal nach dem Passwort):

   ```sh
   umask 077
   sudo tar -czf - /etc/rancher /etc/systemd/system/k3s-agent.service \
     /etc/systemd/system/k3s-agent.service.env /etc/NetworkManager/system-connections \
     /etc/sysctl.d > "$HOME/k3s-agent-backup.tar.gz"
   ```

   Fehlende Dateien oder Fehler prüfen, nicht ein unvollständiges Archiv als
   Erfolg behandeln. Archive anschließend auf den geschützten PC-Backup-Pfad
   kopieren und Lesbarkeit/Prüfsummen prüfen. Sie enthalten Zugangsdaten.
2. master512-Netzwerk-/Sysctl-Konfiguration wurde zusätzlich als
   `master512-network-config.tar.gz` im geschützten PC-Backup-Verzeichnis gesichert
   und auf Lesbarkeit geprüft. Effektive K3s-Startoptionen vor dem Neuaufbau mit
   dem neuen Plan vergleichen. Tokens nicht ausgeben.
3. Restore-Probe der SQLite-Kopie in isolierter Umgebung oder vollständiges
   Systemabbild für den Rückweg vorbereiten; der erfolgreiche Integrity-Check
   allein ist noch kein vollständiger Wiederanlauftest.
4. Frische Inventur auf neu hinzugekommene Anwendungen und Volumes durchführen.
5. Wartungsfenster für die Neuinstallation aller fünf Nodes ausdrücklich freigeben.
   Währenddessen sind Kubernetes und ArgoCD nicht verfügbar. Die bisherige
   Zustimmung galt der Vorbereitung und Sicherung, nicht einer Deinstallation.

## Reihenfolge im freigegebenen Wartungsfenster

1. Änderungen einfrieren, Agents kontrolliert stoppen, dann Server stoppen.
   Abschließende kalte Kopie des vollständigen SQLite-DB-Verzeichnisses inklusive
   Server-Token und Konfiguration erstellen, auf den PC kopieren und prüfen.
2. Erst dann die alte K3s-Installation auf allen fünf Nodes bereinigen und neu
   installieren. Kein Upgrade des alten SQLite-Datastores als Umstellung verwenden.
3. Auf master512 die vorbereitete [Server-Konfiguration](dual-stack/k3s-server.yaml)
   als `/etc/rancher/k3s/config.yaml` installieren. Version ausdrücklich auf
   `v1.36.4+k3s1` festlegen; Netzumstellung und Versionsupgrade getrennt halten.
   Die Werte gelten für einen neuen Cluster und identisch für eventuelle weitere Server.
4. Agents mit derselben K3s-Version, `K3S_URL=https://192.168.0.77:6443` und dem
   neuen Server-Token verbinden. Token vertraulich übertragen, nicht in Git speichern.
   Alte Node-Passwörter nicht unkontrolliert mit der neuen Installation mischen.
5. Vor Infrastruktur-Deployment: alle Nodes Ready, jeweils IPv4- und IPv6-PodCIDR;
   Dual-Stack-Service-Test, Pod-DNS, IPv4/IPv6-Verbindungen zwischen Nodes und
   ULA-IPv6-Egress mit Flannel-Masquerading prüfen. Fehlende RA-Route beheben.
6. Namespaces für Bootstrap/Secrets anlegen. Aus den geschützten Exporten gezielt
   Cloudflare-API-Token und ACME-Account in `cert-manager`, ArgoCD-Login-Secret und
   `argocd-server-tls` in `argocd` wiederherstellen. UID, resourceVersion,
   creationTimestamp, managedFields und alte ownerReferences entfernen; nur die
   benötigten Secret-Daten/Typen und beabsichtigte Metadaten übernehmen.
   Keine Kubernetes-ServiceAccount-Tokens, Node-Passwörter, alten ClusterIPs oder
   pauschal alle System-Secrets in den neuen Cluster importieren.
7. ArgoCD mit Chart `10.9.0` und `bootstrap/argocd-values.yaml` bootstrappen;
   vorhandenes Login-Secret erhalten. Root-Application aus `bootstrap/root-app.yaml`
   einmalig bootstrappen. Danach übernimmt ausschließlich ArgoCD die Reconciliation.
8. Die vorbereiteten Werte aus `docs/dual-stack/metallb-config.values.yaml` und
   `traefik-public.values.yaml` in die jeweiligen Application-valuesObject übernehmen
   und committen/pushen. Das erst jetzt machen, weil der neue Cluster Dual Stack hat.
9. Public-Service muss `.240` und `2a02:21b4:4a85:b800::240` besitzen; Private bleibt
   ausschließlich `.241`. VIP-Verfügbarkeit und aktuelles Präfix vorher bestätigen.
10. Für ArgoCD IPv6-Firewall TCP 443 zur VIP erlauben, keine IPv4-Portweiterleitung.
    Direkt von externem IPv6-Anschluss TLS/Anmeldung testen. Erst dann Cloudflare
    AAAA `argocd` auf `2a02:21b4:4a85:b800::240`, DNS only. Einen widersprechenden
    CNAME für denselben Host ersetzen. Details: [ArgoCD-Zugriff](argocd-access.md).

## Rückweg

Die alte IPv4-SQLite-Sicherung dient nur dem Rückweg zum alten IPv4-Cluster,
nicht der Wiederherstellung in den neuen Dual-Stack-Cluster. Beim Rückweg neue
Instanzen stoppen, exakt passende alte K3s-Version/Netzkonfiguration, kaltes
DB-Backup und ursprünglichen Server-Token wiederherstellen; Agents mit den
passenden alten Einstellungen verbinden. Vorher in Git wieder IPv4-Werte setzen,
damit ArgoCD keine Dual-Stack-Service-Spezifikation in den IPv4-Cluster schreibt.
Neue DNS-AAAA-/Firewall-Einstellungen gegebenenfalls zurücknehmen. Nach dem
Neuaufbau erzeugte Daten sind nicht im alten Backup enthalten.

Referenzen: [K3s Backup/Restore](https://docs.k3s.io/datastore/backup-restore),
[K3s Dual Stack](https://docs.k3s.io/networking/basic-network-options#dual-stack-ipv4--ipv6-networking).
