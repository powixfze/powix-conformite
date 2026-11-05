# Site « Powix Conformité » — powix.io

Site statique une page, FR + EN, sans framework, sans CDN, sans cookie, sans traceur, sans formulaire.

```
site/
├── index.html          page FR  → https://powix.io/
├── en/index.html       page EN  → https://powix.io/en/
├── og-image.png        image Open Graph FR (1200×630)
├── og-image-en.png     image Open Graph EN (1200×630)
├── vercel.json         en-têtes de sécurité + URLs propres (Vercel uniquement)
├── .vercelignore       exclut screenshots/, tools/ et ce README du déploiement CLI
├── screenshots/        captures de contrôle (ne pas déployer)
└── tools/              scripts Playwright de capture (ne pas déployer)
```

Le dossier `site/` est la racine du site : rien à compiler, rien à installer.

---

## 1. Déploiement recommandé : Vercel (nameservers déjà chez Vercel)

Contexte : le domaine `powix.io` est enregistré chez **Name.com** et ses nameservers pointent déjà vers **Vercel** (`ns1.vercel-dns.com` / `ns2.vercel-dns.com`). Le DNS est donc géré dans Vercel ; il suffit de rattacher le domaine au projet.

### 1.1 Créer le projet

**Option A — depuis GitHub (recommandé : chaque `git push` redéploie)**

1. Pousser le contenu de `site/` dans un dépôt Git (par ex. `powix/site`). Ne pas inclure `screenshots/` ni `tools/` (ou les laisser : ils sont inoffensifs, mais seraient servis publiquement ; le plus propre est de les retirer du dépôt).
2. Vercel → **Add New… → Project** → importer le dépôt.
3. Réglages :
   - **Framework Preset** : `Other`
   - **Root Directory** : `./` (ou `site` si le dépôt contient d'autres dossiers)
   - **Build Command** : vide — **Output Directory** : vide (laisser « Other », Vercel sert les fichiers tels quels)
4. **Deploy**. Le site est disponible sur `https://<projet>.vercel.app`.

**Option B — depuis le poste, sans Git**

```bash
npm i -g vercel
cd site
vercel login
vercel --prod        # répondre : projet statique, racine = ./, pas de build
```

### 1.2 Rattacher le domaine

1. Projet → **Settings → Domains** → **Add** → `powix.io` → Add.
2. Ajouter aussi `www.powix.io` et choisir **Redirect to powix.io** (308).
3. Comme le DNS est déjà chez Vercel, les enregistrements `A`/`CNAME` sont créés automatiquement : pas de ligne à poser chez Name.com. Vérification en quelques secondes, certificat HTTPS (Let's Encrypt) émis et renouvelé automatiquement ; HTTP redirigé vers HTTPS.

Si le domaine est déjà attaché à un autre projet Vercel, le retirer de celui-ci d'abord (Settings → Domains → Remove), puis l'ajouter au nouveau.

### 1.3 Enregistrements de messagerie (MX / TXT) — ne pas les perdre

Tout le DNS de powix.io se gère ici : **Vercel → compte/équipe → Domains → powix.io → DNS Records**.

| Type | Nom | Valeur | Usage |
|---|---|---|---|
| MX | `@` | selon le fournisseur de messagerie (priorité + hôte) | réception sur `yamani@` / `dev@` |
| TXT | `@` | `v=spf1 include:<fournisseur> ~all` | SPF |
| TXT / CNAME | `<sélecteur>._domainkey` | fourni par la messagerie | DKIM |
| TXT | `_dmarc` | `v=DMARC1; p=quarantine; rua=mailto:yamani@powix.io` | DMARC |

Rattacher le domaine au projet **ne touche pas** aux MX/TXT existants ; vérifier simplement, avant et après, que les enregistrements de messagerie sont bien présents dans DNS Records.

### 1.4 Ce que fait `vercel.json`

- `cleanUrls: true` + `trailingSlash: true` : `/en/` sert `en/index.html` ; `/index.html` redirige vers `/`.
- En-têtes de sécurité sur toutes les URLs : **CSP** sans aucune ressource externe (`default-src 'none'` ; styles et script inline autorisés ; images `self` + `data:` pour le favicon), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`, **HSTS** 1 an, COOP/CORP.
- Si un jour une ressource externe est ajoutée (police, image, script), la CSP devra être élargie, sinon le navigateur la bloquera.

Contrôle après mise en ligne : `curl -sI https://powix.io/ | grep -i -E "strict-transport|content-security|x-content"`.

---

## 2. Alternative : Cloudflare Pages

Nécessite de **changer les nameservers chez Name.com** (le DNS passerait de Vercel à Cloudflare). À ne faire que si l'on quitte Vercel, car tous les enregistrements (y compris MX/TXT) doivent alors être recréés chez Cloudflare.

1. Cloudflare → **Add a site** → `powix.io` → plan Free. Cloudflare importe les enregistrements qu'il détecte : **vérifier manuellement MX, SPF, DKIM, DMARC** et compléter.
2. Chez **Name.com** → My Domains → powix.io → **Nameservers** → remplacer `ns1/ns2.vercel-dns.com` par les deux nameservers indiqués par Cloudflare. Propagation : de quelques minutes à 24 h.
3. Cloudflare → **Workers & Pages → Create → Pages** :
   - depuis Git : connecter le dépôt, Build command vide, Build output directory `/` (ou `site`) ;
   - ou **Upload assets** : glisser le contenu de `site/`.
4. Pages → projet → **Custom domains** → `powix.io` (Cloudflare crée le CNAME lui-même) ; ajouter `www.powix.io` puis une **Bulk Redirect** ou une Redirect Rule `www → apex`.
5. HTTPS automatique. Les en-têtes de `vercel.json` ne s'appliquent pas : créer un fichier `_headers` à la racine :

```
/*
  Content-Security-Policy: default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline'; img-src 'self' data:; font-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'; upgrade-insecure-requests
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=(), interest-cohort=()
  Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## 3. Alternative : GitHub Pages

1. Dépôt public (ou privé avec GitHub Pro) contenant `index.html`, `en/index.html`, les deux PNG ; ajouter un fichier `CNAME` contenant `powix.io`.
2. Settings → **Pages** → Source : branche `main`, dossier `/` (root).
3. DNS (chez Vercel tant que les nameservers y sont, sinon chez Name.com/Cloudflare) :
   - `A @` → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
   - `AAAA @` → `2606:50c0:8000::153`, `8001::153`, `8002::153`, `8003::153`
   - `CNAME www` → `<utilisateur>.github.io`
4. Settings → Pages → Custom domain `powix.io` → cocher **Enforce HTTPS** une fois le DNS vérifié.
5. GitHub Pages ne permet pas d'en-têtes personnalisés (pas de CSP/HSTS côté serveur) : c'est la raison de préférer Vercel ou Cloudflare.

---

## 4. Mettre à jour les prix — un seul endroit par langue

Les montants n'apparaissent **que** dans la section `<section id="packs">` de chaque page, délimitée par les commentaires :

- FR : `index.html`, entre `<!-- PRIX — SEUL ENDROIT À MODIFIER (page FR) -->` et `<!-- FIN DU BLOC PRIX -->`
- EN : `en/index.html`, entre `<!-- PRICES — THE ONLY PLACE TO EDIT (EN page) -->` et `<!-- END OF PRICE BLOCK -->`

Le reste de la page (FAQ, bloc MSP, hero) renvoie aux cartes sans répéter de chiffre. Dans chaque carte :

| Élément | Balise |
|---|---|
| Prix principal | `<span class="amount">1 900 €</span>` |
| Montant TTC (FR) / mention de liste (EN) | `<p class="ttc">…</p>` — TTC = HT × 1,20 |
| Prix « puis » / remise / option | `<p class="then">…</p>` |
| Délai | `<span class="delay">…</span>` |
| Badge « prix de lancement » du pack A | `<span class="tag">…</span>` — à supprimer (et passer `amount` à 2 400 €) après le 3e client |

La page EN affiche la grille **Pays-Bas / Belgique / Luxembourg** (hors TVA, autoliquidation) ; la page FR la grille **France** (HT + TTC). Le pourcentage MSP (−25 %) est dans la section `#msp` de chaque page.

Vérification rapide après modification : `grep -n "€" index.html en/index.html`.

---

## 5. Points à compléter par le commanditaire (repérés en jaune sur la page, classe `todo`)

1. Capital social (mentions légales, FR et EN).
2. N° de TVA intracommunautaire.
3. Hébergeur du site : « Vercel Inc., 440 N Barranca Ave #4133, Covina, CA 91723, USA » si Vercel est retenu (à confirmer sur la page légale de Vercel au moment du déploiement), ou Cloudflare / GitHub selon le choix.
4. Assurance RC Pro : assureur, n° de police, territoires couverts (bloc confiance, FR et EN).
5. Lien PDF de l'exemple de dossier : dans la section `#exemple`, retirer les balises de commentaire autour du bouton « Télécharger l'exemple (PDF) » et déposer le fichier à la racine.
6. Image Open Graph : `og-image.png` et `og-image-en.png` sont générées ; à remplacer si une identité visuelle différente est retenue.

---

## 6. Contrôles effectués

- HTML + CSS validés avec le validateur Nu (vnu.jar) : 0 erreur, 0 avertissement sur les deux pages.
- Aucune ressource externe (`grep -E 'https?://'` ne renvoie que powix.io dans les métadonnées), aucun cookie, aucun formulaire.
- Menu mobile, accordéon FAQ (`<details>`, lisible sans JS) et ancres testés avec Playwright ; zéro erreur console.
- Captures : `screenshots/{fr,en}-{mobile-375x812,desktop-1280x900}.png` (premier écran) et `…-full.png` (page entière), deviceScaleFactor 2.
- Pour regénérer : `node tools/shots.js` (captures) et `node tools/og.js` (images Open Graph) — Playwright requis.
