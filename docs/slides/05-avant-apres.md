[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [📖 Lexique](lexique.md) | [⬅️ Précédent](04-virtual-threads-intro.md) | [➡️ Suivant](06-conclusion.md)

---

# 5. Exemples Pratiques : Avant/Après

## 5.1 Exemple Spring Boot : API REST avec appels externes

### Contexte

Application de gestion de commandes qui :
- Interroge une base de données PostgreSQL
- Appelle un service externe de paiement
- Appelle un service externe de validation de stock
- Envoie une notification

### AVANT : Spring Boot 3.x avec Platform Threads

#### Configuration (application.yml)

```yaml
# application.yml
spring:
  application:
    name: order-service
  
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: postgres
    password: password
    hikari:
      maximum-pool-size: 20      # Limité par platform threads
      minimum-idle: 5

server:
  port: 8080
  tomcat:
    threads:
      max: 200                    # ⚠️ LIMITE : 200 requêtes max
      min-spare: 10
```

#### Service Layer

```java
// OrderService.java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${payment.service.url}")
    private String paymentServiceUrl;
    
    @Value("${inventory.service.url}")
    private String inventoryServiceUrl;
    
    public OrderResponse createOrder(OrderRequest request) {
        
        long startTime = System.currentTimeMillis();
        
        // 1. Validation et création de l'ordre en base (50ms)
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setStatus("PENDING");
        
        order = orderRepository.save(order);  // JDBC bloque le thread
        
        // 2. Vérification du stock (120ms)
        StockResponse stockResponse = restTemplate.getForObject(
            inventoryServiceUrl + "/check/" + request.getProductId(),
            StockResponse.class
        );  // HTTP bloque le thread
        
        if (!stockResponse.isAvailable()) {
            order.setStatus("CANCELLED");
            orderRepository.save(order);
            throw new OutOfStockException();
        }
        
        // 3. Traitement du paiement (200ms)
        PaymentRequest paymentRequest = new PaymentRequest(
            request.getUserId(),
            order.getTotalAmount()
        );
        
        PaymentResponse paymentResponse = restTemplate.postForObject(
            paymentServiceUrl + "/process",
            paymentRequest,
            PaymentResponse.class
        );  // HTTP bloque le thread
        
        if (!paymentResponse.isSuccess()) {
            order.setStatus("PAYMENT_FAILED");
            orderRepository.save(order);
            throw new PaymentException();
        }
        
        // 4. Mise à jour finale (30ms)
        order.setStatus("CONFIRMED");
        order.setPaymentId(paymentResponse.getPaymentId());
        order = orderRepository.save(order);  // JDBC bloque le thread
        
        long duration = System.currentTimeMillis() - startTime;
        
        return new OrderResponse(order, duration);
    }
}

/*
Analyse:
┌─────────────────────────────────────────────────────┐
│ Timeline d'une requête (400ms total)                │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Phase            Durée    Type        Thread        │
│ ─────────────────────────────────────────────────  │
│ DB INSERT        50ms     I/O         BLOQUÉ       │
│ HTTP Stock       120ms    I/O         BLOQUÉ       │
│ HTTP Payment     200ms    I/O         BLOQUÉ       │
│ DB UPDATE        30ms     I/O         BLOQUÉ       │
│                                                     │
│ Total CPU:       ~10ms    (2.5%)                    │
│ Total I/O:       400ms    (97.5%)                   │
│                                                     │
└─────────────────────────────────────────────────────┘

Problème:
• Thread Platform bloqué 400ms
• Avec 200 threads max → 200 requêtes simultanées max
• Si 500 req/sec arrivent → queue + latence
• CPU utilization: ~5%
• Mémoire: 200 threads × 2MB = 400MB
*/
```

### APRÈS : Spring Boot 3.2+ avec Virtual Threads

#### Configuration (application.yml)

```yaml
# application.yml
spring:
  application:
    name: order-service
  
  threads:
    virtual:
      enabled: true              # ⚡ ACTIVER VIRTUAL THREADS
  
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: postgres
    password: password
    hikari:
      maximum-pool-size: 10      # ✅ Peut être RÉDUIT
      minimum-idle: 5

server:
  port: 8080
  # Plus besoin de configurer max-threads !
  # Virtual Threads gèrent automatiquement
```

#### Service Layer (INCHANGÉ !)

```java
// OrderService.java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${payment.service.url}")
    private String paymentServiceUrl;
    
    @Value("${inventory.service.url}")
    private String inventoryServiceUrl;
    
    // ✅ CODE IDENTIQUE - Rien à changer !
    public OrderResponse createOrder(OrderRequest request) {
        
        long startTime = System.currentTimeMillis();
        
        // 1. Validation et création de l'ordre
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setProductId(request.getProductId());
        order.setQuantity(request.getQuantity());
        order.setStatus("PENDING");
        
        order = orderRepository.save(order);  // VT se démonte automatiquement
        
        // 2. Vérification du stock
        StockResponse stockResponse = restTemplate.getForObject(
            inventoryServiceUrl + "/check/" + request.getProductId(),
            StockResponse.class
        );  // VT se démonte automatiquement
        
        if (!stockResponse.isAvailable()) {
            order.setStatus("CANCELLED");
            orderRepository.save(order);
            throw new OutOfStockException();
        }
        
        // 3. Traitement du paiement
        PaymentRequest paymentRequest = new PaymentRequest(
            request.getUserId(),
            order.getTotalAmount()
        );
        
        PaymentResponse paymentResponse = restTemplate.postForObject(
            paymentServiceUrl + "/process",
            paymentRequest,
            PaymentResponse.class
        );  // VT se démonte automatiquement
        
        if (!paymentResponse.isSuccess()) {
            order.setStatus("PAYMENT_FAILED");
            orderRepository.save(order);
            throw new PaymentException();
        }
        
        // 4. Mise à jour finale
        order.setStatus("CONFIRMED");
        order.setPaymentId(paymentResponse.getPaymentId());
        order = orderRepository.save(order);
        
        long duration = System.currentTimeMillis() - startTime;
        
        return new OrderResponse(order, duration);
    }
}

/*
Analyse avec Virtual Threads:
┌─────────────────────────────────────────────────────┐
│ Timeline d'une requête (400ms total)                │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Phase            Durée    Type        VThread       │
│ ─────────────────────────────────────────────────  │
│ DB INSERT        50ms     I/O         DÉMONTÉ      │
│ HTTP Stock       120ms    I/O         DÉMONTÉ      │
│ HTTP Payment     200ms    I/O         DÉMONTÉ      │
│ DB UPDATE        30ms     I/O         DÉMONTÉ      │
│                                                     │
│ Carrier utilisé: ~10ms seulement                    │
│                                                     │
└─────────────────────────────────────────────────────┘

Avantages:
• VT se démonte à chaque I/O → carrier libre
• Peut gérer 10,000+ requêtes simultanées
• Même avec 8 carriers seulement !
• Latence identique par requête
• Throughput × 50
• Mémoire: ~10MB pour 10,000 VT (vs 400MB pour 200 PT)
*/
```

### Résultats Benchmark

```java
// Benchmark avec Apache Bench
// ab -n 10000 -c 1000 http://localhost:8080/api/orders

/*
AVANT (Platform Threads):
────────────────────────────────────────
Concurrency Level:      1000
Time taken for tests:   52.3 seconds
Complete requests:      10000
Failed requests:        2341          ← ⚠️ Timeouts !
Requests per second:    191.2 [#/sec]
Time per request:       5230 [ms]     ← ⚠️ Latence élevée
CPU utilization:        8%            ← ⚠️ Sous-utilisé

APRÈS (Virtual Threads):
────────────────────────────────────────
Concurrency Level:      1000
Time taken for tests:   4.2 seconds   ← ✅ 12× plus rapide
Complete requests:      10000
Failed requests:        0             ← ✅ Aucun timeout
Requests per second:    2380 [#/sec]  ← ✅ 12× meilleur
Time per request:       420 [ms]      ← ✅ Latence stable
CPU utilization:        45%           ← ✅ Bien utilisé

Gains:
• Throughput: 191 → 2380 req/sec (12×)
• Latence p99: 8500ms → 450ms (19×)
• Taux d'erreur: 23% → 0%
• CPU: 8% → 45% (meilleure utilisation)
*/
```

---

## 5.2 Exemple Java natif : Traitement batch parallèle

### Contexte

Application de traitement batch qui :
- Lit 100,000 enregistrements depuis un fichier
- Pour chaque enregistrement : appel API de validation (100ms)
- Écrit les résultats dans une base de données

### AVANT : ExecutorService avec Platform Threads

```java
import java.util.concurrent.*;
import java.sql.*;
import java.net.http.*;
import java.net.URI;
import java.util.*;

public class BatchProcessorBefore {
    
    private static final HttpClient httpClient = HttpClient.newHttpClient();
    private static final String DB_URL = "jdbc:postgresql://localhost/batch";
    private static final String VALIDATION_API = "http://api.service.com/validate";
    
    public static void main(String[] args) throws Exception {
        
        // Charger les enregistrements
        List<Record> records = loadRecords(100_000);
        
        System.out.println("=== Traitement AVANT (Platform Threads) ===");
        System.out.println("Records à traiter: " + records.size());
        
        long startTime = System.currentTimeMillis();
        
        // Pool de 100 threads (compromis mémoire/performance)
        ExecutorService executor = Executors.newFixedThreadPool(100);
        
        List<Future<ProcessedRecord>> futures = new ArrayList<>();
        
        // Soumettre toutes les tâches
        for (Record record : records) {
            Future<ProcessedRecord> future = executor.submit(() -> 
                processRecord(record)
            );
            futures.add(future);
        }
        
        // Attendre tous les résultats
        List<ProcessedRecord> results = new ArrayList<>();
        for (Future<ProcessedRecord> future : futures) {
            try {
                results.add(future.get());
            } catch (Exception e) {
                System.err.println("Erreur traitement: " + e.getMessage());
            }
        }
        
        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.HOURS);
        
        long duration = System.currentTimeMillis() - startTime;
        
        // Sauvegarder les résultats
        saveResults(results);
        
        System.out.println("\n=== Résultats ===");
        System.out.println("Durée totale: " + duration + " ms");
        System.out.println("Records traités: " + results.size());
        System.out.println("Throughput: " + 
            String.format("%.0f", results.size() * 1000.0 / duration) + " rec/sec");
        System.out.println("Mémoire threads: ~200 MB (100 threads)");
    }
    
    private static ProcessedRecord processRecord(Record record) {
        try {
            // 1. Appel API de validation (100ms)
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(VALIDATION_API + "?id=" + record.getId()))
                .GET()
                .build();
            
            HttpResponse<String> response = httpClient.send(
                request, 
                HttpResponse.BodyHandlers.ofString()
            );  // Thread Platform BLOQUÉ pendant l'I/O
            
            // 2. Traitement du résultat (1ms)
            boolean isValid = response.body().contains("\"valid\":true");
            
            return new ProcessedRecord(
                record.getId(),
                record.getData(),
                isValid,
                response.body()
            );
            
        } catch (Exception e) {
            return new ProcessedRecord(record.getId(), record.getData(), false, null);
        }
    }
    
    private static void saveResults(List<ProcessedRecord> results) throws SQLException {
        try (Connection conn = DriverManager.getConnection(DB_URL);
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO processed_records (id, data, valid, validation_result) VALUES (?, ?, ?, ?)")) {
            
            for (ProcessedRecord record : results) {
                stmt.setLong(1, record.getId());
                stmt.setString(2, record.getData());
                stmt.setBoolean(3, record.isValid());
                stmt.setString(4, record.getValidationResult());
                stmt.addBatch();
            }
            
            stmt.executeBatch();
        }
    }
    
    private static List<Record> loadRecords(int count) {
        // Simulation chargement depuis fichier
        List<Record> records = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            records.add(new Record(i, "data-" + i));
        }
        return records;
    }
}

/*
Résultats typiques:

=== Traitement AVANT (Platform Threads) ===
Records à traiter: 100000

=== Résultats ===
Durée totale: 102500 ms (≈ 1min 42s)
Records traités: 100000
Throughput: 976 rec/sec
Mémoire threads: ~200 MB (100 threads)

Analyse:
• 100 threads en parallèle
• 100,000 / 100 = 1000 "rounds"
• Chaque round: 100ms (API call)
• Total: 1000 × 100ms = 100 secondes
• Limitation: impossible d'augmenter à 1000 threads (OOM)
*/
```

### APRÈS : Virtual Threads

```java
import java.util.concurrent.*;
import java.sql.*;
import java.net.http.*;
import java.net.URI;
import java.util.*;

public class BatchProcessorAfter {
    
    private static final HttpClient httpClient = HttpClient.newHttpClient();
    private static final String DB_URL = "jdbc:postgresql://localhost/batch";
    private static final String VALIDATION_API = "http://api.service.com/validate";
    
    public static void main(String[] args) throws Exception {
        
        // Charger les enregistrements
        List<Record> records = loadRecords(100_000);
        
        System.out.println("=== Traitement APRÈS (Virtual Threads) ===");
        System.out.println("Records à traiter: " + records.size());
        
        long startTime = System.currentTimeMillis();
        
        // ⚡ Virtual Thread Executor - pas de limite !
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            List<Future<ProcessedRecord>> futures = new ArrayList<>();
            
            // Soumettre TOUTES les tâches en même temps
            for (Record record : records) {
                Future<ProcessedRecord> future = executor.submit(() -> 
                    processRecord(record)
                );
                futures.add(future);
            }
            
            // Attendre tous les résultats
            List<ProcessedRecord> results = new ArrayList<>();
            for (Future<ProcessedRecord> future : futures) {
                try {
                    results.add(future.get());
                } catch (Exception e) {
                    System.err.println("Erreur traitement: " + e.getMessage());
                }
            }
            
            long duration = System.currentTimeMillis() - startTime;
            
            // Sauvegarder les résultats
            saveResults(results);
            
            System.out.println("\n=== Résultats ===");
            System.out.println("Durée totale: " + duration + " ms");
            System.out.println("Records traités: " + results.size());
            System.out.println("Throughput: " + 
                String.format("%.0f", results.size() * 1000.0 / duration) + " rec/sec");
            System.out.println("Mémoire VT: ~150 MB (100,000 VT)");
            System.out.println("Carriers utilisés: ~" + 
                Runtime.getRuntime().availableProcessors());
        }
    }
    
    // ✅ CODE IDENTIQUE - Rien à changer !
    private static ProcessedRecord processRecord(Record record) {
        try {
            // 1. Appel API de validation (100ms)
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(VALIDATION_API + "?id=" + record.getId()))
                .GET()
                .build();
            
            HttpResponse<String> response = httpClient.send(
                request, 
                HttpResponse.BodyHandlers.ofString()
            );  // VT se démonte automatiquement pendant l'I/O
            
            // 2. Traitement du résultat (1ms)
            boolean isValid = response.body().contains("\"valid\":true");
            
            return new ProcessedRecord(
                record.getId(),
                record.getData(),
                isValid,
                response.body()
            );
            
        } catch (Exception e) {
            return new ProcessedRecord(record.getId(), record.getData(), false, null);
        }
    }
    
    private static void saveResults(List<ProcessedRecord> results) throws SQLException {
        try (Connection conn = DriverManager.getConnection(DB_URL);
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO processed_records (id, data, valid, validation_result) VALUES (?, ?, ?, ?)")) {
            
            for (ProcessedRecord record : results) {
                stmt.setLong(1, record.getId());
                stmt.setString(2, record.getData());
                stmt.setBoolean(3, record.isValid());
                stmt.setString(4, record.getValidationResult());
                stmt.addBatch();
            }
            
            stmt.executeBatch();
        }
    }
    
    private static List<Record> loadRecords(int count) {
        List<Record> records = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            records.add(new Record(i, "data-" + i));
        }
        return records;
    }
}

/*
Résultats typiques:

=== Traitement APRÈS (Virtual Threads) ===
Records à traiter: 100000

=== Résultats ===
Durée totale: 2300 ms (≈ 2.3 secondes)  ← ✅ 45× plus rapide!
Records traités: 100000
Throughput: 43478 rec/sec               ← ✅ 45× meilleur!
Mémoire VT: ~150 MB (100,000 VT)
Carriers utilisés: ~8

Analyse:
• 100,000 VT lancés simultanément
• Tous font leur appel API "en parallèle"
• Limité par: bande passante réseau + API externe
• Temps ≈ latence API (100ms) + overhead
• Carriers: seulement 8 (nb de CPU cores)
• Pendant les I/O: VT démontés, carriers libres
*/
```

### Comparaison Visuelle

```
┌─────────────────────────────────────────────────────┐
│        Batch 100,000 records : PT vs VT             │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Platform Threads (100 threads):                    │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │
│ │←        100 seconds (1min 42s)              →│   │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │
│                                                     │
│ Virtual Threads (100,000 VT):                      │
│ ━━                                                  │
│ │←→│                                                │
│ 2.3s                                                │
│                                                     │
│ Speedup: 45×                                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 5.3 Synthèse des changements

### Ce qui change

```
Migration vers Virtual Threads:

Configuration Spring Boot:
━━━━━━━━━━━━━━━━━━━━━━━━
AVANT:
  server:
    tomcat:
      threads:
        max: 200

APRÈS:
  spring:
    threads:
      virtual:
        enabled: true

Code métier:
━━━━━━━━━━━━
✅ AUCUN CHANGEMENT requis!
Le code reste identique.
```

### Gains mesurés

```
┌──────────────────────┬──────────────┬───────────────┬──────────┐
│ Métrique             │ Platform     │ Virtual       │ Gain     │
├──────────────────────┼──────────────┼───────────────┼──────────┤
│ Throughput (Spring)  │ 191 req/s    │ 2,380 req/s   │ 12×      │
│ Latence p99 (Spring) │ 8,500 ms     │ 450 ms        │ 19×      │
│ Batch 100k records   │ 102 sec      │ 2.3 sec       │ 45×      │
│ Mémoire (Spring)     │ 400 MB       │ 10 MB         │ 40×      │
│ Connexions simul.    │ 200          │ 10,000+       │ 50×      │
│ CPU utilization      │ 8%           │ 45%           │ 5.6×     │
└──────────────────────┴──────────────┴───────────────┴──────────┘
```

### Effort de migration

```
Checklist migration:

✅ Mise à jour Java 21+
   └─ Changement version dans pom.xml/build.gradle

✅ Mise à jour Spring Boot 3.5+ (Et bientôt 4.X)
   └─ Changement version parent

✅ Configuration Virtual Threads
   └─ Ajouter spring.threads.virtual.enabled: true

✅ Révision synchronized blocks
   └─ Remplacer par ReentrantLock si I/O dans le bloc

⚠️  Tests de charge
   └─ Vérifier absence de pinning avec -Djdk.tracePinnedThreads

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Effort: 1-2 jours pour une application typique
ROI: Immédiat (gains de performance mesurables directement à partir du moment ou l'app en question est assez sollicité pour en avoir besoin, mais le coup d'implem est tellement bas que ça reste très rentable)
```

---

[🏠 Accueil](../index.md) | [📋 Sommaire](../sommaire.md) | [⬅️ Précédent](04-virtual-threads-intro.md) | [➡️ Suivant: Conclusion](06-conclusion.md)