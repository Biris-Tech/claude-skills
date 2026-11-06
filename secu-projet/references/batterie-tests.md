# Batterie de tests — 9 domaines

Préalable : **tout ce qui suit s'exécute sur la cible autorisée de l'utilisateur**
(voir la garde dans `../SKILL.md`). Trois familles :

| | Fréquence | Nature |
|---|---|---|
| **A** automatisé en CI | chaque PR | fait échouer la construction |
| **B** manuel périodique | trimestriel + gros changements | demande un cerveau |
| **C** exercice à blanc | semestriel | vérifie qu'on *réagit* |

Verdict global : pas un score, une **couverture**. Chaque ligne est VÉRIFIÉE
(preuve) ou NON VÉRIFIÉE. Jamais « OK » sans preuve.

---

## 0. Inventaire (préalable, aussi bloquant que N0 en perf)

| Test | Méthode | Attendu |
|---|---|---|
| Sous-domaines oubliés | `subfinder -d domaine`, recoupé avec `crt.sh` | chacun a un propriétaire connu |
| Ports exposés | `nmap -sV --top-ports 1000 cible` | 80/443 seulement ; base jamais joignable |
| Services vivants | `httpx -l hosts.txt -title -tech-detect -status-code` | aucune admin ni outil interne exposé |
| Récupération de sous-domaine | `nuclei -t http/takeovers/ -l hosts.txt` | aucun résultat |
| Inventaire des données | revue manuelle | quelles données sensibles, où, qui y accède |

Trouve presque toujours quelque chose : un `staging.`, un `old.`, un `metabase.`
que personne n'a fermé.

---

## 1. Autorisations — priorité n°1, aucun scanner ne la trouve

Aucun outil ne connaît qui a droit à quoi. C'est pour ça que c'est la faille la
plus répandue et la plus coûteuse.

**Test manuel (B)** — deux comptes A et B, chacun ses données. Connecté en A,
rejouer chaque requête de B en changeant l'identifiant d'objet :

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN_A" \
  https://cible/factures/$ID_DE_B
```

Attendu : **403 ou 404, jamais 200** (404 préférable, il ne confirme pas
l'existence). L'extension **Autorize** de Burp automatise ce rejeu A/B pendant
la navigation.

**Automatisation (A)** — c'est la **suite de tests**, pas un scanner. Un test
par endpoint sensible : « B appelle la ressource de A → 403 ». Seul moyen
d'attraper la régression le jour où quelqu'un ajoute un endpoint sans le
contrôle.

À tester aussi :
- **Assignation de masse** : `PATCH /moi` avec `{"role":"admin"}` ou
  `{"isAdmin":true}` → champ ignoré, pas enregistré.
- **Contrôle serveur seul** : appeler directement l'endpoint dont le bouton est
  masqué côté interface.
- **Révocation** : après déconnexion, rejouer le jeton → échec. Après changement
  de mot de passe, rejouer l'ancienne session → échec.
- **JWT** : `jwt_tool jeton -X a` (signature retirée) et algorithme `none` →
  rejet dans les deux cas.
- **Sessions** : cookies `HttpOnly` + `Secure` + `SameSite`, rotation de l'ID à
  la connexion et à l'élévation de privilège.

---

## 2. Secrets et configuration — priorité n°2

```bash
gitleaks detect --source . --log-opts="--all" --redact
trufflehog git file://. --only-verified   # teste si la clé est encore vivante
```

`--all` = tout l'historique. Un secret trouvé se **tourne**, il ne se supprime
pas simplement.

Endpoints oubliés :

```bash
for p in .env .git/HEAD .git/config swagger.json api-docs actuator/env \
         debug phpinfo.php .DS_Store backup.sql; do
  curl -s -o /dev/null -w "$p -> %{http_code}\n" https://cible/$p
done
```

Attendu : **404 partout**. Puis `ffuf -w wordlist.txt -u https://cible/FUZZ
-mc 200,301,403`. Sur GraphQL, l'introspection (`{__schema{types{name}}}`) doit
être refusée en prod.

**CORS** : `curl -H "Origin: https://evil.com" -I https://cible/me` →
`Access-Control-Allow-Origin` ne doit **pas** renvoyer `evil.com`. Reflet de
l'origine + `Allow-Credentials: true` = API offerte à n'importe quel site.

Débogage : messages d'erreur génériques en prod, pas de trace d'appels, pas de
cartes de source publiées, identifiants d'usine changés.

---

## 3. Injections

| Point | Test | Famille |
|---|---|---|
| SQL — code | `semgrep --config p/sql-injection` + grep des requêtes concaténées | A |
| SQL — comportement | `sqlmap -u 'https://cible/produit?id=1' --batch --level 3` | B |
| SQL — test unitaire | injecter `' OR 1=1--` dans chaque paramètre, assert résultat vide (pas erreur) | A |
| XSS | `<img src=x onerror=alert(1)>` dans chaque champ, vérifier affichage **et** exports (PDF, CSV, mail) | B |
| XSS — statique | `semgrep` sur `dangerouslySetInnerHTML`, `v-html`, `innerHTML` | A |
| CSP | coller les en-têtes dans **CSP Evaluator** (Google) | A |

**SSRF** — sur chaque endroit où le serveur va chercher une URL fournie :

```bash
for u in http://169.254.169.254/latest/meta-data/ http://127.0.0.1:22 \
         http://[::1]/ http://2130706433/; do
  curl -s -X POST https://cible/api/import-image -d "url=$u" \
    -o /dev/null -w "$u -> %{http_code}\n"
done
```

Tous refusés. Puis le cas vicieux : une URL **publique qui redirige (302)** vers
`169.254.169.254` — beaucoup de validations ne vérifient que la première URL.
`169.254.169.254` est l'endpoint de métadonnées du cloud, par lequel on
récupère des identifiants d'instance : la cible la plus payante d'un SSRF.

**Téléversement** : tenter `.php`, `.svg` (XSS exécutable), double extension
`img.jpg.php`, type MIME menti, fichier de 2 Go, nom `../../etc/passwd`. Le
fichier accepté ne doit **pas être exécutable** à l'URL où il est servi.

---

## 4. En-têtes et TLS

```bash
curl -sI https://cible | grep -iE \
 'strict-transport|content-security|x-content-type|referrer-policy|permissions-policy|set-cookie'
testssl.sh https://cible
```

CSP présente (vrai filet anti-XSS), `frame-ancestors` (anti-détournement de
clic), HSTS, cookies `HttpOnly`+`Secure`+`SameSite`. **Mozilla Observatory** pour
une note d'ensemble. **CSRF** : construire une page sur un autre domaine qui
poste vers une mutation, l'ouvrir connecté — si l'action passe, problème.
Surveiller l'**expiration des certificats** par une sonde (alerte J-21), pas un
agenda.

---

## 5. Dépendances

`osv-scanner -r .` ou `npm audit --audit-level=high`, `trivy image` et
`trivy config .` — en CI, seuil bloquant. Contrôle manuel que personne ne fait :
la liste des **scripts tiers en front**, chacun avec un propriétaire nommé.
Chaque script tiers est une exécution de code arbitraire sur la page, avec accès
au DOM et aux cookies accessibles — auto-héberger ou poser un SRI.

---

## 6. Données et sauvegardes

| Test | Méthode | Famille |
|---|---|---|
| Données perso dans les journaux | envoyer une requête avec un marqueur (`motdepasse=CANARI_X9`), chercher `CANARI_X9` dans **tous** les outils de logs | B |
| Restauration de sauvegarde | restaurer réellement dans un env vierge, chronométrer, vérifier l'intégrité | **C** |
| Clés | la clé est-elle ailleurs que la donnée ? qui peut la lire ? | B |
| Données de prod en test | un dump de test contient-il de vraies adresses mail ? | B |

La restauration est l'équivalent exact du test de charge en perf : la règle que
tout le monde coche sans l'avoir vérifiée. Une sauvegarde jamais restaurée
n'existe pas.

---

## 7. API

`ffuf` sur les anciennes versions (`/v1/`, `/api/old/`) pour les endpoints
zombies non maintenus. Une liste sans paramètre ne doit pas renvoyer 50 000
enregistrements (pagination bornée côté serveur). GraphQL : limite de profondeur
et de complexité, abus de lots bridé.

---

## 8. Détection — le test le plus révélateur (C)

> Lancer 50 connexions échouées depuis une IP. **Quelqu'un est-il prévenu, et en
> combien de temps ?**

Variantes : exporter 10 000 enregistrements d'un coup ; passer un compte en
admin. Si rien ne se déclenche, la détection est une intention. Mesurer deux
délais chiffrés : **temps de détection** et **temps de révocation de tous les
jetons**. Avoir un **plan d'incident écrit** (qui on appelle, comment on révoque
tout, comment on communique) et un `security.txt`.

---

## 9. Comptes, humains, DNS — souvent absent des audits techniques

Pas de scanner, une revue (B) :
- **Double facteur vérifié** sur le registrar du domaine, le cloud, GitHub.
  Prendre le domaine = prendre le site, les mails et les certificats, sans
  toucher au code.
- Verrouillage du domaine : `dig +short domaine NS` et l'état
  `clientTransferProhibited` au whois.
- Liste des accès confrontée aux personnes réellement présentes ; accès retirés
  aux départs.
- Protection de branche active ; revue obligatoire sur l'authentification et le
  paiement.

---

## Fiche de restitution

```
Projet :                    Date :            Cible testée :
Autorisation confirmée : oui/non      Cible = préprod/test : oui/non

Couverture :
  0 inventaire      ___ vérifiées
  1 autorisations   ___            ← priorité, détailler chaque endpoint
  2 secrets/config  ___
  3 injections      ___
  4 en-têtes/TLS    ___
  5 dépendances     ___
  6 données/backup  ___
  7 API             ___
  8 détection       ___
  9 comptes/DNS     ___

Failles ouvertes, triées par exploitabilité × donnée exposée :
  1.
  2.
Délai de détection mesuré : ___    Délai de révocation : ___
Points NON VÉRIFIABLES faute d'instrumentation :
```
