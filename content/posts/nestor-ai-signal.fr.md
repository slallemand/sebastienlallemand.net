---
title: "Nestor, mon assistant IA maison — vocal et domotique via Signal"
date: 2026-05-16
type: "post"
showTableOfContents: true
tags: ['n8n', 'signal', 'home-assistant', 'ai', 'infomaniak', 'self-hosting', 'workflow', 'mcp', 'whisper', 'ollama']
---

J'ai arrêté de chercher l'application parfaite pour piloter ma maison et gérer mon quotidien. J'en ai construit une. Elle s'appelle **Nestor**.

Nestor est un assistant personnel accessible depuis un groupe Signal. Il comprend le texte et la voix, sait allumer les lumières, fermer les volets, donner la météo, et se souvient du fil de la conversation. Il tourne entièrement en self-hosted, sans envoyer mes données à Google ou Apple.

![Conversation avec Nestor dans Signal](/images/nestor-signal.png)

---

## Le problème que je voulais résoudre

J'utilise déjà Signal pour toutes mes communications. Je voulais éviter d'installer une énième application pour interagir avec ma domotique ou poser une question rapide à un LLM. L'idée : **un seul point d'entrée, dans l'application que j'ai déjà ouverte.**

La contrainte principale était la confidentialité. Mes messages Signal sont chiffrés de bout en bout. Mon Home Assistant tourne sur ma propre infrastructure. Je voulais que mon assistant s'inscrive dans cette logique : des données qui restent chez moi.

---

## L'architecture

```
Signal (groupe "NestorAI")
  │
  ├─ Sécurité : AllowList → seul le groupe autorisé est traité
  │
  ├─ Message texte ──────────────────────────────────────────────┐
  │                                                              │
  └─ Message vocal → téléchargement → Whisper → transcription ──┘
                                                                 │
                                                        Agent LLM (Gemma 4 31B)
                                                                 │
                                              ┌──────────────────┴──────────────────┐
                                           Météo                              HASS MCP
                                        (Home Assistant)               (domotique complète)
                                                                 │
                                                       Réponse → Signal
```

Quatre composants clés : **Signal** comme interface, **n8n** comme moteur d'orchestration, **Infomaniak AI** pour le LLM et la transcription vocale, et **Home Assistant** pour la domotique.

---

## Signal comme interface : un choix délibéré

J'aurais pu utiliser Telegram, WhatsApp ou une application web. J'ai choisi Signal pour deux raisons.

D'abord, le chiffrement de bout en bout est non-négociable pour des messages qui vont contenir des questions sur ma maison ou mon agenda. Ensuite, je l'utilise déjà : pas d'application supplémentaire à ouvrir, pas de compte supplémentaire à gérer.

L'intégration repose sur **signal-cli** et son API REST. n8n se connecte à ce service pour recevoir les messages entrants et envoyer les réponses.

La sécurité est gérée par une liste d'autorisation explicite : seuls les messages provenant du groupe nommé `"NestorAI"` sont traités. Tout le reste est ignoré silencieusement. L'ID du groupe est résolu dynamiquement à chaque démarrage — si le groupe est recréé, le workflow continue de fonctionner sans modification.

---

## La magie des messages vocaux

C'est la fonctionnalité qui change vraiment les usages. Depuis mon téléphone, je peux envoyer un message vocal comme je le ferais à un ami, et Nestor le comprend.

Le pipeline de transcription se déroule en cinq étapes :

1. **Détection** : si `messageText` est vide, c'est un message vocal
2. **Téléchargement** : récupération du fichier audio depuis signal-cli
3. **Transcription** : envoi à l'API Whisper d'Infomaniak
4. **Attente + retry** : l'API est asynchrone, on attend 1 s puis on interroge avec 5 tentatives
5. **Extraction** : le texte transcrit rejoint le pipeline texte standard

À partir de là, peu importe que le message soit écrit ou oral — le même agent LLM le traite.

---

## Le workflow n8n

![Workflow NestorAI dans n8n](/images/nestor-workflow.png)

## L'agent Nestor : mémoire et outils

Le cœur du système est un **agent LLM** basé sur Gemma 4 31B, servi par Infomaniak AI. Un modèle Gemma3 local via Ollama prend le relais en cas d'indisponibilité.

### La mémoire de session

Nestor se souvient des 10 derniers échanges par session. La clé de session est le numéro Signal de l'expéditeur : chaque membre du groupe a donc sa propre fenêtre de contexte. Pas besoin de répéter "comme je t'ai dit tout à l'heure" — Nestor suit le fil.

### Les outils

| Outil | Source | Rôle |
|---|---|---|
| **Météo** | Entité Home Assistant `weather.<ville>` | Conditions météo locales actuelles |
| **HASS MCP** | Serveur MCP de Home Assistant (`/api/mcp`) | Contrôle domotique : lumières, volets, capteurs, scènes… |

L'intégration via **MCP (Model Context Protocol)** est particulièrement intéressante pour Home Assistant. Plutôt que de créer un outil par équipement, Nestor accède à l'API MCP de Home Assistant et interagit avec tous les équipements exposés en une seule connexion. "Ferme les volets du salon", "Quelle est la température dans la chambre ?", "Allume la lumière de bureau à 40%" — tout passe par le même outil.

### Les indicateurs de frappe

Un détail d'UX qui fait toute la différence : dès que Nestor identifie un message valide, il envoie l'indicateur "en train d'écrire" dans Signal. L'utilisateur sait immédiatement que son message a été pris en compte, même si le LLM met quelques secondes à répondre. L'indicateur s'arrête juste avant l'envoi de la réponse.

---

## Ce que j'ai appris

**Signal + MCP + LLM = interface naturelle pour la maison.** La combinaison est plus ergonomique que je ne l'espérais. Dire "mets la clim à 22 et ferme les volets" en vocal pendant qu'on est dans la cuisine est plus naturel qu'ouvrir l'application Home Assistant et naviguer jusqu'aux bons contrôles.

**Le fallback local est indispensable.** Infomaniak AI est disponible en permanence dans mon expérience, mais avoir Ollama en local comme filet de sécurité change la relation qu'on a avec le service principal. Une panne ne rend pas l'assistant inutilisable.

**La résolution dynamique de l'ID de groupe.** Coder en dur l'ID d'un groupe Signal est un piège classique — il change si le groupe est recréé. Résoudre l'ID au démarrage via `GetGroups` + comparaison par nom rend le workflow beaucoup plus robuste.

**Whisper est remarquablement précis.** Les messages vocaux en français sont transcrits avec très peu d'erreurs, même dans un environnement bruyant. Le pipeline asynchrone avec retry absorbe bien les variations de latence de l'API.

---

## Pour aller plus loin

Le workflow est disponible sur [GitHub](https://github.com/slallemand/n8n-workflows/tree/main/NestorAI). Pour l'adapter :

- **Changer le groupe Signal autorisé** : modifier la valeur dans le nœud `AllowList`
- **Ajouter un outil** : brancher un nœud Tool sur l'entrée `ai_tool` de l'agent — calendrier, emails, tâches…
- **Changer le LLM** : modifier le modèle dans `OpenAI Chat Model` (Infomaniak) ou `Ollama Chat Model`
- **Ajuster la taille de la mémoire** : modifier `contextWindowLength` dans `Simple Memory`

L'architecture s'inspire du workflow [Angie](https://n8n.io/workflows/2462-angie-personal-ai-assistant-with-telegram-voice-and-text/) de Derek Cheung, dont j'ai repris la logique voix/texte/mémoire en remplaçant Telegram par Signal et en ajoutant la couche domotique.
