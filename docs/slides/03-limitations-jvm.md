[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [📖 Lexique](lexique.md) | [⬅️ Précédent](02-jvm-platform-threads.md) | [➡️ Suivant](04-virtual-threads-intro.md)

---

# 3. Limitations Réelles des Platform Threads

## 3.1 Le problème fondamental : Thread-per-Request

### Architecture classique d'un serveur

```
Serveur HTTP classique (Tomcat, Jetty, etc.)

┌─────────────────────────────────────────────────┐
│           Thread Pool (200 threads)             │
├─────────────────────────────────────────────────┤
│                                                 │
│  Client 1 → [Thread 1] → Handler → Response     │
│  Client 2 → [Thread 2] → Handler → Response     │
│  Client 3 → [Thread 3] → Handler → Response     │
│  ...                                            │
│  Client 200 → [Thread 200] → Handler → Response │
│                                                 │
│   Client 201 → [QUEUE] En attente...            │
│   Client 202 → [QUEUE] En attente...            |
│   Client 203 → [QUEUE] En attente...            |
│                                                 │
└─────────────────────────────────────────────────┘

Problème:
• 200 threads max = 200 requêtes simultanées max
• Au-delà: les requêtes attendent dans une queue
• Si tous les threads sont bloqués en I/O → CPU à 5%
• Mais impossible d'accepter plus de clients!
```

### Démonstration avec un serveur HTTP simple

Ici, on va créer un serveur HTTP simple en Java qui utilise un ThreadPoolExecutor avec une taille de pool limitée et une queue limitée. 
Le serveur simule un traitement bloquant (par exemple, une opération I/O) en dormant pendant 5 secondes pour chaque requête. 
En parallèle on va jouer un petit script pour envoyer 30 requêtes simultanées et observer le comportement du serveur.

```java
import com.sun.net.httpserver.HttpServer;

public class ThreadPerRequestDemo {

	private static final AtomicInteger activeRequests = new AtomicInteger(0);
	private static final AtomicInteger rejectedRequests = new AtomicInteger(0);

	public static void main(String[] args) throws IOException {

		HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

		ThreadPoolExecutor executor = new ThreadPoolExecutor(
			5,
			5,
			60L, TimeUnit.SECONDS,
			new ArrayBlockingQueue<>(10)
		);

		executor.setRejectedExecutionHandler((r, exec) -> {
			int rejected = rejectedRequests.incrementAndGet();
			System.err.printf("❌ REJETÉ ! Total rejections: %d%n", rejected);
			throw new RejectedExecutionException("Queue pleine");
		});

		server.setExecutor(executor);

		// Handler qui simule un I/O bloquant
		server.createContext("/api/data", exchange -> {
			int active = activeRequests.incrementAndGet();
			int queued = executor.getQueue().size();
			String threadName = Thread.currentThread().getName();

			System.out.printf("[%s] Requête reçue (Active: %d, Queue: %d)%n",
				threadName, active, queued);

			try {
				Thread.sleep(5000);

				String response = String.format(
					"Traité par %s (Active: %d, Queue: %d)",
					threadName, active, queued
				);

				exchange.sendResponseHeaders(200, response.length());
				OutputStream os = exchange.getResponseBody();
				os.write(response.getBytes());
				os.close();

			} catch (InterruptedException e) {
				Thread.currentThread().interrupt();
			} finally {
				activeRequests.decrementAndGet();
			}
		});

		// Endpoint de monitoring amélioré
		server.createContext("/stats", exchange -> {
			String stats = String.format(
				"Active: %d, Queue: %d, Pool size: %d, Rejected: %d",
				executor.getActiveCount(),
				executor.getQueue().size(),
				executor.getPoolSize(),
				rejectedRequests.get()
			);

			exchange.sendResponseHeaders(200, stats.length());
			OutputStream os = exchange.getResponseBody();
			os.write(stats.getBytes());
			os.close();
		});

		server.start();
		System.out.println("🚀 Serveur démarré sur http://localhost:8080");
		System.out.println("   Thread pool: 5 threads");
		System.out.println("   Queue: 10 requêtes max");
		System.out.println("   Capacité totale: 15 requêtes simultanées");
		System.out.println();
		System.out.println("📍 Endpoints:");
		System.out.println("   GET http://localhost:8080/api/data");
		System.out.println("   GET http://localhost:8080/stats");
	}
}
```

```java

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class BadGuyWithLotOfRequestToDo {

	private static final String SERVER_URL = "http://localhost:8080/api/data";
	private static final int TOTAL_REQUESTS = 20; // 15 capacités + 5 rejets

	private static final AtomicInteger successCount = new AtomicInteger(0);
	private static final AtomicInteger errorCount = new AtomicInteger(0);

	public static void main(String[] args) throws InterruptedException {
		HttpClient httpClient = HttpClient.newBuilder()
			.connectTimeout(Duration.ofSeconds(10))
			.build();

		ExecutorService executor = Executors.newFixedThreadPool(20);

		System.out.println("🚀 Envoi de " + TOTAL_REQUESTS + " requêtes simultanées...");
		System.out.println("⏱️  Capacité serveur: 5 threads + 10 queue = 15 max");
		System.out.println();

		long startTime = System.currentTimeMillis();

		for (int i = 1; i <= TOTAL_REQUESTS; i++) {
			final int requestId = i;
			executor.submit(() -> {
				try {
					HttpRequest request = HttpRequest.newBuilder()
						.uri(URI.create(SERVER_URL))
						.GET()
						.build();

					HttpResponse<String> response = httpClient.send(
						request,
						HttpResponse.BodyHandlers.ofString()
					);

					if (response.statusCode() == 200) {
						successCount.incrementAndGet();
						System.out.printf("✅ Requête #%d OK (Thread: %s)%n",
							requestId, Thread.currentThread().getName());
					}
				} catch (Exception e) {
					errorCount.incrementAndGet();
					System.err.printf("❌ Requête #%d ERREUR: %s%n",
						requestId, e.getMessage());
				}
			});
		}

		executor.shutdown();
		executor.awaitTermination(1, TimeUnit.MINUTES);

		long duration = System.currentTimeMillis() - startTime;

		System.out.println("\n📊 Résultats:");
		System.out.println("   Succès: " + successCount.get());
		System.out.println("   Erreurs: " + errorCount.get());
		System.out.println("   Durée: " + duration + " ms");

		if (errorCount.get() > 0) {
			System.out.println("\n⚠️  Serveur saturé ! Des requêtes ont été rejetées.");
		}
	}

}
```

Cette exemple démontre clairement les limitations d'une architecture Thread-per-Request classique :
 - Le serveur ne peut gérer qu'un nombre limité de requêtes simultanées (15 dans cet exemple, volontairement bas).
 - Lorsque la charge dépasse cette limite, les requêtes supplémentaires sont rejetées.
---

## 3.2 Le coût du Context Switching intensif (POUR APPROFONDIR)

### Benchmark : Impact du nombre de threads

```java
import java.util.concurrent.*;
import java.util.List;
import java.util.ArrayList;

public class ContextSwitchingBenchmark {

	public static void main(String[] args) throws InterruptedException {

		System.out.println("=== Benchmark Context Switching ===\n");

		//On essaye de voir le temps de latence de traitement sur différents jalons
		int[] threadCounts = {10, 50, 100, 500, 1000, 5000, 10_000};

		System.out.println("Durée : Temps total (en millisecondes) pour que tous les threads terminent leur travail (calculs simple dans la boucle).");
		System.out.println("Opértion/sec : Nombre total d’opérations (incréments dans toutes les boucles) divisé par la durée du test en secondes. mesure finalement le débit du système");
		System.out.println("Overhead : Indicateur de surcharge basé sur le nombre de threads par rapport aux cœurs disponibles.");

		for (int numThreads : threadCounts) {
			benchmarkWithThreads(numThreads);
		}

	}

	private static void benchmarkWithThreads(int numThreads)
		throws InterruptedException {

		CountDownLatch startLatch = new CountDownLatch(1);
		CountDownLatch doneLatch = new CountDownLatch(numThreads);

		int incrementsPerThread = 100_000;

		List<Thread> threads = new ArrayList<>();

		// On commence par créer les threads
		for (int i = 0; i < numThreads; i++) {
			Thread t = new Thread(() -> {
				try {
					startLatch.await();

					//Juste un calcul pour simuler du travail
					long sum = 0;
					for (int j = 0; j < incrementsPerThread; j++) {
						sum += j;
					}

				} catch (InterruptedException e) {
					Thread.currentThread().interrupt();
				} finally {
					doneLatch.countDown();
				}
			});
			t.start();
			threads.add(t);
		}

		Thread.sleep(100);

		// Ici on démarre tous les threads
		long startTime = System.nanoTime();
		startLatch.countDown();

		doneLatch.await();
		long duration = System.nanoTime() - startTime;

		//On essaye de calculer un débit d'opérations
		long totalOperations = (long) numThreads * incrementsPerThread;
		double durationMs = duration / 1_000_000.0;
		double opsPerSec = totalOperations / (duration / 1_000_000_000.0);

		System.out.printf("Threads: %5d | Durée: %7.0f ms | " +
				"Opérations/sec: %12.0f | Overhead: %s%n",
			numThreads, durationMs, opsPerSec,
			getOverheadIndicator(numThreads)
		);

		// finito pipo ricardo
		for (Thread t : threads) {
			t.join();
		}
	}

	private static String getOverheadIndicator(int numThreads) {
		int cores = Runtime.getRuntime().availableProcessors();
		if (numThreads <= cores) return "✓ Optimal";
		if (numThreads <= cores * 2) return "⚠ Acceptable";
		if (numThreads <= cores * 10) return "⚠⚠ Élevé";
		return "❌ Critique";
	}
}

```
---

### Impact sur les architectures modernes

```
Application Microservice moderne:

Besoins réels:
• API Gateway: 10,000 connexions simultanées
• User Service: 5,000 connexions
• Order Service: 3,000 connexions
• Payment Service: 2,000 connexions
Total: 20,000 connexions simultanées

Avec Platform Threads:
┌────────────────────────────────────────┐
│ API Gateway                            │
│ • 200 threads → 200 connexions max     │
│ • Les 9,800 autres en QUEUE            │
└────────────────────────────────────────┘
┌────────────────────────────────────────┐
│ User Service                           │
│ • 200 threads → 200 connexions max     │
│ • Les 4,800 autres en QUEUE            │
└────────────────────────────────────────┘

Solution actuelle (coûteuse):
• Horizontal Scaling: 50 instances pour Gateway
• 25 instances pour User Service
• Coût cloud: $$$$$
• Complexité: Load balancing, orchestration
```

---

## 3.3 I/O Bloquant : Le grand gaspillage (Pour approfondir)

### Analyse d'une application réelle

```java
import java.sql.*;
import java.net.URI;
import java.net.http.*;
import java.time.Duration;

public class RealWorldScenario {
    
    private static final HttpClient httpClient = HttpClient.newHttpClient();
    
    public static void main(String[] args) throws Exception {
        
        System.out.println("=== Simulation application réelle ===\n");
        
        // Profiler une requête typique
        profileRequest();
    }
    
    private static void profileRequest() throws Exception {
        
        long totalStart = System.nanoTime();
        long lastCheckpoint = totalStart;
        
        System.out.println("[Phase 1] Validation paramètres");
        Thread.sleep(5); // Simulation
        printDuration("Validation", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 2] Requête base de données");
        // Simulation JDBC
        Thread.sleep(50);
        printDuration("DB Query", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 3] Appel API externe (auth)");
        // Simulation HTTP
        Thread.sleep(120);
        printDuration("Auth API", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 4] Traitement business logic");
        Thread.sleep(10);
        printDuration("Business Logic", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 5] Requête cache (Redis)");
        Thread.sleep(15);
        printDuration("Cache", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 6] Appel API externe (notification)");
        Thread.sleep(80);
        printDuration("Notification API", lastCheckpoint);
        lastCheckpoint = System.nanoTime();
        
        System.out.println("\n[Phase 7] Sérialisation réponse");
        Thread.sleep(5);
        printDuration("Serialization", lastCheckpoint);
        
        long totalDuration = System.nanoTime() - totalStart;
        
        System.out.println("\n" + "=".repeat(50));
        System.out.println("DURÉE TOTALE: " + totalDuration / 1_000_000 + " ms");
        System.out.println("=".repeat(50));
        
        // Analyse
        analyzeEfficiency();
    }
    
    private static void printDuration(String phase, long lastCheckpoint) {
        long duration = (System.nanoTime() - lastCheckpoint) / 1_000_000;
        System.out.printf("  ✓ %s: %d ms%n", phase, duration);
    }
    
    private static void analyzeEfficiency() {
        System.out.println("\nAnalyse d'efficacité:");
        System.out.println("┌─────────────────────────┬──────────┬────────────┐");
        System.out.println("│ Phase                   │ Durée    │ Type       │");
        System.out.println("├─────────────────────────┼──────────┼────────────┤");
        System.out.println("│ Validation              │    5 ms  │ CPU ✓      │");
        System.out.println("│ DB Query                │   50 ms  │ I/O ⚠      │");
        System.out.println("│ Auth API                │  120 ms  │ I/O ⚠      │");
        System.out.println("│ Business Logic          │   10 ms  │ CPU ✓      │");
        System.out.println("│ Cache                   │   15 ms  │ I/O ⚠      │");
        System.out.println("│ Notification API        │   80 ms  │ I/O ⚠      │");
        System.out.println("│ Serialization           │    5 ms  │ CPU ✓      │");
        System.out.println("├─────────────────────────┼──────────┼────────────┤");
        System.out.println("│ TOTAL                   │  285 ms  │            │");
        System.out.println("└─────────────────────────┴──────────┴────────────┘");
        
        System.out.println("\n📊 Répartition:");
        System.out.println("  • Temps CPU actif:  20 ms (  7%)  ← Travail utile");
        System.out.println("  • Temps I/O bloqué: 265 ms ( 93%)  ← GASPILLAGE!");
        
        System.out.println("\n⚠️  Problème:");
        System.out.println("  Le thread Platform reste BLOQUÉ pendant 265ms");
        System.out.println("  → 2 MB de mémoire inutilisés");
        System.out.println("  → 1 slot du thread pool monopolisé");
        System.out.println("  → CPU idle");
        
        System.out.println("\n💡 Avec 200 threads Platform:");
        System.out.println("  • Si 200 requêtes arrivent simultanément");
        System.out.println("  • 200 threads tous BLOQUÉS 93% du temps");
        System.out.println("  • CPU utilization: ~7%");
        System.out.println("  • Mais impossible d'accepter plus de requêtes!");
    }
}

/* Output:

=== Simulation application réelle ===

[Phase 1] Validation paramètres
  ✓ Validation: 5 ms

[Phase 2] Requête base de données
  ✓ DB Query: 50 ms

[Phase 3] Appel API externe (auth)
  ✓ Auth API: 120 ms

[Phase 4] Traitement business logic
  ✓ Business Logic: 10 ms

[Phase 5] Requête cache (Redis)
  ✓ Cache: 15 ms

[Phase 6] Appel API externe (notification)
  ✓ Notification API: 80 ms

[Phase 7] Sérialisation réponse
  ✓ Serialization: 5 ms

==================================================
DURÉE TOTALE: 285 ms
==================================================

Analyse d'efficacité:
┌─────────────────────────┬──────────┬────────────┐
│ Phase                   │ Durée    │ Type       │
├─────────────────────────┼──────────┼────────────┤
│ Validation              │    5 ms  │ CPU ✓      │
│ DB Query                │   50 ms  │ I/O ⚠      │
│ Auth API                │  120 ms  │ I/O ⚠      │
│ Business Logic          │   10 ms  │ CPU ✓      │
│ Cache                   │   15 ms  │ I/O ⚠      │
│ Notification API        │   80 ms  │ I/O ⚠      │
│ Serialization           │    5 ms  │ CPU ✓      │
├─────────────────────────┼──────────┼────────────┤
│ TOTAL                   │  285 ms  │            │
└─────────────────────────┴──────────┴────────────┘

📊 Répartition:
  • Temps CPU actif:  20 ms (  7%)  ← Travail utile
  • Temps I/O bloqué: 265 ms ( 93%)  ← GASPILLAGE!

⚠️  Problème:
  Le thread Platform reste BLOQUÉ pendant 265ms
  → 2 MB de mémoire inutilisés
  → 1 slot du thread pool monopolisé
  → CPU idle

💡 Avec 200 threads Platform:
  • Si 200 requêtes arrivent simultanément
  • 200 threads tous BLOQUÉS 93% du temps
  • CPU utilization: ~7%
  • Mais impossible d'accepter plus de requêtes!
*/
```

---

## 3.4 Cas d'usage critiques (Pour approfondir)

### 3.4.1 WebSocket et Connexions longues

```java
import javax.websocket.*;
import javax.websocket.server.ServerEndpoint;
import java.util.Set;
import java.util.concurrent.CopyOnWriteArraySet;

@ServerEndpoint("/chat")
public class ChatWebSocket {
    
    private static final Set<Session> sessions = new CopyOnWriteArraySet<>();
    
    @OnOpen
    public void onOpen(Session session) {
        sessions.add(session);
        System.out.println("Nouvelle connexion: " + session.getId());
        System.out.println("Connexions actives: " + sessions.size());
        
        // PROBLÈME CRITIQUE:
        // Chaque connexion WebSocket = 1 thread Platform BLOQUÉ
        // Le thread reste ouvert tant que la connexion est active
        // Un websocket reste ouvert sur des périodes de temps extrêmement longue 
        
        // Avec 200 threads max:
        // 200 connexions WebSocket = thread pool SATURÉ
        // → Plus aucune requête HTTP ne peut être traitée!
    }
    
    @OnMessage
    public void onMessage(String message, Session session) {
        sessions.forEach(s -> {
            try {
                s.getBasicRemote().sendText(message);
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
    }
    
    @OnClose
    public void onClose(Session session) {
        sessions.remove(session);
        System.out.println("Connexion fermée: " + session.getId());
        System.out.println("Connexions restantes: " + sessions.size());
    }
}

/*
Scénario catastrophe:

1. Serveur avec 200 threads Platform
2. 150 clients WebSocket se connectent
3. → 150 threads Platform BLOQUÉS indéfiniment
4. → Il reste 50 threads pour les requêtes HTTP
5. → Si 50+ requêtes HTTP arrivent → QUEUE
6. → Latence explose pour les requêtes normales

Solution actuelle: Serveur dédié pour WebSocket
Coût: Infrastructure additionnelle, complexité
*/
```

### 3.4.2 Traitement Batch massivement parallèle (Pour approfondir)

```java
import java.util.List;
import java.util.ArrayList;
import java.util.concurrent.*;

public class BatchProcessingProblem {
    
    public static void main(String[] args) throws Exception {
        
        // Scénario: Traiter 100,000 enregistrements comme on pourrait le faire en entreprise sur un batch
        List<Integer> records = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            records.add(i);
        }
        
        System.out.println("Approche 1: Pool fixe de 50 threads");
        long start = System.currentTimeMillis();
        
        ExecutorService executor = Executors.newFixedThreadPool(50);
        List<Future<String>> futures = new ArrayList<>();
        
        for (Integer record : records) {
            futures.add(executor.submit(() -> processRecord(record)));
        }
        
        for (Future<String> future : futures) {
            future.get();
        }
        
        executor.shutdown();
        long duration1 = System.currentTimeMillis() - start;
        
        System.out.println("Temps: " + duration1 + " ms");
        System.out.println("Throughput: " + (100_000.0 / duration1 * 1000) + " records/sec");
        
       System.out.println("\n---------------------------------------------------");
       System.out.println("\n---------------------------------------------------");
        System.out.println("\nApproche 2: Pool de 500 threads");
        start = System.currentTimeMillis();
        
        executor = Executors.newFixedThreadPool(500);
        futures = new ArrayList<>();
        
        for (Integer record : records) {
            futures.add(executor.submit(() -> processRecord(record)));
        }
        
        for (Future<String> future : futures) {
            future.get();
        }
        
        executor.shutdown();
        long duration2 = System.currentTimeMillis() - start;
        
        System.out.println("Temps: " + duration2 + " ms");
        System.out.println("Throughput: " + (100_000.0 / duration2 * 1000) + " records/sec");
        
        
        System.out.println("\n=== Analyse ===");
        System.out.println("50 threads:  " + duration1 + " ms");
        System.out.println("500 threads: " + duration2 + " ms");
        
        if (duration2 < duration1) {
            double improvement = ((duration1 - duration2) / (double) duration1) * 100;
            System.out.println("Amélioration: " + String.format("%.1f", improvement) + "%");
        } else {
            System.out.println("Plus de threads = PLUS LENT!");
            System.out.println("Raison: Context switching overhead > gain parallélisme");
        }
        
        System.out.println("\n Problème:");
        System.out.println("• 500 threads = 1 GB de mémoire (stack)");
        System.out.println("• Context switching intensif");
        System.out.println("• Pas scalable à 1 million de records");
    }
    
    private static String processRecord(int recordId) {
        try {
            // Simulation: I/O bloquant (DB write, API call)
            Thread.sleep(100);
            return "Processed: " + recordId;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "Error: " + recordId;
        }
    }
}

/* Output typique:

=== Traitement avec Platform Threads ===

Approche 1: Pool fixe de 50 threads
Temps: 200800 ms (≈ 3min 20s)
Throughput: 498.0 records/sec

Approche 2: Pool de 500 threads
Temps: 21200 ms (≈ 21s)
Throughput: 4716.9 records/sec

=== Analyse ===
50 threads:  200800 ms
500 threads: 21200 ms
Amélioration: 89.4%

Problème:
• 500 threads = 1 GB de mémoire (stack)
• Context switching intensif
• Pas scalable à 1 million de records

Avec 1 million de records:
• 500 threads → 200+ secondes
• Pour aller plus vite → plus de threads
• Mais 5000 threads = OutOfMemoryError!
*/
```

### 3.4.3 Microservices avec appels en cascade (Pour approfondir)

```java
import java.net.http.*;
import java.net.URI;
import java.util.concurrent.*;

public class MicroserviceCascade {
    
    private static final HttpClient httpClient = HttpClient.newHttpClient();
    
    public static void main(String[] args) throws Exception {
        
        System.out.println("=== Simulation cascade microservices ===\n");
        
        // Service A appelle Service B qui appelle Service C
        String result = serviceA();
        System.out.println("Résultat: " + result);
    }
    
    // Service A: API Gateway
    private static String serviceA() throws Exception {
        System.out.println("[Service A] Requête reçue");
        long start = System.nanoTime();
        
        // Appel Service B (bloque le thread)
        String resultB = serviceB();
        
        // Traitement
        Thread.sleep(10);
        
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("[Service A] Terminé en " + duration + "ms");
        
        return "A(" + resultB + ")";
    }
    
    // Service B: User Service
    private static String serviceB() throws Exception {
        System.out.println("  [Service B] Requête reçue");
        long start = System.nanoTime();
        
        // Appel Service C (bloque le thread)
        String resultC = serviceC();
        
        // Traitement
        Thread.sleep(15);
        
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("  [Service B] Terminé en " + duration + "ms");
        
        return "B(" + resultC + ")";
    }
    
    // Service C: Database Service
    private static String serviceC() throws Exception {
        System.out.println("    [Service C] Requête reçue");
        long start = System.nanoTime();
        
        // Requête DB (bloque le thread)
        Thread.sleep(80);
        
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("    [Service C] Terminé en " + duration + "ms");
        
        return "C(data)";
    }
}

/* Output:

=== Simulation cascade microservices ===

[Service A] Requête reçue
  [Service B] Requête reçue
    [Service C] Requête reçue
    [Service C] Terminé en 80ms
  [Service B] Terminé en 95ms
[Service A] Terminé en 105ms
Résultat: A(B(C(data)))

Analyse du problème:

Timeline des threads:

Service A thread: [████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██]
                   ▲                                  ▲
                   Actif                           Actif
                   │← Bloqué en attente Service B →│

Service B thread: [████░░░░░░░░░░░░░░░░░░░░░░░░░██]
                   ▲                            ▲
                   Actif                     Actif
                   │← Bloqué attente Service C →│

Service C thread: [████░░░░░░░░░░░░░░░░░░░░░░██]
                   ▲                        ▲
                   Actif                 Actif
                   │← Bloqué sur DB →│

Problème:
• 3 threads Platform mobilisés pour 1 requête
• Chaque thread bloqué 90% du temps
• Avec 1000 requêtes simultanées → 3000 threads!
• OutOfMemoryError garanti

Impact production:
Service A: 200 threads → max 200 requêtes/sec
Service B: 200 threads → max 200 requêtes/sec
Service C: 200 threads → max 200 requêtes/sec

Si Service C est lent (spike de latence):
→ Tous les threads de A et B se bloquent
→ Effet domino sur toute l'architecture
→ Timeout en cascade
*/
```

---

## 3.5 Les "Solutions" actuelles et leurs limites

### 3.5.1 Augmenter le nombre de threads

```java
// Configuration Tomcat typique
server.tomcat.threads.max=1000  // Au lieu de 200

/* Problèmes:
Consommation mémoire: 1000 × 2MB = 2GB juste pour les stacks
Context switching: Performance dégradée
Toujours une limite: 1000 threads = 1000 connexions max
Si 2000 requêtes simultanées → toujours un problème!
Ne scale pas indéfiniment (OutOfMemoryError)

Résultat:
• Coût mémoire élevé
• Performance dégradée (context switching)
• Problème non résolu, juste retardé
*/
```

### 3.5.2 Programmation Réactive (WebFlux)

```java
// Avec Reactor (WebFlux)
public Mono<OrderDetails> getOrderDetails(Long orderId) {
    return orderRepository.findById(orderId)
        .flatMap(order -> 
            Mono.zip(
                userService.getUser(order.getUserId()),
                paymentService.getPayment(orderId),
                inventoryService.checkStock(orderId)
            ).map(tuple -> new OrderDetails(
                order,
                tuple.getT1(),
                tuple.getT2(),
                tuple.getT3()
            ))
        );
}

/* Avantages:
- Non-bloquant: excellent throughput
- Peu de threads nécessaires
- Scale très bien

Inconvénients:
- Courbe d'apprentissage TRÈS raide
- Debugging cauchemardesque:
   Stack traces incompréhensibles avec 50 niveaux
- "Viral": tout le code doit devenir réactif
   Si une lib n'est pas réactive → bloque tout
- Code complexe pour des opérations simples
- Nécessite drivers réactifs (R2DBC vs JDBC)
- Beaucoup de libs Java ne supportent pas le réactif
- Gestion d'erreur complexe
*/
```

### 3.5.3 Async/CompletableFuture

```java
// Avec CompletableFuture
public CompletableFuture<OrderDetails> getOrderDetails(Long orderId) {
    return orderRepository.findByIdAsync(orderId)
        .thenCompose(order -> 
            CompletableFuture.allOf(
                userService.getUserAsync(order.getUserId()),
                paymentService.getPaymentAsync(orderId)
            ).thenApply(v -> new OrderDetails(order, /*...*/ ))
        );
}

/* Avantages:
- Non-bloquant
- API standard Java
- Meilleur que les threads bloquants

Inconvénients:
- Code verbeux et complexe
- Gestion erreur difficile (exceptionally, handle)
- Composition compliquée (thenCompose, thenCombine, etc.)
- Debug difficile
- Toujours limité par le thread pool sous-jacent
- "Viral": propage la complexité

Exemple de code pour 3 appels parallèles + 1 séquentiel:
CompletableFuture.supplyAsync(() -> getUser())
    .thenCompose(user -> 
        CompletableFuture.allOf(
            CompletableFuture.supplyAsync(() -> getOrders(user)),
            CompletableFuture.supplyAsync(() -> getPayments(user)),
            CompletableFuture.supplyAsync(() -> getPreferences(user))
        ).thenApply(v -> combine(user, ...))
    ).exceptionally(ex -> handleError(ex))
     .thenAccept(result -> sendResponse(result));

Code synchrone équivalent :
User user = getUser();
Orders orders = getOrders(user);
Payments payments = getPayments(user);
Preferences prefs = getPreferences(user);
Result result = combine(user, orders, payments, prefs);
sendResponse(result);


*/
```

### 3.5.4 Horizontal Scaling (plus de serveurs)

```
Solution actuelle en production:

┌──────────────────────────────────────────┐
│ Load Balancer                            │
└─────┬────────────────────────────────────┘
      │
      ├─► Instance 1 (200 threads)
      ├─► Instance 2 (200 threads)
      ├─► Instance 3 (200 threads)
      ├─► Instance 4 (200 threads)
      ├─► Instance 5 (200 threads)
      ├─► ... (50 instances)
      └─► Instance 50 (200 threads)

Total: 10,000 threads "effectifs"
Coût: 50 instances EC2 m5.xlarge
Prix: ~$3,500/mois sur AWS

/* Problèmes:
- Coût élevé: $42,000/an
- Complexité: Load balancing, orchestration, monitoring
- Latence: Session affinity, cross-instance calls
- Waste: Chaque instance sous-utilisée (CPU à 10%)
- Management: Déploiements, mises à jour, scaling

Alternative avec Virtual Threads:
- 5 instances seulement (même capacité)
- Coût: ~$350/mois ($4,200/an)
- Économie: $37,800/an (90% de réduction!)
- Complexité réduite
- Meilleure utilisation CPU (70-80%)
*/
```

---

## 3.6 Tableau récapitulatif des limitations

```
┌─────────────────────────────────────────────────────────────────┐
│          Limitations des Platform Threads                       │
├─────────────────────┬───────────────────────────────────────────┤
│ Limitation          │ Impact                                    │
├─────────────────────┼───────────────────────────────────────────┤
│ Mémoire             │ • 2 MB par thread                         │
│                     │ • Max ~5000 threads (16GB RAM)            │
│                     │ • OutOfMemoryError au-delà                │
├─────────────────────┼───────────────────────────────────────────┤
│ Context Switching   │ • Overhead 10-70% avec 1000+ threads      │
│                     │ • CPU time gaspillé                       │
│                     │ • Cache thrashing                         │
├─────────────────────┼───────────────────────────────────────────┤
│ Création            │ • 0.5-1 ms par thread                     │
│                     │ • Coûteux pour short-lived tasks          │
│                     │ • Thread pools requis                     │
├─────────────────────┼───────────────────────────────────────────┤
│ Blocking I/O        │ • 70-95% du temps en attente              │
│                     │ • Thread inutilisé mais                   │
│                     │      consomme des ressources              │            
│                     │ • Throughput limité                       │
├─────────────────────┼───────────────────────────────────────────┤
│ Scalabilité         │ • Thread-per-request → max connections    │
│                     │ • WebSocket → thread pool saturation      │
│                     │ • Batch processing limité                 │
├─────────────────────┼───────────────────────────────────────────┤
│ Coût Cloud          │ • Horizontal scaling coûteux              │
│                     │ • Sous-utilisation CPU (10-20%)           │
│                     │ • Gaspillage ressources                   │
└─────────────────────┴───────────────────────────────────────────┘
```
### Comparaison des "solutions"
```
┌─────────────────────┬─────────────┬─────────────┬──────────────┬─────────────┐
│ Solution            │ Complexité  │ Performance │ Scalabilité  │ Maintenance │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ Plus de threads     │ BON         │ PASSABLE    │ MAUVAIS      │ PASSABLE    │
│ (1000+ threads)     │ Simple      │ Limité      │ OOM rapide   │ Config      │
│                     │             │ context     │              │ difficile   │
│                     │             │ switching   │              │             │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ Thread Pools        │ BON         │ BON         │ PASSABLE     │ BON         │
│ (fixed size)        │ Standard    │ Réutilise   │ Toujours     │ APIs        │
│                     │ Java        │ threads     │ limité       │ simples     │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ CompletableFuture   │ PASSABLE    │ BON         │ BON          │ PASSABLE    │
│ (async)             │ Verbeux     │ Non-        │ Pool sous-   │ Callbacks   │
│                     │ Callbacks   │ bloquant    │ jacent       │ complexes   │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ Reactive (WebFlux)  │ MAUVAIS     │ EXCELLENT   │ EXCELLENT    │ MAUVAIS     │
│ Reactor/RxJava      │ Très raide  │ Throughput  │ Très peu     │ Debug       │
│                     │ Viral       │ maximal     │ de threads   │ difficile   │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ Horizontal          │ PASSABLE    │ BON         │ EXCELLENT    │ PASSABLE    │
│ Scaling             │ Orchestr.   │ Linéaire    │ Ajouter      │ Déploiements│
│                     │ nécessaire  │             │ instances    │ multiples   │
├─────────────────────┼─────────────┼─────────────┼──────────────┼─────────────┤
│ Virtual Threads     │ EXCELLENT   │ EXCELLENT   │ EXCELLENT    │ EXCELLENT   │
│ (Java 21)           │ Code sync   │ Non-        │ Millions     │ Code simple │
│                     │ simple      │ bloquant    │ de threads   │ Debug std   │
└─────────────────────┴─────────────┴─────────────┴──────────────┴─────────────┘

Légende:
• MAUVAIS : Inadapté ou problématique
• PASSABLE : Fonctionne mais avec compromis
• BON : Satisfaisant pour la plupart des cas
• EXCELLENT : Optimal, sans compromis significatif
```

---


## 3.7 Métriques réelles : Avant LTS21/25 et VT

### Application E-commerce typique

```
Profil d'une application e-commerce classique:

┌────────────────────────────────────────────────┐
│ Métriques Production (Platform Threads)        │
├────────────────────────────────────────────────┤
│                                                │
│ Infrastructure:                                │
│ • 20 instances EC2 m5.xlarge (4 vCPU, 16GB)    │
│ • Thread pool: 200 threads par instance        │
│ • Total: 4,000 threads "effectifs"             │
│                                                │
│ Performance:                                   │
│ • Peak load: 500 req/sec                       │
│ • Latence p50: 250ms                           │
│ • Latence p99: 1,200ms                         │
│ • CPU utilization: 15-25%                      │
│ • Memory usage: 60%                            │
│                                                │
│ Coûts mensuels:                                │
│ • EC2: 20 × $70 = $1,400/mois                  │
│ • Load Balancer: $100/mois                     │
│ • Total: $1,500/mois                           │
│                                                │
│ Problèmes:                                     │
│ • Sous-utilisation CPU massive (75% idle)      │
│ • Latence p99 élevée (queueing)                │
│ • Coûts élevés pour la capacité fournie        │
│ • Impossible de gérer plus de 500 req/sec      │
│   sans ajouter des instances                   │
│                                                │
└────────────────────────────────────────────────┘
```

### Analyse d'une journée type

```
Load profile sur 24h:

Requêtes/sec
    1000│                    ╭─╮
        │                  ╭─╯ ╰─╮
     500│       ╭──────────╯     ╰────╮
        │     ╭─╯                     ╰─╮
     250│  ╭──╯                         ╰──╮
        │╭─╯                               ╰─╮
       0└┴───┴───┴───┴───┴───┴───┴───┴───┴───┴─
        0h  4h  8h  12h 16h 20h 24h

Observations:
• Peak: 16h-18h (rentrée bureau) → 600 req/sec
• Avec 4,000 threads → proche saturation
• Night: 0h-6h → 50 req/sec
• 3,900 threads INUTILISÉS mais coûtent!

Avec Virtual Threads:
• Même infrastructure
• Peut gérer 5,000+ req/sec
• Coût identique
• Ou: diviser par 10 le nombre d'instances
  → $150/mois au lieu de $1,500/mois
```

---

## 3.8 Résumé : Le mur des Platform Threads

### Les chiffres qui font mal

```
Limites dures des Platform Threads:

Avec Platform Threads, choisissez 2 sur 3:        │
│  • Simple + Performant = Pas scalable           │
│  • Scalable + Performant = Complexe (reactive)  │
│  • Simple + Scalable = Pas performant 

┌──────────────────────────┬──────────────────┐
│ Métrique                 │ Limite Platform  │
├──────────────────────────┼──────────────────┤
│ Threads par machine      │ ~5,000 max       │
│ Connexions simultanées   │ ~5,000 max       │
│ Mémoire par thread       │ 2 MB             │
│ Context switch overhead  │ 10-70% (>1000)   │
│ Temps création           │ 0.5 ms           │
│ Efficacité I/O bloquant  │ 5-30% CPU usage  │
│ Coût horizontal scaling  │ $$             │
└──────────────────────────┴──────────────────┘

Ce que ça signifie en production:
- Impossible de gérer 100,000 connexions WebSocket
- Batch de 1M records → 30+ minutes ou OOM
- Microservices → cascade de blocages
- CPU idle à 90% mais "pas de capacité"
- Coûts cloud × 10 pour compenser

La conclusion est claire:
Le modèle Platform Thread ne peut PAS scale
pour les applications modernes.

➡️  Solution: Virtual Threads (chapitre suivant)
```

### Points clés à retenir

```
✅ Ce qu'il faut retenir sur les limitations:

1. Thread-per-request ne scale pas
   • Max 5,000 connexions simultanées
   • Au-delà: queue ou crash
   
2. I/O bloquant = gaspillage massif
   • 70-95% du temps en attente
   • CPU idle mais capacité saturée
   
3. Context switching = overhead critique
   • >1,000 threads → 50% overhead
   • Performance s'effondre
   
4. Mémoire = limite dure
   • 2 MB par thread
   • OutOfMemoryError inévitable
   
5. Les "solutions" actuelles ont des compromis
   • Reactive: complexe
   • Async: verbeux
   • Scaling: coûteux
   
6. Le problème est fondamental
   • Pas un bug à fixer
   • Mais une limite architecturale
   • Nécessite une nouvelle approche

Enfin nous y arrivons : Virtual Threads

   • Millions de threads possibles
   • Code simple (synchrone)
   • Performance excellente
   • Sans les compromis!
```

---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [⬅️ Précédent](02-jvm-platform-threads.md) | [➡️ Suivant: Introduction aux Virtual Threads](04-virtual-threads-intro.md)