# Sécurité web : headers, CORS, CSP et protection contre l'abus

## Vue d'ensemble

Au-delà de l'authentification et du contrôle d'accès, le navigateur applique un ensemble de règles de sécurité, dont certaines peuvent être configurées par le serveur via des en-têtes HTTP. Ces mécanismes ne remplacent pas une architecture sécurisée côté serveur, mais réduisent la surface d'exploitation de certaines classes d'attaques côté client.

## Headers de sécurité

Points étudiés :

- `Strict-Transport-Security` (HSTS) — indique au navigateur d'utiliser HTTPS pour les requêtes ultérieures vers l'origine concernée ;
- `X-Content-Type-Options: nosniff` — empêche le navigateur de deviner un type MIME différent de celui déclaré ;
- `frame-ancestors` (CSP) / `X-Frame-Options` — contrôle de l'intégration dans des frames et défense contre le clickjacking, `frame-ancestors` étant le mécanisme moderne, `X-Frame-Options` restant utile pour la compatibilité avec certains clients ;
- `Referrer-Policy` — contrôle les informations transmises dans l'en-tête `Referer` ;
- `Permissions-Policy` — contrôle quelles origines peuvent utiliser certaines fonctionnalités du navigateur, selon les directives configurées ;
- `Cross-Origin-Resource-Policy` (CORP), `Cross-Origin-Opener-Policy` (COOP), `Cross-Origin-Embedder-Policy` (COEP) — mécanismes d'isolation entre contextes et ressources cross-origin, notamment utiles pour réduire certaines classes d'attaques par canaux auxiliaires.

## CORS

Le Cross-Origin Resource Sharing détermine si une page peut lire la réponse d'une requête effectuée vers une autre origine.

Points étudiés :

- politique de même origine (same-origin policy) comme comportement par défaut du navigateur ;
- en-têtes `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`, `Access-Control-Allow-Methods` ;
- requêtes simples vs requêtes nécessitant une requête préliminaire (`preflight`, méthode `OPTIONS`) ;
- `Access-Control-Allow-Origin: *` ne peut pas être utilisé pour autoriser une requête CORS avec credentials — les navigateurs rejettent cette combinaison. Une politique CORS trop permissive (allowlist mal contrôlée, réflexion de l'origine sans validation) peut néanmoins exposer des données à des origines non prévues ;
- CORS protège la lecture de la réponse par le navigateur, mais n'empêche pas la requête serveur à serveur elle-même d'être envoyée — ce n'est pas un mécanisme de contrôle d'accès côté serveur.

La spécification normative actuelle de CORS n'est pas la Recommendation W3C historique (obsolète depuis 2017) mais le [Fetch Living Standard](https://fetch.spec.whatwg.org/) du WHATWG, mis à jour en continu.

## CSP

Le Content Security Policy restreint les sources depuis lesquelles une page peut charger et exécuter des ressources (scripts, styles, images, etc.), en défense principalement contre les attaques XSS.

Points étudiés :

- directives `default-src`, `script-src`, `style-src`, `frame-ancestors`, `object-src` ;
- `nonce` et hachages (`sha256-...`) pour autoriser des scripts inline spécifiques sans `unsafe-inline` ;
- mode `report-only` pour tester une politique sans la faire respecter ;
- limites : CSP peut réduire l'exploitabilité de certaines injections XSS, notamment grâce aux nonces et aux hachages, mais ne corrige pas la vulnérabilité d'injection elle-même.

CSP Level 2 est la version publiée comme W3C Recommendation ; CSP Level 3 est toujours un Working Draft, bien que largement implémenté par les navigateurs pour ses fonctionnalités les plus stables (`nonce`, `strict-dynamic`).

## Rate limiting et protection contre l'abus

Points étudiés :

- limitation par IP, par compte, par clé d'API, ou par combinaison de ces critères ;
- algorithmes et modèles courants : fenêtre fixe, fenêtre glissante, token bucket, leaky bucket ;
- réponse `429 Too Many Requests`, en-tête `Retry-After` ;
- rate limiting sur les tentatives d'authentification comme défense contre le credential stuffing et le brute force, distinct du rate limiting général anti-abus ;
- rate limiting applicatif vs limitation au niveau infrastructure (reverse proxy, WAF, CDN).

## Limites

Les mécanismes déclaratifs (CSP, CORS, HSTS) dépendent de leur interprétation correcte par le client : un client qui ne les applique pas, ou qui ne fournit pas les mécanismes de sécurité du navigateur, n'est pas protégé. Ces mécanismes réduisent des classes d'attaques côté navigateur, mais ne dispensent pas d'une validation et d'un contrôle d'accès rigoureux côté serveur.

## Sources

- IETF — [RFC 6797](https://www.rfc-editor.org/rfc/rfc6797.html) — HTTP Strict Transport Security (HSTS)
- WHATWG — [Fetch Standard](https://fetch.spec.whatwg.org/) — spécification normative actuelle de CORS
- W3C — [Content Security Policy Level 2](https://www.w3.org/TR/CSP2/) — Recommendation publiée
- W3C — [Content Security Policy Level 3](https://www.w3.org/TR/CSP3/) — Working Draft, non finalisé mais largement implémenté
- OWASP — [HTTP Security Response Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- OWASP — [Denial of Service Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) — couvre le rate limiting
