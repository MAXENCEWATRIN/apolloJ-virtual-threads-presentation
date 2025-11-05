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
      maximum-pool-size: 200
      minimum-idle: 5

server:
  port: 8080
  tomcat:
    threads:
      max: 2000
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
        
        // 1. Validation et création de la commande en base (50ms)
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
│ ────────────────────────────────────────────────────│
│ DB INSERT        50ms     I/O         BLOQUÉ        │
│ HTTP Stock       120ms    I/O         BLOQUÉ        │
│ HTTP Payment     200ms    I/O         BLOQUÉ        │
│ DB UPDATE        30ms     I/O         BLOQUÉ        │
│                                                     │
│ Total CPU:       ~10ms    (2.5%)                    │
│ Total I/O:       400ms    (97.5%)                   │
│                                                     │
└─────────────────────────────────────────────────────┘

Problème:
• Thread Platform bloqué 400ms
• Avec 2000 threads max → 2000 requêtes simultanées max
• Si 5000 req/sec arrivent → queue + latence
• CPU utilization: ~5%
• Mémoire: 2000 threads × 2MB = 4000MB
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
      enabled: true              
  
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: postgres
    password: password
    hikari:
      maximum-pool-size: 10      # Moins de connexions nécessaires avec VT
      minimum-idle: 5

server:
  port: 8080
  # Plus besoin de configurer un max-threads si c'était le cas chez vous avant
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
    
    public OrderResponse createOrder(OrderRequest request) {
        
        long startTime = System.currentTimeMillis();
        
        // 1. Validation et création de la commande
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
│ ────────────────────────────────────────────────────│
│ DB INSERT        50ms     I/O         DÉMONTÉ       │
│ HTTP Stock       120ms    I/O         DÉMONTÉ       │
│ HTTP Payment     200ms    I/O         DÉMONTÉ       │
│ DB UPDATE        30ms     I/O         DÉMONTÉ       │
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

---

## 5.2 Synthèse des changements

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