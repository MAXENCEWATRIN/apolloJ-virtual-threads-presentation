# sommaire.md

[🏠 Retour à l'accueil](index.md)

---

# 📋 Sommaire

## 1. Fondamentaux

### 1.1 [Qu'est-ce qu'un Thread ?](slides/01-qu-est-ce-qu-un-thread.md)
- Thread physique au niveau système
- Caractéristiques et coûts
- Context switching

### 1.2 [La JVM et les Threads Platform](slides/02-jvm-platform-threads.md)
- Architecture JVM
- Mapping 1:1 avec l'OS
- Consommation mémoire

---

## 2. Les Limitations

### 2.1 [Limitations Réelles sur la JVM](slides/03-limitations-jvm.md)
- Le problème thread-per-request
- Cas d'usage problématiques
- Métriques et impacts

---

## 3. La Solution : Virtual Threads

### 3.1 [Introduction aux Virtual Threads](slides/04-virtual-threads-intro.md)
- Concept fondamental
- Mécanisme mounting/unmounting
- Caractéristiques et avantages

### 3.2 [Comparaison Avant/Après](slides/05-avant-apres.md)
- Code avant Virtual Threads
- Code avec Virtual Threads
- Gains de performance

---

## 4. Alternatives Historiques

### 4.1 [Avant Java 21 : Les Alternatives](slides/06-alternatives-historiques.md)
- Thread Pools classiques
- CompletableFuture
- Reactive Programming (WebFlux)
- Kotlin Coroutines
- Comparaison des approches

---

## 5. Mise en Pratique

### 5.1 [Virtual Threads Sans Spring Boot](slides/07-pratique-java-pur.md)
- API de base
- ExecutorService
- Structured Concurrency
- Serveur HTTP simple

### 5.2 [Projet Spring Boot](slides/08-spring-boot-projet.md)
- Configuration Maven/Gradle
- Configuration application
- Service Layer
- Controllers et REST API
- @Async avec Virtual Threads
- Monitoring

### 5.3 [Tests et Benchmarks](slides/09-tests-benchmarks.md)
- Tests de performance
- Benchmarks comparatifs
- Métriques et résultats

---

## 6. Déploiement

### 6.1 [Environnement On-Premise](slides/10-deploiement-on-premise.md)
- Configuration serveur
- Dockerfile
- Monitoring

### 6.2 [Environnement Cloud (AWS/K8s)](slides/11-deploiement-cloud.md)
- Architecture Kubernetes
- Configuration EKS/ECS
- Ressources et limites
- Auto-scaling
- Observabilité

---

## 7. Écosystème et Comparaisons

### 7.1 [État des Langages Concurrents](slides/12-langages-concurrents.md)
- C# async/await
- Kotlin Coroutines
- Go Goroutines
- Rust Tokio
- Python asyncio
- Comparatifs et classements

---

## 8. Conclusion

### 8.1 [Recommandations et Meilleures Pratiques](slides/13-recommendations.md)
- Quand utiliser Virtual Threads
- Pièges à éviter (pinned threads)
- Migration depuis code existant
- Checklist de migration

### 8.2 [Ressources et Références](slides/14-ressources.md)
- Documentation officielle
- Articles et talks
- Projets exemples
- Communauté

---

<br>

[🏠 Retour à l'accueil](index.md) | [▶️ Commencer →](slides/01-qu-est-ce-qu-un-thread.md)