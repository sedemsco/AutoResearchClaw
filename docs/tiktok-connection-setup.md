# Connexion TikTok for Business — Guide de rattachement du jeton d'accès

**Dernière vérification :** 2026-09-20
**Contexte :** Le connecteur MCP `TikTok_for_Business` est configuré et le jeton répond
`code:0 OK`, mais aucun asset (advertiser account ni Business Center) n'y est rattaché.
Ce document explique comment régénérer un jeton fonctionnel.

## Diagnostic actuel

| Appel MCP | Résultat |
|---|---|
| `auth_advertiser_get` | `{"list": []}` — aucun advertiser autorisé |
| `bc_get` | `{"list": [], "total_number": 0}` — aucun Business Center |
| Connexion API | Valide (`code:0`) |

**Conclusion :** le jeton est *nu* — flux OAuth à refaire en cochant explicitement
au moins un ad account.

---

## Prérequis

- Compte **TikTok Ads Manager** (ads.tiktok.com) avec au moins un compte publicitaire actif.
- Compte **TikTok for Business Developer** (business-api.tiktok.com).
- Optionnel mais recommandé : un **Business Center** (business.tiktok.com) pour gérer
  plusieurs ad accounts et fournisseurs sous une même entité.

## Étape 1 — Créer une App développeur TikTok

1. Aller sur https://business-api.tiktok.com/portal.
2. `My Apps` → `Create an App`.
3. Renseigner :
   - **App name**, **App description**.
   - **Redirect URL** — l'URL où TikTok renverra le `auth_code`.
   - **Scopes / Permissions** — cocher au minimum : `Ad Account Management`,
     `Campaign Management`, `Ads Management`, `Reporting`.
     Ajouter `Catalog`, `Pixel`, `Custom Audiences`, etc. selon les besoins.
4. Noter l'`App ID` et le `Secret`.

## Étape 2 — Flux OAuth utilisateur

1. Construire l'URL d'autorisation :

   ```
   https://business-api.tiktok.com/portal/auth?app_id=APP_ID&state=STATE&redirect_uri=REDIRECT_URI
   ```

2. Ouvrir dans un navigateur → se connecter avec le compte TikTok Ads.
3. **Étape critique :** cocher chaque compte publicitaire à autoriser. Un jeton sans
   ad account coché reste vide (c'est le cas actuel).
4. TikTok redirige vers `REDIRECT_URI?auth_code=XXXX&state=STATE`.
5. Échanger le `auth_code` contre un `access_token` :

   ```bash
   curl -X POST https://business-api.tiktok.com/open_api/v1.3/oauth2/access_token/ \
     -H "Content-Type: application/json" \
     -d '{
       "app_id": "APP_ID",
       "secret": "SECRET",
       "auth_code": "AUTH_CODE"
     }'
   ```

   Réponse :

   ```json
   {
     "code": 0,
     "message": "OK",
     "data": {
       "access_token": "…",
       "advertiser_ids": ["…"],
       "scope": [1, 2, 3, …]
     }
   }
   ```

## Étape 3 — Variante Business Center (multi-comptes)

1. Dans **Business Center** : `Assets → Ad Accounts → Add` pour rattacher chaque compte.
2. Attribuer la propriété (`Own`) au BC.
3. `Settings → Developer Apps → Add App` pour autoriser votre App développeur.
4. Refaire le flux OAuth de l'étape 2 : tous les ad accounts du BC deviennent
   disponibles au moment du consentement.

## Étape 4 — Injecter le nouveau jeton dans le MCP

Deux voies possibles selon la façon dont le connecteur est provisionné :

- **Via l'UI Claude** : paramètres → *Connectors* → *TikTok for Business* → *Reconnect*
  ou *Sign in again*.
- **Via variable d'environnement** : mettre à jour la clé attendue par le serveur MCP
  (typiquement `TIKTOK_ACCESS_TOKEN` ou une entrée dans `.mcp.json` /
  `claude_desktop_config.json`), puis redémarrer la session Claude.

## Étape 5 — Vérification

Depuis Claude :

```
auth_advertiser_get      → doit retourner ≥1 advertiser_id
bc_get                   → doit retourner ≥1 BC si le flux BC a été utilisé
advertiser_info_get({advertiser_ids: […]})
```

Une fois `advertiser_ids` peuplé, tous les outils MCP TikTok deviennent opérationnels
(rapports, création Smart+, catalogue, pixels, audiences, etc.).

## Documentation officielle

- Portail dev : https://business-api.tiktok.com/portal
- Docs API : https://business-api.tiktok.com/portal/docs?id=1738373141733378
- OAuth : https://business-api.tiktok.com/portal/docs?id=1739965703387649
