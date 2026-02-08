--- 
title: "RAG pas ouf"
date: 2026-02-08
---

# Mise en place d'un assistant IA local pour l'interrogation d'un coffre-fort Obsidian (RAG)

### Résumé
Ce document décrit la conception d'un système de génération augmentée par récupération (RAG) permettant d'interroger une base de connaissances Obsidian de manière totalement locale. L'architecture repose sur l'utilisation du modèle DeepSeek-R1 via Ollama et l'orchestration par la bibliothèque LangChain. L'objectif est de transformer des notes statiques en une base de données interactive sans dépendre de services cloud tiers.

---

## Principes de fonctionnement
Le système traite les données selon une séquence logique permettant à l'intelligence artificielle d'accéder au contexte des notes personnelles :

1.  **Ingestion** : Le script parcourt le répertoire Obsidian pour extraire le contenu des fichiers Markdown, en excluant les dossiers de configuration internes.
2.  **Segmentation** : Les documents sont découpés en segments de 1000 caractères. Ce partitionnement assure que les informations transmises au modèle restent dans sa fenêtre de contexte.
3.  **Vectorisation** : Chaque segment est converti en vecteur numérique grâce au modèle nomic-embed-text. Cette étape traduit le sens sémantique des phrases en coordonnées mathématiques.
4.  **Indexation** : Les vecteurs sont stockés dans une base locale FAISS, ce qui permet des recherches de similarité rapides lors d'une question.


## Configuration technique
La mise en œuvre nécessite l'installation préalable de plusieurs outils :

* **Moteur d'exécution** : Ollama (chargé d'héberger les modèles).
* **Modèles** : `deepseek-r1:7b` pour la réflexion et `nomic-embed-text` pour l'indexation.
* **Environnement** : Python 3.14+ avec les bibliothèques LangChain et FAISS.

### Installation des dépendances
La préparation de l'environnement s'effectue via les commandes suivantes :
```bash
ollama pull deepseek-r1:7b
ollama pull nomic-embed-text
pip install langchain langchain-ollama faiss-cpu glob2

```


## Implémentation du script : ObsidianBrain.py

Le script suivant centralise la logique d'indexation et l'interface de discussion. Il convient de modifier la variable `VAULT_PATH` pour indiquer le chemin local du dossier Obsidian.

```python
import os
from langchain_ollama import OllamaEmbeddings, ChatOllama
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# CONFIGURATION
VAULT_PATH = "D:/Documents/Obsidian/MonVault" 
DB_PATH = "faiss_obsidian_index"

llm = ChatOllama(model="deepseek-r1:7b")
embeddings = OllamaEmbeddings(model="nomic-embed-text")

# LOGIQUE D'INDEXATION
if os.path.exists(DB_PATH):
    print("--- Chargement de l'index existant ---")
    vectorstore = FAISS.load_local(DB_PATH, embeddings, allow_dangerous_deserialization=True)
else:
    print("--- Initialisation de l'indexation du Vault ---")
    all_docs = []
    for root, dirs, files in os.walk(VAULT_PATH):
        if ".obsidian" in root: continue
        for file in files:
            if file.endswith(".md"):
                try:
                    loader = TextLoader(os.path.join(root, file), encoding="utf-8", autodetect_encoding=True)
                    all_docs.extend(loader.load())
                except Exception: pass

    text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
    splits = text_splitter.split_documents(all_docs)
    vectorstore = FAISS.from_documents(documents=splits, embedding=embeddings)
    vectorstore.save_local(DB_PATH)

retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={'k': 8})

# PIPELINE DE REPONSE
template = """L'assistant utilise les notes suivantes pour répondre à l'utilisateur.
CONTEXTE : {context}
QUESTION : {question}
REPONSE :"""

prompt = ChatPromptTemplate.from_template(template)
rag_chain = ({"context": retriever | (lambda docs: "\n\n".join(d.page_content for d in docs)), 
              "question": RunnablePassthrough()} | prompt | llm | StrOutputParser())

# INTERACTION
while True:
    query = input("\nQuestion posée au cerveau local (ou 'exit') : ")
    if query.lower() in ['exit', 'quit']: break
    print(f"\n{rag_chain.invoke(query)}")

```


## Maintenance et optimisation

* **Actualisation des données** : Le système ne détecte pas les modifications de fichiers en temps réel. Pour intégrer de nouvelles notes, il est nécessaire de supprimer le répertoire `faiss_obsidian_index` afin de forcer une nouvelle indexation au prochain lancement.
* **Qualité des réponses** : La précision dépend de la structure des notes. L'usage de titres explicites et de mots-clés dans Obsidian facilite le travail de récupération du moteur de recherche vectoriel.
* **Performance** : Sur des configurations matérielles limitées, augmenter la taille des segments (chunk_size) peut réduire la charge de calcul, au prix d'une précision moindre.

## Conclusion et perspectives

Cette installation fournit une base fonctionnelle pour l'exploitation privée de données personnelles par une IA. Le projet peut évoluer vers une automatisation de l'indexation ou l'intégration d'une interface utilisateur plus ergonomique via des outils comme Streamlit.

**Prochaines étapes :**

* Développement d'un script de surveillance (watcher) pour automatiser la mise à jour de l'index.
* Exploration de l'hybridation de la recherche pour inclure des critères lexicaux classiques.

