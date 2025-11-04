[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [📖 Lexique](lexique.md) | [➡️ Suivant](02-jvm-platform-threads.md)

---

# 1. Qu'est-ce qu'un Thread ?

## 1.1 Définition

Un **thread** (ou fil d'exécution) est la plus petite unité d'exécution qu'un système d'exploitation peut gérer. C'est une séquence d'instructions qui peut être exécutée de manière indépendante.

### Un exemple par analogie (le restaurant) 🍽️

Imaginez un restaurant :
- **Le restaurant** = votre application
- **La cuisine** = le CPU
- **Les serveurs** = les threads
- **Les clients** = les requêtes/tâches

Plus vous avez de serveurs, plus vous pouvez servir de clients simultanément. Mais :
- Chaque serveur coûte cher (salaire)
- Trop de serveurs dans une petite cuisine entraîne des complications (Chaud devant !)
- Les serveurs doivent se coordonner (context switching)

---

## 1.2 Thread au niveau du système d'exploitation

### Architecture complète

```
┌─────────────────────────────────────────────────┐
│         Application (Processus)                 │
│                                                 │
│  ┌────────┐  ┌────────┐  ┌────────┐             │
│  │Thread 1│  │Thread 2│  │Thread 3│             │
│  │ Stack  │  │ Stack  │  │ Stack  │             │
│  │ ~2 MB  │  │ ~2 MB  │  │ ~2 MB  │             │
│  └───┬────┘  └───┬────┘  └───┬────┘             │
│      │           │           │                  │
└──────┼───────────┼───────────┼──────────────────┘
       │           │           │
       │  Appels système (syscalls)
       │           │           │
┌──────▼───────────▼───────────▼────────────────┐
│           Kernel Space (OS)                   │
│                                               │
│  ┌──────────────────────────────────┐         │
│  │    Thread Scheduler              │         │
│  │  - Gère les priorités            │         │
│  │  - Context switching             │         │
│  │  - Allocation CPU time           │         │
│  └──────────────────────────────────┘         │
│                                               │
│  Thread Control Blocks (TCB):                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ TCB #1  │ │ TCB #2  │ │ TCB #3  │          │
│  │ State   │ │ State   │ │ State   │          │
│  │ Priority│ │ Priority│ │ Priority│          │
│  │Registers│ │Registers│ │Registers│          │
│  └─────────┘ └─────────┘ └─────────┘          │
└──────┬───────────┬───────────┬────────────────┘
       │           │           │
┌──────▼───────────▼───────────▼──────────────────┐
│              CPU Hardware                       │
│                                                 │
│   ┌────────┐  ┌────────┐  ┌────────┐            │
│   │ Core 1 │  │ Core 2 │  │ Core 3 │            │
│   └────────┘  └────────┘  └────────┘            │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 1.3 Caractéristiques d'un Thread OS (Platform Thread)

# Caractéristiques des Threads

**Mémoire Stack**
- Valeur typique : 1-2 MB
- Espace mémoire alloué pour les variables locales et l'historique d'appels

**Temps de création**
- Valeur typique : 0.2 - 1 ms
- Temps nécessaire pour créer un nouveau thread via l'OS

**Context Switching**
- Valeur typique : 1 - 10 µs
- Temps pour sauvegarder l'état d'un thread et restaurer un autre

**Thread Control Block**
- Valeur typique : ~1.5 KB
- Métadonnées du thread (ID, priorité, registres CPU, etc.)

**Limite pratique**
- Valeur typique : 1000-5000 (Dépend de la plateforme physique)
- Nombre max de threads avant dégradation des performances

**Gestion**
- Valeur typique : OS Kernel
- C'est le système d'exploitation qui gère la vie du thread

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
3. **Vider/recharger** les caches CPU (Peut-être couteux en ressource)


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
│ 100 threads se partagent 4 cores       │
│ Context switching constant             │
│ Overhead: ~10-20%                      │
└────────────────────────────────────────┘

Scénario 3: 10,000 threads sur 4 cores
┌────────────────────────────────────────┐
│ 10,000 threads se battent pour 4 cores │
│ Context switching qui surcharge le CPU │
│ Overhead: 50-80%                       │
│ CPU en pleine limitation               │
└────────────────────────────────────────┘
```

---

## 1.6 Mesure réelle des coûts

### Benchmark : Création de threads

```java
public class ThreadCreationBenchmark {

	static void main() {
		int numThreads = 10_000;

		// Mesure du temps de création
		long start = System.nanoTime();

		List<Thread> threads = new ArrayList<>();
		for (int i = 0; i < numThreads; i++) {
			int finalI = i;
			Thread t = new Thread(() -> {
				System.out.println("Je suis le thread n°" + finalI + " et je ne sers à rien !");
			});
			threads.add(t);
		}

		long creationTime = System.nanoTime() - start;
		System.out.println("Temps de création: " + creationTime / 1_000_000 + " ms");
		System.out.println("Par thread: " + creationTime / numThreads / 1000 + " µs");

		start = System.nanoTime();
		threads.forEach(Thread::start);
		threads.forEach(t -> {
			try { t.join(); } catch (InterruptedException ex) {}
		});
		long executionTime = System.nanoTime() - start;

		System.out.println("Temps total: " + executionTime / 1_000_000 + " ms");
	}
}
```

### Benchmark : Consommation mémoire

```java
import java.util.ArrayList;
import java.util.List;

public class ThreadMemoryBenchmark {

	public static void main(String[] args) throws Exception {
		Runtime runtime = Runtime.getRuntime();

		// Forcer le GC avant de mesurer la mémoire
		for (int i = 0; i < 5; i++) {
			System.gc();
			Thread.sleep(100);
		}

		long memoryBefore = runtime.totalMemory() - runtime.freeMemory();

		List<Thread> threads = new ArrayList<>();
		int threadCount = 1000;

		for (int i = 0; i < threadCount; i++) {
			Thread t = new Thread(() -> {
				try {
					byte[] buffer = new byte[1024];
					Thread.sleep(Long.MAX_VALUE);
				} catch (InterruptedException ex) {}
			});
			t.start();
			threads.add(t);
		}

		Thread.sleep(3000);

		long memoryAfter = runtime.totalMemory() - runtime.freeMemory();
		long memoryUsed = memoryAfter - memoryBefore;

		System.out.println("Threads créés: " + threadCount);
		System.out.println("Mémoire utilisée: " + memoryUsed / 1024 / 1024 + " MB");
		System.out.println("Par thread: " + memoryUsed / threadCount / 1024 + " KB");

		System.out.println("\nNote: Cette mesure inclut seulement la mémoire allouée dans la HEAP. Le stack natif (~1MB/thread)");
		System.out.println("n'est pas visible via l'API Java standard. La mesure n'est pas démontrable précisément.");

		threads.forEach(Thread::interrupt);
	}
}
```

---

## 1.7 Les limites physiques

### Pourquoi ne peut-on pas avoir un million de threads OS ?

Logiquement, on pourrait se dire que la limite matérielle peut facilement être dépassée avec les technologies modernes ?
Essayons de comprendre pourquoi c'est faux.

```
Calcul simple:
- 1 thread = 2 MB de stack
- 1,000,000 threads = 2,000,000 MB = 2 TB de RAM : 
    Avant même de parler de puissance de calcul CPU, la RAM pose problème. 
    La RAM est un des éléments matériel les plus volatiles en terme de prix sur le marché, les prix varies du simple au double sur un trimestre. 

Même avec 128 GB de RAM:
- Max théorique: ~64,000 threads
- Max pratique (avec overhead): ~10,000 threads

Au-delà:
- OutOfMemoryError
- Thrashing (swap constant)
- Context switching overhead > 50%
- Le système devient inutilisable
```
---

## 1.8 Concepts clés à retenir

### ✅ Ce qu'il faut retenir

1. **Un thread OS est lourd**
   - ~2 MB de mémoire
   - Coût de création élevé
   - Géré par le kernel (donc par la machine hôte)

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
---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [➡️ Suivant: JVM et Platform Threads](02-jvm-platform-threads.md)