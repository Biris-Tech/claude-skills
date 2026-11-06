# Bots : brute-force et scans d'injection

Deux menaces de bots, deux logiques de défense **opposées**. Les confondre est
l'erreur courante.

---

## Bourrage d'identifiants et force brute

### L'erreur de conception n°1 : ne limiter que par IP

Un attaquant moderne ne fait pas 10 000 essais depuis une IP. Il fait **un essai
sur 10 000 comptes depuis 10 000 IP résidentielles** (bourrage d'identifiants
avec des mots de passe déjà fuités ailleurs). Une limite par IP ne voit rien
passer.

### Trois compteurs simultanés

| Niveau | Seuil indicatif | Ce qu'il attrape |
|---|---|---|
| **Par IP** | 10 échecs / 10 min | le script bête |
| **Par compte**, toutes IP | 5 échecs / 15 min, puis attente croissante | le pulvérisage distribué |
| **Global**, tout le site | alerte si le taux d'échec > 3× la normale | la campagne en cours |

Le troisième est le plus précieux et le plus rare : il dit « on est attaqué
*maintenant* », indépendamment de la source.

### Les correctifs, par ordre d'efficacité

1. **Double facteur** — le seul vrai correctif. Tout le reste ralentit ; le
   second facteur annule le bénéfice d'un mot de passe volé. Obligatoire sur les
   comptes à privilèges, proposé partout.
2. **Vérification contre les fuites** à l'inscription et au changement, via
   l'API Pwned Passwords en **k-anonymat** (on n'envoie que 5 caractères du
   condensat SHA-1, jamais le mot de passe). Un mot de passe déjà fuité est le
   vecteur même du bourrage.
3. **Pas d'énumération** : réponse **et temps de réponse** identiques pour un
   compte inexistant et un mauvais mot de passe. Si on ne hache que quand le
   compte existe, le temps de réponse révèle la liste des utilisateurs.
   Tester : `curl -w '%{time_total}'` sur les deux cas, comparer.
4. **Attente croissante, pas verrouillage définitif** : un verrouillage
   permanent après N échecs devient un déni de service contre ses propres
   utilisateurs (il suffit d'échouer exprès sur leur compte).
5. **CAPTCHA seulement après quelques échecs**, jamais d'emblée.

### Protéger tous les points d'entrée, pas le formulaire web

- API mobile, flux OAuth, réinitialisation de mot de passe.
- **Vérification de code à usage unique** : très souvent sans limite. 6 chiffres
  sans plafond = 1 million d'essais, triviale à forcer. Plafonner et expirer.
- Inscription : les bots créent aussi des comptes en masse.

### C'est ici qu'un service anti-bot est à sa place

Contrairement aux autorisations, le travail est vraiment de la mitigation de
volume : réputation d'IP, empreinte de client, volumétrie. Cloudflare ou
équivalent retire 95 % du bruit avant l'application. Le WAF est utile **ici**,
là où il est inutile contre les autorisations.

### Comment tester

```bash
# limite par IP : doit renvoyer des 429 rapidement
hey -n 200 -c 10 -m POST \
  -d '{"email":"test@exemple.test","password":"x"}' \
  https://cible/login
```

Puis le vrai test : 200 essais sur **le même compte** depuis des sources
différentes. **Point à vérifier d'abord** : si l'application fait confiance à
l'en-tête `X-Forwarded-For` sans savoir combien de relais elle a devant elle, la
limite par IP est **contournable en un en-tête** — un bot le change à chaque
requête. Vérifier que seul le dernier relais de confiance est pris en compte.

---

## Scans d'injection SQL

Changement complet de logique. Ces bots sont **opportunistes et stupides** : ils
balancent `' OR 1=1--` sur tous les paramètres de tous les sites qu'ils
croisent. Ils n'ont rien ciblé en particulier.

### Les défenses, dans cet ordre

1. **Requêtes paramétrées partout.** C'est tout. Correctement paramétré, les
   bots peuvent cogner dix ans. **Seule défense qui ferme le problème** ; tout
   le reste ne fait que réduire le bruit. Vérifier que l'ORM n'est pas contourné
   par du SQL brut concaténé quelque part.
2. **Privilèges minimaux du compte de base** : pas de `DROP`, pas de `FILE`,
   lecture seule où c'est possible, un compte distinct par service. Transforme
   une injection réussie en fuite partielle plutôt qu'en perte totale — la
   deuxième ligne, celle qui sert le jour où la première a un trou.
3. **Jamais d'erreur SQL dans la réponse.** Une erreur affichée fait gagner des
   heures à l'attaquant ; sans elle il doit passer à l'injection aveugle, cent
   fois plus lente et bruyante. Tester : une apostrophe simple dans chaque
   paramètre doit donner une erreur **générique**, pas un message du moteur.
4. **Règles WAF** (jeu OWASP CRS) : utile pour le volume, jamais comme
   correctif. Un WAF devant du SQL concaténé est un filtre devant une porte
   ouverte.
5. **Bannissement sur comportement** : une IP qui collectionne les
   400/404/403 est un scanner. Dix erreurs en une minute → bannissement d'une
   heure (`fail2ban` ou une règle de taux chez l'hébergeur). Économise surtout
   les journaux et le processeur.

### Le point de bascule à comprendre

Voir des tentatives d'injection dans les journaux est **normal** — tout site
exposé en reçoit dès le premier jour. Ce n'est pas un signal d'alerte.

Les deux seuls signaux qui comptent :
- **Une tentative a-t-elle réussi** : pic de temps de réponse, requêtes
  anormalement longues, erreurs 500 corrélées à des paramètres bizarres, volume
  de données sortant inhabituel.
- **Est-ce que je le verrais** : sans la détection du domaine 8 de la batterie,
  la réponse est non, et la question reste ouverte.

### Comment tester (cible autorisée)

- `sqlmap -u 'https://cible/produit?id=1' --batch --level 3 --risk 2` sur ses
  propres URL : doit ne rien trouver.
- Injecter une apostrophe dans chaque paramètre : vérifier qu'aucune erreur de
  moteur SQL ne remonte.
- Vérifier que le compte applicatif ne **peut pas** exécuter `DROP` même si une
  injection passait (test en base jetable).
