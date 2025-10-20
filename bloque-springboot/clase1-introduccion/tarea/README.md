# Tarea para Casa - Clase 1

**Fecha de entrega:** Antes de la próxima clase (verificar en Moodle)
**Peso en la nota final:** 10%
**Modalidad:** Individual

---

## Objetivo

Desarrollar una REST API completa con Spring Boot aplicando los conceptos de la Clase 1, demostrando dominio de arquitectura en capas, endpoints REST, y mejores prácticas.

---

## Descripción de la Tarea

Debes crear un microservicio REST API para uno de los siguientes dominios:

1. **Banking (Banca)** - Sistema de gestión de cuentas bancarias
2. **Retail (Comercio)** - Sistema de gestión de productos e inventario
3. **Telecom (Telecomunicaciones)** - Sistema de gestión de clientes y planes

**Importante:** Elige UN solo dominio y desarrolla la API completa para ese dominio.

---

## Requisitos Funcionales

Tu API debe implementar un CRUD completo (Create, Read, Update, Delete) con los siguientes endpoints:

### Endpoints Obligatorios

```
GET    /api/{recursos}           - Listar todos los recursos
GET    /api/{recursos}/{id}      - Obtener un recurso por ID
POST   /api/{recursos}           - Crear un nuevo recurso
PUT    /api/{recursos}/{id}      - Actualizar un recurso existente
DELETE /api/{recursos}/{id}      - Eliminar un recurso
```

### Endpoints Adicionales (Mínimo 2)

Agrega al menos 2 de los siguientes:

- **Filtrado**: `GET /api/{recursos}/filter?campo=valor`
- **Búsqueda**: `GET /api/{recursos}/search?query=texto`
- **Estadísticas**: `GET /api/{recursos}/stats`
- **Operación específica del dominio** (ver ejemplos abajo)

---

## Dominios y Ejemplos

### Opción 1: Banking (Banca)

**Recurso principal:** `Account` (Cuenta bancaria)

**Modelo:**
```java
public class Account {
    private Long id;
    private String accountNumber;      // Ej: "ACC-001234"
    private String accountHolder;      // Nombre del titular
    private String accountType;        // SAVINGS, CHECKING, INVESTMENT
    private Double balance;            // Saldo actual
    private String currency;           // USD, BOB, EUR
    private Boolean active;
}
```

**Endpoints adicionales sugeridos:**
- `GET /api/accounts/filter?accountType=SAVINGS`
- `GET /api/accounts/search?holder=Juan`
- `GET /api/accounts/stats` - Total de cuentas, suma de saldos, etc.
- `PATCH /api/accounts/{id}/deposit` - Depositar dinero (operación específica)
- `PATCH /api/accounts/{id}/withdraw` - Retirar dinero (operación específica)

---

### Opción 2: Retail (Comercio)

**Recurso principal:** `Product` (Producto)

**Modelo:**
```java
public class Product {
    private Long id;
    private String sku;                // Código de producto: "PROD-001"
    private String name;               // Nombre del producto
    private String category;           // ELECTRONICS, CLOTHING, FOOD, etc.
    private Double price;              // Precio unitario
    private Integer stock;             // Cantidad en inventario
    private Boolean available;         // Disponible para venta
}
```

**Endpoints adicionales sugeridos:**
- `GET /api/products/filter?category=ELECTRONICS`
- `GET /api/products/search?name=laptop`
- `GET /api/products/stats` - Total productos, valor del inventario, etc.
- `GET /api/products/low-stock?threshold=10` - Productos con stock bajo
- `PATCH /api/products/{id}/restock` - Reabastecer inventario

---

### Opción 3: Telecom (Telecomunicaciones)

**Recurso principal:** `Customer` (Cliente)

**Modelo:**
```java
public class Customer {
    private Long id;
    private String customerId;         // ID único: "CUST-001234"
    private String fullName;           // Nombre completo
    private String phoneNumber;        // Número telefónico
    private String plan;               // PREPAID, POSTPAID, UNLIMITED
    private Double balance;            // Saldo disponible (prepago)
    private Boolean active;            // Estado de la línea
}
```

**Endpoints adicionales sugeridos:**
- `GET /api/customers/filter?plan=PREPAID`
- `GET /api/customers/search?name=Maria`
- `GET /api/customers/stats` - Total clientes por plan, etc.
- `PATCH /api/customers/{id}/recharge` - Recargar saldo (prepago)
- `PATCH /api/customers/{id}/change-plan` - Cambiar plan

---

## Requisitos Técnicos

### 1. Arquitectura en Capas

Tu proyecto debe tener la siguiente estructura:

```
src/main/java/dev/alefiengo/{dominio}/
├── Application.java              # Clase principal
├── controller/
│   └── {Recurso}Controller.java  # Controlador REST
├── service/
│   └── {Recurso}Service.java     # Lógica de negocio
└── model/
    └── {Recurso}.java            # Modelo de datos
```

**Ejemplo para Banking:**
```
src/main/java/dev/alefiengo/banking/
├── BankingApplication.java
├── controller/
│   └── AccountController.java
├── service/
│   └── AccountService.java
└── model/
    └── Account.java
```

---

### 2. Anotaciones Requeridas

**Controller:**
```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {
    // Inyección de dependencias por constructor
    private final AccountService accountService;

    public AccountController(AccountService accountService) {
        this.accountService = accountService;
    }

    @GetMapping
    public List<Account> getAllAccounts() { }

    @PostMapping
    public Account createAccount(@RequestBody Account account) { }

    // ... resto de endpoints
}
```

**Service:**
```java
@Service
public class AccountService {
    private List<Account> accounts = new ArrayList<>();

    // Métodos de lógica de negocio
}
```

**Model:**
```java
public class Account {
    // Campos, constructores, getters, setters
}
```

---

### 3. Configuración (application.yml)

```yaml
server:
  port: 8080

spring:
  application:
    name: {tu-dominio}-api

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always

info:
  app:
    name: {Tu Dominio} API
    version: 1.0.0
    author: Tu Nombre
```

---

### 4. Spring Boot Actuator

Tu aplicación debe incluir Actuator con los siguientes endpoints funcionales:

- `GET /actuator/health` - Health check
- `GET /actuator/info` - Información de la aplicación

---

## Requisitos de Entrega

### 1. Repositorio Git (GitHub o GitLab)

Crea un repositorio público con el siguiente nombre:

```
{dominio}-api-springboot
```

Ejemplos:
- `banking-api-springboot`
- `retail-api-springboot`
- `telecom-api-springboot`

---

### 2. Estructura del Repositorio

```
{dominio}-api-springboot/
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   └── test/
├── pom.xml
├── README.md                   # Documentación del proyecto
├── POSTMAN_COLLECTION.json     # Colección de Postman
└── .gitignore
```

---

### 3. README.md del Proyecto

Tu README debe incluir:

```markdown
# {Nombre del Proyecto}

## Descripción
Breve descripción del proyecto y su propósito.

## Tecnologías
- Java 17
- Spring Boot 3.x
- Maven
- Spring Boot Actuator

## Requisitos Previos
- Java 17+
- Maven 3.9+

## Instalación y Ejecución

\```bash
# Clonar el repositorio
git clone https://github.com/{tu-usuario}/{dominio}-api-springboot.git

# Navegar al directorio
cd {dominio}-api-springboot

# Ejecutar la aplicación
mvn spring-boot:run
\```

## Endpoints

### Listar todos los recursos
\```
GET http://localhost:8080/api/{recursos}
\```

### Crear un recurso
\```
POST http://localhost:8080/api/{recursos}
Content-Type: application/json

{
  "campo1": "valor1",
  "campo2": "valor2"
}
\```

[... Documentar todos los endpoints ...]

## Pruebas

Importa la colección de Postman incluida en `POSTMAN_COLLECTION.json`

## Autor
Tu Nombre - [GitHub](https://github.com/tu-usuario)
```

---

### 4. Colección de Postman

Exporta una colección de Postman que incluya:

- Todos los endpoints implementados
- Ejemplos de request body para POST/PUT
- Variables de entorno (si aplica)

**Archivo:** `POSTMAN_COLLECTION.json`

---

### 5. Código Limpio

- Nombres de variables y métodos en inglés
- Código indentado y formateado consistentemente
- Sin código comentado innecesario
- Sin warnings del compilador

---

## Entrega en Moodle

Sube a Moodle un archivo de texto (.txt) con:

1. **URL de tu repositorio** (GitHub o GitLab)
2. **Nombre completo**
3. **Dominio elegido** (Banking, Retail o Telecom)
4. **Nota adicional** (opcional): Cualquier observación sobre tu implementación

**Ejemplo de archivo de entrega:**

```
Nombre: Juan Pérez Rodríguez
Dominio: Banking
Repositorio: https://github.com/juanperez/banking-api-springboot

Notas:
- Implementé 3 endpoints adicionales
- Agregué validación de saldo negativo en el método withdraw
```

---

## Criterios de Evaluación

Tu tarea será evaluada según la rúbrica detallada en [RUBRICA.md](RUBRICA.md).

**Resumen de criterios:**

| Criterio | Peso |
|----------|------|
| Funcionalidad (CRUD completo) | 40% |
| Arquitectura en capas | 20% |
| Endpoints adicionales | 15% |
| Documentación (README) | 10% |
| Colección de Postman | 5% |
| Código limpio y buenas prácticas | 10% |

**Total:** 100%

---

## Ejemplos de Referencia

En el directorio `ejemplos/` encontrarás proyectos de referencia completos para cada dominio:

- [ejemplo-banking/](ejemplos/ejemplo-banking/) - Implementación de referencia para Banking
- [ejemplo-retail/](ejemplos/ejemplo-retail/) - Implementación de referencia para Retail
- [ejemplo-telecom/](ejemplos/ejemplo-telecom/) - Implementación de referencia para Telecom

**Importante:** Estos son ejemplos de referencia. Tu implementación debe ser original.

---

## Preguntas Frecuentes

### ¿Puedo usar otro dominio que no esté en la lista?

No. Debes elegir uno de los tres dominios especificados (Banking, Retail, Telecom).

### ¿Debo usar base de datos?

No. Para esta tarea usa `ArrayList` o `HashMap` como almacenamiento en memoria. La persistencia en base de datos se verá en la Clase 2.

### ¿Puedo trabajar en equipo?

No. Esta es una tarea individual. Cada estudiante debe entregar su propio proyecto.

### ¿Qué pasa si entrego tarde?

Revisa la política de entregas tardías en Moodle. Generalmente se aplica una penalización por día de retraso.

### ¿Puedo agregar más funcionalidades?

Sí, puedes agregar funcionalidades extra más allá de los requisitos mínimos. Esto puede sumar puntos adicionales.

### ¿Debo escribir tests?

No es obligatorio para esta tarea, pero es recomendado. Los tests suman puntos adicionales.

---

## Recursos de Apoyo

- [Cheatsheet de la Clase 1](../cheatsheet.md)
- [FAQ de la Clase 1](../FAQ.md)
- [Spring Initializr](https://start.spring.io/)
- [Prácticas de la Clase 1](../practicas/)

---

## Consejos

1. **Empieza temprano** - No dejes la tarea para el último día
2. **Prueba constantemente** - Verifica cada endpoint después de implementarlo
3. **Commits frecuentes** - Haz commits pequeños y descriptivos
4. **README claro** - Un buen README facilita la evaluación
5. **Revisa la rúbrica** - Asegúrate de cumplir todos los criterios
6. **Consulta ejemplos** - Usa los proyectos de referencia como guía
7. **Pregunta dudas** - Si algo no está claro, pregunta en clase o por Moodle

---

**¡Éxito en tu proyecto!**

[← Volver a Clase 1](../README.md)
