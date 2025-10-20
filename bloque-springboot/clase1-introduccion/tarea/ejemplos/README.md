# Proyectos de Ejemplo - Tarea Clase 1

Este directorio contiene proyectos de referencia completos para cada dominio de la tarea.

**IMPORTANTE**: Estos son ejemplos de referencia para guiarte. Tu implementación debe ser original y desarrollada por ti.

---

## Proyectos Disponibles

### 1. ejemplo-banking/

**Dominio**: Sistema de gestión de cuentas bancarias

**Funcionalidades implementadas:**
- CRUD completo de cuentas bancarias
- Filtrado por tipo de cuenta
- Búsqueda por titular
- Estadísticas de saldos
- Operaciones de depósito y retiro

**Endpoints**:
```
GET    /api/accounts
GET    /api/accounts/{id}
POST   /api/accounts
PUT    /api/accounts/{id}
DELETE /api/accounts/{id}
GET    /api/accounts/filter?accountType=SAVINGS
GET    /api/accounts/search?holder=Juan
GET    /api/accounts/stats
PATCH  /api/accounts/{id}/deposit
PATCH  /api/accounts/{id}/withdraw
```

**Tecnologías:**
- Spring Boot 3.3.x
- Java 17
- Maven
- Spring Boot Actuator
- ArrayList (almacenamiento en memoria)

---

### 2. ejemplo-retail/

**Dominio**: Sistema de gestión de productos e inventario

**Funcionalidades implementadas:**
- CRUD completo de productos
- Filtrado por categoría
- Búsqueda por nombre
- Estadísticas de inventario
- Alerta de stock bajo
- Reabastecimiento de inventario

**Endpoints:**
```
GET    /api/products
GET    /api/products/{id}
POST   /api/products
PUT    /api/products/{id}
DELETE /api/products/{id}
GET    /api/products/filter?category=ELECTRONICS
GET    /api/products/search?name=laptop
GET    /api/products/stats
GET    /api/products/low-stock?threshold=10
PATCH  /api/products/{id}/restock
```

**Tecnologías:**
- Spring Boot 3.3.x
- Java 17
- Maven
- Spring Boot Actuator
- ArrayList (almacenamiento en memoria)

---

### 3. ejemplo-telecom/

**Dominio**: Sistema de gestión de clientes y planes

**Funcionalidades implementadas:**
- CRUD completo de clientes
- Filtrado por plan
- Búsqueda por nombre
- Estadísticas de clientes
- Recarga de saldo (prepago)
- Cambio de plan

**Endpoints:**
```
GET    /api/customers
GET    /api/customers/{id}
POST   /api/customers
PUT    /api/customers/{id}
DELETE /api/customers/{id}
GET    /api/customers/filter?plan=PREPAID
GET    /api/customers/search?name=Maria
GET    /api/customers/stats
PATCH  /api/customers/{id}/recharge
PATCH  /api/customers/{id}/change-plan
```

**Tecnologías:**
- Spring Boot 3.3.x
- Java 17
- Maven
- Spring Boot Actuator
- ArrayList (almacenamiento en memoria)

---

## Estructura de Cada Proyecto

Todos los proyectos siguen la misma estructura de arquitectura en capas:

```
ejemplo-{dominio}/
├── src/
│   ├── main/
│   │   ├── java/dev/alefiengo/{dominio}/
│   │   │   ├── Application.java              # Clase principal
│   │   │   ├── controller/
│   │   │   │   └── {Recurso}Controller.java  # REST controller
│   │   │   ├── service/
│   │   │   │   └── {Recurso}Service.java     # Lógica de negocio
│   │   │   └── model/
│   │   │       └── {Recurso}.java            # Modelo de datos
│   │   └── resources/
│   │       └── application.yml               # Configuración
│   └── test/
│       └── java/
├── pom.xml
├── README.md
├── POSTMAN_COLLECTION.json
└── .gitignore
```

---

## Cómo Usar los Ejemplos

### 1. Exploración

**NO copies directamente el código**. En su lugar:

1. Lee el README de cada proyecto
2. Revisa la estructura del código
3. Entiende cómo están implementadas las funcionalidades
4. Observa los patrones usados
5. Importa la colección de Postman
6. Prueba los endpoints

### 2. Aprendizaje

**Aspectos a observar:**

- ¿Cómo está organizado el código en capas?
- ¿Cómo se manejan las validaciones?
- ¿Cómo se implementan los endpoints adicionales?
- ¿Cómo está configurado Actuator?
- ¿Qué buenas prácticas se aplican?

### 3. Implementación Original

**Tu implementación debe:**

- Ser escrita por ti (no copiada)
- Tener tu propio estilo
- Incluir tus propias mejoras
- Demostrar tu comprensión

**Puedes:**
- Usar los ejemplos como guía conceptual
- Adaptar patrones a tu código
- Inspirarte en la estructura
- Consultar cómo resolver problemas específicos

**NO puedes:**
- Copiar y pegar código directamente
- Renombrar variables y presentarlo como tuyo
- Usar sin entender cómo funciona

---

## Ejecutar un Proyecto de Ejemplo

### Requisitos

- Java 17+
- Maven 3.9+

### Pasos

```bash
# 1. Navegar al proyecto
cd ejemplo-banking/

# 2. Ejecutar
mvn spring-boot:run

# 3. La aplicación estará disponible en:
http://localhost:8080

# 4. Probar health check
curl http://localhost:8080/actuator/health

# 5. Probar endpoint
curl http://localhost:8080/api/accounts
```

### Importar Postman Collection

Cada proyecto incluye `POSTMAN_COLLECTION.json`:

1. Abre Postman
2. Import → File
3. Selecciona el archivo `POSTMAN_COLLECTION.json` del proyecto
4. Prueba los endpoints

---

## Comparación de Dominios

### Banking (Banca)

**Complejidad**: Media
**Lógica adicional**: Validaciones de saldo, transacciones
**Endpoints especiales**: Depósito, retiro

**Ideal para:**
- Practicar validaciones de negocio
- Operaciones con estado (saldo)
- Cálculos numéricos

---

### Retail (Comercio)

**Complejidad**: Media
**Lógica adicional**: Gestión de inventario, alertas de stock
**Endpoints especiales**: Reabastecimiento, stock bajo

**Ideal para:**
- Practicar filtros y búsquedas
- Alertas condicionales
- Estadísticas de inventario

---

### Telecom (Telecomunicaciones)

**Complejidad**: Media
**Lógica adicional**: Planes diferenciados (prepago/postpago), recarga
**Endpoints especiales**: Recarga, cambio de plan

**Ideal para:**
- Practicar enumeraciones (enum)
- Lógica condicional por tipo
- Operaciones de estado

---

## Diferencias Clave con la Tarea

### Los ejemplos incluyen:

1. **Código completo funcional**
2. **Comentarios explicativos**
3. **Tests unitarios básicos** (opcional para tu tarea)
4. **README detallado**
5. **Colección de Postman completa**
6. **Logging** (opcional para tu tarea)

### Tu tarea debe incluir:

1. **Los requisitos mínimos** (ver [tarea/README.md](../README.md))
2. **Tu propio código** (original)
3. **Tu propio README** (documentación)
4. **Tu propia colección de Postman**
5. **Arquitectura en capas** (Controller → Service → Model)

---

## Niveles de Implementación

### Nivel Básico (60-70 puntos)

- CRUD completo funcional
- 2 endpoints adicionales básicos
- Arquitectura en capas básica
- README simple

**Referencia**: Funcionalidad core de los ejemplos

---

### Nivel Intermedio (70-85 puntos)

- CRUD completo + endpoints adicionales complejos
- Validaciones en el service
- Arquitectura en capas clara
- README completo con ejemplos
- Colección de Postman organizada

**Referencia**: Ejemplos completos + tu creatividad

---

### Nivel Avanzado (85-100 puntos)

- Todo lo anterior +
- Custom health indicator
- Métricas personalizadas
- Logging
- Tests unitarios
- DTOs separados
- Exception handling

**Referencia**: Desafío avanzado de Práctica 3

---

## Preguntas Frecuentes

### ¿Puedo usar el código de los ejemplos?

NO directamente. Puedes:
- Entender cómo funcionan
- Aprender patrones
- Usarlos como referencia conceptual
- Copiar y pegar código

### ¿Qué pasa si mi código es muy similar a los ejemplos?

Es normal que haya similitudes en:
- Estructura de proyecto (es estándar)
- Anotaciones de Spring (son las mismas)
- Nombres de métodos comunes (getAllX, createX, etc.)

Pero debe ser DIFERENTE en:
- Implementación de lógica
- Nombres de variables
- Orden y organización del código
- Validaciones específicas

### ¿Puedo combinar características de diferentes dominios?

NO. Elige UN solo dominio y desarrolla la API completa para ese dominio.

### ¿Los ejemplos tienen todo lo que necesito?

Los ejemplos cubren los requisitos mínimos + algunos extras. Para obtener la nota máxima, necesitas agregar tus propias mejoras más allá de lo que ves en los ejemplos.

---

## Recursos Adicionales

- [Prácticas de Clase 1](../../practicas/)
- [Desafíos](../../desafios/)
- [Cheatsheet](../../cheatsheet.md)
- [FAQ](../../FAQ.md)
- [Rúbrica de Evaluación](../RUBRICA.md)

---

**Nota Importante**: El objetivo de estos ejemplos es ayudarte a aprender, NO a copiar. El profesor puede identificar código copiado y esto resultará en penalización o 0 puntos. Desarrolla tu propia implementación basándote en lo aprendido en clase.

---

[← Volver a Tarea](../README.md) | [← Volver a Clase 1](../../README.md)
