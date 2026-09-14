# Authentification et contrôle d'accès : du web à l'infrastructure

## Vue d'ensemble

L'authentification répond à une question simple : « êtes-vous bien qui vous prétendez être ? » Mais le contexte change radicalement la réponse : un utilisateur qui se connecte à sa banque, un téléphone qui se déverrouille au visage, et un serveur qui accepte une connexion ne posent pas le même problème, ni les mêmes contraintes de menace.

Ce document couvre deux terrains volontairement mis côte à côte : les services grand public et l'infrastructure serveur. Pour chaque méthode, l'objectif va jusqu'au concret — commandes, API, formats de fichiers, librairies — et pas seulement le principe théorique.

## Authentification, autorisation, contrôle d'accès, session : quatre notions distinctes

Une confusion fréquente consiste à ranger dans « authentification » des mécanismes qui répondent en réalité à une question différente. Les distinguer clarifie tout le reste du document.

| Notion | Question posée | Exemples |
|---|---|---|
| **Authentification** | Qui êtes-vous ? | Clé SSH, certificat SSH, Kerberos, mTLS, mot de passe, WebAuthn |
| **Autorisation** | Qu'avez-vous le droit de faire ? | `sudo`, permissions IAM, rôles applicatifs |
| **Contrôle d'accès réseau** | Depuis où pouvez-vous atteindre le service ? | Pare-feu, liste blanche d'IP, VPN |
| **Gestion de session** | Combien de temps votre authentification reste-t-elle valable ? | Cookie de session, durée de vie d'un token, `signCount` |

Une clé SSH prouve une identité (authentification) ; `sudo` décide ensuite ce que cette identité peut exécuter (autorisation) ; un pare-feu décide si la connexion peut même atteindre le serveur (contrôle réseau) ; un cookie de session maintient l'état authentifié entre deux requêtes (session). Un certificat SSH combine les deux premières notions : il porte une identité **et** une durée de validité. Ce découpage sert de fil conducteur à la Partie 2.

## Comparatif — services grand public

| Méthode | Sécurité | Friction utilisateur | Résistance au phishing | Dépendances externes |
|---|---|---|---|---|
| Mot de passe | Variable | Faible | Faible | Faible |
| OTP / 2FA | Variable | Moyenne | Faible à moyenne | Faible |
| Magic link | Variable | Faible | Faible à moyenne | Email |
| OAuth / connexion sociale | Variable | Faible | Variable | Élevée |
| Biométrie native (Face ID, empreinte) | Élevée localement | Très faible | Élevée lorsqu'elle déverrouille une clé cryptographique liée au domaine (WebAuthn/passkey) | Faible |
| Authentification bancaire forte (SCA) | Élevée | Moyenne | Dépend du protocole et de l'implémentation (3DS2) | Réglementaire |
| Identité numérique nationale | Élevée | Moyenne | Dépend du protocole et de l'implémentation, pas garantie par le concept lui-même | État |
| Passkey / WebAuthn | Élevée | Faible | Élevée (liée cryptographiquement à l'origine) | Faible |

## Comparatif — infrastructure et VPS

**Authentification (identité)**

| Méthode | Sécurité | Scalabilité | Révocation |
|---|---|---|---|
| Mot de passe SSH | Faible | Faible | Immédiate |
| Clé SSH publique/privée | Élevée | Moyenne | Manuelle par serveur |
| Certificat SSH (CA) | Élevée | Élevée | Centralisée, courte durée de vie |
| Kerberos | Élevée | Élevée (grand parc) | Centralisée (KDC) |
| mTLS | Élevée | Élevée | Via CRL/OCSP ou courte durée de vie |
| Identité de workload cloud (rôle + credentials temporaires) | Élevée | Élevée | Immédiate (révocation du rôle) |

**Contrôle d'accès réseau**

| Méthode | Sécurité | Modèle |
|---|---|---|
| IP fixe / pare-feu | Élevée en périmètre fermé, mais fondée sur l'origine réseau, pas sur une identité | Périmétrique |
| WireGuard | Élevée (authentifie aussi les pairs cryptographiquement) | Canal chiffré + périmètre logique |

**Architecture d'accès**

| Élément | Rôle |
|---|---|
| Bastion / jump host | Point d'entrée unique, réduit la surface exposée |
| Overlay mesh (Tailscale, Headscale) | Automatise la distribution des clés et l'application de règles d'accès entre machines |

---

# Partie 1 — Authentification dans les services du quotidien

## Mots de passe

Un mot de passe est un secret partagé : le serveur doit vérifier que l'utilisateur le connaît sans jamais le stocker en clair, faute de quoi une fuite de base de données livre directement l'ensemble des identifiants.

**Stockage — implémentation concrète.** Hacher avec Argon2id plutôt qu'un hash généraliste. En Node.js, `argon2.hash(password)` / `argon2.verify(hash, password)` ; en PHP, `password_hash($password, PASSWORD_ARGON2ID)` / `password_verify()` ; en Python, `argon2-cffi`. Le coût mémoire, le nombre d'itérations et le degré de parallélisme doivent être calibrés selon l'environnement d'exécution plutôt que fixés à une valeur universelle : l'OWASP fournit des paramètres de référence à titre de point de départ, à ajuster ensuite en mesurant le temps de hachage réel obtenu côté serveur (viser un temps perceptible mais qui ne dégrade pas l'expérience de connexion, typiquement entre 250 ms et 1 seconde).

**Vérification contre les mots de passe compromis.** L'API « Pwned Passwords » de Have I Been Pwned permet de vérifier si un mot de passe apparaît dans des fuites connues sans jamais le transmettre en clair : seul le préfixe des 5 premiers caractères du hash SHA-1 est envoyé (k-anonymity), le client comparant localement le reste du hash parmi les résultats renvoyés.

**Rate limiting concret.** Un compteur Redis par identifiant (`INCR login_attempts:<user>` avec `EXPIRE`) bloque après N tentatives échouées ; des librairies comme `express-rate-limit` (Node) ou `django-ratelimit` (Python) encapsulent ce pattern.

**Récupération de compte.** Le token de réinitialisation suit le même traitement qu'un magic link (voir plus bas) : généré aléatoirement, haché avant stockage, à usage unique, expiration courte.

## OTP / 2FA

**TOTP — implémentation concrète.** Génération du secret côté serveur à l'enregistrement (`crypto.randomBytes(20)` encodé en base32), affiché sous forme de QR code encodant une URI standard `otpauth://totp/MonService:utilisateur?secret=XXXX&issuer=MonService`. Des librairies comme `otplib` (Node), `pyotp` (Python) ou `speakeasy` calculent le code attendu à partir du secret et du temps courant, avec une fenêtre de tolérance (`window`) d'une ou deux périodes de 30 secondes pour absorber les décalages d'horloge.

Contrairement à un mot de passe, ce secret ne peut pas être simplement haché : le serveur doit pouvoir le récupérer en clair à chaque connexion pour recalculer le code attendu et le comparer à celui saisi — un hachage à sens unique rendrait ce calcul impossible. Il doit donc être protégé comme un secret cryptographique, typiquement chiffré au repos avec une clé de chiffrement gérée séparément de la base de données (voir la section Secrets management plus bas), pour qu'une fuite de la seule base ne livre pas des secrets TOTP directement exploitables.

**SMS.** Envoi via un fournisseur tiers (Twilio, Vonage) — techniquement trivial à intégrer, ce qui explique en partie pourquoi le SIM swapping en fait le facteur le plus attaqué en pratique.

**Codes de récupération.** Générés en lot à l'activation du 2FA, hachés individuellement comme des mots de passe, chacun marqué consommé après utilisation.

## Magic links

**Implémentation concrète.** Génération d'un token aléatoire à haute entropie (`crypto.randomBytes(32).toString('hex')`), stockage non pas du token lui-même mais de son hash SHA-256 en base (comme un mot de passe), avec une colonne `expires_at` (10-15 minutes) et un flag `used`. Le lien envoyé contient le token en clair ; à la validation, le serveur hache le token reçu, le compare à celui stocké, puis marque immédiatement la ligne comme consommée avant de créer la session — l'ordre importe pour éviter une course en cas de double clic ou de pré-visite automatique par un scanner email.

## OAuth / SSO et connexion sociale

**Implémentation concrète.** Côté fournisseur (Google Cloud Console, GitHub Developer Settings), on déclare une application OAuth avec une `redirect_uri` exacte et on récupère un `client_id`/`client_secret`. Des librairies comme `passport.js` (Node, stratégies `passport-google-oauth20`, `passport-github2`) ou `authlib` (Python) gèrent la construction de l'URL d'autorisation, l'échange du code contre un token, et la validation.

**PKCE en pratique.** Génération d'un `code_verifier` aléatoire, calcul de son empreinte SHA-256 encodée en base64url comme `code_challenge`, envoyé lors de la requête d'autorisation ; le `code_verifier` original est renvoyé lors de l'échange final du code, permettant au serveur d'autorisation de vérifier la correspondance.

**Point de vigilance.** Toujours générer un paramètre `state` aléatoire, le stocker en session avant redirection, et vérifier qu'il correspond exactement à la valeur retournée — un `state` absent ou non vérifié ouvre la voie au CSRF sur le flux de connexion.

## Passkeys / WebAuthn

**Implémentation concrète.** Des librairies serveur comme `@simplewebauthn/server` (Node), `webauthn4j` (Java) ou `py_webauthn` (Python) exposent typiquement quatre endpoints : `/register/options`, `/register/verify`, `/login/options`, `/login/verify`. Côté navigateur, `navigator.credentials.create()` pour l'enregistrement et `navigator.credentials.get()` pour la connexion, basés sur l'API WebAuthn native — aucune dépendance JavaScript tierce n'est nécessaire côté client.

**Suivi via signCount.** Chaque réponse d'authentification WebAuthn inclut un compteur (`signCount`) que le serveur peut comparer à la dernière valeur connue. Ce champ peut fournir un signal de détection de clonage lorsque l'authenticator l'incrémente de manière exploitable, mais son interprétation dépend du comportement propre de l'authenticator : certains modèles renvoient systématiquement zéro ou ne l'utilisent pas de façon fiable, ce qui empêche d'en tirer une conclusion automatique et catégorique. Il s'agit d'un signal complémentaire, pas d'une preuve à lui seul.

## Créer une connexion par empreinte digitale

Il n'existe pas de « connexion par empreinte » où l'empreinte transiterait vers un serveur — ce qui se construit réellement, c'est une connexion par clé cryptographique déverrouillée localement par l'empreinte, sur le modèle WebAuthn ci-dessus.

**Web.** Demander un authenticator « platform » via `authenticatorAttachment: "platform"` dans `navigator.credentials.create()`. Le flux reste celui de WebAuthn : challenge serveur, déverrouillage local par empreinte, signature, seule la signature revient au serveur.

**Android natif.** L'API `BiometricPrompt` débloque une clé stockée dans l'Android Keystore, générée au premier enregistrement via `KeyGenParameterSpec.Builder().setUserAuthenticationRequired(true)`.

**iOS natif.** Le framework `LocalAuthentication` (`LAContext.evaluatePolicy`) débloque une clé du Keychain protégée par `kSecAccessControlBiometryCurrentSet`, qui invalide automatiquement la clé si de nouvelles empreintes sont enregistrées sur l'appareil.

**Côté serveur, dans tous les cas.** Stocker uniquement des clés publiques par utilisateur, générer un challenge aléatoire à usage unique par connexion, vérifier la signature — exactement l'architecture WebAuthn décrite ci-dessus, la biométrie elle-même ne quittant jamais l'appareil.

## Authentification bancaire forte (SCA)

**Implémentation concrète.** Un e-commerçant délègue presque toujours le 3D Secure à un prestataire de paiement (Stripe, Adyen, Mollie) dont le SDK déclenche le flux 3DS2 — redirection vers l'application bancaire pour validation biométrique locale — conformément aux spécifications techniques réglementaires (RTS) de la DSP2. La résistance au phishing de ce mécanisme dépend de cette implémentation précise (liaison à la transaction, validation dans l'app bancaire) et non du seul fait qu'une réglementation l'impose. Le commerçant ne reçoit que le résultat final, jamais de donnée biométrique ni de credential bancaire.

## Identité numérique nationale

**Implémentation concrète (FranceConnect).** Un service qui veut proposer « Se connecter avec FranceConnect » s'enregistre comme fournisseur de service auprès de la DINUM, obtient un `client_id`/`client_secret`, puis implémente un flux OAuth2/OIDC vers les endpoints FranceConnect (`/authorize`, `/token`, `/userinfo`), avec des scopes comme `identite_pivot`. Sa garantie de sécurité réelle dépend donc, comme pour n'importe quel flux OAuth2/OIDC, de la rigueur de cette implémentation (validation du `state`, du `redirect_uri`) et pas uniquement du fait que l'identité sous-jacente soit vérifiée par l'État.

## Cartes et badges sans contact

**Implémentation concrète.** Communication selon la norme ISO/IEC 14443, échange de commandes APDU entre lecteur et puce. Une transaction EMV sans contact génère un cryptogramme (ARQC) unique par transaction à partir d'une clé stockée dans la puce, empêchant le rejeu. Pour lire un badge NFC depuis une application : API Web NFC (`NDEFReader`) ou SDK natifs (`CoreNFC` iOS, `NfcAdapter` Android).

## Authentification adaptative (risk-based)

**Implémentation concrète.** Des librairies comme FingerprintJS calculent, côté client, un identifiant de corrélation d'appareil (canvas fingerprinting, polices installées, résolution, fuseau horaire) renvoyé au serveur à chaque connexion. Une base de géolocalisation IP (MaxMind GeoIP2) détecte une connexion incohérente avec la dernière position connue. Ces signaux alimentent un score de risque (règles pondérées : nouvelle IP = +30, nouvel appareil = +40, nouveau pays = +50) déclenchant une exigence de second facteur au-delà d'un seuil.

Un fingerprint navigateur ne constitue pas un facteur d'authentification en soi : il peut être instable dans le temps, volontairement contourné (navigation privée, extensions anti-fingerprinting) ou reproduit par un attaquant qui collecte les mêmes signaux. Il ne doit être utilisé que comme signal complémentaire dans une logique de détection de risque, jamais comme preuve d'identité à part entière.

## Vérification d'identité (KYC)

**Implémentation concrète.** Rarement construit en interne : des prestataires spécialisés (Onfido, Veriff, Stripe Identity) exposent une API où le service intègre un widget de capture (photo de pièce d'identité + selfie avec détection de vivacité), puis reçoit un webhook asynchrone avec le résultat, sans stocker lui-même les documents bruts.

---

# Partie 2 — Authentification, autorisation et contrôle d'accès pour serveurs et VPS

Cette partie applique explicitement le découpage introduit plus haut : certains mécanismes établissent une identité (authentification), d'autres décident de ce que cette identité peut faire (autorisation), d'autres encore filtrent l'accès réseau sans jamais authentifier personne (contrôle réseau), et une dernière catégorie structure l'architecture globale d'accès sans être elle-même un mécanisme d'authentification.

## Authentification (identité machine et service)

### SSH par clé publique

```
ssh-keygen -t ed25519 -C "gabriel@laptop"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@vps
```
La clé publique atterrit dans `~/.ssh/authorized_keys`. Désactiver ensuite l'authentification par mot de passe (`PasswordAuthentication no` dans `sshd_config`) pour fermer la porte au brute force.

### Certificats SSH

Génération d'une CA dédiée, puis signature des clés utilisateurs :
```
ssh-keygen -s ca_key -I identifiant_utilisateur -n username -V +12h user_key.pub
```
Sur chaque serveur, `TrustedUserCAKeys /etc/ssh/ca.pub` dans `sshd_config` : plus besoin de connaître chaque clé individuellement, seulement de faire confiance à la CA. Le certificat combine identité et durée de validité — il expire de lui-même après la durée `-V`. Vault propose un moteur de secrets SSH (`vault write ssh/sign/role public_key=@user_key.pub`) qui automatise cette signature à la demande.

### Kerberos

```
kinit utilisateur@REALM
klist
```
Les services possèdent un `keytab` — fichier contenant leur propre clé secrète partagée avec le KDC (Key Distribution Center) — déclaré dans leur configuration pour valider les tickets présentés sans recontacter le KDC à chaque requête. Une fois un ticket obtenu, l'utilisateur accède à plusieurs services sans se ré-authentifier jusqu'à expiration.

### mTLS

Génération d'une CA interne et de certificats client :
```
openssl req -x509 -new -key ca.key -out ca.pem
openssl req -new -key client.key -out client.csr
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key -out client.pem
```
Côté nginx : `ssl_client_certificate /etc/nginx/ca.pem; ssl_verify_client on;`. Côté Node.js : `https.createServer({ ca, requestCert: true, rejectUnauthorized: true }, app)`. Les deux parties s'authentifient mutuellement avant tout échange, sans secret transmis en clair.

### Identité de workload et credentials temporaires

Le principe général, indépendant du fournisseur, consiste à remplacer des credentials statiques (clé d'API stockée en dur) par une identité attachée directement à la charge de travail, assortie de credentials temporaires délivrés automatiquement. AWS l'implémente via les rôles IAM et l'Instance Metadata Service, Azure via Managed Identity, GCP via Workload Identity — trois implémentations différentes du même principe.

**Illustration concrète (AWS EC2/IMDSv2).** Un rôle IAM attaché à l'instance (instance profile) permet au SDK installé sur le VPS de récupérer automatiquement des identifiants temporaires, sans clé statique sur le disque. L'ancienne version du service de métadonnées (IMDSv1) répond à une simple requête GET sans authentification, ce qui en a fait une cible d'attaques SSRF (Server-Side Request Forgery) : une application vulnérable qui peut être trompée pour requêter une URL arbitraire peut être dirigée vers ce endpoint et récupérer directement les identifiants du rôle. IMDSv2 corrige cela en exigeant d'abord une requête PUT avec un en-tête spécifique pour obtenir un jeton de session :
```
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/mon-role
```
La plupart des vulnérabilités SSRF, limitées à des requêtes GET sans en-têtes personnalisés, ne peuvent pas reproduire cette requête PUT initiale. Forcer IMDSv2 via Terraform : `metadata_options { http_tokens = "required" }`.

### Authentification machine-à-machine

- **API key statique** : en-tête `Authorization: Bearer <clé>` vérifié contre une valeur hachée en base, à faire tourner régulièrement, jamais dans un dépôt Git.
- **OAuth Client Credentials** : requête directe sans navigateur ni utilisateur —
```
curl -X POST https://auth.exemple.com/token \
  -d grant_type=client_credentials \
  -d client_id=xxx -d client_secret=yyy
```
retourne un token de courte durée pour chaque appel API.
- **JWT signé** : librairies `jsonwebtoken` (Node) ou `pyjwt` (Python), signés en RS256 (asymétrique) plutôt qu'HS256 quand plusieurs services doivent pouvoir vérifier le token sans partager le secret de signature.
- **Kubernetes** : chaque pod reçoit un token de compte de service monté dans `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- **Service mesh (SPIFFE/SPIRE)** : attribue automatiquement une identité cryptographique (certificat X.509 courte durée) à chaque charge de travail, base du mTLS entre services sans gestion manuelle de certificats.

## Autorisation

### PAM, sudo et facteurs supplémentaires

`sudo` lui-même relève de l'autorisation, pas de l'authentification : une fois l'identité déjà établie (session ouverte), il décide quelles commandes cette identité peut exécuter avec quels privilèges, et journalise l'usage. On peut en revanche ajouter un facteur d'authentification supplémentaire au moment de l'ouverture de session ou de l'élévation, ce qui relève cette fois bien de l'authentification :
```
apt install libpam-google-authenticator
google-authenticator
```
puis `auth required pam_google_authenticator.so` dans `/etc/pam.d/sshd`, et `ChallengeResponseAuthentication yes` + `AuthenticationMethods publickey,keyboard-interactive` dans `sshd_config` pour exiger la clé SSH **et** le code TOTP avant l'ouverture de session — l'autorisation `sudo` intervenant ensuite, séparément, une fois connecté.

## Contrôle d'accès réseau

Cette catégorie ne répond pas à « qui êtes-vous ? » mais à « depuis où pouvez-vous atteindre le service ? ». Elle est complémentaire à l'authentification, jamais un substitut.

### IP fixe et filtrage par pare-feu

```
ufw allow from 1.2.3.4 to any port 22
ufw default deny incoming
```
ou l'équivalent au niveau du security group du fournisseur cloud. Une IP autorisée ne prouve l'identité de personne : elle prouve seulement l'origine réseau de la requête, partagée par tous les utilisateurs d'un même NAT et potentiellement usurpable selon le contexte. Une fois une machine du réseau autorisé compromise, ce contrôle devient inopérant — d'où la préférence croissante pour des approches zero-trust où chaque requête s'authentifie indépendamment de son origine.

### Fail2ban

```
apt install fail2ban
```
`/etc/fail2ban/jail.local` :
```
[sshd]
enabled = true
maxretry = 3
bantime = 3600
```
`fail2ban-client status sshd` pour vérifier les bans actifs. Complémentaire au filtrage par IP fixe : il bannit dynamiquement une origine réseau sur la base d'un comportement, sans authentifier personne non plus.

### WireGuard

Se situe à la frontière entre authentification et contrôle réseau : le tunnel repose sur une authentification cryptographique réelle des pairs (voir ci-dessous), mais son usage principal reste d'établir un canal réseau chiffré faisant ensuite office de périmètre logique.

```
wg genkey | tee privatekey | wg pubkey > publickey
```
`/etc/wireguard/wg0.conf` côté serveur :
```
[Interface]
PrivateKey = <clé privée serveur>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <clé publique du client>
AllowedIPs = 10.0.0.2/32
```
`wg-quick up wg0`. Le protocole utilise Curve25519 (échange de clés), ChaCha20 (chiffrement), Poly1305 (authentification des messages) et un handshake Noise_IK qui authentifie mutuellement les pairs via leurs clés statiques avant tout échange de données — contrairement à une IP fixe, ce n'est donc pas un contrôle purement périmétrique : le champ `AllowedIPs` unifie routage réseau et authentification cryptographique (crypto key routing), un paquet n'étant accepté que s'il est chiffré avec la clé du pair associée à l'IP source déclarée.

## Architecture d'accès

Ni authentification ni contrôle réseau à eux seuls : ces éléments structurent la manière dont l'accès est organisé.

### Bastion / jump host

Un seul VPS exposé publiquement, tous les autres accessibles uniquement depuis son IP interne.
```
ssh -J user@bastion user@serveur-interne
```
ou directive `ProxyJump bastion` dans `~/.ssh/config`. Réduit la surface exposée à un seul point à durcir et journaliser finement — un point de défaillance unique s'il est compromis. Des outils comme Teleport ajoutent l'émission de certificats SSH à courte durée de vie et l'enregistrement de session par-dessus cette architecture.

### Overlay mesh (type Tailscale, Headscale)

```
tailscale up
```
authentifie la machine via SSO (Google, GitHub, OIDC d'entreprise) auprès d'un serveur de coordination, qui distribue automatiquement les clés WireGuard entre pairs et applique des règles d'accès déclaratives (fichier ACL JSON) entre machines. Pour héberger soi-même ce serveur de coordination, Headscale est l'implémentation open source compatible.

## Gestion des secrets

### Secrets management (Vault, KMS, HSM)

```
vault kv put secret/db password=xxxx
vault read database/creds/mon-role
```
La seconde commande illustre un secret dynamique : Vault crée un utilisateur de base de données temporaire à la demande, avec un bail (`lease`) qui expire automatiquement — aucun mot de passe statique à faire circuler dans une variable d'environnement. Un HSM va plus loin : la clé privée n'est jamais extractible du matériel, qui n'expose qu'une opération de signature/déchiffrement.

---

## Limites générales

Aucune méthode, quel que soit le contexte, ne constitue à elle seule une garantie absolue de sécurité. La sécurité globale dépend tout autant de la gestion des sessions après authentification, de l'autorisation, de la robustesse des flux de récupération de compte, de la sécurité de l'infrastructure sous-jacente, et de la qualité de l'implémentation d'ensemble. Un mécanisme fort mal entouré — clé SSH robuste mais serveur non patché, passkey solide mais flux de récupération de compte faible — n'apporte qu'une sécurité illusoire.

## Sources

- IETF/IRTF — [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106.html) — Argon2 Memory-Hard Function for Password Hashing
- IETF — [RFC 4226](https://www.rfc-editor.org/rfc/rfc4226.html) — HOTP
- IETF — [RFC 6238](https://www.rfc-editor.org/rfc/rfc6238.html) — TOTP
- IETF — [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) — OAuth 2.0
- IETF — [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html) — PKCE
- IETF — [RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html) — Kerberos Network Authentication Service (V5)
- OpenID Foundation — [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- W3C — [Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/)
- FIDO Alliance — [Passkeys](https://fidoalliance.org/passkeys/)
- OWASP — [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- OWASP — [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- NIST — [SP 800-63B-4](https://csrc.nist.gov/pubs/sp/800/63/b/4/final)
- WireGuard — [Protocol & Cryptography](https://www.wireguard.com/protocol/)
- Dowling, Paterson — [A Cryptographic Analysis of the WireGuard Protocol](https://www.wireguard.com/papers/dowling-paterson-computational-2018.pdf)
- Noise Protocol Framework — [noiseprotocol.org](https://noiseprotocol.org/noise.html)
- Tailscale — [How Tailscale works](https://tailscale.com/blog/how-tailscale-works)
- AWS — [Instance Metadata Service documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- Datadog Security Labs — [Misconfiguration Spotlight: Securing the EC2 Instance Metadata Service](https://securitylabs.datadoghq.com/articles/misconfiguration-spotlight-imds/)
- Microsoft — [Azure Managed Identities overview](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- Google Cloud — [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- HashiCorp — [Vault documentation](https://developer.hashicorp.com/vault/docs)
- Commission européenne — [Règlement eIDAS](https://digital-strategy.ec.europa.eu/en/policies/eidas-regulation)
- Have I Been Pwned — [Pwned Passwords API (k-anonymity)](https://haveibeenpwned.com/API/v3#PwnedPasswords)
