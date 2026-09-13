# Authentification : du mot de passe aux passkeys

## Vue d'ensemble

L'authentification répond à une question simple : « êtes-vous bien qui vous prétendez être ? »

Les mécanismes utilisés pour y répondre diffèrent cependant fortement en matière de sécurité, de résistance au phishing, de friction utilisateur, de dépendances externes et de complexité d'implémentation.

## Comparatif

| Méthode | Sécurité | Friction utilisateur | Résistance au phishing | Dépendances externes | Complexité |
|---|---|---|---|---|---|
| Mot de passe | Variable | Faible | Faible | Faible | Faible |
| OTP / 2FA | Variable | Moyenne | Faible à moyenne | Faible | Moyenne |
| Magic link | Variable | Faible | Faible à moyenne | Email | Faible |
| OAuth / SSO | Variable | Faible | Variable | Élevée | Moyenne |
| Passkey / WebAuthn | Élevée | Faible | Élevée | Faible | Élevée |

Ces appréciations sont indicatives : la sécurité réelle dépend notamment de l'implémentation, de la configuration et du contexte d'utilisation.

## Mots de passe

Les mots de passe restent l'un des mécanismes d'authentification les plus répandus.

Points étudiés :

- stockage avec une fonction de hachage de mot de passe adaptée, telle qu'Argon2id ou bcrypt, plutôt qu'un hash cryptographique généraliste ;
- politiques de longueur et de complexité ;
- gestionnaires de mots de passe ;
- attaques par force brute ;
- credential stuffing ;
- réutilisation des mots de passe ;
- récupération et réinitialisation des comptes.

## OTP / 2FA

L'authentification à plusieurs facteurs ajoute un élément supplémentaire à la vérification de l'identité.

Points étudiés :

- TOTP et HOTP ;
- applications d'authentification ;
- SMS et appels téléphoniques ;
- codes de récupération ;
- risques liés au SIM swapping ;
- limites des codes à usage unique face au phishing.

## Magic links

Les magic links permettent d'authentifier un utilisateur à partir d'un lien envoyé par email.

Points étudiés :

- génération et expiration des liens ;
- jetons à usage unique ;
- protection contre la réutilisation ;
- sécurité de la boîte email ;
- interception et transfert des liens ;
- compromis entre simplicité et dépendance à l'email.

## OAuth / SSO

OAuth 2.0 est un framework d'autorisation permettant à une application d'obtenir un accès délégué à des ressources. OpenID Connect ajoute une couche d'authentification et d'identité au-dessus d'OAuth 2.0. Les systèmes de SSO peuvent s'appuyer sur OpenID Connect ou sur d'autres protocoles de fédération.

Points étudiés :

- Authorization Code Flow ;
- PKCE, mécanisme de protection du Authorization Code Flow, désormais recommandé pour les clients publics et largement recommandé pour les autres clients ;
- OAuth 2.0 (RFC 6749) et le projet OAuth 2.1, qui consolide plusieurs bonnes pratiques et mises à jour ultérieures d'OAuth 2.0 (Internet-Draft) ;
- rôles du client, du serveur d'autorisation et du resource server ;
- gestion des tokens ;
- redirections ;
- risques liés aux mauvaises implémentations ;
- différence entre OAuth 2.0 et OpenID Connect.

## Passkeys / WebAuthn

Les passkeys reposent notamment sur la cryptographie à clé publique et offrent une résistance structurelle au phishing.

Points étudiés :

- génération d'une paire de clés ;
- rôle de l'authenticator ;
- challenge cryptographique ;
- vérification de l'origine ;
- résistance structurelle au phishing ;
- WebAuthn (API W3C) et FIDO2 (ensemble plus large de spécifications de l'écosystème FIDO incluant WebAuthn et CTAP) — deux notions liées mais distinctes ;
- formats CBOR et COSE ;
- synchronisation des passkeys selon les écosystèmes.

## Vérification d'identité

La vérification d'identité répond à une problématique différente de l'authentification : établir ou vérifier qu'une personne correspond à une identité déclarée ou revendiquée.

Points étudiés :

- cas d'usage ;
- vérification documentaire ;
- biométrie ;
- KYC ;
- contraintes réglementaires ;
- protection des données ;
- coûts et friction utilisateur.

## Limites

Aucune méthode d'authentification ne constitue à elle seule une garantie absolue de sécurité.

La sécurité d'un système dépend également de la récupération de compte, de la gestion des sessions, de la protection des appareils, du contrôle d'accès, de l'infrastructure et de l'implémentation globale.

## Sources

- IETF/IRTF — [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106.html) — Argon2 Memory-Hard Function for Password Hashing (document Informational du CFRG, pas Standards Track)
- IETF — [RFC 4226](https://www.rfc-editor.org/rfc/rfc4226.html) — HOTP: An HMAC-Based One-Time Password Algorithm
- IETF — [RFC 6238](https://www.rfc-editor.org/rfc/rfc6238.html) — TOTP: Time-Based One-Time Password Algorithm
- IETF — [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) — The OAuth 2.0 Authorization Framework
- IETF — [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html) — Proof Key for Code Exchange (PKCE)
- OpenID Foundation — [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- W3C — [Web Authentication (WebAuthn) Level 3](https://www.w3.org/TR/webauthn-3/) — Recommendation, succède à WebAuthn Level 2 (2021)
- FIDO Alliance — [Passkeys](https://fidoalliance.org/passkeys/) — documentation officielle sur les passkeys et leur relation à FIDO2
- OWASP — [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- OWASP — [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- NIST — [SP 800-63B-4](https://csrc.nist.gov/pubs/sp/800/63/b/4/final) — Digital Identity Guidelines: Authentication and Authenticator Management (supersède SP 800-63B)
