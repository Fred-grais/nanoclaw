---
name: instagram-manager
description: Gère le compte Instagram Business @pawtraitsofnobility : publication de posts et récupération des posts existants. TRIGGER when the user wants to publish a photo on Instagram, see published posts on @pawtraitsofnobility, or manage the Pawtraits of Nobility Instagram account.
allowed-tools: Bash
---

# Instagram Manager

Gère le compte Instagram Business @pawtraitsofnobility : publication de posts et récupération des posts existants.

## Quand utiliser ce skill

Utilise ce skill quand l'utilisateur demande :
- Publier une photo sur Instagram
- Voir les posts publiés sur @pawtraitsofnobility
- Gérer le compte Instagram de Pawtraits of Nobility

## Informations du compte

- **Instagram Business Account ID** : `17841439615510629`
- **Username** : `@pawtraitsofnobility`
- **Facebook Page ID** : `1085865291266057`
- **Facebook Business ID** : `1406726254560249`
- **CTA URL** : `https://pawtraits-of-nobility.vercel.app/`

## 🔐 Authentification — System User Token

L'authentification utilise un **System User Token** (user système "nanoclaw" dans Meta Business Suite), qui n'expire pas avec les sessions utilisateur.

- **System User Token** : `/home/node/.claude/skills/instagram-manager/fb_token.txt`
- **Page Access Token** (dérivé du System User Token) : `/home/node/.claude/skills/instagram-manager/page_token.txt`

### Régénérer le Page Token depuis le System User Token

```bash
FB_TOKEN=$(cat /home/node/.claude/skills/instagram-manager/fb_token.txt)
PAGE_ID="1085865291266057"

RESP=$(curl -s "https://graph.facebook.com/v25.0/${PAGE_ID}?fields=access_token&access_token=${FB_TOKEN}")
NEW_PAGE_TOKEN=$(echo "$RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo -n "$NEW_PAGE_TOKEN" > /home/node/.claude/skills/instagram-manager/page_token.txt
echo "✅ Page Token renouvelé"
```

> ⚠️ Si le System User Token est lui-même expiré (erreur 463), demander à Frédéric un nouveau token depuis Meta Business Suite → Paramètres → Utilisateurs système → nanoclaw → Générer un token.

## 🛍️ Product Tagging (OBLIGATOIRE sur chaque post)

Chaque post publié **doit** inclure le tag du produit à 49€ :

- **Produit** : "Pawtrait of Nobility for your pet (20 × 25 cm)"
- **Product ID** : `26025482050427541`
- **Catalogue ID** : `4066442766941751`
- **Position du tag** : `x: 0.5, y: 0.8` (centre-bas de l'image)

La config complète est dans `/home/node/.claude/skills/instagram-manager/shopping_config.json`

## Workflow standard

### Publier un nouveau post (avec product tag)

1. Récupérer l'image (URL partagée par l'utilisateur)
2. Télécharger l'image et l'uploader sur **litterbox.catbox.moe** (obligatoire — uguu.se est bloqué par les serveurs Facebook)
3. Rédiger une caption fun avec CTA vers https://pawtraits-of-nobility.vercel.app/
4. Créer le media container via `graph.facebook.com` avec product_tags
5. Attendre ~12 secondes puis publier

```bash
SKILL_DIR="/home/node/.claude/skills/instagram-manager"
PAGE_TOKEN=$(cat "$SKILL_DIR/page_token.txt")
IGA_ID="17841439615510629"
PRODUCT_ID="26025482050427541"

# Télécharger l'image
curl -sL "SOURCE_URL" -o /tmp/ig_upload_image.jpg

# Uploader sur litterbox (requis pour graph.facebook.com)
IMAGE_URL=$(curl -s -F "reqtype=fileupload" -F "time=72h" -F "fileToUpload=@/tmp/ig_upload_image.jpg" "https://litterbox.catbox.moe/resources/internals/api.php")
echo "Image URL: $IMAGE_URL"

# Créer container avec product tag
CONTAINER=$(curl -s -X POST "https://graph.facebook.com/v25.0/${IGA_ID}/media" \
  --data-urlencode "image_url=${IMAGE_URL}" \
  --data-urlencode "caption=${CAPTION}" \
  --data-urlencode "product_tags=[{\"product_id\":\"${PRODUCT_ID}\",\"x\":0.5,\"y\":0.8}]" \
  --data-urlencode "access_token=${PAGE_TOKEN}")
CREATION_ID=$(echo "$CONTAINER" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Attendre que le media soit prêt
sleep 12

# Publier
curl -s -X POST "https://graph.facebook.com/v25.0/${IGA_ID}/media_publish" \
  --data-urlencode "creation_id=${CREATION_ID}" \
  --data-urlencode "access_token=${PAGE_TOKEN}"
```

### Publier une Story

```bash
SKILL_DIR="/home/node/.claude/skills/instagram-manager"
PAGE_TOKEN=$(cat "$SKILL_DIR/page_token.txt")
IGA_ID="17841439615510629"

# Uploader sur litterbox
IMAGE_URL=$(curl -s -F "reqtype=fileupload" -F "time=72h" -F "fileToUpload=@/tmp/ig_upload_image.jpg" "https://litterbox.catbox.moe/resources/internals/api.php")

CONTAINER=$(curl -s -X POST "https://graph.facebook.com/v25.0/${IGA_ID}/media" \
  --data-urlencode "image_url=${IMAGE_URL}" \
  --data-urlencode "media_type=STORIES" \
  --data-urlencode "access_token=${PAGE_TOKEN}")
STORY_ID=$(echo "$CONTAINER" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
sleep 12
curl -s -X POST "https://graph.facebook.com/v25.0/${IGA_ID}/media_publish" \
  --data-urlencode "creation_id=${STORY_ID}" \
  --data-urlencode "access_token=${PAGE_TOKEN}"
```

## ⚠️ Règles importantes

- Utiliser **exclusivement** `graph.facebook.com` + Page Token pour tous les appels API
- **Jamais** `graph.instagram.com` (ancien endpoint, ne supporte pas le product tagging)
- Image **obligatoirement** hébergée sur litterbox.catbox.moe avant d'envoyer à l'API
- Product tag **obligatoire** sur chaque post (jamais de post sans tag produit)

## Style des captions

- **Langue : FRANÇAIS obligatoire** — toutes les captions sont rédigées en français
- Ton : **amusant, décalé, avec de la personnalité**
- Structure : Citation imaginaire du chien (en français) + description fun + CTA
- Emojis : généreusement utilisés 🐾🎨👑
- CTA fixe : `👉 Crée le portrait noble de ton animal ici : https://pawtraits-of-nobility.vercel.app/`
- Hashtags : 8-12 hashtags pertinents (race du chien + art + portrait) — les hashtags peuvent rester en anglais pour la portée

## Token expiré ? (erreur OAuthException code 190/463)

1. Essayer d'abord de régénérer le Page Token depuis le System User Token (voir section Authentification ci-dessus)
2. Si le System User Token lui-même est expiré → demander à Frédéric un nouveau token depuis :
   **Meta Business Suite** → Paramètres → Utilisateurs système → nanoclaw → Générer un token
3. Sauvegarder le nouveau System User Token :
   ```bash
   echo -n "NOUVEAU_TOKEN" > /home/node/.claude/skills/instagram-manager/fb_token.txt
   ```
4. Puis régénérer le Page Token (voir étape ci-dessus)
