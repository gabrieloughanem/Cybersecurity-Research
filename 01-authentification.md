# Authentification : du mot de passe aux passkeys

## Vue d'ensemble

L'authentification répond à une question simple : « êtes-vous bien qui vous prétendez être ? »

Les mécanismes utilisés pour y répondre diffèrent cependant fortement en matière de sécurité, de résistance au phishing, de friction utilisateur, de dépendances externes et de complexité d'implémentation.

## Comparatif

| Méthode | Sécurité | Friction utilisateur | Résistance au phishing | Dépendances externes | Complexité |
|---|---|---|---|---|---|
| Mot de passe | Moyenne | Faible | Faible | Faible | Faible |
| OTP / 2FA | Bonne | Moyenne | Faible à moyenne | Faible | Moyenne |
| Magic link | Bonne | Faible | Faible à moyenne | Email | Faible |
| OAuth / SSO | Variable | Faible | Variable | Élevée | Moyenne |
| Passkey / WebAuthn | Très élevée | Faible | Élevée | Faible | Élevée |

Ces appréciations sont indicatives : la sécurité réelle dépend notamment de l'implémentation, de la configuration et du contexte d'utilisation.

## Mots de passe

Les mots de passe restent l'un des mécanismes d'authentification les plus répandus.

Points étudiés :

- stockage via des fonctions de dérivation de clé (Argon2, bcrypt), plutôt qu'un hash simple ;
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

OAuth et les systèmes de SSO permettent notamment de déléguer certaines fonctions d'authentification ou d'autorisation à un fournisseur d'identité.

Points étudiés :

- Authorization Code Flow ;
- PKCE ;
- rôles du client, du serveur d'autorisation et du resource server ;
- gestion des tokens ;
- redirections ;
- risques liés aux mauvaises implémentations ;
- différence entre OAuth 2.0 et OpenID Connect.

## Passkeys / WebAuthn

Les passkeys reposent notamment sur la cryptographie à clé publique et permettent de réduire fortement l'exposition aux attaques par phishing.

Points étudiés :

- génération d'une paire de clés ;
- rôle de l'authenticator ;
- challenge cryptographique ;
- vérification de l'origine ;
- résistance structurelle au phishing ;
- WebAuthn et FIDO2 ;
- formats CBOR et COSE ;
- synchronisation des passkeys selon les écosystèmes.

## Vérification d'identité

La vérification d'identité répond à une problématique différente de l'authentification classique : établir ou vérifier l'identité réelle d'une personne.

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

Les références sont ajoutées au fil des recherches : RFC, spécifications W3C, standards FIDO, documentations techniques et publications spécialisées.
