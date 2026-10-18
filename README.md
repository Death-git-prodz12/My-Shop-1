# MyShop — catalogue modulaire et moteur de marge

Site statique multipage HTML/CSS/ES modules avec Firebase Authentication et adaptateur API optionnel. L’archive inclut un catalogue de démonstration; aucun robot d’extraction commercial ne tourne dans le navigateur.

## Démarrer

Servir le dossier `myshop/` avec un serveur statique (par exemple `npx serve .`) puis ouvrir `index.html`. Les modules ES et `fetch(data/products.json)` nécessitent un serveur HTTP; l’ouverture en `file://` ne fonctionne pas. Déploiement Netlify: publier le dossier `myshop/` comme répertoire de site. GitHub Pages fonctionne aussi en conservant l’arborescence.

## Données produit

Chaque produit utilise le schéma canonique suivant (les champs marqués * sont calculés si absents) :

```json
{
  "id": "example-item",
  "title": "Nom du produit",
  "price": 49.9,
  "costPrice": 28.0,
  "profitMargin": 43.9,
  "platform": "Shopify",
  "sourceUrl": "https://shop.example.com/products/example-item",
  "isPremium": true,
  "category": "Home & Kitchen",
  "rating": 4.6,
  "images": ["https://cdn.example.com/item.jpg"],
  "currency": "USD"
}
```

`normalizeProduct()` convertit n’importe quelle réponse vers ce modèle et ajoute : `profitAmount*`, `marginTier*` (`high` ≥ 40 %, `mid`, `low`, `loss`, `unknown`), `marginSource*` (`computed` ou `declared`), `premiumReasons*`, `premiumSource*` (`declared` ou `auto`), `rawCategory*` (valeur d’origine) et `categoryVia*` (`alias`, `keyword` ou `fallback`). Les alias historiques (`name`, `source`, `url`, `priceRef`, `image`) restent acceptés, et la fonction est idempotente.

### Catégories et repli « Undefined / Divers » (`js/categories.js`)

Un produit n’est **jamais écarté** pour une catégorie inconnue. `resolveCategory()` essaie dans l’ordre : alias exact, mots-clés FR/EN pondérés (catégorie, tags / fil d’Ariane, titre), puis repli sur **Undefined / Divers** (`divers`). La catégorie d’origine reste dans `rawCategory`, et `getIngestionReport()` liste les produits tombés dans le repli pour que vous puissiez enrichir les mots-clés. Pour ajouter une catégorie : l’ajouter à `CAT` et `KEYWORDS`.

### Moteur de marge (`js/pricing.js`)

`computeProfitMargin(price, costPrice)` = `(prix − coût) / prix × 100`, arrondie à 0,1. Elle lit « 29,99 € », « 1 299,00 » ou « 12.34 -- 15.00 », conserve une marge négative (vente à perte) et renvoie `null` si le prix ou le coût est inconnu : **aucune marge n’est inventée**. Dès que prix et coût sont connus, la marge est recalculée et remplace une valeur déclarée incohérente (conservée dans `declaredMargin`). `profitAmount` est le bénéfice brut unitaire avant livraison, taxes, retours et frais de plateforme.

La marge apparaît sur chaque carte (pastille colorée sur la vignette + bénéfice brut) et dans le panneau « Marge bénéficiaire » de `produit.html` (prix, coût, bénéfice, jauge, formule).

### Segment Premium / High-Yield

`isPremium` déclaré par la source (`true` ou `false`) est toujours respecté. S’il est absent, `classifyPremium()` qualifie le produit dès qu’UN critère est rempli (seuils dans `PREMIUM_RULES`) : marge ≥ 40 %, prix ≥ 100, ou note ≥ 4,6 avec ≥ 5 000 avis. Les raisons s’affichent sur la fiche produit. Le segment est mis en avant par un badge doré lumineux sur les cartes, une section « Premium · High-Yield » (indicateurs : nombre, marge moyenne, meilleure marge, plateformes) sur l’accueil et dans `catalogue.html`, le filtre « Premium · High-Yield » (`catalogue.html?premium=1`) et le filtre « Marge minimale ». Ces règles trient et mettent en valeur ; elles ne sont pas un paywall (voir « Firebase et état local » plus bas).

Le JSON inclus reprend les prix de référence de l’ancien catalogue. Les coûts marqués `costMode: "simulation"` et les produits `dataMode: "demo"` sont des **valeurs de simulation**: ne les utilisez pas comme cotations réelles. Les produits sans prix ou coût connu affichent « Prix sur la plateforme » et « Marge à confirmer ».

## Ingestion et API

`js/api.js` expose `INGESTION_SOURCES`, `ingestProducts(records, source, { origin })`, `ingestWithReport()`, `normalizeProduct()`, `getIngestionReport()`, `getProducts()`, `getHistory(id)`, `getCompetitors(id)` et `watchLive(callback)`. La logique pure (modèle, adaptateurs, marge, catégories) est dans `js/ingest.js`, `js/pricing.js` et `js/categories.js`, sans dépendance au navigateur.

Adaptateurs par plateforme (`js/ingest.js`) : Shopify (`products.json`, avec `origin` pour reconstruire l’URL), AliExpress (API affiliés), Amazon (PA-API 5), TikTok Shop (Partner API) et CJ Dropshipping (où `sellPrice` est le **coût**, jamais le prix de vente) ; Temu, Shein et Alibaba passent par un adaptateur générique d’alias courants. Les noms de champs suivent les documentations publiques de ces API : à valider avec vos vrais flux. Un champ déjà au format canonique n’est jamais écrasé. Les connecteurs ingèrent des flux/API autorisés des plateformes (Amazon, AliExpress, Temu, Shopify, TikTok Shop, Shein, Alibaba, CJ Dropshipping). L’extraction effective doit être exécutée côté serveur selon les règles des sources; la page ne contourne ni CAPTCHA ni restrictions d’accès.

Configurer `API_BASE` dans `js/config.js` pour activer:

- `GET {API_BASE}/products` → `[{...product}]`, `{ "products": [...] }` ou `{ "items": [...] }`
- `GET {API_BASE}/products/:id/history` → `[{ "t": "2026-10-18T10:00:00Z", "price": 29.99 }]`
- `GET {API_BASE}/products/:id/competitors` → produits comparables au même schéma
- `STREAM_URL` facultatif → flux Server-Sent Events; sans flux, l’app interroge l’API à intervalle fixe.

Une session Firebase active joint son jeton ID en Bearer aux appels API. Le serveur doit le vérifier et appliquer les règles d’accès. Ne placez pas de secret API dans le JavaScript du navigateur. Si l’API est indisponible, le site revient au catalogue local.

## Firebase et état local

La connexion Google utilise Firebase Auth. La liste de surveillance est isolée par UID dans `localStorage` et synchronisée entre onglets du même navigateur. Elle n’est pas partagée entre appareils. Pour une synchronisation multiappareil, remplacer `js/store.js` par une collection Firestore protégée par des règles basées sur l’UID.

Le garde d’accès exécuté dans le navigateur ne protège pas les données côté serveur. Vérifier les jetons Firebase côté backend et dans les règles Firestore.

## Publication

Pour GitHub Pages, autoriser le domaine `*.github.io` dans Firebase Authentication → Authorized domains. Les logos transparents recadrés utilisés par l’en-tête et la page de connexion sont `assets/MyShop-transparent.png` et `assets/Google-transparent.png`.

## Tests

`node --test` (Node 18+, depuis la racine) exécute `tests/ingest.test.mjs` : marge, lecture des montants, repli « Undefined / Divers », règles Premium, idempotence, adaptateurs et conservation de tous les produits du catalogue local.
