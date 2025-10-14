# Virtual Threads en Java 21 → 25  
### Simplifier la concurrence à grande échelle  
**Atelier interne — [Ton Nom]**  
**[Entreprise] — Octobre 2025**

---

## Qu’est-ce qu’un Thread ?
- Un thread est une unité d’exécution dans un processus.
- Permet le multitâche au sein d’un même programme.
- Les threads partagent la mémoire du processus.
- Exemple visuel : plusieurs ouvriers travaillant sur le même chantier.

---

## Threads physiques dans la JVM
- Chaque `java.lang.Thread` correspond à un **thread natif** du système.
- L’OS gère chaque thread séparément → coût de création et de contexte élevé.
- En Java classique : quelques milliers de threads max dans une appli courante.

