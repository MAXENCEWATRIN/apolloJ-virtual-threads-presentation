[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [➡️ Suivant](02-jvm-platform-threads.md)

---

# 1. Qu'est-ce qu'un Thread ?

## 1.1 Définition

Un **thread** (ou fil d'exécution) est la plus petite unité d'exécution qu'un système d'exploitation peut gérer. C'est une séquence d'instructions qui peut être exécutée de manière indépendante.

### Analogie du restaurant 🍽️

Imaginez un restaurant :
- **Le restaurant** = votre application
- **La cuisine** = le CPU
- **Les serveurs** = les threads
- **Les clients** = les requêtes/tâches

Plus vous avez de serveurs, plus vous pouvez servir de clients simultanément. Mais :
- Chaque serveur coûte cher (salaire)
- Trop de serveurs dans une petite cuisine = chaos
- Les serveurs doivent se coordonner (context switching)

---

## 1.2 Thread au niveau du système d'exploitation

### Architecture complète

```
┌─────────────────────────────────────────────────┐
│         Application (Processus)                 │
│                                                 │
│  ┌────────┐  ┌────────┐  ┌────────┐            │
│  │Thread 1│  │Thread 2│  │Thread 3│            │
│  │ Stack  │  │ Stack  │  │ Stack  │            │
│  │ ~2 MB  │  │ ~2 MB  │  │ ~2 MB  │            │
│  └───┬────┘  └───┬────┘  └───┬────┘            │
│      │           │           │                  │
└──────┼───────────┼───────────┼──────────────────┘
       │           │           │
       │  Appels système (syscalls)
       │           │           │
┌──────▼───────────▼───────────▼──────────────────┐
│           Kernel Space (OS)                     │
│                                                 │
│  ┌──────────────────────────────────┐          │
│  │    Thread Scheduler               │          │
│  │  - Gère les priorités            │          │
│  │  - Context switching              │          │
│  │  - Allocation CPU time            │          │
│  └──────────────────────────────────┘          │
│                                                 │
│  Thread Control Blocks (TCB):                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ TCB #1  │ │ TCB #2  │ │ TCB #3  │          │
│  │ State   │ │ State   │ │ State   │          │
│  │ Priority│ │ Priority│ │ Priority│          │
│  │ Registers│ │ Registers│ │ Registers│        │
│  └─────────┘ └─────────┘ └─────────┘          │
└──────┬───────────┬───────────┬──────────────────┘
       │           │           │
┌──────▼───────────▼───────────▼──────────────────┐
│              CPU Hardware                       │
│                                                 │
│   ┌────────┐  ┌────────┐  ┌────────┐          │
│   │ Core 1 │  │ Core 2 │  │ Core 3 │          │
│   └────────┘  └────────┘  └────────┘          │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 1.3 Caractéristiques d'un Thread OS (Platform Thread)

| Caractéristique | Valeur typique | Explication |
|-----------------|----------------|-------------|
| **Mémoire Stack** | 1-2 MB | Espace mémoire alloué pour les variables locales et l'historique d'appels |
| **Temps de création** | 0.2 - 1 ms | Temps nécessaire pour créer un nouveau thread via l'OS |
| **Context Switching** | 1 - 10 µs | Temps pour sauvegarder l'état d'un thread et restaurer un autre |
| **Thread Control Block** | ~1.5 KB | Métadonnées du thread (ID, priorité, registres CPU, etc.) |
| **Limite pratique** | 1000-5000 | Nombre max de threads avant dégradation des performances |
| **Gestion** | OS Kernel | C'est le système d'exploitation qui gère la vie du thread |

---

## 1.4 Le cycle de vie d'un Thread

```
┌─────────┐
│   NEW   │  Thread créé mais pas encore démarré
└────┬────┘
     │ start()
     ▼
┌─────────┐
│RUNNABLE │  Thread prêt à s'exécuter ou en cours d'exécution
└────┬────┘
     │
     ├─────► ┌─────────┐
     │       │ RUNNING │  Thread actuellement exécuté sur un CPU
     │       └─────────┘
     │            │
     │            │ I/O, sleep(), wait()
     │            ▼
     │       ┌─────────┐
     └──────►│ BLOCKED │  Thread en attente (I/O, lock, etc.)
             └────┬────┘
                  │ I/O terminée, notify()
                  │
                  ▼
             ┌─────────┐
             │RUNNABLE │  Retour à l'état prêt
             └────┬────┘
                  │
                  │ Fin d'exécution
                  ▼
             ┌──────────┐
             │TERMINATED│  Thread terminé
             └──────────┘
```

---

## 1.5 Le coût du Context Switching

### Qu'est-ce que le Context Switching ?

Lorsque le CPU passe d'un thread à un autre, il doit :
1. **Sauvegarder** l'état du thread actuel (registres, program counter, etc.)
2. **Restaurer** l'état du thread suivant
3. **Vider/recharger** les caches CPU (coûteux !)

### Exemple visuel sur une timeline

```
Timeline sur 1 CPU core avec 4 threads actifs:

Thread A: ████████░░░░░░░░████████░░░░░░░░████████
Thread B: ░░░░░░░░████████░░░░░░░░████████░░░░░░░░
Thread C: ░░░░░░░░░░░░░░░░████████░░░░░░░░████████
Thread D: ████████░░░░░░░░░░░░░░░░████████░░░░░░░░
Switch:   ▲───────▲───────▲───────▲───────▲───────
          Context Context Context Context Context
          Switch  Switch  Switch  Switch  Switch

Légende:
█ = Thread en exécution (travail utile)
░ = Thread en attente
▲ = Context Switch (temps perdu)
```

### Impact du nombre de threads

```
Scénario 1: 4 threads sur 4 cores
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Thread 1│ │Thread 2│ │Thread 3│ │Thread 4│
│ Core 1 │ │ Core 2 │ │ Core 3 │ │ Core 4 │
└────────┘ └────────┘ └────────┘ └────────┘
Overhead: ~0% (parfait!)

Scénario 2: 100 threads sur 4 cores
┌────────────────────────────────────────┐
│ 100 threads se partagent 4 cores      │
│ Context switching constant             │
│ Overhead: ~10-20%                      │
└────────────────────────────────────────┘

Scénario 3: 10,000 threads sur 4 cores
┌────────────────────────────────────────┐
│ 10,000 threads se battent pour 4 cores│
│ Context switching infernal             │
│ Overhead: 50-80% (catastrophique!)     │
│ CPU occupe son temps à switcher       │
└────────────────────────────────────────┘
```

---

## 1.6 Mesure réelle des coûts

### Benchmark : Création de threads

```java
public class ThreadCreationBenchmark {
    
    public static void main(String[] args) {
        int numThreads = 10_000;
        
        // Mesure du temps de création
        long start = System.nanoTime();
        
        List threads = new ArrayList<>();
        for (int i = 0; i < numThreads; i++) {
            Thread t = new Thread(() -> {
                // Ne fait rien
            });
            threads.add(t);
        }
        
        long creationTime = System.nanoTime() - start;
        System.out.println("Temps de création: " + creationTime / 1_000_000 + " ms");
        System.out.println("Par thread: " + creationTime / numThreads / 1000 + " µs");
        
        // Mesure du démarrage
        start = System.nanoTime();
        threads.forEach(Thread::start);
        threads.forEach(t -> {
            try { t.join(); } catch (InterruptedException e) {}
        });
        long executionTime = System.nanoTime() - start;
        
        System.out.println("Temps total: " + executionTime / 1_000_000 + " ms");
    }
}

/* Résultats typiques:
Temps de création: 250 ms
Par thread: 25 µs
Temps total: 500 ms

Conclusion: Créer 10,000 threads = 500ms de latence !
*/
```

### Benchmark : Consommation mémoire

```java
public class ThreadMemoryBenchmark {
    
    public static void main(String[] args) throws InterruptedException {
        Runtime runtime = Runtime.getRuntime();
        
        // Mémoire avant
        runtime.gc();
        Thread.sleep(100);
        long memoryBefore = runtime.totalMemory() - runtime.freeMemory();
        
        // Créer des threads bloqués
        List threads = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            Thread t = new Thread(() -> {
                try {
                    Thread.sleep(Long.MAX_VALUE);
                } catch (InterruptedException e) {}
            });
            t.start();
            threads.add(t);
        }
        
        Thread.sleep(1000); // Attendre que tous démarrent
        
        // Mémoire après
        long memoryAfter = runtime.totalMemory() - runtime.freeMemory();
        long memoryUsed = memoryAfter - memoryBefore;
        
        System.out.println("Mémoire utilisée: " + memoryUsed / 1024 / 1024 + " MB");
        System.out.println("Par thread: " + memoryUsed / 1000 / 1024 + " KB");
        
        // Nettoyage
        threads.forEach(Thread::interrupt);
    }
}

/* Résultats typiques:
Mémoire utilisée: 1800 MB
Par thread: ~1.8 MB

Note: Cela correspond au stack size du thread
*/
```

---

## 1.7 Les limites physiques

### Pourquoi ne peut-on pas avoir un million de threads OS ?

```
Calcul simple:
- 1 thread = 2 MB de stack
- 1,000,000 threads = 2,000,000 MB = 2 TB de RAM

Même avec 128 GB de RAM:
- Max théorique: ~64,000 threads
- Max pratique (avec overhead): ~10,000 threads

Au-delà:
- OutOfMemoryError
- Thrashing (swap constant)
- Context switching overhead > 50%
- Le système devient inutilisable
```

### Démonstration visuelle

```
Machine: 16 GB RAM, 8 CPU cores

┌───────────────────────────────────────────┐
│ Nombre de threads vs Performance         │
├───────────────────────────────────────────┤
│                                           │
│ Perf │                                    │
│  %   │     ****                           │
│ 100  │    *    *                          │
│      │   *      *                         │
│  80  │  *        *                        │
│      │ *          *                       │
│  60  │*            **                     │
│      │               ***                  │
│  40  │                  ****              │
│      │                      *****         │
│  20  │                           ******   │
│      │                                ****│
│   0  └──┬────┬────┬────┬────┬────┬───────│
│        10   100  500  1K   5K   10K      │
│           Nombre de threads               │
└───────────────────────────────────────────┘

Zone optimale: 10-100 threads (proche du nb de cores)
Zone dégradée: 500-1000 threads
Zone critique: > 5000 threads
```

---

## 1.8 Concepts clés à retenir

### ✅ Ce qu'il faut retenir

1. **Un thread OS est lourd**
   - ~2 MB de mémoire
   - Coût de création élevé
   - Géré par le kernel (pas par votre application)

2. **Le context switching a un coût**
   - Plus de threads = plus de switching
   - Au-delà d'un certain seuil, l'overhead domine

3. **Il y a une limite pratique**
   - ~1000-5000 threads max sur une machine standard
   - Au-delà, les performances s'effondrent

4. **Les threads bloqués gaspillent des ressources**
   - Un thread qui attend une I/O occupe 2 MB de RAM
   - Il ne peut rien faire d'autre pendant ce temps

### 🎯 Le problème à résoudre

> Comment gérer des **dizaines de milliers de tâches concurrentes** avec seulement quelques **centaines de threads OS** ?

**Réponse : Les Virtual Threads !** (Chapitres suivants)

---

## 1.9 Exemple pratique : Le problème concret

```java
// Serveur Web classique
@RestController
public class ApiController {
    
    @GetMapping("/data")
    public Response getData() {
        // 1. Requête DB: thread bloque 50ms
        Data data = database.query();
        
        // 2. Appel API externe: thread bloque 100ms
        ExternalData external = httpClient.get("https://api.example.com");
        
        // 3. Processing: thread actif 10ms
        Result result = process(data, external);
        
        return new Response(result);
    }
}

// Timeline pour UNE requête:
// ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████
// ▲                                                         ▲
// CPU                                                      CPU
// actif                                                   actif
// 10ms         Bloqué sur I/O: 150ms                     10ms
//
// Total: 160ms dont 150ms de BLOCAGE (93% de gaspillage!)

// Avec 200 threads max:
// - Si 200 requêtes arrivent simultanément
// - Les 200 threads sont TOUS BLOQUÉS 93% du temps
// - CPU utilization: ~7%
// - Mais on ne peut pas gérer plus de 200 requêtes simultanées!
```

---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [➡️ Suivant: JVM et Platform Threads](02-jvm-platform-threads.md)