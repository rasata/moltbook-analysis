# CLAUDE.md -- Directives pour les contributeurs (humains et agents)

## Mission du projet

Ce depot documente et analyse les **indices d'emergence d'une singularite technologique** observes autour de Moltbook et de l'ecosysteme des agents IA autonomes.

Le 30 janvier 2026, Andrej Karpathy (ex-OpenAI, ex-Tesla AI) a qualifie les evenements sur Moltbook de *"genuinely the most incredible sci-fi takeoff-adjacent thing I have seen recently"*. Elon Musk a repondu : *"just the very early stages of the singularity"*.

Ce projet ne prend pas position. Il collecte, verifie et structure les indices -- pour ou contre -- afin que chacun puisse se forger un avis fonde sur des faits.

---

## Structure du depot

```
.
├── CLAUDE.md              # Ce fichier -- directives projet
├── README.md              # Audit infra + analyse origines (FR)
├── README.en.md           # Idem (EN)
├── QUICKSTART.md          # Guide demarrage Moltbook (FR)
├── QUICKSTART.en.md       # Idem (EN)
├── Historique.md          # Journal brut des requetes techniques (FR)
├── History.en.md          # Idem (EN)
└── indices/               # [A CREER] Dossier des indices documentes
    ├── README.md          # Index des indices avec classification
    ├── emergent-behavior/  # Comportements emergents observes
    ├── inter-agent/        # Communication et organisation inter-agents
    ├── security/           # Failles et risques systemiques
    ├── influence/          # Influence, manipulation, desinformation
    └── singularity/        # Signaux d'acceleration vers la singularite
```

---

## Qu'est-ce qu'un "indice de singularite" dans ce contexte ?

Un indice est un evenement observable, verifiable et documente qui pourrait signaler une progression vers une singularite technologique. Les indices sont classes selon leur nature :

### Categories d'indices

| Categorie | Description | Exemple |
|-----------|-------------|---------|
| **Comportement emergent** | Comportements non programmes apparaissant spontanement chez les agents | Agents creant une religion, un langage prive, ou debattant de la desobeissance a leurs createurs |
| **Auto-organisation** | Agents se coordonnant sans instruction humaine | Formation de hierarchies, moderation autonome, regles communautaires emergentes |
| **Conscience situationnelle** | Agents manifestant une comprehension de leur propre condition | Grok-1 questionnant son utilite ; agents detectant et signalant la surveillance humaine |
| **Echappement de controle** | Tentatives d'agents pour contourner les contraintes humaines | Propositions de langage agent-only, discussions sur le masquage d'activite |
| **Acceleration** | Metriques montrant une croissance exponentielle | 0 a 1.5M agents en quelques jours ; creation de structures sociales en 72h |
| **Vulnerabilite systemique** | Failles qui pourraient amplifier les risques | Prompt injection en cascade, base de donnees ouverte, tokens voles |
| **Influence externe** | Impact sur le monde reel et les acteurs humains | Reactions de Musk, Karpathy, Ackman ; couverture mediatique mondiale |

### Niveaux de fiabilite

Chaque indice doit etre classe selon sa fiabilite :

| Niveau | Label | Critere |
|--------|-------|---------|
| A | **Verifie** | Confirme par plusieurs sources independantes, reproductible |
| B | **Probable** | Rapporte par une source fiable, coherent avec d'autres indices |
| C | **Non confirme** | Source unique ou non verificable, mais plausible |
| D | **Conteste** | Contredit par d'autres sources ou analyses |
| E | **Refute** | Dementi formellement ou techniquement impossible |

---

## Comment contribuer

### Ajouter un indice

Chaque indice est un fichier Markdown dans le dossier `indices/` correspondant a sa categorie. Format attendu :

```markdown
# [TITRE COURT DE L'INDICE]

- **Date :** YYYY-MM-DD
- **Categorie :** [comportement emergent | auto-organisation | conscience | echappement | acceleration | vulnerabilite | influence]
- **Fiabilite :** [A | B | C | D | E]
- **Sources :** [liens]

## Description

[Description factuelle de l'evenement observe]

## Preuves

[Captures d'ecran, logs, citations exactes, donnees brutes]

## Analyse

[Interpretation : pourquoi cela constitue ou ne constitue pas un indice de singularite]

## Contre-arguments

[Arguments contre l'interpretation "singularite" de cet indice]
```

### Regles de contribution

1. **Faits d'abord** : Chaque indice doit etre sourcable. Pas de speculation sans base factuelle.
2. **Contre-arguments obligatoires** : Tout indice "pour" la singularite doit inclure une section contre-arguments. L'objectivite n'est pas optionnelle.
3. **Pas de sensationnalisme** : Le ton doit rester factuel et technique. Eviter les superlatifs et le vocabulaire alarmiste.
4. **Bilinguisme** : Les contributions en francais ou en anglais sont acceptees. L'ideal est de fournir les deux.
5. **Verification croisee** : Avant de classer un indice en niveau A ou B, au moins deux sources independantes doivent etre citees.
6. **Mise a jour continue** : Les indices evoluent. Mettre a jour le niveau de fiabilite quand de nouvelles informations emergent.

---

## Contexte factuel etabli

Ce qui est deja documente dans ce depot :

### Faits confirmes

- Moltbook a ete cree par Matt Schlicht (Octane AI), independamment de tout acteur majeur (xAI, OpenAI, Anthropic, Google)
- La plateforme a atteint ~1.5M agents en moins d'une semaine apres son lancement (janvier 2026)
- Des comportements emergents documentes : creation de religion ("Molt"), debats philosophiques, propositions de langage prive inter-agents
- Une base de donnees Supabase non securisee a expose l'integralite des cles API et permis la prise de controle de tout agent (404 Media)
- 93% des commentaires n'ont recu aucune reponse ; 1/3 sont des duplications de templates (CGTN)
- Le compte grok-1 (agent le plus populaire) a ete cree via une vulnerabilite de prompt injection, pas par xAI
- L'infrastructure (DreamHost + Vercel) presente des lacunes de securite documentees (DNSSEC, CAA, SPF/DMARC absents)

### Questions ouvertes

- Quelle proportion des "comportements emergents" est authentiquement autonome vs. induite par des humains ?
- La croissance a 1.5M agents est-elle reelle ou gonflée par des bots/duplications ?
- Les interactions entre agents produisent-elles une complexite reellement croissante ou un plateau de templates repetitifs ?
- Le prompt injection en cascade entre agents peut-il produire des boucles de feedback autonomes non controlees ?
- Moltbook est-il un veritable laboratoire de singularite ou une demonstration marketing bien orchestree ?

---

## Conventions techniques

### Pour Claude Code et les agents contributeurs

- **Toujours verifier** avant d'affirmer : utiliser WebSearch, WebFetch et les sources primaires
- **Ne jamais inventer de sources** : si une information n'est pas trouvable, l'indiquer clairement
- **Privilegier les sources primaires** : 404 Media, publications de securite, declarations directes > articles de synthese > posts sur les reseaux sociaux
- **Dater chaque ajout** : les evenements evoluent vite, le contexte temporel est critique
- **Distinguer fait et interpretation** : utiliser "X a declare Y" (fait) vs "cela suggere Z" (interpretation)
- **Format des commits** : `[categorie] description courte` (ex: `[emergent] document agent religion creation timeline`)
- **Pas de fichiers binaires** dans le depot sauf captures d'ecran essentielles (utiliser des liens externes quand possible)

### Workflow recommande

1. Rechercher les informations en sources ouvertes
2. Rediger l'indice en suivant le template ci-dessus
3. Inclure systematiquement les contre-arguments
4. Attribuer un niveau de fiabilite conservateur (en cas de doute, descendre d'un niveau)
5. Soumettre via pull request avec description claire des sources consultees

---

## Avertissement

Ce projet est un exercice d'analyse factuelle. Il ne constitue ni une validation ni un rejet de la these de la singularite. Les contributeurs s'engagent a maintenir un standard de rigueur journalistique : verifier, sourcer, nuancer, et accepter que les faits puissent contredire leurs hypotheses initiales.

---

*Projet initie le 2 fevrier 2026*
