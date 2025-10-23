# Conceptos Fundamentales - Clase 2

Esta clase profundiza en la anatomía de una aplicación Spring Boot 3.x y en los componentes necesarios para construir una API REST con persistencia en PostgreSQL. Usa esta lectura como soporte antes y después de los laboratorios.

---

## 1. Spring Framework vs Spring Boot

| Aspecto                    | Spring Framework                               | Spring Boot                                             |
|---------------------------|------------------------------------------------|--------------------------------------------------------|
| Configuración             | Manual (XML/Java Config), muchas decisiones    | Autoconfiguración según dependencias                   |
| Servidor                  | Externo (Tomcat, Jetty desplegado aparte)      | Servidor embebido listo para ejecutar (`spring-boot:run`)
| Dependencias              | Gestionar cada starter individualmente         | Starters opinados (`spring-boot-starter-web`, JPA, etc.)|
| Estructura inicial        | Generada a mano                                | Spring Initializr en segundos                          |
| Objetivo                  | Flexibilidad total                             | Productividad y convenciones listas para producción    |

**Claves:** Spring Boot no reemplaza a Spring Framework; lo extiende con convenciones, autoconfiguración y tooling para acelerar el arranque.

---

## 2. Anatomía de una aplicación Spring Boot

```java
@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
}
```

- `@SpringBootApplication` combina configuraciones que permiten detectar componentes automáticamente y aplicar autoconfiguración.
- `SpringApplication.run` levanta el contexto de Spring, registra beans y arranca el servidor embebido (Tomcat por defecto).
- `src/main/resources/application.yml` centraliza configuración (puerto, datasource, logging). Preferimos `.yml` por legibilidad.

---

## 3. Componentes y estereotipos principales

| Anotación          | Uso | Descripción |
|--------------------|-----|-------------|
| `@Component`       | Genérico | Bean detectado por `@ComponentScan`. Base para otras anotaciones. |
| `@RestController`  | Web (REST) | Controladores que exponen endpoints JSON/XML. |
| `@Repository`      | Datos | Interfaz que Spring Data implementa para acceder a la base de datos. |
| `@Configuration`   | Configuración | Clases que definen beans manualmente (`@Bean`). |
| `@Entity`          | Persistencia | Mapea una clase a una tabla con JPA. |

> En la Clase 3 introduciremos `@Service` y otros estereotipos orientados a capas adicionales.

**Convención:** nombre de paquetes en minúsculas (`dev.alefiengo.productservice`), clases y métodos en inglés.

---

## 4. Inversión de Control (IoC) e Inyección de Dependencias (DI)

- **IoC**: el framework controla la creación y ciclo de vida de los beans.
- **DI en esta clase**: inyectamos el `ProductRepository` directamente en el controlador para consumirlo sin crear instancias manuales.

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductRepository repository;

    public ProductController(ProductRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<Product> list() {
        return repository.findAll();
    }
}
```

---

## 5. Flujo de una petición REST

```
Cliente HTTP ──> DispatcherServlet ──> Controlador (@RestController) ──> Repositorio (@Repository) ──> Base de datos
```

1. **DispatcherServlet** enruta la petición al controlador adecuado.
2. **Controlador** parsea la entrada y llama al repositorio.
3. **Repositorio** ejecuta la operación JPA correspondiente.

(En la Clase 3 agregaremos capas adicionales como servicios, DTOs y manejadores globales para escalar la arquitectura.)

---

## 6. Persistencia con Spring Data JPA

### Entidad básica
```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String description;
    private BigDecimal price;
    private Integer stock;
    // getters y setters
}
```

### Repositorio
```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByNameContainingIgnoreCase(String name);
}
```

- `JpaRepository` provee CRUD básico (`findAll`, `save`, `delete`, etc.).
- Métodos derivados (`findByNameContainingIgnoreCase`) generan consultas automáticamente.
- Configuración típica en `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: product_user
    password: product_password
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

---

## 7. Ecosistema comparativo (recordatorio breve)

| Framework   | Lenguaje  | Enfoque | Cuando considerarlo |
|-------------|-----------|--------|---------------------|
| Spring Boot | Java/Kotlin | Enterprise, full-stack, ecosistema maduro | Proyectos backend corporativos, integración con Kafka, cloud-native. |
| NestJS      | TypeScript  | Modular, inspirado en Angular | Equipos con stack JavaScript, APIs rápidas con soporte TypeScript. |
| FastAPI     | Python      | Async, tipado opcional | ML/IA, prototipos rápidos, equipos data science. |
| Laravel     | PHP         | Framework MVC completo | Aplicaciones web monolíticas con stack PHP. |

---

## 8. Lecturas y documentación recomendada

- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Framework Core](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html)
- [Guía oficial: Building a RESTful Web Service](https://spring.io/guides/gs/rest-service/)
- [Guía: Accessing Data with JPA](https://spring.io/guides/gs/accessing-data-jpa/)

---

**Siguiente paso:** aplicar estos conceptos en los laboratorios (Lab 00–05) para obtener un servicio `product-service` listo para evolucionar en Clase 3.
