# CLAUDE.md — Générateur de Dialogues Pédagogiques
## `dialogue-generator` · GitHub Pages · HTML/JS statique

---

## 1. VUE D'ENSEMBLE

Application web monopage (un seul fichier `index.html`) permettant à des enseignants de langues de générer des dialogues pédagogiques multilingues audio via l'API Gemini de Google.

**Déploiement** : GitHub Pages (statique, zéro serveur)  
**Modèles utilisés** :
- Génération de texte : `gemini-2.5-flash-preview-05-20`
- Génération audio : `gemini-2.5-flash-preview-tts`

**Utilisateurs cibles** : Enseignants non-techniciens (niveau numérique faible)  
**Principe de sécurité** : La clé API reste dans le `localStorage` du navigateur de l'enseignant. Elle n'est jamais envoyée à un serveur tiers.

---

## 2. FONCTIONNALITÉS REQUISES

### 2.1 Gestion de la clé API
- Champ de saisie masqué pour la clé API Gemini personnelle
- Stockage dans `localStorage` (persistant entre sessions)
- Bouton "Oubli de la clé" pour effacer
- Lien vers le tutoriel de création de clé (voir section 8)
- Validation silencieuse au premier appel (pas de bouton "tester")

### 2.2 Paramètres du dialogue
| Paramètre | Type | Valeurs |
|-----------|------|---------|
| Langue cible | `<select>` | Français, Anglais, Espagnol, Arabe, Néerlandais, Allemand, Italien, Portugais, Polonais, Russe, Chinois, Japonais |
| Niveau CECR | `<select>` | A1, A2, B1, B2, C1, C2 |
| Nombre de locuteurs | `<select>` | 1, 2, 3, 4 |
| Noms des locuteurs | `<input>` × N | Prénoms libres (ex. Yasmine, Tom...) |
| Genre vocal par locuteur | `<select>` | Masculin / Féminin |
| Consigne libre | `<textarea>` | Thème, contexte, contraintes linguistiques |
| Nombre de répliques | `<select>` | 4, 6, 8, 10, 12, 16 |

### 2.3 Génération
1. **Phase 1 – Texte** : Appel à `gemini-2.5-flash-preview-05-20` → dialogue structuré en JSON
2. **Phase 2 – Audio** : Appel à `gemini-2.5-flash-preview-tts` → audio WAV multi-locuteurs

Les deux phases sont déclenchées par un seul bouton "GÉNÉRER".  
Un indicateur de progression distinct pour chaque phase (spinner + message).

### 2.4 Affichage du dialogue
- Chaque réplique affichée dans une bulle colorée par locuteur
- Prénom du locuteur en en-tête de chaque bulle
- Couleurs différentes pour chaque locuteur (4 couleurs max issues de la palette retro)

### 2.5 Audio
- Lecteur audio HTML5 standard intégré (pas de lecteur custom)
- Le fichier WAV généré est lisible directement dans la page
- Bouton de téléchargement du fichier WAV

### 2.6 Export
- **Copier** : Copie le texte brut dans le presse-papiers
- **Imprimer / PDF** : `window.print()` avec CSS d'impression propre (bulles, noms, sans UI)
- Pas d'export Word (trop complexe, non prioritaire)

### 2.7 Compteur de quota (symbolique)
Voir section 5 — spécification détaillée.

---

## 3. ARCHITECTURE TECHNIQUE

```
index.html
├── <style>         ← CSS inline complet (retrofuturisme + layout)
├── <body>          ← UI complète
│   ├── #screen-key    ← Écran d'accueil / saisie clé API
│   ├── #screen-main   ← Interface principale
│   └── #screen-result ← Résultat (dialogue + audio)
└── <script>        ← JS inline complet
    ├── StorageManager    (localStorage)
    ├── QuotaTracker      (compteur local)
    ├── GeminiClient      (appels API)
    ├── DialogueBuilder   (construction du prompt)
    ├── AudioPlayer       (gestion WAV)
    └── UI                (rendu, transitions)
```

**Zéro dépendance externe JS.**  
Google Fonts via CDN uniquement (Oswald + Inter).

---

## 4. FLUX UTILISATEUR (UX)

### Premier lancement
```
[Écran Accueil]
  → Message de bienvenue clair
  → Explication en 3 lignes de ce que fait l'app
  → Lien "Comment obtenir ma clé API ?" (ouvre tutoriel)
  → Champ : coller votre clé API Google
  → Bouton "COMMENCER"
```

### Usage courant
```
[Écran Principal]
  ┌─────────────────────────────────┐
  │ Compteur de quota (en haut)    │
  ├─────────────────────────────────┤
  │ Langue + Niveau                │
  │ Nombre de locuteurs             │
  │ [Locuteur 1] Prénom | Voix ♂/♀ │
  │ [Locuteur 2] Prénom | Voix ♂/♀ │
  │ ...                            │
  │ Consigne libre (textarea)      │
  │ Nombre de répliques            │
  │                                │
  │      [GÉNÉRER LE DIALOGUE]     │
  └─────────────────────────────────┘
```

### Génération en cours
```
[Indicateur Phase 1] "Rédaction du dialogue..." ⟳
[Indicateur Phase 2] "Synthèse vocale..."       ⟳ (après phase 1)
```

### Résultat
```
[Écran Résultat]
  → Bulles de dialogue colorées par locuteur
  → Lecteur audio
  → [Copier] [Imprimer] [↓ Télécharger WAV]
  → [← Générer un nouveau dialogue]
```

---

## 5. COMPTEUR DE QUOTA

### Principe
Le compteur est **local et symbolique** : il ne consulte pas l'API Google.  
Il compte les appels effectués par le navigateur de l'enseignant, stockés en `localStorage`.

### Données trackées (par `localStorage`)
```json
{
  "quota_count_today": 3,
  "quota_date": "2026-03-31",
  "quota_total_all_time": 47
}
```

### Logique de reset
- À chaque chargement de la page, comparer `quota_date` avec la date du jour
- Si date différente → remettre `quota_count_today` à 0 et mettre à jour `quota_date`
- Le "jour" se base sur **minuit heure de Bruxelles** (Europe/Brussels), soit 9h heure du Pacifique

### Affichage
Widget compact en haut de l'écran principal :

```
┌─────────────────────────────────────────┐
│ ⚡ QUOTA DU JOUR   [████████░░]  8/10  │
│ Renouvellement : aujourd'hui à 9h00     │
│ ℹ️ Quota indicatif — vérifiez dans AI Studio │
└─────────────────────────────────────────┘
```

- Barre de progression : teal → orange → rouge selon remplissage
  - 0–70% : teal
  - 70–90% : jaune
  - 90–100% : orange
- Compteur affiché : `X / 10` (on affiche 10 par sécurité, non 250, pour un usage raisonnable par prof)
- Quand quota atteint : message doux + lien AI Studio + heure de reset
- **Mention obligatoire** : "Quota indicatif. Pour voir votre quota réel : [lien AI Studio]"

### Note d'honnêteté
Ne pas prétendre que le compteur est précis. Libellé exact :  
> *"Ce compteur estime votre utilisation locale. Votre quota réel peut varier."*

---

## 6. GESTION D'ERREURS

| Erreur | Message affiché |
|--------|----------------|
| Clé API invalide | "Clé API non reconnue. Vérifiez qu'elle est bien copiée depuis AI Studio." + lien |
| Quota dépassé (429) | "Quota journalier atteint. Revenez demain à 9h00 ou vérifiez votre quota réel." + lien AI Studio |
| Erreur réseau | "Connexion impossible. Vérifiez votre réseau et réessayez." |
| Erreur TTS | "La synthèse vocale a échoué. Le texte du dialogue reste disponible." (dégradation gracieuse) |

---

## 7. CONSTRUCTION DU PROMPT

### Prompt système (génération texte)
```
Tu es un expert en didactique des langues.
Génère un dialogue pédagogique en {LANGUE} de niveau {NIVEAU_CECR}.
Le dialogue implique {N} locuteurs : {LISTE_PRENOMS}.
Contexte/consigne de l'enseignant : {CONSIGNE}
Nombre de répliques : {N_REPLIQUES}

RÉPONDS UNIQUEMENT avec un objet JSON valide, sans markdown, sans explication.
Format exact :
{
  "titre": "string",
  "langue": "string",
  "niveau": "string",
  "locuteurs": ["Prénom1", "Prénom2"],
  "repliques": [
    {"locuteur": "Prénom1", "texte": "..."},
    {"locuteur": "Prénom2", "texte": "..."}
  ]
}
```

### Prompt TTS (génération audio)
```
TTS the following dialogue. Use distinct voices for each speaker.
{LISTE_VOIX : "Prénom1 (voice: Kore)", "Prénom2 (voice: Puck)"}

{TEXTE_DU_DIALOGUE_FORMATÉ}
```

### Mapping voix Gemini TTS
| Genre demandé | Voix Gemini utilisée |
|---------------|---------------------|
| Féminin | `Kore` (naturelle, claire) |
| Masculin | `Puck` (naturel, posé) |
| Féminin 2 | `Aoede` |
| Masculin 2 | `Charon` |

---

## 8. TUTORIEL CLÉ API (contenu intégré dans l'app)

Affiché dans un panneau dépliable ou une modale, en français simple :

```
Comment obtenir votre clé API Google gratuite ?

1. Rendez-vous sur https://aistudio.google.com
2. Connectez-vous avec votre compte Google
3. Cliquez sur "Get API Key" (en haut à gauche)
4. Cliquez sur "Create API key in new project"
5. Copiez la clé affichée (elle commence par "AIza...")
6. Collez-la dans le champ ci-dessous

⚠️ Votre clé reste sur votre appareil. Elle n'est jamais partagée.
```

---

## 9. CHARTE GRAPHIQUE — RETROFUTURISME

### Palette obligatoire
```css
:root {
  --retro-teal:   #127676;  /* Structure, titres, validation */
  --retro-orange: #E4632E;  /* Actions, CTA, erreurs */
  --retro-jaune:  #E3A535;  /* Highlights, hover, bonus */
  --retro-ink:    #0D1617;  /* Fond sombre, texte principal */
  --retro-paper:  #F2EFE6;  /* Fond clair, zones de lecture */
}
```

### Typographie
```css
/* Import CDN (autorisé pour projet externe) */
@import url('https://fonts.googleapis.com/css2?family=Oswald:wght@400;600;700&family=Inter:wght@400;500;600&display=swap');

/* Titres */
font-family: 'Oswald', 'Anton', Impact, sans-serif;
text-transform: uppercase;
letter-spacing: 0.1em;

/* Corps */
font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
```

### Règle d'or : tension froid/chaud
Toujours associer une couleur froide (teal/ink) ET une chaude (orange/jaune) dans chaque zone visuelle.

### Distribution chromatique
- **60%** `paper` ou `ink` — fond, zones de contenu
- **30%** `teal` — structure, bordures, navigation, titres
- **10%** `orange` + `jaune` combinés — CTA, feedback, highlights

### Composants UI obligatoires

**Bouton principal (pill)**
```css
.retro-btn-pill {
  display: inline-flex;
  border-radius: 50px;
  border: 3px solid var(--retro-teal);
  overflow: hidden;
  cursor: pointer;
}
.retro-btn-pill-text {
  background: var(--retro-jaune);
  color: var(--retro-ink);
  padding: 0.75rem 2rem;
  font-family: 'Oswald', sans-serif;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  font-size: 1rem;
}
.retro-btn-pill-icon {
  background: var(--retro-orange);
  color: var(--retro-paper);
  width: 56px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.2rem;
}
```

**Cartes / containers**
```css
.retro-card {
  background: var(--retro-paper);
  border: 3px solid var(--retro-teal);
  border-radius: 24px 8px 24px 8px; /* asymétrie art nouveau */
  box-shadow: inset 0 0 0 2px var(--retro-orange);
  padding: 1.5rem;
}
```

**Cadre de page**
```css
.page-frame {
  background: var(--retro-paper);
  border: 4px solid var(--retro-orange);
  min-height: calc(100vh - 24px);
  margin: 12px;
  padding: 2rem;
  position: relative;
}
/* Coins floraux : SVG inline aux 4 coins (motifs teal, opacity 0.6) */
```

**Séparateurs**
Lignes avec ornement central (caractère `❧` ou `✿`), couleur teal, dégradé transparent→teal→transparent.

**Bulles de dialogue**
Chaque locuteur a une couleur de fond distincte :
- Locuteur 1 : fond teal clair (`rgba(18,118,118,0.12)`), bordure teal
- Locuteur 2 : fond orange clair (`rgba(228,99,46,0.10)`), bordure orange
- Locuteur 3 : fond jaune clair (`rgba(227,165,53,0.12)`), bordure jaune
- Locuteur 4 : fond ink clair (`rgba(13,22,23,0.08)`), bordure ink

### Cohérence sémantique des couleurs
| Couleur | Sens constant |
|---------|--------------|
| `teal` | Succès, validation, structure |
| `orange` | Action requise, erreur, CTA |
| `jaune` | Highlight, hover, information secondaire |
| `ink` | Texte, profondeur |
| `paper` | Fond, espace de lecture |

### Contraste
Texte sur fond : ratio minimum 4.5:1 partout.  
⚠️ Paper sur Orange = 4.3:1 → utiliser `ink` comme couleur de texte sur fonds orange, ou texte ≥18px bold.

---

## 10. COMPORTEMENT CSS D'IMPRESSION

```css
@media print {
  .page-frame, .retro-card,
  #quota-widget, #btn-generate,
  #screen-key, .retro-btn-pill { display: none !important; }

  .dialogue-bubble {
    break-inside: avoid;
    border: 1px solid #ccc;
    margin-bottom: 8px;
    padding: 8px;
  }
  .speaker-name { font-weight: bold; }
}
```

---

## 11. CONTRAINTES TECHNIQUES

- **Fichier unique** : tout dans `index.html` (style + script inline)
- **Zéro framework JS** : vanilla uniquement
- **Google Fonts** : via CDN (autorisé pour GitHub Pages)
- **Audio** : format WAV (sortie native Gemini TTS)
- **localStorage** : clé API + quota tracker + date
- **Responsive** : `max-width: 680px; margin: auto` — mobile first
- **Accessibilité** : labels sur tous les champs, contraste ≥ 4.5:1

---

## 12. FICHIERS DU PROJET

```
/
├── index.html          ← Application complète (fichier unique)
├── CLAUDE.md           ← Ce fichier
└── README.md           ← Instructions déploiement GitHub Pages
```

---

## 13. MESSAGES D'INTERFACE (WORDING)

Tous en français, ton chaleureux et simple (pas de jargon technique).

| Contexte | Texte |
|---------|-------|
| Titre app | `DIALOGUE STUDIO` |
| Sous-titre | `Générateur de dialogues pédagogiques` |
| Bouton générer | `GÉNÉRER LE DIALOGUE` |
| Phase 1 | `Rédaction du dialogue en cours…` |
| Phase 2 | `Synthèse des voix en cours…` |
| Succès | `Votre dialogue est prêt !` |
| Quota widget | `QUOTA DU JOUR` |
| Reset heure | `Renouvellement à 9h00 (heure de Bruxelles)` |
| Disclaimer quota | `Estimation locale — quota réel sur AI Studio` |
| Clé absente | `Pour commencer, collez votre clé API Google ci-dessous.` |

---

## 14. NOTES DE DÉVELOPPEMENT

### Appel API texte (exemple fetch)
```javascript
const response = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
      generationConfig: { temperature: 0.8, maxOutputTokens: 2048 }
    })
  }
);
```

### Appel API TTS (exemple fetch)
```javascript
const response = await fetch(
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{ parts: [{ text: ttsPrompt }] }],
      generationConfig: {
        responseModalities: ['AUDIO'],
        speechConfig: {
          multiSpeakerVoiceConfig: {
            speakerVoiceConfigs: speakerConfigs // tableau dynamique
          }
        }
      }
    })
  }
);
// La réponse contient data.candidates[0].content.parts[0].inlineData.data (base64 WAV)
```

### Conversion base64 → Blob audio
```javascript
function base64ToAudioBlob(base64) {
  const binary = atob(base64);
  const bytes = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
  return new Blob([bytes], { type: 'audio/wav' });
}
```

### Détection date locale (Bruxelles)
```javascript
function getTodayBrussels() {
  return new Intl.DateTimeFormat('fr-BE', {
    timeZone: 'Europe/Brussels',
    year: 'numeric', month: '2-digit', day: '2-digit'
  }).format(new Date());
}
```
