---
title: "Ma veille techno quotidienne, pilotée par l'IA — 100% souveraine"
date: 2026-05-16
type: "post"
showTableOfContents: true
tags: ['n8n', 'rss', 'freshrss', 'pgvector', 'postgresql', 'ai', 'infomaniak', 'self-hosting', 'embedding', 'workflow']
---

Chaque matin, je reçois un mail avec les dix articles les plus pertinents de mes flux RSS, résumés en français, classés par thème, avec un lien cliquable pour chacun. Ça tient en cinq minutes de lecture. Zéro abonnement payant, zéro donnée envoyée à une GAFAM.

Voilà comment j'ai construit ça — et pourquoi la souveraineté numérique était un critère non négociable.

---

## Le problème : trop d'information, pas assez de signal

Je suis abonné à une centaine de flux RSS : blogs techniques, actualités Red Hat et OpenShift, veille domotique, suivi de l'IA... Chaque jour, ça représente entre 100 et 300 nouveaux articles. Personne ne lit ça.

Les solutions classiques ne me convenaient pas :

- **Feedly, Inoreader** : des services cloud qui hébergent tes données de lecture, proposent l'IA comme feature premium, et dont tu dépends entièrement
- **Les newsletters** : agrégation éditoriale humaine, biais éditoriaux, rythme imposé
- **Lire directement dans FreshRSS** : efficace mais chronophage, sans priorisation automatique

Ce que je voulais : un outil qui *connaît mes centres d'intérêt*, qui *lit à ma place*, qui me donne le *résumé de ce qui compte* — et qui tourne chez moi.

---

## Le critère souverain

Je travaille quotidiennement avec des données techniques internes (architectures, configs, projets). Hors de question d'envoyer ça à OpenAI ou n'importe quel cloud américain soumis au Cloud Act.

Mes contraintes :
- Hébergement **self-hosted** pour les composants critiques (base de données, orchestration)
- IA fournie par un acteur **européen**, avec des données qui restent en Europe
- Pas de vendor lock-in sur le modèle d'embedding

J'ai trouvé ma réponse dans la combinaison **n8n + PostgreSQL/pgvector + Infomaniak AI**.

---

## L'architecture

```
FreshRSS (self-hosted)
    ↓ GReader API
n8n (self-hosted) — Phase 1 : Ingestion
    ↓ HTTP Request → Infomaniak AI (embeddings)
    ↓ SQL INSERT
PostgreSQL + pgvector (self-hosted)
    ↓
n8n — Phase 2 : Digest (enchaîné automatiquement)
    ↓ SQL SELECT (similarité cosinus)
    ↓ HTTP Request → Infomaniak AI (LLM résumé)
    ↓ SMTP
Mail quotidien
```

Tout tourne dans mon infrastructure. Le seul appel externe va vers **Infomaniak**, hébergeur suisse, infrastructure en Europe, RGPD natif.

---

## Les composants, un par un

### FreshRSS — l'agrégateur souverain

[FreshRSS](https://freshrss.org/) est un agrégateur RSS open source auto-hébergeable. Il expose une API compatible Google Reader (GReader), ce qui le rend facilement pilotable par des outils tiers.

Je l'utilise pour centraliser mes abonnements en catégories : *Actu IT*, *Domotique*, etc. C'est mon point d'entrée unique.

### n8n — l'orchestrateur

[n8n](https://n8n.io/) est une plateforme d'automatisation de workflows, open source, self-hostable. Penser à Zapier ou Make, mais sans les limites de confidentialité d'un SaaS.

Le workflow est découpé en deux phases qui s'enchaînent :

**Phase 1 — Ingestion (7h du matin)**

1. Authentification sur FreshRSS via l'API GReader
2. Récupération des articles des dernières 24h, par catégorie
3. Nettoyage HTML, détection de langue (FR/EN par comptage de mots-outils)
4. Déduplication en mémoire
5. Génération d'un embedding par article via Infomaniak AI
6. Insertion dans PostgreSQL avec déduplication en base (`WHERE NOT EXISTS`)

**Phase 2 — Digest (enchaîné)**

1. Liste de centres d'intérêt définie dans un nœud `Set`
2. Chaque intérêt est embedé séparément
3. Requête SQL de similarité cosinus par intérêt (`LIMIT 5` chacun)
4. Agrégation, déduplication, top 10
5. Résumé HTML généré par le LLM
6. Envoi par mail SMTP

### PostgreSQL + pgvector — la mémoire vectorielle

[pgvector](https://github.com/pgvector/pgvector) est une extension PostgreSQL qui ajoute un type `VECTOR` et des opérateurs de distance. Ça transforme Postgres en base de données vectorielle sans infrastructure supplémentaire.

La table est simple :

```sql
CREATE TABLE rss_articles (
    id        UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    text      TEXT NOT NULL,
    metadata  JSONB DEFAULT '{}',
    embedding VECTOR(3584)
);
```

La colonne `metadata` stocke titre, URL, source, langue et timestamp Unix. La colonne `embedding` stocke le vecteur de 3584 dimensions.

### Infomaniak AI — le cerveau, en Europe

[Infomaniak](https://www.infomaniak.com/fr/hebergement/ai-tools) est un hébergeur suisse qui propose une API d'IA compatible OpenAI. Deux modèles sont utilisés ici :

- **BGE-Multilingual-Gemma2** (9B params, 3584 dims) pour les embeddings — multilingue FR/EN natif
- **Gemma 4 31B** pour la génération du résumé HTML

Tout reste en Europe. Aucune donnée ne transite par des serveurs américains.

---

## Le cœur technique : la recherche par similarité

C'est là que la magie opère. Chaque article est stocké sous forme d'un vecteur de 3584 nombres — une représentation mathématique de son sens. Chaque centre d'intérêt est également converti en vecteur.

La recherche consiste à trouver les articles dont le vecteur est le plus *proche* du vecteur de l'intérêt. On utilise la **distance cosinus** (opérateur `<=>` dans pgvector), qui mesure l'angle entre deux vecteurs — la métrique correcte pour les embeddings de texte.

```sql
SELECT titre, lien,
       (embedding <=> '[...]'::vector)
         * CASE WHEN language = 'fr' THEN 0.75 ELSE 1.0 END AS distance
FROM rss_articles
WHERE to_timestamp(published_at::bigint) > NOW() - INTERVAL '30 days'
ORDER BY distance ASC
LIMIT 5
```

Le multiplicateur `0.75` sur les articles français est un léger bonus de pertinence pour compenser un éventuel biais du modèle vers l'anglais.

### Pourquoi un embedding par intérêt ?

J'aurais pu concaténer tous mes intérêts en une seule chaîne et générer un vecteur unique. Ce serait plus simple, mais bien moins efficace.

Imaginez : "voiture électrique" et "software supply chain" sont des sujets radicalement différents. Un vecteur moyen ne représente ni l'un ni l'autre correctement — il tombe quelque part au milieu de nulle part. En embeddant chaque intérêt séparément et en faisant N requêtes, chaque sujet trouve *ses propres* articles pertinents.

### Pourquoi BGE-Multilingual-Gemma2 ?

Le choix du modèle d'embedding est critique. J'ai commencé avec `all-MiniLM-L12-v2` : un modèle anglophone de 384 dimensions, rapide et populaire. Résultat : mes requêtes en français produisaient des vecteurs incohérents, et les articles remontés n'avaient aucun rapport avec mes intérêts.

BGE-Multilingual-Gemma2 (BAAI) change tout : 9 milliards de paramètres, 3584 dimensions, entraîné nativement sur du français et de l'anglais. Les résultats sont immédiatement pertinents.

> ⚠️ Le modèle doit être *identique* entre l'ingestion et la recherche. Changer de modèle impose de recalculer tous les embeddings existants.

---

## Le résultat

Chaque matin vers 7h30, je reçois un mail structuré :

- **Section Actu IT** : résumés des articles Red Hat, OpenShift, DevSecOps, IA...
- **Section Domotique** : Home Assistant, nouveautés domotique...
- Pour chaque article : 2 phrases de résumé en français + lien cliquable
- Une recommandation finale du LLM

Le tout généré en HTML propre, lisible dans n'importe quel client mail.

---

## Ce que j'ai appris

**La distance cosinus, pas euclidienne.** L'opérateur `<->` dans pgvector mesure la distance euclidienne, qui est inadaptée aux embeddings de texte. `<=>` mesure l'angle — c'est la bonne métrique.

**LangChain peut trahir.** J'avais d'abord utilisé les nœuds LangChain natifs de n8n pour l'ingestion. Le document loader traitait les JSON comme du texte brut et fragmentait chaque article en une dizaine de lignes. Contourner complètement LangChain avec des appels HTTP directs + SQL a tout résolu.

**Les timestamps en base.** FreshRSS retourne les dates au format Unix (secondes). PostgreSQL ne peut pas caster directement un entier en `timestamptz`. La conversion correcte : `to_timestamp((metadata->>'published_at')::bigint)`.

**pgvector et les grandes dimensions.** Les index `ivfflat` et `hnsw` sont limités à 2000 dimensions dans pgvector < 0.7.0. BGE produit 3584 dims : sans le type `halfvec` (pgvector ≥ 0.7.0), pas d'index possible. Pour quelques centaines d'articles, le scan séquentiel est parfaitement suffisant.

---

## Pour aller plus loin

Le workflow est disponible sur [GitHub](https://github.com/slallemand/n8n-workflows/tree/main/RSS-DailyDigest). Pour l'adapter :

- **Remplacer FreshRSS** par n'importe quelle source : le nœud RSS natif de n8n, un appel HTTP vers un autre agrégateur
- **Changer le fournisseur d'IA** : n'importe quelle API compatible OpenAI fonctionne (Mistral, Ollama en local...) — l'important est de garder le *même modèle* entre ingestion et recherche
- **Adapter les intérêts** : un simple champ texte dans le workflow, modifiable sans toucher au code

La stack complète tient sur un VPS modeste. C'est la veille techno que j'aurais voulu avoir depuis dix ans.
