# Authentification : du quotidien à l'infrastructure serveur

## Vue d'ensemble

L'authentification répond à une question simple : « êtes-vous bien qui vous prétendez être ? » Mais le contexte change radicalement la réponse : un utilisateur qui se connecte à sa banque, un téléphone qui se déverrouille au visage, et un serveur qui accepte une connexion SSH ne posent pas du tout le même problème, ni les mêmes contraintes de menace.

Ce document couvre deux terrains volontairement mis côte à côte : l'authentification des services grand public et l'authentification d'infrastructure. Pour chaque méthode, l'objectif est d'aller jusqu'au concret — les commandes, API, formats de fichiers et librairies réellement utilisés pour l'implémenter — et pas seulement le principe théorique.

## Comparatif — services grand public

| Méthode | Sécurité | Friction utilisateur | Résistance au phishing | Dépendances externes |
|---|---|---|---|---|
| Mot de passe | Variable | Faible | Faible | Faible |
| OTP / 2FA | Variable | Moyenne | Faible à moyenne | Faible |
| Magic link | Variable | Faible | Faible à moyenne | Email |
| OAuth / connexion sociale | Variable | Faible | Variable | Élevée |
| Biométrie native (Face ID, empreinte) | Élevée localement | Très faible | Élevée (rien à hameçonner) | Faible |
| Authentification bancaire forte (SCA) | Élevée | Moyenne | Élevée | Réglementaire |
| Identité numérique nationale | Élevée | Moyenne | Élevée | État |
| Passkey / WebAuthn | Élevée | Faible | Élevée | Faible |

## Comparatif — infrastructure et VPS

| Méthode | Sécurité | Scalabilité | Révocation | Complexité opérationnelle |
|---|---|---|---|---|
| Mot de passe SSH | Faible | Faible | Immédiate | Faible |
| Clé SSH publique/privée | Élevée | Moyenne | Manuelle par serveur | Faible |
| Certificat SSH (CA) | Élevée | Élevée | Centralisée, courte durée de vie | Moyenne |
| IP fixe / firewall | Élevée en périmètre fermé | Faible | Immédiate | Faible |
| WireGuard | Élevée | Élevée | Manuelle (retrait de clé) | Moyenne |
| Overlay mesh (type Tailscale) | Élevée | Élevée | Centralisée via coordinateur | Faible à moyenne |
| mTLS | Élevée | Élevée | Via CRL/OCSP ou courte durée de vie | Élevée |
| Kerberos | Élevée | Élevée (grand parc) | Centralisée (KDC) | Élevée |
| IAM cloud (rôles, STS) | Élevée | Élevée | Immédiate (révocation de rôle) | Moyenne |

---

# Partie 1 — Authentification dans les services du quotidien

## Mots de passe

Un mot de passe est un secret partagé : le serveur doit vérifier que l'utilisateur connaît le bon secret sans jamais le stocker en clair, faute de quoi une fuite de base de données livre directement l'ensemble des identifiants.

**Stockage — implémentation concrète.** Hacher avec Argon2id plutôt qu'un hash généraliste. En Node.js, la librairie `argon2` expose `argon2.hash(password)` (qui génère et intègre automatiquement un sel aléatoire dans la sortie) et `argon2.verify(hash, password)` pour la vérification. En PHP natif, `password_hash($password, PASSWORD_ARGON2ID)` et `password_verify()` font la même chose sans dépendance externe. En Python, `argon2-cffi` expose une API équivalente (`PasswordHasher().hash()` / `.verify()`). Les paramètres à régler explicitement sont le coût mémoire (`memoryCost`, viser au moins 19 Mo recommandés par l'OWASP) et le nombre d'itérations, à calibrer selon le temps de hachage toléré côté serveur (typiquement 250 ms à 1 seconde).

**Vérification contre les mots de passe compromis.** L'API « Pwned Passwords » de Have I Been Pwned permet de vérifier si un mot de passe apparaît dans des fuites connues, sans jamais transmettre le mot de passe en clair : seul le préfixe des 5 premiers caractères du hash SHA-1 est envoyé (k-anonymity), et le client compare localement le reste du hash parmi les résultats renvoyés.

**Rate limiting concret.** Un compteur Redis par identifiant (`INCR login_attempts:<user>` avec `EXPIRE` de quelques minutes) permet de bloquer après N tentatives échouées sans base de données relationnelle dédiée ; des librairies comme `express-rate-limit` (Node) ou `django-ratelimit` (Python) encapsulent ce pattern.

**Récupération de compte.** Le token de réinitialisation suit le même traitement qu'un magic link (voir plus bas) : généré aléatoirement, haché avant stockage, à usage unique, expiration courte.

## OTP / 2FA

**TOTP — implémentation concrète.** Génération du secret côté serveur à l'enregistrement (`crypto.randomBytes(20)` encodé en base32), affiché sous forme de QR code encodant une URI standard `otpauth://totp/MonService:utilisateur?secret=XXXX&issuer=MonService`, scannable par n'importe quelle application d'authentification (Google Authenticator, Aegis, Bitwarden). Côté serveur, des librairies comme `otplib` (Node), `pyotp` (Python) ou `speakeasy` calculent le code attendu à partir du secret et du temps courant, avec une fenêtre de tolérance (`window`) d'une ou deux périodes de 30 secondes pour absorber les décalages d'horloge. Le secret doit être stocké chiffré en base (pas en clair), car sa fuite permet de générer indéfiniment des codes valides.

**SMS.** Envoi via un fournisseur tiers (Twilio, Vonage) — techniquement trivial à intégrer, mais c'est justement cette simplicité qui explique pourquoi le SIM swapping en fait le facteur le plus attaqué en pratique.

**Codes de récupération.** Générés en lot à l'activation du 2FA (par exemple 10 codes à usage unique), hachés individuellement comme des mots de passe, chacun marqué consommé après utilisation.

## Magic links

**Implémentation concrète.** Génération d'un token aléatoire à haute entropie (`crypto.randomBytes(32).toString('hex')`), stockage non pas du token lui-même mais de son hash SHA-256 en base (comme un mot de passe, pour qu'une fuite de base ne livre pas des liens valides directement utilisables), avec une colonne `expires_at` (10-15 minutes) et un flag `used`. Le lien envoyé contient le token en clair (`https://service.com/auth?token=xxx`) ; à la validation, le serveur hache le token reçu et le compare à celui stocké, puis marque immédiatement la ligne comme consommée avant de créer la session — l'ordre importe pour éviter une course (race condition) en cas de double clic ou de pré-visite automatique par un scanner email.

## OAuth / SSO et connexion sociale

**Implémentation concrète.** Côté fournisseur (Google Cloud Console, GitHub Developer Settings, etc.), on déclare une application OAuth avec une `redirect_uri` exacte et on récupère un `client_id`/`client_secret`. Le flux se code rarement à la main : des librairies comme `passport.js` (Node, avec ses stratégies `passport-google-oauth20`, `passport-github2`) ou `authlib` (Python) gèrent la construction de l'URL d'autorisation, l'échange du code contre un token, et la validation.

**PKCE en pratique.** Génération d'un `code_verifier` aléatoire, calcul de son empreinte SHA-256 encodée en base64url comme `code_challenge`, envoyé lors de la requête d'autorisation ; le `code_verifier` original est renvoyé lors de l'échange final du code contre le token, permettant au serveur d'autorisation de vérifier la correspondance. La plupart des SDK OAuth modernes (`oauth4webapi`, bibliothèques officielles Google/Microsoft) génèrent et vérifient PKCE automatiquement sans intervention manuelle.

**Point de vigilance concret.** Toujours générer un paramètre `state` aléatoire, le stocker en session avant redirection, et vérifier qu'il correspond exactement à la valeur retournée par le fournisseur — un `state` absent ou non vérifié ouvre la voie au CSRF sur le flux de connexion.

## Passkeys / WebAuthn

**Implémentation concrète.** Des librairies serveur comme `@simplewebauthn/server` (Node), `webauthn4j` (Java) ou `py_webauthn` (Python) exposent typiquement quatre endpoints : `/register/options` (génère le challenge et les paramètres d'enregistrement), `/register/verify` (vérifie l'attestation renvoyée par le navigateur et stocke la clé publique), `/login/options` (génère un nouveau challenge de connexion) et `/login/verify` (vérifie la signature). Côté navigateur, `navigator.credentials.create()` pour l'enregistrement et `navigator.credentials.get()` pour la connexion, tous deux basés sur l'API WebAuthn native déjà présente dans tous les navigateurs modernes — aucune dépendance JavaScript tierce n'est nécessaire côté client.

**Suivi anti-clonage.** Chaque réponse d'authentification WebAuthn inclut un compteur (`signCount`) que le serveur doit comparer à la dernière valeur connue : s'il n'a pas strictement augmenté, cela indique potentiellement un clonage de l'authenticator, et la connexion doit être refusée ou signalée.

## Créer une connexion par empreinte digitale

Il n'existe pas de « connexion par empreinte » où l'empreinte transiterait vers un serveur — ce qui se construit réellement, c'est une connexion par clé cryptographique déverrouillée localement par l'empreinte, sur le modèle WebAuthn ci-dessus.

**Web.** Demander un authenticator « platform » plutôt qu'une clé externe via l'option `authenticatorAttachment: "platform"` dans l'appel `navigator.credentials.create()`. Le flux reste celui de WebAuthn : challenge serveur, déverrouillage local par empreinte, signature, seule la signature revient au serveur.

**Android natif.** L'API `BiometricPrompt` affiche l'invite système, débloquant une clé stockée dans l'Android Keystore. Génération de la paire de clés au premier enregistrement via `KeyGenParameterSpec.Builder` avec `.setUserAuthenticationRequired(true)`, puis chaque connexion signe un challenge serveur avec la clé privée débloquée par l'empreinte.

**iOS natif.** Le framework `LocalAuthentication` (`LAContext.evaluatePolicy`) déclenche Touch ID/Face ID pour débloquer une clé dans le Keychain protégée par `kSecAccessControlBiometryCurrentSet`, qui invalide automatiquement la clé si de nouvelles empreintes sont enregistrées sur l'appareil.

**Côté serveur, dans tous les cas.** Stocker uniquement des clés publiques par utilisateur, générer un challenge aléatoire à usage unique par connexion, vérifier la signature, suivre le `signCount` — exactement la même architecture que la section WebAuthn ci-dessus.

## Authentification bancaire forte (SCA)

**Implémentation concrète.** Techniquement, un e-commerçant n'implémente presque jamais le 3D Secure lui-même : il délègue à un prestataire de paiement (Stripe, Adyen, Mollie) dont le SDK gère le déclenchement du flux 3DS2 — une iframe ou une redirection vers l'application bancaire du client pour la validation biométrique locale — conformément aux spécifications techniques réglementaires (RTS) de la DSP2. Le commerçant reçoit uniquement le résultat final de l'authentification (succès/échec), jamais de donnée biométrique ni de credential bancaire.

## Identité numérique nationale

**Implémentation concrète (FranceConnect).** Un service qui veut proposer « Se connecter avec FranceConnect » doit s'enregistrer comme fournisseur de service auprès de la DINUM, obtenir un `client_id`/`client_secret`, puis implémenter un flux OAuth2/OIDC standard vers les endpoints FranceConnect (`/authorize`, `/token`, `/userinfo`), avec des scopes spécifiques comme `identite_pivot` pour récupérer nom, prénom et date de naissance vérifiés par l'État plutôt que déclarés par l'utilisateur.

## Cartes et badges sans contact

**Implémentation concrète.** La communication repose sur la norme ISO/IEC 14443 et l'échange de commandes APDU (Application Protocol Data Unit) entre le lecteur et la puce. Une transaction EMV sans contact génère un cryptogramme (ARQC) unique par transaction à partir d'une clé stockée dans la puce, empêchant le rejeu d'une transaction capturée. Pour lire un badge NFC depuis une application, l'API Web NFC (`NDEFReader` en JavaScript) ou les SDK natifs (`CoreNFC` sur iOS, `NfcAdapter` sur Android) exposent l'accès bas niveau au tag ou à la carte.

## Authentification adaptative (risk-based)

**Implémentation concrète.** Des librairies comme FingerprintJS calculent une empreinte d'appareil côté client (canvas fingerprinting, polices installées, résolution, fuseau horaire) renvoyée au serveur à chaque connexion. Côté serveur, une base de géolocalisation IP (MaxMind GeoIP2) permet de détecter une connexion incohérente avec la dernière position connue. Ces signaux alimentent un score de risque simple (somme pondérée de règles : nouvelle IP = +30, nouvel appareil = +40, nouveau pays = +50) déclenchant une exigence de second facteur au-delà d'un seuil — pas besoin de machine learning sophistiqué pour un premier niveau utile.

## Vérification d'identité (KYC)

**Implémentation concrète.** Rarement construit en interne : des prestataires spécialisés (Onfido, Veriff, Stripe Identity) exposent une API où le service intègre un widget de capture (photo de la pièce d'identité + selfie avec détection de vivacité), puis reçoit un webhook asynchrone avec le résultat de la vérification quelques secondes à minutes plus tard, sans jamais stocker lui-même les documents bruts.

---

# Partie 2 — Authentification pour serveurs, VPS et infrastructure

## SSH par clé publique

**Implémentation concrète.**
```
ssh-keygen -t ed25519 -C "gabriel@laptop"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@vps
```
La clé publique atterrit dans `~/.ssh/authorized_keys` sur le serveur, une clé par ligne. Désactiver l'authentification par mot de passe dans `/etc/ssh/sshd_config` (`PasswordAuthentication no`) une fois la clé en place, pour fermer la porte au brute force par mot de passe.

## Certificats SSH

**Implémentation concrète.** Générer une autorité de certification dédiée (une simple paire de clés Ed25519 supplémentaire), puis signer les clés utilisateurs :
```
ssh-keygen -s ca_key -I identifiant_utilisateur -n username -V +12h user_key.pub
```
Sur chaque serveur, `sshd_config` déclare `TrustedUserCAKeys /etc/ssh/ca.pub` : il n'a plus besoin de connaître chaque clé individuellement, seulement de faire confiance à la CA. Le certificat expire de lui-même après la durée `-V` spécifiée. HashiCorp Vault propose un moteur de secrets SSH (`vault write ssh/sign/role public_key=@user_key.pub`) qui automatise cette signature à la demande.

## Bastion / jump host

**Implémentation concrète.** Un seul VPS exposé publiquement, tous les autres accessibles uniquement depuis son IP interne. Depuis le client :
```
ssh -J user@bastion user@serveur-interne
```
ou en configuration permanente dans `~/.ssh/config` via la directive `ProxyJump bastion`. Des outils comme Teleport ajoutent par-dessus l'enregistrement de session et l'émission de certificats SSH à courte durée de vie automatiquement.

## PAM, sudo et 2FA sur SSH

**Implémentation concrète.**
```
apt install libpam-google-authenticator
google-authenticator
```
puis ajout de `auth required pam_google_authenticator.so` dans `/etc/pam.d/sshd`, et `ChallengeResponseAuthentication yes` + `AuthenticationMethods publickey,keyboard-interactive` dans `sshd_config` pour exiger la clé SSH **et** le code TOTP.

## Fail2ban

**Implémentation concrète.**
```
apt install fail2ban
```
Configuration dans `/etc/fail2ban/jail.local` :
```
[sshd]
enabled = true
maxretry = 3
bantime = 3600
```
Vérification des bans actifs : `fail2ban-client status sshd`.

## IP fixe et filtrage par pare-feu

**Implémentation concrète.**
```
ufw allow from 1.2.3.4 to any port 22
ufw default deny incoming
```
ou l'équivalent au niveau du security group du fournisseur cloud, ce qui a l'avantage de filtrer avant même que le trafic n'atteigne le VPS.

## WireGuard

**Implémentation concrète.**
```
wg genkey | tee privatekey | wg pubkey > publickey
```
Fichier `/etc/wireguard/wg0.conf` côté serveur :
```
[Interface]
PrivateKey = <clé privée serveur>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <clé publique du client>
AllowedIPs = 10.0.0.2/32
```
Activation : `wg-quick up wg0`. Le champ `AllowedIPs` est à la fois une règle de routage et une règle d'authentification (crypto key routing) : un paquet prétendant venir de `10.0.0.2` n'est accepté que s'il est chiffré avec la clé du pair associé à cette IP.

## Overlay mesh (type Tailscale, Headscale)

**Implémentation concrète.**
```
tailscale up
```
authentifie la machine via SSO (Google, GitHub, OIDC d'entreprise) auprès du serveur de coordination, qui distribue automatiquement les clés WireGuard entre pairs. Les règles d'accès se déclarent en JSON (fichier ACL) : par exemple restreindre l'accès au port 5432 d'une base de données à un groupe `tag:backend` uniquement. Pour héberger soi-même le serveur de coordination sans dépendre du service Tailscale, Headscale est l'implémentation open source compatible.

## mTLS

**Implémentation concrète.** Génération d'une CA interne et de certificats client avec `openssl` :
```
openssl req -x509 -new -key ca.key -out ca.pem
openssl req -new -key client.key -out client.csr
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key -out client.pem
```
Côté serveur nginx :
```
ssl_client_certificate /etc/nginx/ca.pem;
ssl_verify_client on;
```
ou côté Node.js, `https.createServer({ ca, requestCert: true, rejectUnauthorized: true }, app)`.

## Kerberos

**Implémentation concrète.** Côté client, obtention d'un ticket initial :
```
kinit utilisateur@REALM
klist
```
Les services (serveur de fichiers, base de données) possèdent un `keytab` — fichier contenant leur propre clé secrète partagée avec le KDC — déclaré dans leur configuration pour valider les tickets présentés sans contacter le KDC à chaque requête.

## Secrets management (Vault, KMS, HSM)

**Implémentation concrète.**
```
vault kv put secret/db password=xxxx
vault read database/creds/mon-role
```
La seconde commande illustre un secret dynamique : Vault crée un utilisateur de base de données temporaire à la demande, avec un bail (`lease`) qui expire automatiquement — aucun mot de passe de base de données statique à faire circuler dans une variable d'environnement.

## IAM cloud et identifiants temporaires

**Implémentation concrète.** Attacher un rôle IAM directement à l'instance EC2 (instance profile) : le SDK AWS installé sur le VPS récupère alors automatiquement des identifiants temporaires via le service de métadonnées, sans clé d'API statique configurée nulle part sur le disque. Forcer IMDSv2 via Terraform :
```
metadata_options {
  http_tokens = "required"
}
```
Le flux d'accès lui-même illustre le mécanisme d'authentification à deux temps qui bloque le SSRF classique :
```
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/mon-role
```
La requête PUT initiale, avec un en-tête personnalisé, est ce qu'une vulnérabilité SSRF classique (limitée à des GET sans en-têtes custom) ne peut généralement pas reproduire.

## Authentification machine-à-machine

**Implémentation concrète par mécanisme :**

- **API key statique** : un simple en-tête `Authorization: Bearer <clé>` vérifié contre une valeur hachée en base — à faire tourner régulièrement (rotation), jamais dans un dépôt Git.
- **OAuth Client Credentials** : requête directe sans navigateur ni utilisateur —
```
curl -X POST https://auth.exemple.com/token \
  -d grant_type=client_credentials \
  -d client_id=xxx -d client_secret=yyy
```
retourne un token de courte durée à inclure ensuite dans chaque appel API.
- **JWT signé** : générés avec des librairies comme `jsonwebtoken` (Node) ou `pyjwt` (Python), signés en RS256 (clé asymétrique) plutôt qu'HS256 quand plusieurs services doivent pouvoir vérifier le token sans partager le secret de signature.
- **Kubernetes** : chaque pod reçoit automatiquement un token de compte de service monté dans `/var/run/secrets/kubernetes.io/serviceaccount/token`, utilisable pour s'authentifier auprès de l'API du cluster ou, via des projections de token à durée de vie courte (`serviceAccountToken` volume), auprès de services externes.
- **Service mesh (SPIFFE/SPIRE)** : attribue automatiquement une identité cryptographique (certificat X.509 de courte durée) à chaque charge de travail, servant ensuite de base au mTLS entre services sans gestion manuelle de certificats.

---

## Limites générales

Aucune méthode d'authentification, quel que soit le contexte, ne constitue à elle seule une garantie absolue de sécurité. La sécurité globale dépend tout autant de la gestion des sessions après authentification, du contrôle d'accès (autorisation, distincte de l'authentification), de la robustesse des flux de récupération de compte, de la sécurité de l'infrastructure sous-jacente, et de la qualité de l'implémentation d'ensemble. Un mécanisme fort mal entouré — clé SSH robuste mais serveur non patché, passkey solide mais flux de récupération de compte faible — n'apporte qu'une sécurité illusoire.

## Sources

- IETF/IRTF — [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106.html) — Argon2 Memory-Hard Function for Password Hashing
- IETF — [RFC 4226](https://www.rfc-editor.org/rfc/rfc4226.html) — HOTP: An HMAC-Based One-Time Password Algorithm
- IETF — [RFC 6238](https://www.rfc-editor.org/rfc/rfc6238.html) — TOTP: Time-Based One-Time Password Algorithm
- IETF — [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) — The OAuth 2.0 Authorization Framework
- IETF — [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html) — Proof Key for Code Exchange (PKCE)
- IETF — [RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html) — The Kerberos Network Authentication Service (V5)
- OpenID Foundation — [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- W3C — [Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/)
- FIDO Alliance — [Passkeys](https://fidoalliance.org/passkeys/)
- OWASP — [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- OWASP — [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- NIST — [SP 800-63B-4](https://csrc.nist.gov/pubs/sp/800/63/b/4/final) — Digital Identity Guidelines
- WireGuard — [Protocol & Cryptography](https://www.wireguard.com/protocol/)
- Dowling, Paterson — [A Cryptographic Analysis of the WireGuard Protocol](https://www.wireguard.com/papers/dowling-paterson-computational-2018.pdf)
- Noise Protocol Framework — [noiseprotocol.org](https://noiseprotocol.org/noise.html)
- Tailscale — [How Tailscale works](https://tailscale.com/blog/how-tailscale-works)
- AWS — [Instance Metadata Service documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- Datadog Security Labs — [Misconfiguration Spotlight: Securing the EC2 Instance Metadata Service](https://securitylabs.datadoghq.com/articles/misconfiguration-spotlight-imds/)
- HashiCorp — [Vault documentation](https://developer.hashicorp.com/vault/docs)
- Commission européenne — [Règlement eIDAS](https://digital-strategy.ec.europa.eu/en/policies/eidas-regulation)
- Have I Been Pwned — [Pwned Passwords API (k-anonymity)](https://haveibeenpwned.com/API/v3#PwnedPasswords)
