# chapters/02-jvm-platform-threads.md

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [⬅️ Précédent](01-qu-est-ce-qu-un-thread.md) | [➡️ Suivant](03-limitations-jvm.md)

---

# 2. La JVM et les Platform Threads

## 2.1 Architecture : Du Thread Java au Thread OS

### Le mapping 1:1

Depuis Java 1.2, Java utilise un modèle **1:1** : chaque thread Java correspond à **exactement un thread du système d'exploitation**.

```
┌─────────────────────────────────────────────────────────┐
│                    JVM Process                          │
│                                                         │
│  ┌───────────────────────────────────────────────┐    │
│  │          Java Application Code                 │    │
│  │                                                │    │
│  │  Thread t = new Thread(() -> {                │    │
│  │      System.out.println("Hello");             │    │
│  │  });                                          │    │
│  │  t.start();                                   │    │
│  └────────────────┬──────────────────────────────┘    │
│                   │                                     │
│  ┌────────────────▼──────────────────────────────┐    │
│  │         Java Thread Objects (Heap)            │    │
│  │                                                │    │
│  │  ┌──────────────┐      ┌──────────────┐      │    │
│  │  │ Thread obj 1 │      │ Thread obj 2 │      │    │
│  │  │ - name       │      │ - name       │      │    │
│  │  │ - priority   │      │ - priority   │      │    │
│  │  │ - state      │      │ - state      │      │    │
│  │  │ - target     │      │ - target     │      │    │
│  │  └──────┬───────┘      └──────┬───────┘      │    │
│  └─────────┼──────────────────────┼──────────────┘    │
│            │                      │                     │
│            │    Native Method Interface (JNI)         │
│            │                      │                     │
│  ┌─────────▼──────────────────────▼──────────────┐    │
│  │          Native Thread Layer                  │    │
│  │                                                │    │
│  │      pthread_create() / CreateThread()        │    │
│  └────────────────┬──────────────────────────────┘    │
│                   │                                     │
└───────────────────┼─────────────────────────────────────┘
                    │ Appels système (syscalls)
┌───────────────────▼─────────────────────────────────────┐
│                Operating System (Linux/Windows)         │
│                                                         │
│  ┌──────────────┐      ┌──────────────┐               │
│  │ OS Thread 1  │      │ OS Thread 2  │               │
│  │ (pthread_t)  │      │ (pthread_t)  │               │
│  │              │      │              │               │
│  │ Stack: 2 MB  │      │ Stack: 2 MB  │               │
│  └──────┬───────┘      └──────┬───────┘               │
│         │                     │                         │
│         │   Kernel Scheduler  │                         │
│         │                     │                         │
└─────────┼─────────────────────┼─────────────────────────┘
          │                     │
┌─────────▼─────────────────────▼─────────────────────────┐
│                    CPU Hardware                         │
│         Core 1          Core 2          Core 3          │
└─────────────────────────────────────────────────────────┘
```

---

## 2.2 Anatomie d'un Thread Java

### Structure mémoire complète

```
┌─────────────────────────────────────────────────────┐
│               JVM Memory Layout                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │         Java Heap (Shared)              │       │
│  │                                         │       │
│  │  • Thread objects                       │       │
│  │  • Application objects                  │       │
│  │  • ThreadLocal values (data)            │       │
│  │                                         │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │      Method Area (Shared)               │       │
│  │  • Class metadata                       │       │
│  │  • Static fields                        │       │
│  │  • Runtime constant pool                │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐               │
│  │Thread 1 Stack│  │Thread 2 Stack│  Per-thread   │
│  │              │  │              │               │
│  │ ┌──────────┐ │  │ ┌──────────┐ │               │
│  │ │ Frame 3  │ │  │ │ Frame 2  │ │ Frames =     │
│  │ ├──────────┤ │  │ ├──────────┤ │ method calls │
│  │ │ Frame 2  │ │  │ │ Frame 1  │ │               │
│  │ ├──────────┤ │  │ └──────────┘ │               │
│  │ │ Frame 1  │ │  │              │               │
│  │ └──────────┘ │  │              │               │
│  │              │  │              │               │
│  │ • Local vars │  │ • Local vars │               │
│  │ • Operands   │  │ • Operands   │               │
│  │ • Frame data │  │ • Frame data │               │
│  └──────────────┘  └──────────────┘               │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐               │
│  │  Thread 1    │  │  Thread 2    │  Per-thread   │
│  │  PC Register │  │  PC Register │               │
│  │  (pointeur)  │  │  (pointeur)  │               │
│  └──────────────┘  └──────────────┘               │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐               │
│  │Native Method │  │Native Method │  Per-thread   │
│  │Stack (JNI)   │  │Stack (JNI)   │               │
│  └──────────────┘  └──────────────┘               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Code Java vs représentation mémoire

```java
public class ThreadMemoryExample {
    
    // Stocké dans Method Area (partagé entre tous les threads)
    private static int sharedCounter = 0;
    
    // ThreadLocal : une copie par thread (dans Heap, mais isolée)
    private static ThreadLocal<Integer> threadLocalCounter = 
        ThreadLocal.withInitial(() -> 0);
    
    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            // Variables locales → Thread Stack
            int localVar = 10;
            
            // Accès variable partagée → Method Area
            sharedCounter++;
            
            // Accès ThreadLocal → Heap (copie dédiée)
            threadLocalCounter.set(threadLocalCounter.get() + 1);
            
            // Appel méthode → Frame sur Thread Stack
            processData(localVar);
        });
        
        thread1.start();
    }
    
    private static void processData(int value) {
        // Nouvelle frame ajoutée au stack du thread
        int result = value * 2;
        System.out.println(result);
    }
}
```

**Représentation mémoire lors de l'exécution** :

```
Thread 1 Stack:                  Heap:                    Method Area:
┌──────────────────┐            ┌──────────────┐         ┌──────────────┐
│ processData()    │            │ Thread obj   │         │sharedCounter │
│ - value: 10      │            │ - name: "T1" │         │ = 1          │
│ - result: 20     │            │ - state      │         └──────────────┘
├──────────────────┤            ├──────────────┤
│ lambda$main()    │            │ThreadLocal   │
│ - localVar: 10   │───────────▶│ Thread1: 1   │
│                  │            │ Thread2: 0   │
└──────────────────┘            └──────────────┘
```

---

## 2.3 Création d'un Thread : Ce qui se passe réellement

### Code Java simple

```java
Thread thread = new Thread(() -> {
    System.out.println("Hello from thread: " + Thread.currentThread().getName());
});
thread.start();
```

### Séquence d'opérations détaillée

```
┌─────────────────────────────────────────────────────────┐
│ 1. new Thread(Runnable)                                 │
│    • Allocation objet Thread dans le Heap JVM           │
│    • Initialisation des champs (name, priority, etc.)   │
│    • Coût: ~quelques microscondes                       │
│    • Aucun thread OS créé à ce stade !                  │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 2. thread.start()                                       │
│    a) Vérification état (IllegalThreadStateException)   │
│    b) Appel native: start0()                            │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 3. JNI Call → JVM Native Code                          │
│    • JVM_StartThread() (hotspot/src/share/vm/prims)    │
│    • Création JavaThread interne                        │
│    • Allocation OSThread                                │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 4. OS Thread Creation (syscall)                        │
│    Linux: pthread_create(&tid, &attr, start_func, arg) │
│    Windows: CreateThread(...)                          │
│    • Allocation stack: ~2 MB                           │
│    • Création TCB dans kernel                          │
│    • Coût: ~0.2-1 ms                                   │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 5. OS Scheduler                                        │
│    • Thread ajouté à la run queue                      │
│    • Attente d'un CPU disponible                       │
│    • Peut prendre du temps si système chargé           │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Exécution run()                                     │
│    • Thread obtient un CPU core                        │
│    • Exécution du code Runnable                        │
│    • Création frames sur le thread stack               │
└─────────────────────────────────────────────────────────┘
```

### Visualisation avec code instrumenté

```java
public class ThreadCreationInstrumented {
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Avant création Thread Java ===");
        printThreadInfo();
        
        long beforeAlloc = System.nanoTime();
        
        // ÉTAPE 1: Allocation objet Thread (Heap)
        Thread thread = new Thread(() -> {
            System.out.println("\n=== Dans le nouveau thread ===");
            printThreadInfo();
            
            // Simuler du travail
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {}
        }, "MonThread-Custom");
        
        long afterAlloc = System.nanoTime();
        System.out.println("\nTemps allocation objet: " + 
            (afterAlloc - beforeAlloc) / 1000 + " µs");
        
        System.out.println("\n=== Après new Thread(), avant start() ===");
        System.out.println("État du thread: " + thread.getState()); // NEW
        System.out.println("Thread OS créé? Non, pas encore!");
        printThreadInfo(); // Toujours un seul thread OS
        
        long beforeStart = System.nanoTime();
        
        // ÉTAPE 2: Création thread OS
        thread.start();
        
        long afterStart = System.nanoTime();
        System.out.println("\nTemps start() (création OS thread): " + 
            (afterStart - beforeStart) / 1000 + " µs");
        
        Thread.sleep(10); // Laisser le thread démarrer
        
        System.out.println("\n=== Après start() ===");
        System.out.println("État du thread: " + thread.getState()); // RUNNABLE
        printThreadInfo(); // Maintenant 2 threads OS
        
        thread.join();
        
        System.out.println("\n=== Après terminaison ===");
        System.out.println("État du thread: " + thread.getState()); // TERMINATED
        printThreadInfo(); // Retour à 1 thread OS
    }
    
    private static void printThreadInfo() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        System.out.println("Nombre de threads JVM: " + threadBean.getThreadCount());
        System.out.println("Threads actifs: " + Thread.activeCount());
    }
}

/* Output typique:

=== Avant création Thread Java ===
Nombre de threads JVM: 6
Threads actifs: 1

Temps allocation objet: 15 µs

=== Après new Thread(), avant start() ===
État du thread: NEW
Thread OS créé? Non, pas encore!
Nombre de threads JVM: 6
Threads actifs: 1

Temps start() (création OS thread): 450 µs

=== Après start() ===
État du thread: RUNNABLE
Nombre de threads JVM: 7
Threads actifs: 2

=== Dans le nouveau thread ===
Nombre de threads JVM: 7
Threads actifs: 2

=== Après terminaison ===
État du thread: TERMINATED
Nombre de threads JVM: 6
Threads actifs: 1

Conclusion:
- new Thread(): rapide (~15µs), pas de thread OS
- start(): lent (~450µs), crée un thread OS
*/
```

---

## 2.4 Cas pratique : Thread avec JDBC

### Exemple d'accès base de données

```java
import java.sql.*;

public class JdbcThreadExample {
    
    private static final String DB_URL = "jdbc:postgresql://localhost:5432/mydb";
    private static final String USER = "user";
    private static final String PASS = "password";
    
    public static void main(String[] args) throws InterruptedException {
        
        // Lancer 10 threads qui font des requêtes DB
        Thread[] threads = new Thread[10];
        
        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            threads[i] = new Thread(() -> {
                queryDatabase(threadId);
            }, "DB-Thread-" + i);
            
            threads[i].start();
        }
        
        // Attendre tous les threads
        for (Thread t : threads) {
            t.join();
        }
    }
    
    private static void queryDatabase(int threadId) {
        Connection conn = null;
        Statement stmt = null;
        
        try {
            // POINT 1: Thread obtient connexion
            System.out.println("[" + Thread.currentThread().getName() + 
                "] Demande connexion DB...");
            
            long start = System.nanoTime();
            conn = DriverManager.getConnection(DB_URL, USER, PASS);
            long connTime = (System.nanoTime() - start) / 1_000_000;
            
            System.out.println("[" + Thread.currentThread().getName() + 
                "] Connexion obtenue en " + connTime + "ms");
            
            // POINT 2: Thread exécute requête (BLOQUE sur I/O réseau)
            stmt = conn.createStatement();
            
            start = System.nanoTime();
            System.out.println("[" + Thread.currentThread().getName() + 
                "] Exécution requête SQL...");
            
            // ⚠️ ICI LE THREAD SE BLOQUE
            // Il attend la réponse du serveur PostgreSQL
            // Pendant ce temps: 2 MB de stack inutilisés!
            ResultSet rs = stmt.executeQuery(
                "SELECT id, name FROM users WHERE id = " + threadId
            );
            
            long queryTime = (System.nanoTime() - start) / 1_000_000;
            
            // POINT 3: Thread traite les résultats
            if (rs.next()) {
                System.out.println("[" + Thread.currentThread().getName() + 
                    "] Résultat reçu en " + queryTime + "ms: " +
                    "id=" + rs.getInt("id") + 
                    ", name=" + rs.getString("name"));
            }
            
            rs.close();
            
        } catch (SQLException e) {
            System.err.println("[" + Thread.currentThread().getName() + 
                "] Erreur: " + e.getMessage());
        } finally {
            // POINT 4: Libération ressources
            try {
                if (stmt != null) stmt.close();
                if (conn != null) conn.close();
            } catch (SQLException e) {}
        }
    }
}
```

### Timeline détaillée d'un thread JDBC

```
Thread "DB-Thread-0" sur 100ms d'exécution:

Timeline:
0ms    10ms   20ms        70ms   80ms     100ms
│      │      │           │      │        │
├──────┼──────┼───────────┼──────┼────────┤
│ CPU  │ I/O  │   WAIT    │ I/O  │  CPU   │
│ actif│ wait │  (blocked)│ wait │ actif  │
└──────┴──────┴───────────┴──────┴────────┘

Détails:
┌─────────────────────────────────────────────────┐
│ 0-10ms: CPU actif                               │
│  • getConnection()                              │
│  • Préparation requête SQL                      │
│  • Sérialisation paramètres                     │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ 10-20ms: I/O wait (envoi requête)              │
│  • Écriture sur socket réseau                   │
│  • Thread BLOQUÉ (write syscall)                │
│  • Carrier thread pourrait faire autre chose!   │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ 20-70ms: WAIT (attente réponse DB)             │
│  • Thread complètement BLOQUÉ                   │
│  • Attend read() sur socket                     │
│  • 2 MB de stack INUTILISÉS                     │
│  • Le serveur DB traite la requête              │
│  ⚠️ 50ms de pure attente = GASPILLAGE!          │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ 70-80ms: I/O wait (réception réponse)          │
│  • Lecture socket réseau                        │
│  • Désérialisation données                      │
│  • Thread BLOQUÉ (read syscall)                 │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ 80-100ms: CPU actif                             │
│  • Traitement ResultSet                         │
│  • Conversion types                             │
│  • Fermeture ressources                         │
└─────────────────────────────────────────────────┘

Bilan:
- CPU actif: 30ms (30%)
- I/O bloqué: 70ms (70%)
- Efficacité: MAUVAISE (70% de gaspillage)
```

---

## 2.5 Cas pratique : Thread avec HTTP Client

### Appel API REST

```java
import java.net.URI;
import java.net.http.*;
import java.time.Duration;

public class HttpThreadExample {
    
    private static final HttpClient httpClient = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(10))
        .build();
    
    public static void main(String[] args) throws InterruptedException {
        
        // Lancer 100 threads qui appellent une API
        Thread[] threads = new Thread[100];
        
        long globalStart = System.currentTimeMillis();
        
        for (int i = 0; i < 100; i++) {
            final int requestId = i;
            threads[i] = new Thread(() -> {
                callExternalAPI(requestId);
            }, "HTTP-Thread-" + i);
            
            threads[i].start();
        }
        
        // Attendre tous les threads
        for (Thread t : threads) {
            t.join();
        }
        
        long totalTime = System.currentTimeMillis() - globalStart;
        System.out.println("\n=== RÉSUMÉ ===");
        System.out.println("100 requêtes HTTP en " + totalTime + "ms");
        System.out.println("Throughput: " + (100.0 / totalTime * 1000) + " req/s");
    }
    
    private static void callExternalAPI(int requestId) {
        try {
            long start = System.nanoTime();
            
            // Préparer requête
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://jsonplaceholder.typicode.com/posts/" + requestId))
                .GET()
                .build();
            
            System.out.println("[" + Thread.currentThread().getName() + 
                "] Envoi requête HTTP...");
            
            // ⚠️ ICI LE THREAD SE BLOQUE
            // send() est SYNCHRONE et BLOQUANT
            // Le thread attend la réponse réseau
            HttpResponse<String> response = httpClient.send(
                request,
                HttpResponse.BodyHandlers.ofString()
            );
            
            long duration = (System.nanoTime() - start) / 1_000_000;
            
            System.out.println("[" + Thread.currentThread().getName() + 
                "] Réponse reçue en " + duration + "ms (status: " + 
                response.statusCode() + ", body length: " + 
                response.body().length() + ")");
            
        } catch (Exception e) {
            System.err.println("[" + Thread.currentThread().getName() + 
                "] Erreur: " + e.getMessage());
        }
    }
}

/* Output typique:

[HTTP-Thread-0] Envoi requête HTTP...
[HTTP-Thread-1] Envoi requête HTTP...
[HTTP-Thread-2] Envoi requête HTTP...
...
[HTTP-Thread-0] Réponse reçue en 145ms (status: 200, body length: 292)
[HTTP-Thread-1] Réponse reçue en 156ms (status: 200, body length: 292)
...

=== RÉSUMÉ ===
100 requêtes HTTP en 3200ms
Throughput: 31.25 req/s

Problème:
- 100 threads créés = 200 MB de mémoire
- Chaque thread bloqué ~150ms sur I/O réseau
- CPU utilization: ~5%
- Avec 10,000 requêtes → OutOfMemoryError ou thrashing
*/
```

### Comparaison : approche asynchrone (sans VT)

```java
public class HttpAsyncExample {
    
    private static final HttpClient httpClient = HttpClient.newHttpClient();
    
    public static void main(String[] args) throws InterruptedException {
        
        long globalStart = System.currentTimeMillis();
        
        // Approche async: pas de création de threads !
        List<CompletableFuture<HttpResponse<String>>> futures = new ArrayList<>();
        
        for (int i = 0; i < 100; i++) {
            final int requestId = i;
            
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://jsonplaceholder.typicode.com/posts/" + requestId))
                .GET()
                .build();
            
            // sendAsync() retourne immédiatement
            // Pas de blocage !
            CompletableFuture<HttpResponse<String>> future = 
                httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                    .thenApply(response -> {
                        System.out.println("Réponse " + requestId + 
                            ": status=" + response.statusCode());
                        return response;
                    });
            
            futures.add(future);
        }
        
        // Attendre toutes les réponses
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        
        long totalTime = System.currentTimeMillis() - globalStart;
        System.out.println("\n=== RÉSUMÉ ASYNC ===");
        System.out.println("100 requêtes HTTP en " + totalTime + "ms");
        System.out.println("Throughput: " + (100.0 / totalTime * 1000) + " req/s");
    }
}

/* Output typique:

=== RÉSUMÉ ASYNC ===
100 requêtes HTTP en 180ms
Throughput: 555.55 req/s

Avantages:
- Beaucoup plus rapide (180ms vs 3200ms)
- Pas de création de threads
- Meilleur throughput

Inconvénients:
- Code plus complexe (CompletableFuture, lambdas)
- Difficile à debugger
- Gestion erreurs complexe
*/
```

---

## 2.6 La gestion des threads par la JVM

### Thread Lifecycle Management

```java
public class ThreadLifecycleDemo {
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread thread = new Thread(() -> {
            System.out.println("Thread démarré: " + Thread.currentThread().getName());
            
            try {
                // Simulation travail
                for (int i = 0; i < 5; i++) {
                    System.out.println("Travail " + i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                System.out.println("Thread interrompu!");
                return;
            }
            
            System.out.println("Thread terminé");
        }, "Worker-Thread");
        
        // État NEW
        System.out.println("État: " + thread.getState()); // NEW
        System.out.println("Vivant? " + thread.isAlive()); // false
        
        // Démarrage
        thread.start();
        
        // État RUNNABLE (en cours d'exécution ou prêt)
        Thread.sleep(10);
        System.out.println("État: " + thread.getState()); // RUNNABLE
        System.out.println("Vivant? " + thread.isAlive()); // true
        
        // État TIMED_WAITING (pendant Thread.sleep)
        Thread.sleep(150);
        System.out.println("État: " + thread.getState()); // TIMED_WAITING
        
        // Attendre la fin
        thread.join();
        
        // État TERMINATED
        System.out.println("État: " + thread.getState()); // TERMINATED
        System.out.println("Vivant? " + thread.isAlive()); // false
    }
}
```

### États détaillés d'un thread

```
┌─────────────────────────────────────────────────────┐
│                  Thread States                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  NEW (thread créé, pas démarré)                    │
│      │                                              │
│      │ start()                                      │
│      ▼                                              │
│  RUNNABLE (prêt ou en exécution)                   │
│      │                                              │
│      ├─► En execution sur CPU                      │
│      │   (running)                                  │
│      │                                              │
│      ├─► En attente CPU                            │
│      │   (ready - dans run queue)                  │
│      │                                              │
│      │ sleep(n) / wait(timeout) / join(timeout)    │
│      ▼                                              │
│  TIMED_WAITING (attente avec timeout)              │
│      │                                              │
│      │ timeout expiré                               │
│      ▼                                              │
│  RUNNABLE                                           │
│      │                                              │
│      │ wait() / join()                              │
│      ▼                                              │
│  WAITING (attente indéfinie)                       │
│      │                                              │
│      │ notify() / notifyAll() / interrupt()        │
│      ▼                                              │
│  RUNNABLE                                           │
│      │                                              │
│      │ synchronized block / lock                    │
│      ▼                                              │
│  BLOCKED (attente d'un lock)                       │
│      │                                              │
│      │ lock obtenu                                  │
│      ▼                                              │
│  RUNNABLE                                           │
│      │                                              │
│      │ fin du run()                                 │
│      ▼                                              │
│  TERMINATED (thread terminé)                       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Exemple complet des états

```java
public class ThreadStatesExample {
    
    private static final Object lock = new Object();
    private static volatile boolean flag = false;
    
    public static void main(String[] args) throws InterruptedException {
        
        // Thread qui va passer par tous les états
        Thread thread = new Thread(() -> {
            try {
                // État RUNNABLE (en exécution)
                System.out.println("Thread en cours d'exécution");
                
                // État TIMED_WAITING
                Thread.sleep(500);
                
                // État WAITING
                synchronized (lock) {
                    while (!flag) {
                        lock.wait();
                    }
                }
                
                // État BLOCKED (attente du lock)
                synchronized (lock) {
                    System.out.println("Lock obtenu");
                }
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "StateDemo");
        
        // État NEW
        System.out.println("1. État NEW: " + thread.getState());
        
        thread.start();
        Thread.sleep(50);
        
        // État RUNNABLE
        System.out.println("2. État RUNNABLE: " + thread.getState());
        
        Thread.sleep(200);
        
        // État TIMED_WAITING
        System.out.println("3. État TIMED_WAITING: " + thread.getState());
        
        Thread.sleep(400);
        
        // État WAITING
        System.out.println("4. État WAITING: " + thread.getState());
        
        // Débloquer le thread
        synchronized (lock) {
            flag = true;
            lock.notify();
        }
        
        thread.join();
        
        // État TERMINATED
        System.out.println("5. État TERMINATED: " + thread.getState());
    }
}
```

---

## 2.7 Thread Pools : La gestion collective

### ExecutorService et Thread Pools

```java
import java.util.concurrent.*;

public class ThreadPoolExample {
    
    public static void main(String[] args) throws InterruptedException {
        
        // Pool de taille fixe : 4 threads OS
        ExecutorService executor = Executors.newFixedThreadPool(4);
        
        System.out.println("=== Pool créé avec 4 threads ===\n");
        
        // Soumettre 10 tâches
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            
            executor.submit(() -> {
                String threadName = Thread.currentThread().getName();
                System.out.println("[" + threadName + "] Début tâche " + taskId);
                
                try {
                    // Simulation travail bloquant
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                
                System.out.println("[" + threadName + "] Fin tâche " + taskId);
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(30, TimeUnit.SECONDS);
        
        System.out.println("\n=== Toutes les tâches terminées ===");
    }
}

/* Output typique:

=== Pool créé avec 4 threads ===

[pool-1-thread-1] Début tâche 0
[pool-1-thread-2] Début tâche 1
[pool-1-thread-3] Début tâche 2
[pool-1-thread-4] Début tâche 3
[pool-1-thread-1] Fin tâche 0
[pool-1-thread-1] Début tâche 4    ← Thread réutilisé!
[pool-1-thread-2] Fin tâche 1
[pool-1-thread-2] Début tâche 5    ← Thread réutilisé!
...

Observation:
- Seulement 4 threads OS créés
- Les 10 tâches s'exécutent séquentiellement sur ces 4 threads
- Réutilisation des threads (pas de création/destruction)
- Temps total: ~3 secondes (10 tâches / 4 threads ≈ 2.5 rounds)
*/
```

### Architecture interne d'un Thread Pool

```
┌─────────────────────────────────────────────────────┐
│              ExecutorService                        │
│                                                     │
│  ┌───────────────────────────────────────────┐    │
│  │         Work Queue (BlockingQueue)        │    │
│  │                                           │    │
│  │  [Task 5] [Task 6] [Task 7] [Task 8] ... │    │
│  │                                           │    │
│  │  Tâches en attente d'exécution            │    │
│  └───────────┬───────────────────────────────┘    │
│              │                                      │
│              │ take()                               │
│              ▼                                      │
│  ┌───────────────────────────────────────────┐    │
│  │         Thread Pool (Workers)             │    │
│  │                                           │    │
│  │  ┌──────────┐  ┌──────────┐             │    │
│  │  │ Thread 1 │  │ Thread 2 │  ← Platform │    │
│  │  │ (Worker) │  │ (Worker) │     Threads │    │
│  │  │          │  │          │             │    │
│  │  │ Task 1   │  │ Task 2   │             │    │
│  │  └──────────┘  └──────────┘             │    │
│  │                                           │    │
│  │  ┌──────────┐  ┌──────────┐             │    │
│  │  │ Thread 3 │  │ Thread 4 │             │    │
│  │  │ (Worker) │  │ (Worker) │             │    │
│  │  │          │  │          │             │    │
│  │  │ Task 3   │  │ Task 4   │             │    │
│  │  └──────────┘  └──────────┘             │    │
│  │                                           │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
                    │
                    │ Mapping 1:1
                    ▼
┌─────────────────────────────────────────────────────┐
│              Operating System                       │
│                                                     │
│  [OS Thread 1] [OS Thread 2] [OS Thread 3] [OS Thread 4] │
│       │             │             │             │         │
│       └─────────────┴─────────────┴─────────────┘         │
│                     │                                      │
│              CPU Scheduler                                 │
└─────────────────────────────────────────────────────────────┘
```

### Types de Thread Pools

```java
public class ThreadPoolTypes {
    
    public static void main(String[] args) {
        
        // 1. Fixed Thread Pool
        // Nombre fixe de threads, queue illimitée
        ExecutorService fixed = Executors.newFixedThreadPool(10);
        // Création: 10 threads OS
        // Avantage: Contrôle précis des ressources
        // Inconvénient: Queue peut grossir indéfiniment (OOM)
        
        
        // 2. Cached Thread Pool
        // Threads créés à la demande, réutilisés pendant 60s
        ExecutorService cached = Executors.newCachedThreadPool();
        // Création: 0 thread au départ, augmente selon la charge
        // Avantage: S'adapte à la charge
        // Inconvénient: Peut créer trop de threads (OOM)
        
        
        // 3. Single Thread Executor
        // Un seul thread, exécution séquentielle
        ExecutorService single = Executors.newSingleThreadExecutor();
        // Création: 1 thread OS
        // Avantage: Ordre d'exécution garanti
        // Inconvénient: Pas de parallélisme
        
        
        // 4. Scheduled Thread Pool
        // Pour tâches périodiques ou différées
        ScheduledExecutorService scheduled = 
            Executors.newScheduledThreadPool(5);
        // Création: 5 threads OS
        // Avantage: Scheduling intégré
        // Utilisation: Tâches cron-like
        
        
        // 5. Work Stealing Pool (Java 8+)
        // Threads qui "volent" le travail des autres
        ExecutorService workStealing = 
            Executors.newWorkStealingPool();
        // Création: Runtime.availableProcessors() threads
        // Avantage: Optimal pour CPU-bound tasks
        // Basé sur ForkJoinPool
    }
}
```

---

## 2.8 Synchronisation et Monitors

### Le problème de la concurrence

```java
public class RaceConditionExample {
    
    private static int counter = 0;
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread[] threads = new Thread[100];
        
        // 100 threads incrémentent counter 1000 fois chacun
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter++; // ⚠️ PAS ATOMIQUE !
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("Valeur attendue: 100000");
        System.out.println("Valeur obtenue: " + counter);
        System.out.println("Pertes: " + (100000 - counter));
    }
}

/* Output typique:

Valeur attendue: 100000
Valeur obtenue: 87234
Pertes: 12766

Pourquoi? counter++ n'est PAS atomique!

En bytecode:
1. GETFIELD counter      // Lire valeur
2. ICONST_1              // Push 1
3. IADD                  // Additionner
4. PUTFIELD counter      // Écrire résultat

Si 2 threads exécutent simultanément:
Thread A: Lit 42
Thread B: Lit 42
Thread A: Calcule 43
Thread B: Calcule 43
Thread A: Écrit 43
Thread B: Écrit 43  ← Une incrémentation perdue!
*/
```

### Solution : synchronized

```java
public class SynchronizedExample {
    
    private static int counter = 0;
    private static final Object lock = new Object();
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread[] threads = new Thread[100];
        
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    // Bloc synchronized : un seul thread à la fois
                    synchronized (lock) {
                        counter++;
                    }
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("Valeur attendue: 100000");
        System.out.println("Valeur obtenue: " + counter);
    }
}

/* Output:

Valeur attendue: 100000
Valeur obtenue: 100000  ← Correct!

Comment ça marche:
- Chaque objet Java a un "monitor" (verrou intrinsèque)
- synchronized(obj) acquiert le monitor de obj
- Un seul thread peut détenir le monitor à la fois
- Les autres threads sont BLOQUÉS (état BLOCKED)
*/
```

### Visualisation du monitor

```
Timeline avec synchronized:

Thread 1: ████░░░░░░░░░░░░████░░░░░░░░████
Thread 2: ░░░░████░░░░░░░░░░░░████░░░░░░░░
Thread 3: ░░░░░░░░████░░░░░░░░░░░░████░░░░
          │   │   │   │   │   │   │   │
          Lock│   │Unlock│   │Unlock│   │
              Lock│   Lock│       Lock│
                  Unlock│           Unlock

Légende:
█ = Thread possède le lock (exécution)
░ = Thread attend le lock (BLOCKED)

Problème:
- Beaucoup de temps d'attente
- Context switching fréquent
- Contention du lock
```

### Monitor détaillé au niveau JVM

```
┌─────────────────────────────────────────────────────┐
│            Object Monitor (intrinsic lock)          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Owner Thread: Thread-3                            │
│  (thread qui détient actuellement le lock)          │
│                                                     │
│  ┌───────────────────────────────────────────┐    │
│  │         Entry Set (BLOCKED)                │    │
│  │                                            │    │
│  │  [Thread-1] [Thread-5] [Thread-8] ...     │    │
│  │                                            │    │
│  │  Threads en attente du lock                │    │
│  │  État: BLOCKED                             │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
│  ┌───────────────────────────────────────────┐    │
│  │         Wait Set (WAITING)                 │    │
│  │                                            │    │
│  │  [Thread-2] [Thread-4] ...                 │    │
│  │                                            │    │
│  │  Threads ayant appelé wait()               │    │
│  │  État: WAITING                             │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Exemple wait/notify

```java
public class WaitNotifyExample {
    
    private static final Object lock = new Object();
    private static boolean dataReady = false;
    private static String sharedData = null;
    
    public static void main(String[] args) throws InterruptedException {
        
        // Thread consommateur
        Thread consumer = new Thread(() -> {
            synchronized (lock) {
                System.out.println("[Consumer] En attente de données...");
                
                try {
                    // wait() libère le lock et met le thread en WAITING
                    while (!dataReady) {
                        lock.wait();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
                
                System.out.println("[Consumer] Données reçues: " + sharedData);
            }
        }, "Consumer");
        
        // Thread producteur
        Thread producer = new Thread(() -> {
            try {
                Thread.sleep(1000); // Simule préparation
            } catch (InterruptedException e) {
                return;
            }
            
            synchronized (lock) {
                System.out.println("[Producer] Production de données...");
                sharedData = "Hello from Producer!";
                dataReady = true;
                
                // notify() réveille un thread dans le Wait Set
                lock.notify();
                System.out.println("[Producer] notify() envoyé");
            }
        }, "Producer");
        
        consumer.start();
        Thread.sleep(100); // S'assurer que consumer attend
        producer.start();
        
        consumer.join();
        producer.join();
    }
}

/* Output:

[Consumer] En attente de données...
[Producer] Production de données...
[Producer] notify() envoyé
[Consumer] Données reçues: Hello from Producer!

Séquence:
1. Consumer prend le lock
2. Consumer appelle wait() → libère le lock, passe en WAITING
3. Producer prend le lock (libéré par consumer)
4. Producer modifie les données
5. Producer appelle notify() → réveille consumer
6. Producer libère le lock
7. Consumer reprend le lock, sort de wait()
8. Consumer traite les données
*/
```

---

## 2.9 Thread Locals : Isolation par thread

### Concept ThreadLocal

```java
public class ThreadLocalExample {
    
    // Chaque thread a sa propre copie de cette variable
    private static ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> 0);
    private static ThreadLocal<String> username = new ThreadLocal<>();
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread thread1 = new Thread(() -> {
            // Thread 1 définit ses valeurs
            threadId.set(1);
            username.set("Alice");
            
            System.out.println("[Thread 1] threadId: " + threadId.get());
            System.out.println("[Thread 1] username: " + username.get());
            
            doWork();
        }, "Thread-1");
        
        Thread thread2 = new Thread(() -> {
            // Thread 2 définit ses propres valeurs
            threadId.set(2);
            username.set("Bob");
            
            System.out.println("[Thread 2] threadId: " + threadId.get());
            System.out.println("[Thread 2] username: " + username.get());
            
            doWork();
        }, "Thread-2");
        
        thread1.start();
        thread2.start();
        
        thread1.join();
        thread2.join();
    }
    
    private static void doWork() {
        // Chaque thread voit ses propres valeurs
        System.out.println("[" + Thread.currentThread().getName() + 
            "] Dans doWork() - username: " + username.get());
    }
}

/* Output:

[Thread 1] threadId: 1
[Thread 1] username: Alice
[Thread-1] Dans doWork() - username: Alice
[Thread 2] threadId: 2
[Thread 2] username: Bob
[Thread-2] Dans doWork() - username: Bob

Chaque thread a son propre contexte isolé!
*/
```

### Architecture ThreadLocal en mémoire

```
┌─────────────────────────────────────────────────────┐
│                    Heap Memory                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Thread Object 1                                    │
│  ┌───────────────────────────────────────────┐    │
│  │ threadLocals: ThreadLocalMap              │    │
│  │   ┌─────────────────────────────────┐     │    │
│  │   │ Entry[0]: threadId → 1          │     │    │
│  │   │ Entry[1]: username → "Alice"    │     │    │
│  │   └─────────────────────────────────┘     │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
│  Thread Object 2                                    │
│  ┌───────────────────────────────────────────┐    │
│  │ threadLocals: ThreadLocalMap              │    │
│  │   ┌─────────────────────────────────┐     │    │
│  │   │ Entry[0]: threadId → 2          │     │    │
│  │   │ Entry[1]: username → "Bob"      │     │    │
│  │   └─────────────────────────────────┘     │    │
│  └───────────────────────────────────────────┘    │
│                                                     │
│  ThreadLocal Objects (static, partagés)            │
│  ┌─────────────┐  ┌─────────────┐                 │
│  │ threadId    │  │ username    │                  │
│  │ (référence) │  │ (référence) │                  │
│  └─────────────┘  └─────────────┘                  │
│                                                     │
└─────────────────────────────────────────────────────┘

Note: Le ThreadLocal lui-même est partagé,
mais chaque Thread a sa propre Map de valeurs.
```

### Cas d'usage pratique : Contexte de requête

```java
public class RequestContext {
    
    // Contexte de requête HTTP simulé
    private static ThreadLocal<String> requestId = new ThreadLocal<>();
    private static ThreadLocal<String> userId = new ThreadLocal<>();
    private static ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    public static void setContext(String reqId, String user) {
        requestId.set(reqId);
        userId.set(user);
        startTime.set(System.currentTimeMillis());
    }
    
    public static void clearContext() {
        requestId.remove(); // Important: éviter memory leaks!
        userId.remove();
        startTime.remove();
    }
    
    public static String getRequestId() {
        return requestId.get();
    }
    
    public static String getUserId() {
        return userId.get();
    }
    
    public static long getElapsedTime() {
        Long start = startTime.get();
        return start != null ? System.currentTimeMillis() - start : 0;
    }
    
    // Simulation serveur HTTP
    public static void main(String[] args) throws InterruptedException {
        
        // Simuler 3 requêtes HTTP concurrentes
        Thread[] requests = new Thread[3];
        
        for (int i = 0; i < 3; i++) {
            final int requestNum = i;
            requests[i] = new Thread(() -> {
                try {
                    // Initialiser contexte pour ce thread
                    setContext("REQ-" + requestNum, "user-" + requestNum);
                    
                    System.out.println("[" + getRequestId() + "] Début requête");
                    
                    // Simuler traitement
                    processRequest();
                    
                    // Simuler appel service
                    callDatabaseService();
                    
                    System.out.println("[" + getRequestId() + 
                        "] Requête terminée en " + getElapsedTime() + "ms");
                    
                } finally {
                    // Nettoyer contexte (important!)
                    clearContext();
                }
            }, "Request-Handler-" + i);
            
            requests[i].start();
        }
        
        for (Thread t : requests) {
            t.join();
        }
    }
    
    private static void processRequest() throws InterruptedException {
        System.out.println("[" + getRequestId() + "] Traitement pour user: " + 
            getUserId());
        Thread.sleep(100);
    }
    
    private static void callDatabaseService() throws InterruptedException {
        // Le contexte est automatiquement disponible !
        System.out.println("[" + getRequestId() + "] Appel DB pour user: " + 
            getUserId());
        Thread.sleep(200);
    }
}

/* Output:

[REQ-0] Début requête
[REQ-1] Début requête
[REQ-2] Début requête
[REQ-0] Traitement pour user: user-0
[REQ-1] Traitement pour user: user-1
[REQ-2] Traitement pour user: user-2
[REQ-0] Appel DB pour user: user-0
[REQ-1] Appel DB pour user: user-1
[REQ-2] Appel DB pour user: user-2
[REQ-0] Requête terminée en 302ms
[REQ-1] Requête terminée en 301ms
[REQ-2] Requête terminée en 300ms

Avantage: Pas besoin de passer requestId et userId
           en paramètre partout!
*/
```

---

## 2.10 Récapitulatif : Thread Platform en action

### Synthèse visuelle

```
┌─────────────────────────────────────────────────────┐
│          Thread Java (Platform Thread)              │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ Objet Thread (Heap JVM)                     │  │
│  │ - name, priority, state                     │  │
│  │ - target (Runnable)                         │  │
│  │ - threadLocals (Map)                        │  │
│  └────────────┬────────────────────────────────┘  │
│               │                                     │
│               │ Référence vers                      │
│               ▼                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │ Native Thread (JVM internal)                │  │
│  │ - eetop (pointeur vers OS thread)           │  │
│  │ - stack base/size                           │  │
│  └────────────┬────────────────────────────────┘  │
└───────────────┼─────────────────────────────────────┘
                │ JNI
┌───────────────▼─────────────────────────────────────┐
│ OS Thread (pthread_t / HANDLE)                      │
│                                                     │
│ - Thread ID                                         │
│ - Stack: 2 MB                                       │
│ - TCB (Thread Control Block)                        │
│ - Registres CPU sauvegardés                         │
│ - Priority, affinity                                │
└───────────────┬─────────────────────────────────────┘
                │ Scheduler
┌───────────────▼─────────────────────────────────────┐
│ CPU Core                                            │
└─────────────────────────────────────────────────────┘

Coûts:
- Mémoire: ~2 MB par thread
- Création: ~0.5 ms
- Context switch: ~5 µs
- Limite: ~5000 threads
```

### Points clés à retenir

```
✅ Ce qu'il faut retenir sur les Platform Threads:

1. Mapping 1:1 avec l'OS
   • 1 Thread Java = 1 pthread/Windows thread
   • Géré par le kernel OS
   • Coût mémoire: ~2 MB par thread

2. Blocking I/O = gaspillage
   • Thread bloqué sur DB/HTTP/File I/O
   • 2 MB de mémoire inutilisée
   • Ne peut rien faire d'autre

3. Thread Pools aident mais ne résolvent pas
   • Réutilisation de threads
   • Mais toujours limité en nombre
   • Queue peut déborder

4. Synchronisation nécessaire
   • Race conditions sans synchronized
   • Monitor/lock cause contention
   • Threads en BLOCKED state

5. ThreadLocal pour l'isolation
   • Chaque thread a son contexte
   • Utile pour request context
   • Attention aux memory leaks!

❌ Le problème fondamental:
   • Impossible d'avoir 100,000 threads Platform
   • Mais les applications modernes ont besoin
     de gérer 100,000+ connexions simultanées
   
➡️  Solution: Virtual Threads (chapitre suivant)
```

---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [⬅️ Précédent](01-qu-est-ce-qu-un-thread.md) | [➡️ Suivant: Limitations de la JVM](03-limitations-jvm.md)