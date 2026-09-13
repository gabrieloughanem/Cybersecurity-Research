# Sessions et contrôle d'accès

## Vue d'ensemble

Une fois l'identité vérifiée, l'application doit maintenir un état de connexion et déterminer ce que l'utilisateur authentifié est autorisé à faire. Ces deux problématiques sont distinctes : la gestion de session maintient l'authentification dans le temps, tandis que le contrôle d'accès détermine les opérations et les ressources auxquelles l'utilisateur peut accéder.

## Sessions

La session permet de maintenir l'état d'authentification entre plusieurs requêtes sans demander à l'utilisateur de se réauthentifier à chaque fois.

Deux grandes approches sont courantes :

- sessions stateful, avec un identifiant opaque associé à un état conservé côté serveur ;
- tokens autonomes, contenant les informations nécessaires à leur validation, par exemple certains JWT.

Points étudiés :

- génération d'identifiants de session imprévisibles ;
- expiration et renouvellement ;
- rotation des identifiants ;
- révocation et invalidation ;
- fixation de session ;
- stockage et transmission des identifiants ;
- invalidation lors de la déconnexion ou d'un changement de mot de passe ;
- principe du moindre privilège appliqué aux sessions et aux permissions.

## Cookies

Le cookie est un mécanisme courant pour transporter un identifiant de session dans les applications web.

Points étudiés :

- attributs `HttpOnly`, `Secure` et `SameSite` ;
- portée avec `Domain` et `Path` ;
- cookies de première partie et cookies tiers ;
- durée de vie : session ou persistance ;
- implications de `SameSite` pour les requêtes cross-site ;
- différence entre stockage dans un cookie et stockage côté navigateur comme `localStorage`.

Un cookie `HttpOnly` permet notamment d'empêcher l'accès direct à sa valeur depuis JavaScript côté client. Il ne constitue cependant pas une protection générale contre les attaques XSS.

## CSRF

Le Cross-Site Request Forgery (CSRF) exploite le fait qu'un navigateur peut transmettre automatiquement certaines informations d'authentification lorsqu'il effectue une requête vers un site.

Points étudiés :

- mécanisme de l'attaque ;
- tokens CSRF et synchronizer token pattern ;
- rôle de `SameSite` comme défense complémentaire ;
- validation de l'origine lorsque pertinente ;
- risques liés aux requêtes GET qui modifient l'état ;
- différences entre formulaires web et API ;
- distinction entre authentification par cookie et authentification par bearer token transmis explicitement dans `Authorization`.

`SameSite` peut réduire certaines possibilités d'attaque CSRF, mais ne doit pas être considéré comme l'unique mécanisme de défense. Le niveau de protection nécessaire dépend notamment du mécanisme d'authentification et de la manière dont les informations d'authentification sont transmises.

## RBAC

Le Role-Based Access Control (RBAC) associe des permissions à des rôles, eux-mêmes attribués aux utilisateurs.

Points étudiés :

- modélisation des rôles et permissions ;
- principe du moindre privilège ;
- hiérarchie de rôles ;
- séparation des responsabilités ;
- limites du RBAC pour les règles fines ou contextuelles ;
- risques de prolifération des rôles.

## ABAC

L'Attribute-Based Access Control (ABAC) évalue l'autorisation à partir d'attributs liés à l'utilisateur, à la ressource et au contexte d'exécution.

Points étudiés :

- attributs du sujet, de la ressource et de l'environnement ;
- politiques et règles conditionnelles ;
- prise en compte du contexte ;
- flexibilité par rapport au RBAC ;
- complexité accrue des politiques ;
- combinaison de RBAC et d'ABAC dans les systèmes réels.

## Autorisation au niveau des ressources

L'existence d'une permission générale ne garantit pas que l'accès à chaque ressource est correctement contrôlé.

Une application peut par exemple vérifier qu'un utilisateur possède la permission `read_document`, tout en oubliant de vérifier qu'il est effectivement autorisé à consulter ce document précis.

Ce type de défaut peut conduire à une vulnérabilité IDOR (Insecure Direct Object Reference) lorsqu'une référence à un objet est utilisée sans contrôle d'accès approprié. Dans le contexte des API, OWASP désigne ce type de défaillance d'autorisation au niveau des objets sous le terme BOLA (Broken Object Level Authorization). Les deux notions sont proches, mais ne sont pas strictement synonymes : IDOR décrit notamment un cas de référence directe à un objet, tandis que BOLA désigne plus largement une défaillance d'autorisation au niveau des objets dans une API.

Le contrôle d'accès doit donc être appliqué à la fois :

- à l'opération ;
- à la ressource ;
- au contexte lorsque nécessaire.

## Limites

Un modèle de permissions correctement conçu peut devenir inefficace si le cloisonnement des données n'est pas cohérent avec ce modèle.

La séparation entre utilisateurs, organisations, comptes ou ressources doit être prise en compte dans l'architecture et dans les requêtes aux données, plutôt que de reposer uniquement sur l'interface ou sur une vérification superficielle des permissions.

Le modèle de contrôle d'accès, le code applicatif et le schéma de données doivent donc rester cohérents.

## Sources

- IETF — [RFC 6265](https://www.rfc-editor.org/rfc/rfc6265.html) — HTTP State Management Mechanism — mécanisme général des cookies
- IETF — [draft-ietf-httpbis-rfc6265bis](https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/) — Cookies: HTTP State Management Mechanism — Internet-Draft définissant notamment `SameSite`, à distinguer de RFC 6265 qui est la spécification RFC publiée
- IETF — [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html) — JSON Web Token (JWT)
- OWASP — [Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- OWASP — [Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- OWASP — [Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- OWASP — [API1:2023 — Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- OWASP — [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- Sandhu, Ferraiolo, Kuhn (NIST) — [The NIST Model for Role-Based Access Control: Towards a Unified Standard](https://www.nist.gov/publications/nist-model-role-based-access-control-towards-unified-standard) (2000)
- NIST — [SP 800-162](https://csrc.nist.gov/pubs/sp/800/162/upd2/final) — Guide to Attribute Based Access Control (ABAC) Definition and Considerations
