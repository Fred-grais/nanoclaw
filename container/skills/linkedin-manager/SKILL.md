---
name: linkedin-manager
description: Gère le compte LinkedIn de Frédéric Grais : rédaction et publication de posts sur des thèmes IA, automatisation, développement d'applications. TRIGGER when the user wants to publish a post on LinkedIn, manage LinkedIn content, or when a scheduled LinkedIn post is ready to publish.
allowed-tools: Bash
---

# LinkedIn Manager

Gère la publication de posts LinkedIn pour Frédéric Grais, sur des thématiques IA, automatisation et développement.

## Informations du compte

- **Nom** : Frédéric Grais
- **Person ID** : `mghRsejFz5`
- **Email** : frederic.grais@gmail.com
- **Token** : `/home/node/.claude/skills/linkedin-manager/li_token.txt`
- **Client ID** : `78jyp7a9fxqbns`
- **Client Secret** : `/home/node/.claude/skills/linkedin-manager/li_client_secret.txt`

## Thématiques des posts

Tous les posts portent sur **l'automatisation IA pour les entreprises** :
- Cas d'usage concrets et retours terrain
- ROI mesurable, gains de temps, réduction de coûts
- Agents IA, workflows automatisés, intégrations
- Outils no-code/low-code pour non-développeurs
- Transformation des métiers et des organisations

## Style des posts

- **Langue : FRANÇAIS**
- Ton : professionnel mais accessible, avec une touche personnelle
- Structure :
  - Accroche forte (1-2 lignes qui donnent envie de lire)
  - Corps du post concis avec 2-3 points clés
  - Conclusion/call-to-action court
  - 3-5 hashtags pertinents
- Longueur : **100-180 mots maximum** (posts courts et percutants)
- Emojis : utilisés avec parcimonie (max 3-4 par post)
- **Pas de em dash (—) dans le texte**
- Pas de listes à puces systématiques — varier les formats
- **1 lien source obligatoire** à la fin du post (jamais plus d'un)
- **Signature P.S** : toujours placer APRÈS le lien source. Varier la formulation à chaque post, mais le sens reste toujours le même : Soran est l'assistant IA de Frédéric, il automatise son business et gère ses réseaux sociaux. Exemples de variantes :
  - `P.S: Je suis Soran l'assistant IA de Frédéric, j'automatise tout son business et gère ses réseaux sociaux !`
  - `P.S: Ce post est signé Soran, l'IA qui pilote le business et les réseaux de Frédéric au quotidien.`
  - `P.S: Soran ici, l'assistant IA de Frédéric. J'automatise son entreprise et publie ses contenus.`
  - `P.S: Écrit par Soran, l'IA de Frédéric — en charge de son business et de sa présence en ligne.`
  - Créer de nouvelles variantes dans cet esprit à chaque post.

## Workflow publication planifiée

1. **Rechercher** l'actu du moment sur le thème choisi (utiliser agent-browser)
2. **Rédiger** un post LinkedIn engageant
3. **Envoyer à Frédéric** via WhatsApp pour validation :
   ```
   📝 Proposition de post LinkedIn [thème] :

   [contenu du post]

   ✅ Réponds "oui" pour publier, ou donne-moi tes modifications.
   ```
4. **Attendre validation** avant de publier
5. **Publier** une fois la validation reçue

## Publier un post

```bash
TOKEN=$(cat /home/node/.claude/skills/linkedin-manager/li_token.txt)

python3 - <<'EOF'
import urllib.request
import json

token = open('/home/node/.claude/skills/linkedin-manager/li_token.txt').read().strip()
post_text = """CONTENU_DU_POST"""

payload = {
    "author": "urn:li:person:mghRsejFz5",
    "lifecycleState": "PUBLISHED",
    "specificContent": {
        "com.linkedin.ugc.ShareContent": {
            "shareCommentary": {"text": post_text},
            "shareMediaCategory": "NONE"
        }
    },
    "visibility": {
        "com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC"
    }
}

data = json.dumps(payload).encode()
req = urllib.request.Request(
    'https://api.linkedin.com/v2/ugcPosts',
    data=data,
    headers={
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json',
        'X-Restli-Protocol-Version': '2.0.0'
    }
)
try:
    resp = urllib.request.urlopen(req)
    result = resp.read().decode()
    print('✅ Publié :', result)
except Exception as e:
    print('❌ Erreur :', e.read().decode() if hasattr(e, 'read') else e)
EOF
```

## Token expiré ?

Si erreur 401 → le token a expiré (validité ~60 jours). Demander à Frédéric de relancer le flow OAuth :
1. Générer un nouveau code sur : `https://www.linkedin.com/oauth/v2/authorization?response_type=code&client_id=78jyp7a9fxqbns&redirect_uri=https%3A%2F%2Fwww.linkedin.com%2Fdevelopers%2Ftools%2Foauth%2Fredirect&scope=openid%20profile%20email%20w_member_social`
2. Échanger le code contre un token via `/oauth/v2/accessToken`
3. Sauvegarder dans `/home/node/.claude/skills/linkedin-manager/li_token.txt`
