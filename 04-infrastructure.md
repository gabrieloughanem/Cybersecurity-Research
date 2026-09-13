# Infrastructure : SSH, reverse proxy, firewall, DNS, CDN

## Vue d'ensemble

Les mécanismes d'authentification, de session et de sécurité web reposent sur une infrastructure sous-jacente : accès aux serveurs, routage des requêtes, filtrage réseau, résolution de noms et distribution de contenu. Une faille à ce niveau peut contourner des protections correctement conçues aux couches supérieures.

## SSH

Le protocole SSH (Secure Shell) permet l'administration distante sécurisée d'un serveur.

Points étudiés :

- architecture en trois protocoles : transport (authentification du serveur, confidentialité, intégrité), authentification utilisateur, connexion (multiplexage de canaux logiques) ;
- authentification par clé publique et gestion des méthodes d'authentification disponibles ;
- vérification de la clé hôte, notamment via son empreinte ; le modèle Trust On First Use (TOFU) consiste à mémoriser la première clé observée et à détecter ensuite un changement inattendu ;
- durcissement courant : désactivation de l'authentification par mot de passe et de la connexion root directe ;
- changement du port par défaut — mesure de réduction du bruit (scans automatisés), pas une mesure de sécurité au sens strict ;
- fail2ban et limitation des tentatives de connexion.

## Reverse proxy

Le reverse proxy reçoit les requêtes à la place des serveurs applicatifs et les redistribue en interne.

Points étudiés :

- terminaison TLS au niveau du proxy ;
- `X-Forwarded-*` — en-têtes conventionnels largement utilisés pour transmettre des informations sur les proxys intermédiaires, face à `Forwarded`, défini par l'IETF ;
- confiance à accorder aux en-têtes `X-Forwarded-*` uniquement lorsqu'ils proviennent d'un proxy de confiance, sous peine de permettre leur falsification par un client ;
- répartition de charge (load balancing) ;
- isolation des services applicatifs du réseau public.

## Firewall

Le firewall filtre le trafic réseau selon des règles définies.

Points étudiés :

- filtrage par port, protocole, adresse IP source/destination ;
- principe de refus par défaut (`deny by default`) et autorisation explicite des flux nécessaires — définition reprise par le NIST ;
- firewall applicatif (WAF) vs firewall réseau/infrastructure — périmètres de protection différents ;
- segmentation réseau entre services (base de données non exposée publiquement, par exemple).

## DNS

Le DNS traduit les noms de domaine en adresses, avec des implications de sécurité propres.

Points étudiés :

- enregistrements courants (A, AAAA, CNAME, MX, TXT) ;
- DNSSEC — authentification de l'origine et intégrité des données DNS, sans chiffrer les requêtes elles-mêmes ;
- DNS over HTTPS (DoH) et DNS over TLS (DoT) — chiffrement et confidentialité du transport des requêtes entre client et résolveur, distincts de DNSSEC ;
- attaques par empoisonnement de cache (cache poisoning) ;
- rôle des enregistrements TXT, SPF, DKIM et DMARC publiés via DNS dans la sécurité du courrier électronique.

## CDN

Le CDN distribue le contenu depuis des points de présence proches de l'utilisateur.

Points étudiés :

- peut contribuer à absorber des attaques volumétriques (DDoS) en périphérie, avant qu'elles n'atteignent l'infrastructure d'origine ;
- terminaison TLS au niveau du CDN — implique une confiance déléguée au fournisseur pour le trafic déchiffré ;
- possibilité de masquer l'adresse IP réelle du serveur d'origine lorsque celui-ci n'est pas directement exposé ;
- risque de mauvaise configuration exposant directement l'origine (contournement du CDN si l'IP d'origine est découverte).

## Limites

Aucun de ces composants n'est une frontière de sécurité isolée : un reverse proxy mal configuré peut transmettre ou faire confiance à des informations falsifiables, un DNS dépourvu de mécanisme d'authentification des données ne bénéficie pas de la protection cryptographique fournie par DNSSEC, et un CDN ne protège pas une origine directement joignable. La sécurité de l'infrastructure dépend de la cohérence de la chaîne complète, pas d'un seul maillon.

## Sources

- IETF — [RFC 4251](https://www.rfc-editor.org/rfc/rfc4251.html) — The Secure Shell (SSH) Protocol Architecture
- IETF — [RFC 4252](https://www.rfc-editor.org/rfc/rfc4252.html) — The Secure Shell (SSH) Authentication Protocol
- IETF — [RFC 4253](https://www.rfc-editor.org/rfc/rfc4253.html) — The Secure Shell (SSH) Transport Layer Protocol
- IETF — [RFC 4254](https://www.rfc-editor.org/rfc/rfc4254.html) — The Secure Shell (SSH) Connection Protocol
- IETF — [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239.html) — Forwarded HTTP Extension
- IETF — [RFC 4033](https://www.rfc-editor.org/rfc/rfc4033.html) — DNS Security Introduction and Requirements (DNSSEC)
- IETF — [RFC 8484](https://www.rfc-editor.org/rfc/rfc8484.html) — DNS Queries over HTTPS (DoH)
- IETF — [RFC 7858](https://www.rfc-editor.org/rfc/rfc7858.html) — Specification for DNS over Transport Layer Security (DoT)
- NIST — [SP 800-41 Rev. 1](https://csrc.nist.gov/pubs/sp/800/41/r1/final) — Guidelines on Firewalls and Firewall Policy
- OWASP — [Denial of Service Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) — défenses réseau et infrastructure
