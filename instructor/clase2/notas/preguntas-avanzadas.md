# Preguntas Avanzadas y Respuestas Sugeridas · Clase 2

Estas notas están pensadas para aquellas consultas que suelen llegar de alumnos con experiencia previa. Úsalas como referencia rápida durante la sesión.

---

## 1. Pruebas de carga y estrés
**Pregunta típica:** “¿Cómo vamos a comprobar que esta API soporta miles de peticiones? ¿Usaremos Artillery, Gatling, JMeter?”  
**Respuesta sugerida:** “Hoy el foco es construir la API y garantizar persistencia; más adelante (Clases 6-8) veremos integración con Kafka y discutiremos estrategias de pruebas de carga. Personalmente recomiendo JMeter o Gatling para simular tráfico HTTP y, cuando entremos a Kafka, herramientas como `k6` o `kafka-producer-perf-test`.”

---

## 2. ¿Por qué `@SpringBootApplication` y no definir cada anotación manualmente?
**Respuesta:** “Puedes hacerlo, pero perderías autoconfiguración y escaneo automático. `@SpringBootApplication` mantiene el código limpio y está respaldada por convenciones del ecosistema. Solo tiene sentido separarlas si necesitas un control extremadamente fino.”

---

## 3. Diferencia entre `@Component`, `@Service`, `@Repository` y `@Bean`
**Respuesta:**  
- `@Component` es genérico; `@Service` y `@Repository` son estereotipos específicos que facilitan semántica y algunas funcionalidades (traducción de excepciones en `@Repository`).  
- `@Bean` se utiliza dentro de una clase `@Configuration` para registrar manualmente un objeto en el contexto (por ejemplo, un `ObjectMapper` personalizado).  
- Siempre que puedas, deja que Spring descubra beans mediante escaneo (`@Component`/`@Service`). Usa `@Bean` cuando necesites instancias creadas a partir de lógica imperativa.

---

## 4. Inversión de Control vs inyección manual de dependencias
**Respuesta:** “Spring controla el ciclo de vida de los beans; si creas objetos manualmente (`new`), pierdes capacidades como proxies, transacciones y configuración centralizada. La idea es declarar dependencias y dejar que Spring resuelva las instancias.”

---

## 5. Uso de `@Autowired` en propiedades vs constructor
**Respuesta:** “El constructor es la opción recomendada porque facilita pruebas y garantiza que las dependencias estén disponibles al momento de la creación del bean. `@Autowired` en campos funciona, pero dificulta mocking y testing.”

---

## 6. ¿Por qué `JpaRepository` y no directamente `EntityManager`?
**Respuesta:** “`JpaRepository` abstrae operaciones comunes. Si el repositorio estándar se queda corto, siempre se puede extender con consultas personalizadas o usar `@Query`. Para casos muy sofisticados, se utiliza `EntityManager`, pero no queremos esa complejidad tan temprano.”

---

## 7. ¿Cómo se gestionan transacciones complejas?
**Respuesta:** “Spring Data JPA marca métodos con `@Transactional`. En Clase 2 nos enfocamos en CRUD simple; más adelante, cuando abordemos validación y excepciones, veremos cómo ajustar los límites transaccionales.”

---

## 8. Integración con otras bases de datos o NoSQL
**Respuesta:** “Spring Boot tiene starters para MongoDB, Redis, Cassandra, etc. La idea es que dominen el patrón con PostgreSQL y luego puedan aplicar el mismo enfoque a otros motores.”

---

## 9. ¿Dónde aplicar patrones como DTO o MapStruct?
**Respuesta:** “En Clase 2 trabajamos con DTOs sencillos, mapeados a mano. En Clase 3 introduciremos capas más formales y puedes evaluar bibliotecas de mapeo como MapStruct para proyectos grandes.”

---

## 10. ¿Qué hay de la observabilidad (Prometheus, Grafana, tracing)?
**Respuesta:** “Son temas programados para Clases 7 y 8. Hoy mencionamos que Spring Boot Actuator ya está disponible y que iremos habilitando métricas a medida que la arquitectura lo necesite.”

---

## 11. Preparación para despliegue/CI-CD
**Respuesta:** “Concluir este bloque nos permite tener un microservicio dockerizable. En el proyecto final veremos pipelines básicos (Docker Compose + scripts) y hablaremos brevemente de CI/CD.”

---

## 12. ¿Cómo manejar configuraciones por ambiente (dev, test, prod)?
**Respuesta:** “Spring Boot permite perfiles (`application-dev.yml`, `application-prod.yml`) y sobreescritura por variables de entorno. En Clase 4 entraremos a detalle, pero hoy puedes mostrar cómo `spring.profiles.active` cambia el comportamiento.”

---

## 13. ¿Qué hay de las pruebas automatizadas (unitarias/integración)?
**Respuesta:** “En Clase 2 nos enfocamos en funcionalidad. En Clase 3+ añadiremos pruebas con JUnit, `@SpringBootTest` y `@DataJpaTest`. Recomienda mantener tests unitarios para servicios y tests de integración acotados.”

---

## 14. ¿Por qué usar `application.yml` y no `application.properties`?
**Respuesta:** “`.yml` ofrece jerarquías más legibles y permite agrupar configuración de forma natural. Spring soporta ambos formatos, por lo que elige lo que haga más sencillo mantener la configuración.”

---

## 15. ¿Spring MVC o WebFlux (reactivo) para este proyecto?
**Respuesta:** “MVC (bloqueante) es suficiente para el tamaño del curso y facilita el aprendizaje. WebFlux tiene sentido cuando necesitas responder a miles de conexiones simultáneas con operaciones no bloqueantes. Es un tema avanzado que puedes explorar después de dominar el enfoque tradicional.”

---

Mantén estas respuestas a mano; ajusta el nivel de profundidad según la audiencia. Recuerda ser transparente con los límites de cada clase y señalar las sesiones donde se cubrirán los temas avanzados.
