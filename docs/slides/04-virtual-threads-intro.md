[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [📖 Lexique](lexique.md) | [⬅️ Précédent](03-limitations-jvm.md) | [➡️ Suivant](05-avant-apres.md)

---

# 4. La Solution : Virtual Threads (Project Loom)

## 4.1 Qu'est-ce qu'un Virtual Thread ?

### Définition

Un **Virtual Thread** est un thread Java léger géré entièrement par la **JVM** (et non par l'OS), qui est automatiquement monté sur un **Platform Thread** (carrier) pour s'exécuter.

```
Analogie du taxi:

Platform Threads = Taxis (ressource limitée, coûteuse)
Virtual Threads = Passagers (peut être des milliers)
Carrier Threads = Taxis disponibles à la station

┌────────────────────────────────────────────────┐
│  1 Million de Virtual Threads (passagers)      │
│                                                │
│  VT-1  VT-2  VT-3 ... VT-999999  VT-1000000   │
│    ↓    ↓     ↓                                │
│    └────┴─────┴─→ Attendent un carrier        │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│  Pool de Carrier Threads (taxis)               │
│  = Platform Threads                            │
│                                                │
│  [Carrier-1] [Carrier-2] ... [Carrier-N]      │
│      ↑           ↑              ↑              │
│   VT-1 monté  VT-5 monté   VT-42 monté        │
│   (en cours)  (en cours)   (en cours)          │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│  OS Threads (vraies ressources OS)             │
│  pthread-1   pthread-2  ...  pthread-N         │
└────────────────────────────────────────────────┘
                    ↓
┌────────────────────────────────────────────────┐
│  CPU Cores                                     │
│  Core-1      Core-2     ...   Core-N           │
└────────────────────────────────────────────────┘

Nombre de Carriers (N) ≈ Nombre de CPU cores
```

### Comparaison Platform vs Virtual

```
┌──────────────────────────────────────────────────────┐
│         Platform Thread vs Virtual Thread            │
├────────────────────┬─────────────────────────────────┤
│ Platform Thread    │ Virtual Thread                  │
├────────────────────┼─────────────────────────────────┤
│ Géré par l'OS      │ Géré par la JVM                 │
│ 2 MB de stack      │ ~1 KB (grandit si besoin)       │
│ Coût création: ~1ms│ Coût création: ~1µs (1000× ↓)  │
│ Max: ~5,000        │ Max: Des millions               │
│ 1:1 avec OS thread │ N:M (plusieurs VT → 1 carrier)  │
│ Blocking = coûteux │ Blocking = gratuit (unmount)    │
│ Schedulé par l'OS  │ Schedulé par la JVM             │
└────────────────────┴─────────────────────────────────┘
```

---

## 4.2 Le concept de Mounting/Unmounting

### Comment ça marche ?

```
Cycle de vie d'un Virtual Thread:

1. CRÉATION
┌──────────────────────────────────┐
│ Thread.startVirtualThread(() -> │
│     doWork();                    │
│ })                               │
└──────────────────────────────────┘
         │
         ▼
    [VT créé en mémoire JVM]
    ~1 KB, en quelques microsecondes


2. ATTENTE D'UN CARRIER
┌──────────────────────────────────┐
│ VT est dans la queue             │
│ (pas encore sur un carrier)      │
└──────────────────────────────────┘
         │
         ▼
    [VT attend carrier disponible]


3. MOUNTING (montage)
┌──────────────────────────────────┐
│ Carrier Thread devient disponible│
│ VT est MONTÉ sur le carrier      │
└──────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  [Carrier Thread]                │
│       ↑                          │
│       │ Stack VT copié           │
│       │                          │
│    [VT-123]                      │
│   S'exécute                      │
└──────────────────────────────────┘


4. BLOCKING I/O DÉTECTÉ
┌──────────────────────────────────┐
│ VT fait: Thread.sleep()          │
│ ou socket.read()                 │
│ ou JDBC query                    │
└──────────────────────────────────┘
         │
         ▼
    ⚡ UNMOUNTING automatique!


5. UNMOUNTING (démontage)
┌──────────────────────────────────┐
│ JVM détecte le blocage           │
│ → Sauvegarde stack du VT         │
│ → VT mis en "parking"            │
│ → Carrier libéré                 │
└──────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  [Carrier Thread] LIBRE!         │
│                                  │
│  Peut prendre un autre VT        │
└──────────────────────────────────┘
    
    [VT-123 en parking]
    (attend la fin de l'I/O)


6. I/O TERMINÉE
┌──────────────────────────────────┐
│ Socket read() retourne           │
│ ou sleep() terminé               │
└──────────────────────────────────┘
         │
         ▼
    [VT prêt à reprendre]
    → Retour à l'étape 2


7. REMOUNTING
┌──────────────────────────────────┐
│ VT obtient un carrier            │
│ (peut être différent!)           │
│ Stack restauré                   │
│ Exécution reprend                │
└──────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  [Carrier Thread-5]              │
│       ↑                          │
│    [VT-123]                      │
│   Continue l'exécution           │
└──────────────────────────────────┘


8. TERMINAISON
┌──────────────────────────────────┐
│ run() se termine                 │
│ VT libéré de la mémoire          │
│ Carrier redevient disponible     │
└──────────────────────────────────┘
```

### Exemple concret avec timeline

```java
public class MountingUnmountingDemo {
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread.startVirtualThread(() -> {
            System.out.println("[1] VT démarré sur: " + 
                Thread.currentThread());
            
            try {
                // Phase CPU-bound
                System.out.println("[2] Calcul en cours...");
                long sum = 0;
                for (long i = 0; i < 1_000_000; i++) {
                    sum += i;
                }
                System.out.println("[3] Résultat: " + sum + 
                    " sur " + Thread.currentThread());
                
                // ⚡ UNMOUNTING ici!
                System.out.println("[4] Début sleep (VT va se démonter)");
                Thread.sleep(100);
                
                // ⚡ REMOUNTING ici!
                System.out.println("[5] Après sleep (VT remonté) sur: " + 
                    Thread.currentThread());
                
                // Phase CPU-bound
                System.out.println("[6] Autre calcul...");
                sum = 0;
                for (long i = 0; i < 1_000_000; i++) {
                    sum += i;
                }
                System.out.println("[7] Terminé sur: " + 
                    Thread.currentThread());
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        
        Thread.sleep(500);
    }
}

/* Output typique:

[1] VT démarré sur: VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
[2] Calcul en cours...
[3] Résultat: 499999500000 sur VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
[4] Début sleep (VT va se démonter)
[5] Après sleep (VT remonté) sur: VirtualThread[#21]/runnable@ForkJoinPool-1-worker-3
                                                                        ↑
                                                                    Carrier différent!
[6] Autre calcul...
[7] Terminé sur: VirtualThread[#21]/runnable@ForkJoinPool-1-worker-3

Observations:
• VT-21 commence sur worker-1
• Pendant sleep(): VT se démonte automatiquement
• Après sleep(): VT remonte sur worker-3 (différent!)
• Le VT "ne sait pas" qu'il a changé de carrier
• Tout est transparent pour le code
*/
```

### Timeline détaillée

```
Timeline de VT-21 sur 200ms:

Carrier-1: [████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████]
            ▲       ▲                              ▲
            │       │                              │
           VT-21   VT-21                         Autre VT
           monte   démonte                       monte
                   (sleep)

VT-21:     [████████]─────[parking 100ms]─────[████████]
            0-10ms        10-110ms             110-120ms
            Calcul        DÉMONTÉ              Calcul
            sur C-1       (attend)             sur C-3

Carrier-3: [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████]
                                                ▲
                                                │
                                              VT-21
                                              remonte

Légende:
█ = Carrier actif avec VT monté
░ = Carrier libre ou avec autre VT
─ = VT en parking (démonté)

Pendant les 100ms de sleep():
• VT-21 ne consomme AUCUN carrier
• Carrier-1 peut exécuter d'autres VT
• VT-21 occupe ~1KB en mémoire (juste sa stack)
• Pas de thread OS bloqué!
```

---

## 4.3 Architecture interne

### ForkJoinPool : Le scheduler des Virtual Threads

```
┌─────────────────────────────────────────────────────┐
│              Virtual Thread Scheduler               │
│            (ForkJoinPool en mode FIFO)              │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Submission Queue (tous les VT prêts)               │
│  ┌───────────────────────────────────────────┐     │
│  │ [VT-1] [VT-2] [VT-3] ... [VT-999999]     │     │
│  └───────────────────────────────────────────┘     │
│              ↓         ↓         ↓                  │
│                                                     │
│  Worker Threads (Carriers)                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│  │ Worker-1   │  │ Worker-2   │  │ Worker-N   │  │
│  │            │  │            │  │            │  │
│  │ VT-1 monté │  │ VT-5 monté │  │VT-42 monté │  │
│  │ exécution  │  │ exécution  │  │ exécution  │  │
│  └────────────┘  └────────────┘  └────────────┘  │
│                                                     │
│  Parked Virtual Threads (démontés, en attente I/O) │
│  ┌───────────────────────────────────────────┐     │
│  │ [VT-4] [VT-7] [VT-12] ... [VT-88888]     │     │
│  │ (sleep, socket.read, JDBC, etc.)          │     │
│  └───────────────────────────────────────────┘     │
│              ↑                                      │
│              └─ Réveillés quand I/O terminée       │
│                 → retournent dans Submission Queue  │
│                                                     │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│            Platform Threads (OS)                    │
│  [pthread-1]  [pthread-2]  ...  [pthread-N]        │
└─────────────────────────────────────────────────────┘
```

### Configuration du scheduler

```java
public class VirtualThreadSchedulerConfig {
    
    public static void main(String[] args) {
        
        // Propriétés système pour configurer le scheduler
        
        // 1. Parallelism : nombre de carrier threads
        // Par défaut: Runtime.getRuntime().availableProcessors()
        System.setProperty("jdk.virtualThreadScheduler.parallelism", "8");
        
        // 2. MaxPoolSize : pool max si carriers pinnés
        // Par défaut: 256
        System.setProperty("jdk.virtualThreadScheduler.maxPoolSize", "256");
        
        // 3. MinRunnable : nombre min de carriers actifs
        // Par défaut: 1
        System.setProperty("jdk.virtualThreadScheduler.minRunnable", "1");
        
        // Afficher la configuration
        System.out.println("Configuration Virtual Thread Scheduler:");
        System.out.println("  Parallelism: " + 
            System.getProperty("jdk.virtualThreadScheduler.parallelism"));
        System.out.println("  MaxPoolSize: " + 
            System.getProperty("jdk.virtualThreadScheduler.maxPoolSize"));
        System.out.println("  CPUs disponibles: " + 
            Runtime.getRuntime().availableProcessors());
    }
}

/* Configuration recommandée:

En développement (laptop):
• parallelism: non défini (auto = nb CPUs)
• maxPoolSize: 256 (défaut OK)

En production (server):
• parallelism: non défini (auto)
  → JVM détecte nb CPUs du container
• maxPoolSize: 256-512
  → Au cas où certains VT sont pinnés

Machine 8 cores:
• 8 carrier threads actifs
• Peut gérer des millions de VT
• Les VT se partagent les 8 carriers
*/
```

---

## 4.4 Caractéristiques techniques

### Poids mémoire

```java
public class VirtualThreadMemoryDemo {
    
    public static void main(String[] args) throws InterruptedException {
        
        Runtime runtime = Runtime.getRuntime();
        
        // Mémoire avant
        System.gc();
        Thread.sleep(100);
        long memBefore = runtime.totalMemory() - runtime.freeMemory();
        
        System.out.println("=== Création de 100,000 Virtual Threads ===\n");
        System.out.println("Mémoire avant: " + memBefore / 1024 / 1024 + " MB");
        
        List<Thread> threads = new ArrayList<>();
        
        long start = System.currentTimeMillis();
        
        // Créer 100,000 Virtual Threads
        for (int i = 0; i < 100_000; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                try {
                    Thread.sleep(10_000); // Dormir 10 secondes
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            threads.add(vt);
        }
        
        long creationTime = System.currentTimeMillis() - start;
        
        Thread.sleep(500); // Laisser les threads se stabiliser
        
        // Mémoire après
        long memAfter = runtime.totalMemory() - runtime.freeMemory();
        long memUsed = memAfter - memBefore;
        
        System.out.println("\n=== Résultats ===");
        System.out.println("Temps de création: " + creationTime + " ms");
        System.out.println("Temps par thread: " + 
            (creationTime * 1000.0 / 100_000) + " µs");
        System.out.println("\nMémoire après: " + memAfter / 1024 / 1024 + " MB");
        System.out.println("Mémoire utilisée: " + memUsed / 1024 / 1024 + " MB");
        System.out.println("Mémoire par VT: " + (memUsed / 100_000) + " bytes ≈ " +
            (memUsed / 100_000 / 1024.0) + " KB");
        
        System.out.println("\n=== Comparaison ===");
        System.out.println("Platform Threads (100,000): ~200 GB (IMPOSSIBLE!)");
        System.out.println("Virtual Threads (100,000): ~" + 
            memUsed / 1024 / 1024 + " MB (POSSIBLE!)");
        
        // Nettoyer
        threads.forEach(Thread::interrupt);
    }
}

/* Output typique:

=== Création de 100,000 Virtual Threads ===

Mémoire avant: 45 MB

=== Résultats ===
Temps de création: 156 ms
Temps par thread: 1.56 µs

Mémoire après: 195 MB
Mémoire utilisée: 150 MB
Mémoire par VT: 1536 bytes ≈ 1.5 KB

=== Comparaison ===
Platform Threads (100,000): ~200 GB (IMPOSSIBLE!)
Virtual Threads (100,000): ~150 MB (POSSIBLE!)

Observations:
• 100,000 VT créés en 156ms (Platform: plusieurs minutes!)
• ~1.5 KB par VT (Platform: 2 MB)
• Ratio: 1300× moins de mémoire!
• Sur une machine 16 GB: peut créer ~10 MILLIONS de VT
*/
```

### Performance de création

```java
public class VirtualThreadCreationBenchmark {
    
    public static void main(String[] args) throws InterruptedException {
        
        int numThreads = 100_000;
        
        System.out.println("=== Benchmark Création de Threads ===\n");
        
        // Benchmark Platform Threads (limité à 10,000 pour éviter OOM)
        System.out.println("1. Platform Threads (10,000):");
        benchmarkPlatformThreads(10_000);
        
        // Benchmark Virtual Threads
        System.out.println("\n2. Virtual Threads (100,000):");
        benchmarkVirtualThreads(numThreads);
        
        System.out.println("\n=== Conclusion ===");
        System.out.println("Virtual Threads:");
        System.out.println("• 1000× plus rapides à créer");
        System.out.println("• 1000× moins de mémoire");
        System.out.println("• Peuvent être 10× plus nombreux");
    }
    
    private static void benchmarkPlatformThreads(int num) 
            throws InterruptedException {
        
        long start = System.nanoTime();
        
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < num; i++) {
            Thread t = new Thread(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
            });
            t.start();
            threads.add(t);
        }
        
        long creationTime = (System.nanoTime() - start) / 1_000_000;
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("  Temps création: " + creationTime + " ms");
        System.out.println("  Par thread: " + 
            (creationTime * 1000.0 / num) + " µs");
    }
    
    private static void benchmarkVirtualThreads(int num) 
            throws InterruptedException {
        
        long start = System.nanoTime();
        
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < num; i++) {
            Thread t = Thread.startVirtualThread(() -> {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}
            });
            threads.add(t);
        }
        
        long creationTime = (System.nanoTime() - start) / 1_000_000;
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("  Temps création: " + creationTime + " ms");
        System.out.println("  Par thread: " + 
            (creationTime * 1000.0 / num) + " µs");
    }
}

/* Output typique:

=== Benchmark Création de Threads ===

1. Platform Threads (10,000):
  Temps création: 4850 ms
  Par thread: 485.0 µs

2. Virtual Threads (100,000):
  Temps création: 180 ms
  Par thread: 1.8 µs

=== Conclusion ===
Virtual Threads:
• 1000× plus rapides à créer (485µs vs 1.8µs)
• 1000× moins de mémoire
• Peuvent être 10× plus nombreux

Note: 100,000 VT créés en 180ms
     100,000 Platform Threads = IMPOSSIBLE (OOM)
*/
```

---

## 4.5 API Java pour Virtual Threads

### Création de Virtual Threads

```java
public class VirtualThreadCreationAPI {
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== 5 façons de créer des Virtual Threads ===\n");
        
        // 1. Thread.startVirtualThread() - Le plus simple
        System.out.println("1. Thread.startVirtualThread()");
        Thread vt1 = Thread.startVirtualThread(() -> {
            System.out.println("  VT créé avec startVirtualThread()");
        });
        vt1.join();
        
        // 2. Thread.ofVirtual().start() - Plus de contrôle
        System.out.println("\n2. Thread.ofVirtual().start()");
        Thread vt2 = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> {
                System.out.println("  VT nommé: " + 
                    Thread.currentThread().getName());
            });
        vt2.join();
        
        // 3. Thread.ofVirtual().unstarted() - Démarrage manuel
        System.out.println("\n3. Thread.ofVirtual().unstarted()");
        Thread vt3 = Thread.ofVirtual()
            .name("unstarted-vt")
            .unstarted(() -> {
                System.out.println("  VT démarré manuellement");
            });
        System.out.println("  État avant start(): " + vt3.getState());
        vt3.start();
        vt3.join();
        
        // 4. Executors.newVirtualThreadPerTaskExecutor()
        System.out.println("\n4. ExecutorService avec VT");
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                System.out.println("  VT via ExecutorService");
            }).get();
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        // 5. ThreadFactory personnalisé
        System.out.println("\n5. ThreadFactory personnalisé");
        ThreadFactory factory = Thread.ofVirtual()
            .name("custom-vt-", 0)
            .factory();
        
        Thread vt5 = factory.newThread(() -> {
            System.out.println("  VT: " + Thread.currentThread().getName());
        });
        vt5.start();
        vt5.join();
        
        System.out.println("\n=== Vérification ===");
        System.out.println("Tous les threads étaient virtuels: " +
            Stream.of(vt1, vt2, vt3, vt5)
                .allMatch(Thread::isVirtual));
    }
}

/* Output:

=== 5 façons de créer des Virtual Threads ===

1. Thread.startVirtualThread()
  VT créé avec startVirtualThread()

2. Thread.ofVirtual().start()
  VT nommé: my-virtual-thread

3. Thread.ofVirtual().unstarted()
  État avant start(): NEW
  VT démarré manuellement

4. ExecutorService avec VT
  VT via ExecutorService

5. ThreadFactory personnalisé
  VT: custom-vt-0

=== Vérification ===
Tous les threads étaient virtuels: true
*/
```

### Vérifier si un thread est virtuel

```java
public class VirtualThreadDetection {
    
    public static void main(String[] args) throws InterruptedException {
        
        // Platform Thread (thread actuel = main)
        System.out.println("Thread main:");
        printThreadInfo(Thread.currentThread());
        
        // Créer un Platform Thread
        Thread platformThread = new Thread(() -> {
            System.out.println("\nPlatform Thread:");
            printThreadInfo(Thread.currentThread());
        });
        platformThread.start();
        platformThread.join();
        
        // Créer un Virtual Thread
        Thread.startVirtualThread(() -> {
            System.out.println("\nVirtual Thread:");
            printThreadInfo(Thread.currentThread());
        }).join();
    }
    
    private static void printThreadInfo(Thread thread) {
        System.out.println("  Nom: " + thread.getName());
        System.out.println("  ID: " + thread.threadId());
        System.out.println("  isVirtual(): " + thread.isVirtual());
        System.out.println("  isDaemon(): " + thread.isDaemon());
        System.out.println("  Priority: " + thread.getPriority());
        System.out.println("  ThreadGroup: " + thread.getThreadGroup());
        System.out.println("  toString(): " + thread);
    }
}

/* Output:

Thread main:
  Nom: main
  ID: 1
  isVirtual(): false
  isDaemon(): false
  Priority: 5
  ThreadGroup: java.lang.ThreadGroup[name=main,maxpri=10]
  toString(): Thread[#1,main,5,main]

Platform Thread:
  Nom: Thread-0
  ID: 21
  isVirtual(): false
  isDaemon(): false
  Priority: 5
  ThreadGroup: java.lang.ThreadGroup[name=main,maxpri=10]
  toString(): Thread[#21,Thread-0,5,main]

Virtual Thread:
  Nom: <empty>
  ID: 22
  isVirtual(): true
  isDaemon(): true
  isDaemon(): true  ← Toujours daemon!
  Priority: 5
  ThreadGroup: null  ← Pas de ThreadGroup!
  toString(): VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1

Différences clés:
• isVirtual() = true
• Toujours daemon (pas besoin de setDaemon())
• Pas de ThreadGroup (obsolète avec VT)
• toString() indique "VirtualThread" et le carrier
*/
```

---

## 4.6 Blocking gratuit : La magie

### Démonstration du unmounting automatique

```java
import java.util.concurrent.atomic.AtomicInteger;

public class BlockingIsFreeDemo {
    
    private static final AtomicInteger activeCarriers = new AtomicInteger(0);
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== Démonstration: Blocking is Free ===\n");
        System.out.println("CPUs disponibles: " + 
            Runtime.getRuntime().availableProcessors());
        System.out.println("Carriers attendus: ~" + 
            Runtime.getRuntime().availableProcessors());
        
        // Lancer 10,000 Virtual Threads qui bloquent
        System.out.println("\nLancement de 10,000 VT qui dorment 5 secondes...\n");
        
        List<Thread> threads = new ArrayList<>();
        long start = System.currentTimeMillis();
        
        for (int i = 0; i < 10_000; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                try {
                    // ⚡ Ce VT va se DÉMONTER pendant le sleep
                    Thread.sleep(5000);
                    
                    // Après réveil: remonté sur un carrier
                    System.out.print(".");
                    
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
            threads.add(vt);
        }
        
        long launchTime = System.currentTimeMillis() - start;
        System.out.println("Temps de lancement: " + launchTime + " ms");
        
        // Attendre tous les threads
        System.out.println("\nEn attente de tous les VT...");
        for (Thread t : threads) {
            t.join();
        }
        
        long totalTime = System.currentTimeMillis() - start;
        
        System.out.println("\n\n=== Résultats ===");
        System.out.println("Temps total: " + totalTime + " ms");
        System.out.println("10,000 VT dormant 5 secondes chacun");
        System.out.println("Temps théorique (séquentiel): 50,000 secondes");
        System.out.println("Temps réel: ~5 secondes");
        System.out.println("\n⚡ Les 10,000 VT ont dormi EN PARALLÈLE!");
        System.out.println("⚡ Utilisant seulement ~" + 
            Runtime.getRuntime().availableProcessors() + " carriers!");
        
        System.out.println("\nAvec Platform Threads:");
        System.out.println("• 10,000 threads = 20 GB de mémoire → IMPOSSIBLE");
        System.out.println("• Context switching → Performance catastrophique");
        
        System.out.println("\nAvec Virtual Threads:");
        System.out.println("• 10,000 VT = ~15 MB de mémoire → FACILE");
        System.out.println("• Pendant sleep(): VT démontés → carriers libres");
        System.out.println("• Zéro gaspillage de ressources!");
    }
}

/* Output typique:

=== Démonstration: Blocking is Free ===

CPUs disponibles: 8
Carriers attendus: ~8

Lancement de 10,000 VT qui dorment 5 secondes...

Temps de lancement: 125 ms
.............................................................
(10,000 points affichés pendant ~5 secondes)

=== Résultats ===
Temps total: 5156 ms
10,000 VT dormant 5 secondes chacun
Temps théorique (séquentiel): 50,000 secondes
Temps réel: ~5 secondes

⚡ Les 10,000 VT ont dormi EN PARALLÈLE!
⚡ Utilisant seulement ~8 carriers!

Avec Platform Threads:
• 10,000 threads = 20 GB de mémoire → IMPOSSIBLE
• Context switching → Performance catastrophique

Avec Virtual Threads:
• 10,000 VT = ~15 MB de mémoire → FACILE
• Pendant sleep(): VT démontés → carriers libres
• Zéro gaspillage de ressources!

Explication:
1. 10,000 VT lancés rapidement (~125ms)
2. Tous appellent Thread.sleep(5000)
3. JVM détecte le blocage → démonte les 10,000 VT
4. Les ~8 carriers restent LIBRES
5. Après 5 secondes: 10,000 VT se réveillent
6. JVM les remonte progressivement sur les 8 carriers
7. Temps total: ~5 secondes (parallélisme parfait!)
*/
```

### Comparaison avec Platform Threads

```java
public class PlatformVsVirtualBlocking {
    
    public static void main(String[] args) throws Exception {
        
        int numTasks = 1000;
        
        System.out.println("=== Test: 1000 tâches avec I/O bloquant ===\n");
        
        // Test 1: Platform Threads (pool de 50)
        System.out.println("1. Platform Threads (pool de 50):");
        long platformTime = testPlatformThreads(numTasks, 50);
        
        // Test 2: Virtual Threads
        System.out.println("\n2. Virtual Threads:");
        long virtualTime = testVirtualThreads(numTasks);
        
        // Comparaison
        System.out.println("\n=== Comparaison ===");
        System.out.println("Platform Threads: " + platformTime + " ms");
        System.out.println("Virtual Threads:  " + virtualTime + " ms");
        System.out.println("Speedup: " + 
            String.format("%.1f", (double) platformTime / virtualTime) + "×");
    }
    
    private static long testPlatformThreads(int numTasks, int poolSize) 
            throws Exception {
        
        ExecutorService executor = Executors.newFixedThreadPool(poolSize);
        
        long start = System.currentTimeMillis();
        
        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < numTasks; i++) {
            futures.add(executor.submit(() -> {
                try {
                    // Simulation I/O bloquant
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }));
        }
        
        for (Future<?> future : futures) {
            future.get();
        }
        
        long duration = System.currentTimeMillis() - start;
        
        executor.shutdown();
        
        System.out.println("  Pool size: " + poolSize);
        System.out.println("  Durée: " + duration + " ms");
        System.out.println("  Throughput: " + 
            String.format("%.0f", numTasks * 1000.0 / duration) + " tasks/sec");
        
        return duration;
    }
    
    private static long testVirtualThreads(int numTasks) throws Exception {
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            long start = System.currentTimeMillis();
            
            List<Future<?>> futures = new ArrayList<>();
            for (int i = 0; i < numTasks; i++) {
                futures.add(executor.submit(() -> {
                    try {
                        // Simulation I/O bloquant
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }));
            }
            
            for (Future<?> future : futures) {
                future.get();
            }
            
            long duration = System.currentTimeMillis() - start;
            
            System.out.println("  Carriers: ~" + 
                Runtime.getRuntime().availableProcessors());
            System.out.println("  Durée: " + duration + " ms");
            System.out.println("  Throughput: " + 
                String.format("%.0f", numTasks * 1000.0 / duration) + " tasks/sec");
            
            return duration;
        }
    }
}

/* Output typique (machine 8 cores):

=== Test: 1000 tâches avec I/O bloquant ===

1. Platform Threads (pool de 50):
  Pool size: 50
  Durée: 2050 ms
  Throughput: 488 tasks/sec

2. Virtual Threads:
  Carriers: ~8
  Durée: 125 ms
  Throughput: 8000 tasks/sec

=== Comparaison ===
Platform Threads: 2050 ms
Virtual Threads:  125 ms
Speedup: 16.4×

Analyse:
• Platform Threads: 50 tâches parallèles max
  → 1000 / 50 = 20 "rounds" × 100ms = 2000ms
• Virtual Threads: 1000 tâches vraiment parallèles
  → Toutes en même temps = 100ms + overhead
• Speedup: 16× plus rapide!
• Avec seulement 8 carriers vs 50 platform threads!
*/
```

---

## 4.7 Ce qui change et ce qui ne change pas

### Ce qui NE change PAS

```java
public class WhatDoesntChange {
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== Ce qui reste identique ===\n");
        
        Thread.startVirtualThread(() -> {
            
            // 1. Thread.currentThread() fonctionne
            System.out.println("1. currentThread(): " + 
                Thread.currentThread().getName());
            
            // 2. Thread.sleep() fonctionne
            try {
                Thread.sleep(10);
                System.out.println("2. sleep() OK");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            // 3. ThreadLocal fonctionne
            ThreadLocal<String> threadLocal = ThreadLocal.withInitial(() -> "value");
            System.out.println("3. ThreadLocal: " + threadLocal.get());
            
            // 4. Exception handling normal
            try {
                throw new RuntimeException("Test");
            } catch (RuntimeException e) {
                System.out.println("4. Exceptions: OK");
            }
            
            // 5. synchronized fonctionne (mais attention!)
            Object lock = new Object();
            synchronized (lock) {
                System.out.println("5. synchronized: OK");
            }
            
            // 6. wait/notify fonctionnent
            synchronized (lock) {
                try {
                    lock.wait(10);
                    System.out.println("6. wait/notify: OK");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
            
            // 7. InterruptedException normale
            System.out.println("7. Interruption: OK");
            
            // 8. Stack traces normales
            StackTraceElement[] stack = Thread.currentThread().getStackTrace();
            System.out.println("8. Stack trace: " + stack.length + " frames");
            
        }).join();
        
        System.out.println("\n✅ Tout le code existant fonctionne tel quel!");
        System.out.println("✅ Compatibilité 100% avec APIs Java existantes");
    }
}

/* Output:

=== Ce qui reste identique ===

1. currentThread(): VirtualThread[#21]
2. sleep() OK
3. ThreadLocal: value
4. Exceptions: OK
5. synchronized: OK
6. wait/notify: OK
7. Interruption: OK
8. Stack trace: 9 frames

✅ Tout le code existant fonctionne tel quel!
✅ Compatibilité 100% avec APIs Java existantes
*/
```

### Ce qui CHANGE (différences importantes)

```java
public class WhatChanges {
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== Ce qui change avec Virtual Threads ===\n");
        
        Thread vt = Thread.startVirtualThread(() -> {
            System.out.println("Virtual Thread en cours...");
        });
        vt.join();
        
        // 1. Toujours daemon
        System.out.println("1. isDaemon(): " + vt.isDaemon());
        System.out.println("   → Les VT sont TOUJOURS daemon");
        System.out.println("   → JVM peut terminer même si VT en cours");
        
        // 2. Pas de ThreadGroup
        System.out.println("\n2. ThreadGroup: " + vt.getThreadGroup());
        System.out.println("   → null (ThreadGroup obsolète)");
        
        // 3. Priority ignorée
        Thread vt2 = Thread.ofVirtual()
            .name("test")
            .priority(Thread.MAX_PRIORITY)
            .unstarted(() -> {});
        System.out.println("\n3. Priority: " + vt2.getPriority());
        System.out.println("   → Toujours 5 (normal)");
        System.out.println("   → setPriority() ignoré silencieusement");
        
        // 4. Pas de stop(), suspend(), resume()
        System.out.println("\n4. Méthodes obsolètes:");
        System.out.println("   → stop(), suspend(), resume() levèent");
        System.out.println("     UnsupportedOperationException");
        
        // 5. Stack size non configurable
        System.out.println("\n5. Stack:");
        System.out.println("   → Grandit dynamiquement (pas de -Xss)");
        System.out.println("   → Commence petit (~1KB)");
        
        // 6. Création ultra-rapide
        System.out.println("\n6. Création:");
        System.out.println("   → 1000× plus rapide que Platform Thread");
        System.out.println("   → OK pour short-lived tasks");
        
        // 7. Blocking is free
        System.out.println("\n7. Blocking I/O:");
        System.out.println("   → Ne consomme pas de carrier pendant blocage");
        System.out.println("   → Peut avoir des millions de VT bloqués");
    }
}

/* Output:

=== Ce qui change avec Virtual Threads ===

1. isDaemon(): true
   → Les VT sont TOUJOURS daemon
   → JVM peut terminer même si VT en cours

2. ThreadGroup: null
   → null (ThreadGroup obsolète)

3. Priority: 5
   → Toujours 5 (normal)
   → setPriority() ignoré silencieusement

4. Méthodes obsolètes:
   → stop(), suspend(), resume() levèent
     UnsupportedOperationException

5. Stack:
   → Grandit dynamiquement (pas de -Xss)
   → Commence petit (~1KB)

6. Création:
   → 1000× plus rapide que Platform Thread
   → OK pour short-lived tasks

7. Blocking I/O:
   → Ne consomme pas de carrier pendant blocage
   → Peut avoir des millions de VT bloqués
*/
```

---

## 4.8 Pinned Threads : Le seul piège

### Qu'est-ce qu'un pinned thread ?

```
Pinned Thread = Virtual Thread qui NE PEUT PAS se démonter

Situations qui "épinglent" un VT au carrier:

1. Bloc synchronized
2. Méthode native (JNI call)

┌────────────────────────────────────────────────┐
│ Scénario: VT épinglé (pinned)                  │
├────────────────────────────────────────────────┤
│                                                │
│  Thread.startVirtualThread(() -> {             │
│      synchronized (lock) {                     │
│          // ⚠️ VT ÉPINGLÉ au carrier ici!      │
│          socket.read();  // Blocking I/O       │
│          // Le carrier reste BLOQUÉ            │
│      }                                         │
│  });                                           │
│                                                │
└────────────────────────────────────────────────┘

Timeline:
Carrier: [████████████████BLOQUÉ████████████████]
          ▲                                     ▲
          │                                     │
       VT monté                            VT toujours
       + synchronized                      sur le carrier
       
Le carrier ne peut PAS prendre un autre VT!
```

### Démonstration du problème

```java
import java.util.concurrent.locks.ReentrantLock;

public class PinnedThreadDemo {
    
    private static final Object syncLock = new Object();
    private static final ReentrantLock reentrantLock = new ReentrantLock();
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== Démonstration Pinned Threads ===\n");
        
        int numThreads = 1000;
        
        // Test 1: Avec synchronized (PINNED)
        System.out.println("1. Avec synchronized (pinned):");
        long pinnedTime = testPinned(numThreads);
        
        // Test 2: Avec ReentrantLock (PAS PINNED)
        System.out.println("\n2. Avec ReentrantLock (non-pinned):");
        long unpinnedTime = testUnpinned(numThreads);
        
        // Comparaison
        System.out.println("\n=== Résultats ===");
        System.out.println("Avec synchronized: " + pinnedTime + " ms");
        System.out.println("Avec ReentrantLock: " + unpinnedTime + " ms");
        
        if (pinnedTime > unpinnedTime * 1.5) {
            System.out.println("\n⚠️  synchronized cause du pinning!");
            System.out.println("Performance dégradée de " + 
                String.format("%.0f", (pinnedTime / (double) unpinnedTime - 1) * 100) + "%");
        }
    }
    
    private static long testPinned(int numThreads) throws InterruptedException {
        
        long start = System.currentTimeMillis();
        
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < numThreads; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                synchronized (syncLock) {
                    // ⚠️ VT épinglé pendant ce bloc
                    try {
                        // Simulation I/O bloquant
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
            threads.add(vt);
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        long duration = System.currentTimeMillis() - start;
        System.out.println("  Durée: " + duration + " ms");
        
        return duration;
    }
    
    private static long testUnpinned(int numThreads) throws InterruptedException {
        
        long start = System.currentTimeMillis();
        
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < numThreads; i++) {
            Thread vt = Thread.startVirtualThread(() -> {
                reentrantLock.lock();
                try {
                    // ✅ VT peut se démonter ici
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    reentrantLock.unlock();
                }
            });
            threads.add(vt);
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        long duration = System.currentTimeMillis() - start;
        System.out.println("  Durée: " + duration + " ms");
        
        return duration;
    }
}

/* Output typique:

=== Démonstration Pinned Threads ===

1. Avec synchronized (pinned):
  Durée: 850 ms

2. Avec ReentrantLock (non-pinned):
  Durée: 145 ms

=== Résultats ===
Avec synchronized: 850 ms
Avec ReentrantLock: 145 ms

⚠️  synchronized cause du pinning!
Performance dégradée de 486%

Explication:
• Avec synchronized: VT épinglés → carriers bloqués
• Avec ReentrantLock: VT se démontent → carriers libres
• Ratio: ~6× plus lent avec synchronized!
*/
```

### Solution : Éviter le pinning

```java
public class AvoidPinning {
    
    public static void main(String[] args) {
        
        System.out.println("=== Comment éviter le pinning ===\n");
        
        // ❌ MAUVAIS: synchronized avec I/O
        System.out.println("❌ MAUVAIS:");
        System.out.println("""
            synchronized (lock) {
                socket.read();  // VT épinglé!
            }
        """);
        
        // ✅ BON: ReentrantLock avec I/O
        System.out.println("✅ BON:");
        System.out.println("""
            lock.lock();
            try {
                socket.read();  // VT peut se démonter
            } finally {
                lock.unlock();
            }
        """);
        
        // ✅ BON: synchronized sans I/O
        System.out.println("✅ ACCEPTABLE:");
        System.out.println("""
            synchronized (lock) {
                counter++;  // Rapide, pas d'I/O
            }
            // Si le bloc est très court (< 1ms), le pinning est OK
        """);
        
        // ❌ MAUVAIS: synchronized avec DB call
        System.out.println("❌ MAUVAIS:");
        System.out.println("""
            synchronized (lock) {
                ResultSet rs = stmt.executeQuery(...);  // VT épinglé!
            }
        """);
        
        // ✅ BON: Pas de lock pendant I/O
        System.out.println("✅ BON:");
        System.out.println("""
            ResultSet rs = stmt.executeQuery(...);  // Pas de lock
            synchronized (lock) {
                // Traiter les résultats
                processResults(rs);
            }
        """);
        
        System.out.println("\n📝 Règle générale:");
        System.out.println("• synchronized OK pour sections critiques COURTES");
        System.out.println("• ReentrantLock pour sections avec I/O ou longues");
        System.out.println("• Éviter synchronized autour d'appels bloquants");
    }
}
```

### Détection du pinning

```java
public class DetectPinning {
    
    public static void main(String[] args) throws InterruptedException {
        
        System.out.println("=== Détecter le pinning ===\n");
        
        // Activer le mode debug pour pinning
        System.setProperty("jdk.tracePinnedThreads", "full");
        
        System.out.println("Lancement VT avec synchronized + blocking...\n");
        
        Object lock = new Object();
        
        Thread.startVirtualThread(() -> {
            synchronized (lock) {
                try {
                    System.out.println("VT: dans synchronized, appel sleep()");
                    Thread.sleep(100);  // Blocking I/O
                    System.out.println("VT: après sleep()");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }).join();
        
        System.out.println("\n✅ Si pinning détecté, stack trace affichée");
        System.out.println("   dans les logs JVM");
    }
}

/* Avec -Djdk.tracePinnedThreads=full, output:

Thread[#23,ForkJoinPool-1-worker-1,5,CarrierThreads]
    java.base/java.lang.VirtualThread$VThreadContinuation.onPinned
    java.base/jdk.internal.vm.Continuation.onPinned0
    java.base/java.lang.VirtualThread.parkNanos
    java.base/java.lang.System$2.parkVirtualThread
    java.base/jdk.internal.misc.VirtualThreads.park
    java.base/java.lang.Thread.sleepNanos
    java.base/java.lang.Thread.sleep
    DetectPinning.lambda$main$0(DetectPinning.java:19) <== monitors:1
    ...

Le "<== monitors:1" indique synchronized actif = pinning!
*/
```

---

## 4.9 Résumé : Les Virtual Threads en bref

```
┌─────────────────────────────────────────────────────┐
│         Virtual Threads - Points Clés               │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ✅ AVANTAGES                                        │
│                                                     │
│ 1. Légers                                          │
│    • ~1 KB par thread                              │
│    • Des millions possibles                        │
│                                                     │
│ 2. Rapides à créer                                 │
│    • ~1 µs vs ~500 µs (Platform)                   │
│    • 1000× plus rapide                             │
│                                                     │
│ 3. Blocking gratuit                                │
│    • Démontage automatique sur I/O                 │
│    • Carriers restent libres                       │
│    • 0% de gaspillage                              │
│                                                     │
│ 4. Code simple                                     │
│    • Style synchrone/impératif                     │
│    • Pas de callback, Promise, Mono, Flux         │
│    • Compatible 100% code existant                 │
│                                                     │
│ 5. Scalabilité extrême                            │
│    • 100,000+ connexions simultanées               │
│    • WebSocket, long-polling: OK                   │
│    • Batch processing massif: OK                   │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ⚠️  PIÈGES À ÉVITER                                │
│                                                     │
│ 1. Pinning avec synchronized                       │
│    → Utiliser ReentrantLock si I/O dans le bloc    │
│                                                     │
│ 2. ThreadLocal avec millions de VT                 │
│    → Attention à la mémoire si données volumineuses│
│                                                     │
│ 3. Toujours daemon                                 │
│    → JVM peut terminer avec VT en cours            │
│                                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 🎯 QUAND UTILISER                                   │
│                                                     │
│ ✅ I/O-bound applications                           │
│ ✅ Microservices avec appels externes               │
│ ✅ API REST avec DB + cache + services             │
│ ✅ WebSocket / SSE / Long polling                   │
│ ✅ Batch processing parallèle                       │
│ ✅ Tout code qui fait du blocking I/O               │
│                                                     │
│ ⚠️  CPU-bound: pas d'avantage                       │
│    (mais pas de désavantage non plus)              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Tableau comparatif final

```
┌──────────────────┬───────────────┬─────────────────┐
│  Caractéristique │Platform Thread│ Virtual Thread  │
├──────────────────┼───────────────┼─────────────────┤
│ Gestion          │ OS Kernel     │ JVM             │
│ Mémoire/thread   │ ~2 MB         │ ~1 KB           │
│ Création         │ ~500 µs       │ ~1 µs           │
│ Max pratique     │ ~5,000        │ Millions        │
│ Blocking I/O     │ Coûteux       │ Gratuit         │
│ Context switch   │ 10-70% oh.    │ Minimal         │
│ Code style       │ Sync/Async    │ Sync simple     │
│ Compatibilité    │ 100%          │ 100%            │
│ Java version     │ Toutes        │ 21+             │
│ Production ready │ Oui           │ Oui (Java 21+)  │
└──────────────────┴───────────────┴─────────────────┘

Le verdict: Virtual Threads = Game changer!
```

---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [⬅️ Précédent](03-limitations-jvm.md) | [➡️ Suivant: Comparaison Avant/Après](05-avant-apres.md)