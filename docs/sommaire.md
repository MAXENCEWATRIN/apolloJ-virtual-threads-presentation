# sommaire.md

[🏠 Retour à l'accueil](index.md)

---

# 📋 Sommaire

## [1. Qu'est-ce qu'un Thread ?](slides/01-qu-est-ce-qu-un-thread.md)
#### 1.1 Définition
- Unité d'exécution, analogie restaurant
#### 1.2 Thread au niveau OS
- Architecture, mapping, stack
#### 1.3 Caractéristiques d'un Thread OS
- Mémoire, coût, context switching, limites
#### 1.4 Cycle de vie
- États NEW, RUNNABLE, BLOCKED, TERMINATED
#### 1.5 Coût du context switching
- Impact du nombre de threads, benchmarks
#### 1.6 Mesure réelle des coûts
- Création, mémoire, benchmarks
#### 1.7 Limites physiques
- Stack, RAM, limites pratiques
#### 1.8 Concepts clés à retenir
- Synthèse des limitations

## [2. La JVM et les Platform Threads](slides/02-jvm-platform-threads.md)
#### 2.1 Architecture : Du Thread Java au Thread OS
- Mapping 1:1, historique, schéma JVM/OS
#### 2.2 Anatomie d'un Thread Java
- Structure mémoire, Heap, Stack, ThreadLocal
#### 2.3 Création d'un Thread
- Séquence d'opérations, différence BLOCKED/I/O
#### 2.4 Cas pratique : Thread avec JDBC
- Exemple accès DB, timeline d'exécution
#### 2.5 Cas pratique : Thread avec HTTP Client
- Appel API REST, approche asynchrone
#### 2.6 Thread Pools
- ExecutorService, types de pools, schémas
#### 2.7 Synchronisation et Monitors
- Problème de concurrence, synchronized, points clés

## [3. Limitations Réelles des Platform Threads](slides/03-limitations-jvm.md)
#### 3.1 Thread-per-Request
- Architecture serveur, démonstration, saturation
#### 3.2 Coût du context switching intensif
- Benchmarks, impact sur microservices
#### 3.3 I/O bloquant
- Analyse application réelle, gaspillage
#### 3.4 Cas d'usage critiques
- WebSocket, batch, microservices en cascade
#### 3.5 Solutions actuelles et limites
- Plus de threads, réactif, async, scaling horizontal
#### 3.6 Tableau récapitulatif des limitations
- Synthèse des impacts
#### 3.7 Métriques réelles
- Profil e-commerce, analyse journée type
#### 3.8 Résumé : Le mur des Platform Threads
- Chiffres clés, synthèse

## [4. La Solution : Virtual Threads](slides/04-virtual-threads-intro.md)
#### 4.1 Qu'est-ce qu'un Virtual Thread ?
- Définition, analogie, comparaison PT/VT
#### 4.2 Mounting/Unmounting
- Cycle de vie, schéma, mounting/unmounting
#### 4.3 Architecture interne
- Scheduler, ForkJoinPool, configuration
#### 4.4 Caractéristiques techniques
- Poids mémoire, performance création
#### 4.5 API Java pour Virtual Threads
- Création, détection, exemples
#### 4.6 Blocking gratuit
- Démonstration, comparaison PT/VT
#### 4.7 Ce qui change/ne change pas
- Compatibilité, différences, pinning
#### 4.8 Pinned Threads
- Démonstration, solution ReentrantLock
#### 4.9 Résumé
- Points clés, tableau comparatif

## [5. Exemples Pratiques : Avant/Après](slides/05-avant-apres.md)
#### 5.1 Exemple Spring Boot
- API REST, configuration, benchmarks
#### 5.2 Exemple Java natif : batch
- Traitement batch, benchmarks, comparaison
#### 5.3 Synthèse des changements
- Ce qui change, gains mesurés, effort de migration

## [6. Conclusion et Recommandations](slides/06-conclusion.md)
#### 6.1 Project Loom : Virtual Threads
- Avantages, inconvénients, synthèse
#### 6.5 Quand utiliser Virtual Threads ?
- Cas d'usage parfaits, inappropriés

---

<br>

[🏠 Retour à l'accueil](index.md) | [📚 Sources](sources.md) | [▶️ Commencer →](slides/01-qu-est-ce-qu-un-thread.md)
