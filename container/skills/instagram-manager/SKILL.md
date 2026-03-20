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

- **IG User ID** (ancien, graph.instagram.com) : `26252993180996956`
- **Instagram Business Account ID** (nouveau, graph.facebook.com) : `17841439615510629`
- **Username** : `@pawtraitsofnobility`
- **Facebook Page ID** : `1085865291266057`
- **Facebook Business ID** : `1406726254560249`
- **Page Access Token** : `/home/node/.claude/skills/instagram-manager/page_token.txt`
- **Instagram Token** (legacy, stories) : `/home/node/.claude/skills/instagram-manager/ig_token.txt`
- **CTA URL** : `https://pawtraits-of-nobility.vercel.app/`

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
2. Télécharger l'image et l'uploader sur **litterbox.catbox.moe** (obligatoire pour graph.facebook.com — uguu.se est bloqué par les serveurs Facebook)
3. Rédiger une caption fun avec CTA vers https://pawtraits-of-nobility.vercel.app/
4. Créer le media container **via graph.facebook.com** avec product_tags
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
CONTAINER=$(curl -s -X POST "https://graph.facebook.com/v20.0/${IGA_ID}/media" \
  --data-urlencode "image_url=${IMAGE_URL}" \
  --data-urlencode "caption=${CAPTION}" \
  --data-urlencode "product_tags=[{\"product_id\":\"${PRODUCT_ID}\",\"x\":0.5,\"y\":0.8}]" \
  --data-urlencode "access_token=${PAGE_TOKEN}")
CREATION_ID=$(echo "$CONTAINER" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Attendre que le media soit prêt
sleep 12

# Publier
curl -s -X POST "https://graph.facebook.com/v20.0/${IGA_ID}/media_publish" \
  --data-urlencode "creation_id=${CREATION_ID}" \
  --data-urlencode "access_token=${PAGE_TOKEN}"
```

### Publier une Story

Le Page Token fonctionne aussi pour les Stories via graph.facebook.com :

```bash
SKILL_DIR="/home/node/.claude/skills/instagram-manager"
PAGE_TOKEN=$(cat "$SKILL_DIR/page_token.txt")
IGA_ID="17841439615510629"

# Uploader sur litterbox
IMAGE_URL=$(curl -s -F "reqtype=fileupload" -F "time=72h" -F "fileToUpload=@/tmp/ig_upload_image.jpg" "https://litterbox.catbox.moe/resources/internals/api.php")

CONTAINER=$(curl -s -X POST "https://graph.facebook.com/v20.0/${IGA_ID}/media" \
  --data-urlencode "image_url=${IMAGE_URL}" \
  --data-urlencode "media_type=STORIES" \
  --data-urlencode "access_token=${PAGE_TOKEN}")
STORY_ID=$(echo "$CONTAINER" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
sleep 12
curl -s -X POST "https://graph.facebook.com/v20.0/${IGA_ID}/media_publish" \
  --data-urlencode "creation_id=${STORY_ID}" \
  --data-urlencode "access_token=${PAGE_TOKEN}"
```

## ⚠️ Important : utiliser exclusivement graph.facebook.com + Page Token

| Usage | Endpoint | Token |
|-------|----------|-------|
| Posts avec product tag | `graph.facebook.com` + IGA_ID `17841439615510629` | `page_token.txt` |
| Stories | `graph.facebook.com` + IGA_ID `17841439615510629` | `page_token.txt` |

L'ancien token Instagram (`ig_token.txt`) et `graph.instagram.com` ne sont plus nécessaires.

## Style des captions

- **Langue : FRANÇAIS obligatoire** — toutes les captions sont rédigées en français
- Ton : **amusant, décalé, avec de la personnalité**
- Structure : Citation imaginaire du chien (en français) + description fun + CTA
- Emojis : généreusement utilisés 🐾🎨👑
- CTA fixe : `👉 Crée le portrait noble de ton animal ici : https://pawtraits-of-nobility.vercel.app/`
- Hashtags : 8-12 hashtags pertinents (race du chien + art + portrait) — les hashtags peuvent rester en anglais pour la portée

## Token expiré ?

**Page Token (graph.facebook.com)** — Si erreur `OAuthException code 190` :
1. Aller sur https://developers.facebook.com/tools/explorer
2. Sélectionner l'app, cocher toutes les permissions shopping + pages
3. Générer le token utilisateur
4. Récupérer le Page Access Token : `GET /1085865291266057?fields=access_token`
5. Sauvegarder : `echo "TOKEN" > /home/node/.claude/skills/instagram-manager/page_token.txt`

**Instagram Token (graph.instagram.com)** — Si erreur `OAuthException code 190` :
Demander à Frédéric de régénérer un token sur https://developers.facebook.com/tools/explorer
```bash
echo "NOUVEAU_TOKEN" > /home/node/.claude/skills/instagram-manager/ig_token.txt
```
