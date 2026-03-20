---
name: pawtrait-creator
description: Crée un portrait noble d'un animal (pawtrait) via l'API Pawtraits of Nobility. TRIGGER when the user asks to create a "pawtrait", generate a noble portrait of an animal, or use the Pawtraits of Nobility generation service.
allowed-tools: Bash, agent-browser, WebSearch
---

# Pawtrait Creator

Génère des portraits nobles d'animaux via l'API Pawtraits of Nobility.

## Quand utiliser ce skill

Utilise ce skill quand l'utilisateur demande de :
- Créer un « pawtrait » d'un animal
- Générer un portrait noble d'un chien/chat/animal
- Utiliser le service Pawtraits of Nobility

## Styles disponibles

```
queen_duchess, young_prince, court_advisor, ambassador, royal_treasurer,
knight_order, castle_guard_captain, naval_commander, standard_bearer,
painters_muse, master_artist, astronomer, philosopher, alchemist,
italian_noble, spanish_grandee, french_courtier, venetian_merchant,
virtue_loyalty, the_hunter, the_guardian, saintly_companion,
royal_jester, banquet_steward, explorer, falconer
```

Si l'utilisateur ne précise pas de style, demande-lui d'en choisir un parmi la liste ci-dessus (ou propose-en quelques-uns adaptés).

## Source d'images : Google Sheets

Les images sources sont stockées dans la spreadsheet Google Sheets :
- **Spreadsheet ID** : `1rwxhsDcJb8jn2kR-3DsdEyR6jg7IogcVNwnSRS-V63g`
- **Nom** : `nanoclaw-pawtraits-source-imgs`
- **Format** : une URL par ligne, colonne A

### Étape 0 — Récupérer une image depuis la spreadsheet

Si l'utilisateur ne fournit pas d'image directement :

```bash
# Lire les URLs disponibles dans la spreadsheet
IMAGES=$(gws sheets +read --spreadsheet "1rwxhsDcJb8jn2kR-3DsdEyR6jg7IogcVNwnSRS-V63g" --range "A1:A100" 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
values = data.get('values', [])
urls = [row[0] for row in values if row and row[0].strip()]
print('\n'.join(urls))
")

if [ -z "$IMAGES" ]; then
  # Aucune image disponible → prévenir Frédéric
  echo "EMPTY"
else
  # Prendre la première URL disponible
  IMAGE_URL=$(echo "$IMAGES" | head -1)
  echo "Image URL: $IMAGE_URL"
fi
```

**Si la spreadsheet est vide** → envoyer ce message à Frédéric via send_message :
> "🐾 La spreadsheet d'images source est vide ! Ajoute des URLs d'images dans `nanoclaw-pawtraits-source-imgs` sur Google Sheets et je génèrerai les pawtraits automatiquement."

**Si une image est disponible** → l'envoyer à Frédéric pour validation :
> 🐾 Image trouvée dans la spreadsheet : [URL]
> Est-ce que je peux l'utiliser pour créer le pawtrait ? (oui / non)

Attendre la confirmation avant de continuer. Une fois validée, **supprimer l'URL de la spreadsheet** pour éviter de la réutiliser :

```bash
# Supprimer la première ligne (décaler les autres vers le haut)
# Lire toutes les URLs restantes
REMAINING=$(gws sheets +read --spreadsheet "1rwxhsDcJb8jn2kR-3DsdEyR6jg7IogcVNwnSRS-V63g" --range "A1:A100" 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
values = data.get('values', [])
urls = [row[0] for row in values[1:] if row and row[0].strip()]
print(json.dumps([[u] for u in urls]))
")

# Effacer la colonne A
gws sheets spreadsheets values clear --spreadsheet "1rwxhsDcJb8jn2kR-3DsdEyR6jg7IogcVNwnSRS-V63g" --range "A1:A100" 2>/dev/null

# Réécrire les URLs restantes (sans la première)
if [ "$REMAINING" != "[]" ]; then
  gws sheets spreadsheets values update \
    --spreadsheet "1rwxhsDcJb8jn2kR-3DsdEyR6jg7IogcVNwnSRS-V63g" \
    --range "A1" \
    --json "{\"values\": ${REMAINING}, \"majorDimension\": \"ROWS\"}" 2>/dev/null
fi
echo "✅ URL consommée de la spreadsheet"
```

> ⚠️ Ne jamais passer à l'étape suivante sans confirmation explicite de l'utilisateur.

### Étape 1 — Préparer l'image en Base64

L'utilisateur fournit une URL d'image ou un fichier local (ou l'URL a été validée à l'étape 0).

```bash
# Depuis une URL
curl -sL "IMAGE_URL" -o /tmp/pawtrait_input.jpg
IMAGE_B64=$(base64 -w 0 /tmp/pawtrait_input.jpg)

# Depuis un fichier local
IMAGE_B64=$(base64 -w 0 /path/to/image.jpg)
```

### Étape 2 — Appeler l'endpoint de génération

```bash
STYLE="queen_duchess"        # style choisi
DOG_NAME="noble"             # nom quelconque

PAYLOAD=$(python3 -c "
import json, sys
data = {
  '0': {
    'style': '${STYLE}',
    'dogName': '${DOG_NAME}',
    'imageData': sys.stdin.read().strip()
  }
}
print(json.dumps(data))
" <<< "$IMAGE_B64")

RESPONSE=$(curl -s -X POST \
  "https://pawtraits-of-nobility.vercel.app/api/trpc/portraits.generate?batch=1" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")

echo "$RESPONSE"

# Extraire l'ID
PORTRAIT_ID=$(echo "$RESPONSE" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data[0]['result']['data']['id'])
")

echo "Portrait ID: $PORTRAIT_ID"
```

### Étape 3 — Attendre ~1 minute puis vérifier le statut

Attendre 60 secondes, puis appeler le GET endpoint en boucle jusqu'à `status == "completed"` (max 5 tentatives, toutes les 30s).

```bash
sleep 60

for i in 1 2 3 4 5; do
  INPUT=$(python3 -c "import json; print(json.dumps({'0': {'portraitId': '${PORTRAIT_ID}'}}))")
  STATUS_RESP=$(curl -s -G \
    "https://pawtraits-of-nobility.vercel.app/api/trpc/portraits.getStatus?batch=1" \
    --data-urlencode "input=${INPUT}")

  STATUS=$(echo "$STATUS_RESP" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data[0]['result']['data']['status'])
")

  echo "Tentative $i — Status: $STATUS"

  if [ "$STATUS" = "completed" ]; then
    ORIGINAL_URL=$(echo "$STATUS_RESP" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data[0]['result']['data']['originalImageUrl'])
")
    GENERATED_URL=$(echo "$STATUS_RESP" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data[0]['result']['data']['generatedImageUrl'])
")
    echo "✅ Pawtrait généré !"
    echo "Original : $ORIGINAL_URL"
    echo "Portrait : $GENERATED_URL"
    break
  fi

  sleep 30
done
```

### Étape 4 — Stocker les résultats

Une fois la génération terminée, stocker dans `/tmp/pawtraits_db.json` :

```bash
python3 - <<'EOF'
import json, os
from datetime import datetime

db_path = "/tmp/pawtraits_db.json"
db = json.load(open(db_path)) if os.path.exists(db_path) else []

db.append({
    "id": "PORTRAIT_ID",
    "style": "STYLE",
    "originalImageUrl": "ORIGINAL_URL",
    "generatedImageUrl": "GENERATED_URL",
    "createdAt": datetime.utcnow().isoformat()
})

json.dump(db, open(db_path, "w"), indent=2)
print(f"Sauvegardé. Total portraits : {len(db)}")
EOF
```

### Étape 5 — Présenter le résultat à l'utilisateur

Toujours afficher les deux URLs dans la réponse finale :

```
🎨👑 Ton pawtrait noble est prêt !

📷 Image originale :
https://...originalImageUrl...

🖼️ Portrait généré :
https://...generatedImageUrl...
```

- **Ne jamais omettre** l'un ou l'autre des deux attributs (`originalImageUrl` et `generatedImageUrl`)
- Demander à l'utilisateur de valider la qualité avant de passer à l'étape 6
- Message : *"La qualité te convient ? Je peux générer l'image composite (animal + portrait côte à côte) 🖼️🐾"*

> ⚠️ Attendre la confirmation explicite avant de lancer l'étape 6.

### Étape 6 — Générer l'image composite via Replicate (openai/gpt-image-1.5)

Une fois la qualité du pawtrait approuvée, appeler l'API Replicate pour créer une image de l'animal posant à côté de son portrait.

**Token Replicate** : stocké dans `/home/node/.claude/skills/pawtrait-creator/replicate_token.txt`

```bash
# Télécharger les deux images
curl -sL "$ORIGINAL_URL" -o /tmp/composite_original.jpg
curl -sL "$GENERATED_URL" -o /tmp/composite_portrait.jpg

# Encoder en base64 dans des fichiers temporaires
base64 -w 0 /tmp/composite_original.jpg > /tmp/b64_original.txt
base64 -w 0 /tmp/composite_portrait.jpg > /tmp/b64_portrait.txt

# Appel Replicate via python3
python3 - <<'PYEOF'
import json, urllib.request, urllib.error, time

token = open('/home/node/.claude/skills/pawtrait-creator/replicate_token.txt').read().strip()
b64_orig = open('/tmp/b64_original.txt').read().strip()
b64_port = open('/tmp/b64_portrait.txt').read().strip()

payload = {
    "input": {
        "prompt": "I gave you 2 pictures, the first one is the picture of an animal and the second one is a Renaissance Portrait of it. I want you to create an image with that animal posing close to its portrait as if its owner put it there to compare the animal with its portrait. Make sure we can see the whole portrait in the generated image.",
        "input_images": [
            "data:image/jpeg;base64," + b64_orig,
            "data:image/jpeg;base64," + b64_port
        ],
        "aspect_ratio": "1:1",
        "background": "auto",
        "input_fidelity": "low",
        "moderation": "auto",
        "number_of_images": 1,
        "output_compression": 90,
        "output_format": "webp",
        "quality": "low"
    }
}

data = json.dumps(payload).encode()
req = urllib.request.Request(
    'https://api.replicate.com/v1/models/openai/gpt-image-1.5/predictions',
    data=data,
    headers={
        'Authorization': f'Token {token}',
        'Content-Type': 'application/json',
        'Prefer': 'wait'
    }
)
try:
    resp = urllib.request.urlopen(req)
    result = json.loads(resp.read().decode())
except urllib.error.HTTPError as e:
    print(f"Error {e.code}: {e.read().decode()}")
    exit(1)

prediction_id = result.get('id')
status = result.get('status')
print(f"ID: {prediction_id}, Status: {status}")

for i in range(12):
    if status == 'succeeded':
        print(f"✅ Output: {result.get('output')}")
        break
    elif status == 'failed':
        print(f"❌ {result.get('error')}")
        break
    else:
        print(f"Tentative {i+1} — {status}, attente 15s...")
        time.sleep(15)
        poll = urllib.request.Request(
            f'https://api.replicate.com/v1/predictions/{prediction_id}',
            headers={'Authorization': f'Token {token}'}
        )
        result = json.loads(urllib.request.urlopen(poll).read().decode())
        status = result.get('status')
        if status == 'succeeded':
            print(f"✅ Output: {result.get('output')}")
            break
PYEOF
```

### Étape 7 — Présenter l'image composite

```
🖼️🐾 Image composite prête !

https://...url_composite...

Veux-tu que je la poste sur @pawtraitsofnobility ?
```

- Proposer de publier l'**image composite** (pas le pawtrait seul) sur Instagram via le skill **instagram-manager**

## Gestion des erreurs

- Si la génération échoue après 5 tentatives → informer l'utilisateur et conserver l'ID pour réessayer
- Si `imageData` est trop volumineux → compresser l'image avant encodage :
  ```bash
  convert /tmp/pawtrait_input.jpg -resize 1024x1024\> -quality 85 /tmp/pawtrait_input_small.jpg
  ```
