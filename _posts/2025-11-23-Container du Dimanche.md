-----

## Résumé

C'est l'histoire d'un réveil dominical perturbé par un conteneur Docker inattendu. Cet article raconte mon enquête sur un conteneur `n8n-mcp` qui tournait en tâche de fond sans invitation. Au menu : débogage avec `docker events`, découverte de l'interaction secrète entre **Gemini Code Assist**, **VS Code** et l'intégration native **MCP de Docker Desktop**, et une explication du pourquoi mon environnement de dev s'est transformé en champ de bataille numérique dès l'ouverture de mon éditeur.

-----

## Dimanche matin, un café et... un Docker fantôme ?

Dimanche matin. Tout le monde dort, mon café est chaud. Je m'installe devant mon PC pour bricoler tranquillement. J'ouvre mon terminal, je tape machinalement `docker ps` et là... surprise.

Un conteneur tourne. Je n'ai rien lancé. C'est quoi ce truc ?

### Le Suspect : l'image n8n-mcp

```bash
CONTAINER ID    IMAGE                                COMMAND                  STATUS
6b74949ca949    ghcr.io/czlonkowski/n8n-mcp:latest   "/usr/local/bin/dock…"   Up 21 minutes (unhealthy)
```

Il s'agit de la dernière version du serveur MCP pour n8n (le projet est dispo [ici sur GitHub](https://github.com/czlonkowski/n8n-mcp)). Pas *si* étonnant, vu que je bidouille le **Model Context Protocol (MCP)** ces derniers temps.

Mais regardez son statut : **Unhealthy**.
Le pauvre est malade. Un petit `docker inspect` me révèle pourquoi il souffre :

> `curl: (7) Failed to connect to 127.0.0.1 port 3000`

Il essaie désespérément de parler à mon instance locale n8n (`localhost:5678`). Sauf que mon n8n local, lui, il dort encore. Je ne l'ai pas lancé.

## Première piste : C'est ma faute (comme d'habitude)

Mon premier réflexe, c'est de me blâmer. Je me dis : *"C'est sûrement mon `settings.json` que j'utilise avec Gemini Cli."*

J'ai effectivement cette config qui traîne :

```json
"n8n-mcp": {
    "command": "docker",
    "args": [
        "run", "-i", "--rm",
        "-e", "N8N_API_URL=http://host.docker.internal:5678",
        "ghcr.io/czlonkowski/n8n-mcp:latest"
    ]
}
```

Ça ressemble à un coupable idéal. Mais attendez... Un fichier JSON, c'est passif. Ça ne lance pas des commandes `docker run` tout seul pendant que je me frotte les yeux. Il y a autre chose.

## On fouille :  **docker events** est mon ami 

Je ferme tout. Je relance VS Code, mais cette fois, je suis prêt. J'ai un mouchard branché sur le moteur Docker :

```bash
docker events --filter 'type=container'
```

Et là, le terminal se met à cracher des logs. C'est une mine d'or \! Ce n'est pas juste un fichier texte qui est lu. C'est une fonctionnalité "Beta" de Docker Desktop qui s'active.

Les logs affichent des tags révélateurs : `docker-mcp=true` et `docker-mcp-name=notion`.

**Le déclic :** Hier, j'ai installé **Gemini Code Assist**.
Au démarrage de VS Code, l'extension lit ma config et dit à Docker : *"Hey, réveille tout le monde, on va bosser \!"*.

## 11h06 un dimanche matin : Analyse du crash

Les logs racontent ce qui s'est déroulée en quelques secondes (à 11:06 précises) :

1.  **11:06:07 - L'Intrus se lève :** Le conteneur `n8n-mcp` démarre. Il panique immédiatement (pas de n8n local), mais il tient bon.
2.  **11:06:08 - L'Agent Secret :** Un conteneur bizarre, `docker/jcat`, apparaît. C'est un ninja de chez Docker. Sa mission ? Injecter mes tokens Notion en douce et disparaître. Classe.
3.  **11:06:09 - L'Attaque des Clones :** Docker lance *toute* l'armada MCP d'un coup : `mcp/notion`, `mcp/playwright`, `mcp/filesystem`.
4.  **11:06:10 - Le Repli Stratégique :**
      * Le serveur `filesystem` crashe (`exitCode=1`).
      * VS Code, voyant ce bazar (un n8n malade et un filesystem mort), prend peur. Il coupe la connexion.
      * Résultat : Tous les conteneurs MCP s'éteignent proprement (`exitCode=0`).

## Conclusion : Le futur est automatisé (peut-être un peu trop ?)

Ce matin, j'ai appris que mon IDE ne se contente plus d'éditer du texte. Avec l'intégration MCP dans Docker Desktop et les assistants IA comme Gemini, mon environnement est devenu un écosystème vivant.

VS Code a tenté de lancer mes outils pour que l'IA ait du contexte (mes fichiers, mon Notion, mon n8n), mais comme il manquait une brique (le serveur n8n local), tout s'est écroulé comme un château de cartes.

### La leçon du dimanche

Si vous jouez avec MCP et Docker, n'oubliez pas : vos fichiers de configuration sont désormais des **déclencheurs d'actions**.

### Prochaines Étapes

  * Boire mon café avant qu'il ne soit froid.
  * Modifier ma config pour que ces conteneurs ne se lancent que sur demande explicite, pour éviter de transformer mon laptop en serveur de prod à chaque ouverture de VS Code.

