# Authentification et contrôle d'accès : des services grand public aux serveurs

> **Carnet de recherche — Cybersecurity Research**
> Note n° 01 · Objectif : comprendre comment un système vérifie une identité, et pourquoi le choix d'une méthode dépend toujours du contexte.

---

## Introduction

S'authentifier, c'est répondre à une question simple : **« Êtes-vous bien la personne que vous prétendez être ? »**

La réponse varie pourtant beaucoup selon la situation. Se connecter à son espace bancaire, déverrouiller son téléphone avec son visage ou laisser un serveur accepter une connexion à distance sont trois problèmes différents. Les risques, les contraintes et les solutions adaptées ne sont pas les mêmes.

Cette note couvre deux terrains :

- **la première partie** traite des services que chacun utilise au quotidien (sites web, applications, banque en ligne) ;
- **la seconde partie** traite de l'accès aux serveurs, c'est-à-dire aux machines qui hébergent ces services.

L'objectif n'est pas de fournir un manuel de configuration. Il s'agit d'expliquer le fonctionnement de chaque méthode, ses points forts et ses limites.

---

## Quatre notions à bien distinguer

On range souvent sous le mot « authentification » des mécanismes qui répondent en réalité à des questions différentes. Les séparer clarifie tout le reste de la note.

| Notion | Question posée | Analogie |
|---|---|---|
| **Authentification** | Qui êtes-vous ? | Présenter sa carte d'identité à l'accueil |
| **Autorisation** | Qu'avez-vous le droit de faire ? | Le badge qui ouvre certaines portes et pas d'autres |
| **Contrôle d'accès réseau** | D'où venez-vous ? | La barrière à l'entrée du parking, qui ne laisse passer que certains véhicules |
| **Gestion de session** | Combien de temps restez-vous reconnu ? | Le bracelet remis à l'entrée d'un événement, valable pour la journée |

Ces quatre notions sont complémentaires : aucune ne remplace les autres. Un système bien sécurisé vérifie l'identité, limite les droits, filtre les accès et fait expirer les sessions.

---

# Partie 1 — Les services du quotidien

## Vue d'ensemble

| Méthode | Niveau de sécurité | Simplicité pour l'utilisateur | Résistance à l'hameçonnage* |
|---|---|---|---|
| Mot de passe | Variable | Simple | Faible |
| Code à usage unique (SMS, application) | Moyen | Moyenne | Faible à moyenne |
| Lien de connexion par email | Moyen | Simple | Faible à moyenne |
| Connexion via Google, Apple, etc. | Variable | Très simple | Variable |
| Passkey (clé d'accès) | Élevé | Très simple | Élevée |
| Validation bancaire forte | Élevé | Moyenne | Dépend de la mise en œuvre |
| Identité numérique de l'État (FranceConnect) | Élevé | Moyenne | Dépend de la mise en œuvre |

\* *L'hameçonnage (phishing) consiste à piéger un utilisateur avec un faux site imitant le vrai, pour lui faire saisir ses identifiants.*

---

## Le mot de passe

**Le principe.** L'utilisateur et le service partagent un secret. Pour se connecter, l'utilisateur prouve qu'il le connaît.

**Ce qu'un service sérieux doit faire.** Le mot de passe ne doit jamais être conservé tel quel. Le service stocke une **empreinte** du mot de passe, c'est-à-dire une transformation irréversible : on peut vérifier qu'un mot de passe saisi correspond à l'empreinte, mais on ne peut pas retrouver le mot de passe à partir d'elle. La méthode recommandée aujourd'hui s'appelle **Argon2id**. Elle est volontairement lente et gourmande en mémoire, ce qui rend très coûteuses les tentatives massives d'un attaquant.

**Deux bonnes pratiques complémentaires :**

- **refuser les mots de passe déjà compromis**, en les comparant à des bases de mots de passe ayant fuité (le service *Have I Been Pwned* permet cette vérification sans jamais transmettre le mot de passe en clair) ;
- **limiter les tentatives** : après plusieurs échecs, la connexion est ralentie ou temporairement suspendue. Il faut éviter un blocage brutal du compte, qui permettrait à n'importe qui de bloquer volontairement le compte d'un autre utilisateur.

**Sa limite principale.** Un mot de passe peut être deviné, réutilisé sur plusieurs sites, volé ou saisi sur un faux site.

---

## Le code à usage unique (double authentification)

**Le principe.** En plus du mot de passe, l'utilisateur saisit un code temporaire, généralement à six chiffres, valable environ trente secondes.

**Deux variantes :**

- **l'application d'authentification** (Google Authenticator, Authy…) : le téléphone et le service partagent un secret installé une fois pour toutes, via un QR code. Les deux calculent le même code à partir de ce secret et de l'heure courante ;
- **le SMS** : le code est envoyé par message. C'est la variante la plus fragile, car un attaquant peut convaincre un opérateur de transférer le numéro de la victime sur sa propre carte SIM (attaque dite de « SIM swapping »).

**Un point souvent méconnu.** Contrairement à un mot de passe, le secret partagé avec l'application ne peut pas être stocké sous forme d'empreinte, car le service doit pouvoir le relire pour calculer le code attendu. Il doit donc être **chiffré** et protégé par une clé conservée à part.

**Les codes de secours.** Lors de l'activation, le service fournit une liste de codes de récupération à usage unique, en cas de perte du téléphone. Ils doivent être protégés comme des mots de passe.

---

## Le lien de connexion par email (magic link)

**Le principe.** L'utilisateur saisit son adresse email et reçoit un lien. Un clic suffit pour se connecter, sans mot de passe.

**Les règles d'une mise en œuvre fiable :**

- le lien contient un code aléatoire très long, impossible à deviner ;
- il expire rapidement, en dix à quinze minutes ;
- il ne fonctionne qu'une seule fois ;
- le service n'en conserve que l'empreinte, comme pour un mot de passe.

**Un piège concret.** Certaines messageries professionnelles analysent automatiquement les liens reçus pour détecter les menaces, et les « ouvrent » avant l'utilisateur. Le lien est alors consommé et devient inutilisable pour la vraie personne. La parade consiste à afficher d'abord une page intermédiaire avec un bouton « Confirmer la connexion » : seule une action humaine valide le lien.

**Sa limite.** La sécurité du compte repose entièrement sur celle de la boîte email.

---

## La connexion via un compte tiers (Google, Apple, GitHub…)

**Le principe.** Plutôt que de créer un compte, l'utilisateur délègue la vérification de son identité à un fournisseur qu'il utilise déjà. Le service reçoit une confirmation de ce fournisseur, sans jamais voir le mot de passe. Ce mécanisme repose sur les standards **OAuth 2.0** et **OpenID Connect**.

**Les précautions indispensables côté service.** L'échange entre le service et le fournisseur doit être protégé contre deux types de détournement :

- un paramètre de contrôle (appelé `state`) garantit que la réponse reçue correspond bien à une demande émise par l'utilisateur, et non à une demande forgée par un tiers ;
- un mécanisme complémentaire (appelé **PKCE**) garantit que seul l'émetteur de la demande peut finaliser la connexion, même si la réponse est interceptée.

**Sa limite.** Le service dépend d'un acteur extérieur : si le compte Google de l'utilisateur est compromis ou suspendu, l'accès au service l'est aussi.

---

## Les passkeys (clés d'accès)

**Le principe.** C'est aujourd'hui la méthode la plus sûre pour le grand public. Au lieu d'un mot de passe, l'appareil de l'utilisateur (téléphone, ordinateur, clé de sécurité) crée une **paire de clés** :

- une **clé privée**, qui ne quitte jamais l'appareil ;
- une **clé publique**, transmise au service.

À chaque connexion, le service envoie un défi aléatoire. L'appareil le signe avec la clé privée, après avoir vérifié la présence de l'utilisateur (empreinte, visage ou code de déverrouillage). Le service vérifie la signature avec la clé publique.

**Pourquoi c'est robuste :**

- **aucun secret ne circule** : même si le service subit une fuite de données, les clés publiques dérobées ne permettent pas de se connecter ;
- **l'hameçonnage devient inopérant** : la clé est liée à l'adresse exacte du site, et un faux site ne peut donc pas l'utiliser ;
- **l'expérience est plus simple** qu'un mot de passe.

Le standard technique sous-jacent s'appelle **WebAuthn**. Il est porté par le W3C et l'Alliance FIDO.

---

## La connexion par empreinte digitale : une idée reçue

On parle souvent de « connexion par empreinte digitale ». En réalité, **l'empreinte ne quitte jamais l'appareil** et n'est jamais transmise à un service en ligne.

Ce qui se passe réellement : l'empreinte **déverrouille localement une clé cryptographique**, et c'est cette clé qui prouve l'identité auprès du service, selon le modèle des passkeys décrit ci-dessus. Le service ne reçoit qu'une signature, jamais de donnée biométrique.

Ce modèle est le même sur le web, sur Android et sur iOS. Sur iPhone, la clé peut même être automatiquement invalidée si une nouvelle empreinte est ajoutée à l'appareil.

---

## La validation bancaire forte

**Le principe.** Depuis la directive européenne sur les services de paiement (DSP2), les paiements en ligne doivent généralement être validés par deux éléments distincts, par exemple le téléphone de l'utilisateur et son empreinte. C'est le fameux écran « Validez ce paiement dans votre application bancaire » (protocole **3D Secure**).

**Ce que voit le commerçant.** Rien de sensible : il délègue ce processus à son prestataire de paiement (Stripe, Adyen…) et ne reçoit que le résultat final, accepté ou refusé.

**Une nuance importante.** Le niveau de sécurité dépend de la manière dont la banque met en œuvre la validation (lien avec le montant et le bénéficiaire, validation dans l'application officielle). L'obligation réglementaire ne garantit pas, à elle seule, la résistance à la fraude.

---

## L'identité numérique de l'État (FranceConnect)

**Le principe.** FranceConnect permet de se connecter à un service en s'appuyant sur un compte déjà vérifié par l'administration (impots.gouv, Ameli, l'Identité Numérique La Poste…). Techniquement, il fonctionne comme la connexion via Google ou Apple : il repose sur le même standard, OpenID Connect.

**Une nuance importante.** L'identité est vérifiée par l'État, mais la sécurité réelle de la connexion dépend aussi de la rigueur avec laquelle chaque service l'a intégrée. Un service partenaire mal configuré reste vulnérable, quelle que soit la solidité de l'identité d'origine.

---

## Cartes et badges sans contact

**Le principe.** La carte bancaire ou le badge contient une puce qui dialogue avec le lecteur à quelques centimètres. Pour un paiement, la puce calcule une signature **unique pour chaque transaction**, à partir d'une clé qu'elle conserve et qui ne peut pas en être extraite. Intercepter cet échange ne permet donc pas de rejouer le paiement plus tard.

---

## L'authentification adaptative

**Le principe.** Le service évalue le niveau de risque de chaque connexion à partir de signaux de contexte : nouvel appareil, nouveau pays, adresse réseau inhabituelle, horaire atypique. Si le risque dépasse un seuil, il demande une vérification supplémentaire. Dans le cas contraire, la connexion reste fluide.

**Une limite à garder en tête.** L'« empreinte » d'un navigateur (sa configuration, ses polices, sa résolution d'écran…) est un **indice**, pas une preuve d'identité. Elle varie dans le temps, peut être masquée par l'utilisateur, et peut être imitée par un attaquant. Elle sert à détecter une anomalie, jamais à authentifier quelqu'un à elle seule.

---

## La vérification d'identité (KYC)

**Le principe.** Certains services (banques, plateformes financières, sites de location) ont l'obligation légale de vérifier l'identité réelle de leurs clients : c'est le *Know Your Customer*. L'utilisateur photographie sa pièce d'identité puis prend un selfie. Un contrôle vérifie qu'il s'agit d'une personne réelle et présente.

**En pratique.** Ce processus est presque toujours confié à des prestataires spécialisés (Onfido, Veriff, Stripe Identity). Le service ne reçoit que le résultat de la vérification et n'a pas à conserver lui-même les documents d'identité, ce qui réduit fortement son exposition en cas de fuite.

---

# Partie 2 — L'accès aux serveurs

Les serveurs n'ont pas les mêmes besoins que le grand public. L'enjeu n'est plus la simplicité d'usage : il faut protéger des machines critiques, souvent administrées à distance par une poignée de personnes, et parfois des programmes qui dialoguent entre eux sans intervention humaine.

Cette partie reprend le découpage présenté en introduction : **prouver une identité**, **limiter les droits**, **filtrer l'accès réseau** et **organiser l'architecture d'accès**.

## Vue d'ensemble

| Méthode | Catégorie | Niveau de sécurité | Adaptée à |
|---|---|---|---|
| Mot de passe SSH | Authentification | Faible | À éviter |
| Clé SSH | Authentification | Élevé | Quelques serveurs |
| Certificat SSH | Authentification | Élevé | Parc de serveurs, équipes |
| Kerberos | Authentification | Élevé | Grandes organisations |
| Certificats mutuels (mTLS) | Authentification | Élevé | Échanges entre services |
| Identité de workload cloud | Authentification | Élevé | Applications hébergées dans le cloud |
| Pare-feu / liste d'adresses autorisées | Filtrage réseau | Complémentaire | Toutes situations |
| WireGuard (VPN) | Réseau chiffré | Élevé | Accès d'administration |
| Bastion | Architecture | Complémentaire | Plusieurs serveurs |

---

## Prouver une identité

### La clé SSH

**SSH** est le protocole standard pour administrer un serveur à distance. On peut s'y connecter avec un mot de passe, mais cette méthode est exposée aux tentatives automatisées que subit en permanence tout serveur accessible sur Internet.

La bonne pratique consiste à utiliser une **paire de clés**, sur le même principe que les passkeys : la clé privée reste sur l'ordinateur de l'administrateur, et la clé publique est déposée sur le serveur. Une fois les clés en place, on **désactive complètement la connexion par mot de passe**.

**Sa limite.** Sur un grand nombre de serveurs, il faut déposer et retirer manuellement chaque clé sur chaque machine, ce qui devient difficile à suivre lorsqu'un collaborateur quitte l'équipe.

### Le certificat SSH

**Le principe.** Plutôt que de déposer chaque clé sur chaque serveur, on crée une **autorité de certification** interne, en quelque sorte un service qui délivre des laissez-passer. Les serveurs font confiance à cette autorité, et non plus à chaque clé individuellement.

**Ses avantages :**

- un nouveau collaborateur reçoit un certificat, sans intervention sur les serveurs ;
- chaque certificat a une **durée de validité limitée** (par exemple douze heures) et expire de lui-même, sans action manuelle.

Des outils comme **HashiCorp Vault** ou **Teleport** automatisent la délivrance de ces certificats.

### Kerberos

**Le principe.** Utilisé historiquement dans les grandes entreprises (c'est le socle de Microsoft Active Directory), Kerberos fonctionne comme un **guichet central** : l'utilisateur s'identifie une fois auprès de lui, puis reçoit des « tickets » temporaires qui lui ouvrent l'accès aux différents services, sans ressaisir ses identifiants.

**Adapté à :** des parcs importants, avec une gestion centralisée des utilisateurs.

### Les certificats mutuels (mTLS)

**Le principe.** Lors d'une connexion sécurisée classique (le cadenas HTTPS), seul le serveur prouve son identité. Avec le **mTLS** (TLS mutuel), **les deux parties présentent un certificat** et se vérifient réciproquement avant tout échange.

**Adapté à :** la communication entre services internes, lorsque deux programmes doivent avoir la certitude de dialoguer avec le bon interlocuteur.

### L'identité de workload dans le cloud

**Le problème.** Une application a souvent besoin d'accéder à d'autres ressources (stockage, base de données). La mauvaise pratique consiste à inscrire une clé d'accès permanente dans son code ou sa configuration : si elle fuite, elle reste exploitable indéfiniment.

**La solution.** Les grands fournisseurs de cloud (AWS, Azure, Google Cloud) permettent d'attribuer une **identité directement à la machine ou à l'application**. Celle-ci reçoit automatiquement des **accès temporaires**, renouvelés en continu, sans aucune clé stockée sur le disque.

**Une nuance importante.** Retirer les droits d'une application n'annule pas immédiatement les accès temporaires déjà délivrés : ils restent valables jusqu'à leur expiration, sauf à appliquer une règle de révocation explicite.

**Un exemple de risque connu.** Sur AWS, l'ancienne version du service qui distribue ces accès répondait à n'importe quelle requête interne. Une application vulnérable pouvait être manipulée pour interroger ce service à la place d'un attaquant et lui transmettre les accès. La version actuelle (**IMDSv2**) exige une étape préalable que ce type d'attaque ne peut généralement pas reproduire. Elle doit être imposée sur toutes les machines.

### Les échanges entre machines

Lorsque deux programmes communiquent sans intervention humaine, plusieurs approches coexistent, de la plus simple à la plus robuste :

- **la clé d'API** : un secret partagé, simple à mettre en place, mais qui doit être renouvelé régulièrement et ne jamais figurer dans un dépôt de code ;
- **les jetons temporaires** (OAuth « client credentials », JWT) : le programme obtient un jeton de courte durée pour chaque série d'appels ;
- **l'identité automatique** (Kubernetes, SPIFFE) : chaque composant reçoit automatiquement une identité vérifiable, sans gestion manuelle de secrets.

---

## Limiter les droits : l'autorisation

Une fois l'identité établie, il reste à décider **ce que cette personne a le droit de faire**.

Sur un serveur Linux, c'est le rôle de **`sudo`** : il définit quelles commandes d'administration chaque utilisateur peut exécuter, et garde une trace de leur usage. Il ne vérifie pas qui vous êtes, puisque c'est déjà fait. Il décide de ce que vous pouvez faire.

**Renforcer la connexion elle-même.** On peut exiger, en plus de la clé SSH, un **code à usage unique** généré par une application sur téléphone. Un attaquant qui aurait dérobé la clé privée ne pourrait alors pas se connecter sans le téléphone. Sur les versions récentes d'OpenSSH, cette configuration demande une attention particulière : un réglage incomplet peut ajouter une demande de mot de passe non souhaitée.

---

## Filtrer l'accès réseau

Cette catégorie ne répond pas à la question « qui êtes-vous ? » mais à **« d'où venez-vous ? »**. Elle complète l'authentification et ne la remplace jamais.

### Le pare-feu et la liste d'adresses autorisées

**Le principe.** Le serveur n'accepte les connexions d'administration que depuis certaines adresses réseau, par exemple celle du bureau.

**Sa limite.** Une adresse réseau ne prouve l'identité de personne. Elle est partagée par tous les appareils d'un même réseau, et si l'un d'eux est compromis, le filtre ne protège plus rien. C'est pourquoi les approches modernes, dites **« zero trust »**, exigent que chaque connexion prouve son identité, quelle que soit sa provenance.

### Fail2ban

**Le principe.** Cet outil surveille les tentatives de connexion et **bannit temporairement** les adresses qui accumulent les échecs. Il réduit considérablement le bruit des attaques automatisées, sans authentifier personne pour autant.

### WireGuard

**Le principe.** WireGuard est un **VPN** moderne : il crée un tunnel chiffré entre l'ordinateur de l'administrateur et le serveur. Les services sensibles ne sont alors accessibles qu'à l'intérieur de ce tunnel, invisibles depuis Internet.

**Pourquoi il va plus loin qu'un simple filtrage.** Chaque participant au tunnel est identifié par une clé cryptographique, et le serveur n'accepte que les échanges signés par une clé qu'il connaît. Contrairement à une liste d'adresses, WireGuard vérifie donc réellement l'identité de chaque machine. Il se distingue aussi par la simplicité de sa conception, qui facilite son audit.

---

## Organiser l'architecture d'accès

### Le bastion

**Le principe.** Un seul serveur est exposé sur Internet : le **bastion**. Tous les autres ne sont accessibles qu'en passant par lui. On concentre ainsi les efforts de protection et de surveillance sur un point d'entrée unique.

**Sa contrepartie.** Ce point unique devient une cible prioritaire. Il doit être particulièrement durci, mis à jour et journalisé.

### Le réseau maillé (Tailscale, Headscale)

**Le principe.** Ces outils s'appuient sur WireGuard et en automatisent la gestion : chaque machine s'enregistre via le compte professionnel de l'utilisateur (Google, GitHub…), et un serveur central distribue les clés et applique des règles d'accès définies une fois pour toutes. **Headscale** est l'alternative open source qui permet d'héberger soi-même ce serveur central.

---

## Protéger les secrets

Mots de passe de bases de données, clés d'API, certificats : un système manipule de nombreux secrets. Les laisser dans des fichiers de configuration ou des variables d'environnement les expose à une fuite.

Des outils spécialisés comme **HashiCorp Vault** ou les services de gestion de clés des fournisseurs cloud centralisent ces secrets. Ils peuvent même générer des **secrets temporaires à la demande** : un accès à une base de données créé pour une heure, puis supprimé automatiquement.

Au niveau le plus exigeant, un **HSM** (module matériel de sécurité) conserve les clés dans un composant physique dont elles ne peuvent jamais être extraites. Le composant effectue les opérations de signature lui-même, sans jamais révéler la clé.

---

## Conclusion

**Aucune méthode ne constitue à elle seule une garantie absolue.** La sécurité d'un système dépend de l'ensemble de la chaîne :

- la méthode d'authentification ;
- la gestion des sessions après la connexion ;
- la limitation des droits ;
- la robustesse de la récupération de compte en cas de perte d'accès ;
- la mise à jour de l'infrastructure ;
- la qualité de la mise en œuvre.

Un mécanisme solide mal entouré n'apporte qu'une sécurité illusoire : une clé SSH robuste sur un serveur jamais mis à jour, ou une passkey impeccable associée à une récupération de compte par simple email.

Les tendances de fond sont claires : **remplacer les secrets permanents par des accès temporaires**, **ne plus faire confiance à l'origine réseau**, et **remplacer les mots de passe par des clés cryptographiques** qui ne quittent jamais l'appareil de l'utilisateur.

---

## Sources

**Standards et spécifications**

- IETF — [RFC 9106 — Argon2](https://www.rfc-editor.org/rfc/rfc9106.html)
- IETF — [RFC 6238 — Codes à usage unique basés sur le temps (TOTP)](https://www.rfc-editor.org/rfc/rfc6238.html)
- IETF — [RFC 6749 — OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html)
- IETF — [RFC 7636 — PKCE](https://www.rfc-editor.org/rfc/rfc7636.html)
- IETF — [RFC 4120 — Kerberos V5](https://www.rfc-editor.org/rfc/rfc4120.html)
- OpenID Foundation — [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- W3C — [Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/)
- FIDO Alliance — [Passkeys](https://fidoalliance.org/passkeys/)

**Recommandations de référence**

- OWASP — [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- OWASP — [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- NIST — [SP 800-63B-4 — Digital Identity Guidelines](https://csrc.nist.gov/pubs/sp/800/63/b/4/final)

**Infrastructure et cloud**

- WireGuard — [Protocol & Cryptography](https://www.wireguard.com/protocol/)
- Dowling, Paterson — [A Cryptographic Analysis of the WireGuard Protocol](https://www.wireguard.com/papers/dowling-paterson-computational-2018.pdf)
- Tailscale — [How Tailscale works](https://tailscale.com/blog/how-tailscale-works)
- AWS — [Instance Metadata Service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- Datadog Security Labs — [Securing the EC2 Instance Metadata Service](https://securitylabs.datadoghq.com/articles/misconfiguration-spotlight-imds/)
- Microsoft — [Azure Managed Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- Google Cloud — [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- HashiCorp — [Vault documentation](https://developer.hashicorp.com/vault/docs)

**Cadre réglementaire et identité**

- Commission européenne — [Règlement eIDAS](https://digital-strategy.ec.europa.eu/en/policies/eidas-regulation)
- FranceConnect — [Documentation partenaires](https://docs.partenaires.franceconnect.gouv.fr/)
- Have I Been Pwned — [Pwned Passwords API](https://haveibeenpwned.com/API/v3#PwnedPasswords)
