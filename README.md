# Microservicios Castañeda

Backend de gestión de clientes y productos financieros desarrollado con **Java 17 y Spring Boot 3.5.5**, organizado en cinco servicios y desplegable con Docker Compose.

El proyecto aplica el patrón **Backend for Frontend (BFF)** para separar las operaciones del canal web de las del canal móvil. Cada BFF expone su propia API, consume los servicios de negocio y transforma sus respuestas según las necesidades del canal. El caso más representativo es la consulta móvil que reúne los datos de un cliente, sus cuentas y sus productos financieros en una sola respuesta.

## Arquitectura

![Arquitectura de Microservicios Castañeda: BFF web y móvil, servicios de negocio y servidor de autorización](mcs-castaneda-arquitectura.png)

El diagrama representa el diseño general. Los servicios de transacciones y notificaciones aparecen como extensiones del diseño; los cinco módulos implementados son los siguientes:

| Módulo | Responsabilidad | Puerto local |
| --- | --- | --- |
| [`bff-web`](bff-web) | Gestión de clientes, consulta y creación de productos, y asociación de cuentas a clientes. | `8030` |
| [`bff-mobile`](bff-mobile) | Consulta consolidada del cliente y sus productos, catálogo financiero y depósitos. | `8080` |
| [`mcs-customers`](mcs-customers) | Persistencia de clientes, búsqueda por documento, validaciones y bloqueo. | `8083` |
| [`mcs-finance-products`](mcs-finance-products) | Catálogo de productos financieros, cuentas asociadas a clientes y actualización de saldos mediante depósitos. | `8082` |
| [`mcs-security-service`](mcs-security-service) | Servidor de autorización OAuth2, emisión de JWT y publicación de claves para su validación. | `9000` |

Los servicios de clientes y productos utilizan Spring Data JPA sobre una instancia compartida de **PostgreSQL 15**, publicada en el puerto local `5433`. El archivo [`init.sql`](init.sql) define las tablas y los datos iniciales de clientes, productos y cuentas. El servidor de autorización mantiene el registro de clientes OAuth2 en memoria.

## Funcionalidades implementadas

- **Gestión de clientes:** registro, búsqueda por documento, actualización y bloqueo lógico mediante el estado `BLOQUEADO`. Incluye validaciones de correo, teléfono y longitud del documento según su tipo: DNI, RUC o carnet de extranjería. El listado del BFF web filtra los clientes activos.
- **Productos y cuentas:** creación, consulta y actualización del catálogo en el servicio de productos, además de asociación de un producto a un cliente con número de cuenta y saldo.
- **Consulta financiera consolidada:** el BFF móvil descifra el identificador recibido, busca al cliente por documento y consulta sus productos usando el ID obtenido. Combina ambos resultados en un DTO con los datos del cliente y sus cuentas.
- **Depósitos:** el canal móvil envía la operación al servicio financiero, que verifica que el monto sea mayor a cero y actualiza el saldo usando `BigDecimal`.
- **Streaming con Server-Sent Events:** ambos BFF exponen el catálogo en `/api/product-finances/stream`. Es una demostración de emisión progresiva de productos, con un segundo entre elementos.

## Decisiones técnicas

### APIs específicas por canal

Los BFF utilizan **Spring WebFlux, WebClient y Project Reactor** (`Mono` y `Flux`) para consumir y componer respuestas de los servicios. Los servicios de negocio utilizan **Spring MVC y JPA** para las operaciones de persistencia. Esta separación concentra la adaptación de las respuestas en los BFF y mantiene la lógica de clientes y productos en sus respectivos módulos.

Los DTO de las APIs de los BFF se separan de los DTO recibidos de los servicios de negocio. **MapStruct** realiza los mapeos, evitando exponer directamente las entidades de persistencia.

### Autorización y consulta mediante código cifrado

**Spring Authorization Server** registra los clientes `bff-web` y `bff-mobile` con el flujo OAuth2 `client_credentials`. Ambos BFF están configurados como Resource Servers y validan los JWT mediante las claves publicadas en `/oauth2/jwks`.

El BFF móvil incorpora además un servicio de cifrado **RSA de 2048 bits**, con codificación Base64 URL-safe, para el código utilizado en la consulta consolidada. El par de claves se genera al iniciar el servicio.

### Seguimiento de solicitudes y manejo de errores

Los BFF generan un `X-Track-Id` y propagan ese identificador junto con `X-BFF-Source` a los servicios de negocio. Los interceptores validan estas cabeceras y las incorporan al contexto de logging; los aspectos AOP registran la ejecución de los controladores.

El manejo de excepciones se centraliza con `@RestControllerAdvice`. Los clientes HTTP de los BFF interpretan los errores de los servicios y los adaptan a respuestas con estado, mensaje y ruta.

## Tecnologías

| Área | Tecnologías |
| --- | --- |
| Lenguaje y framework | Java 17, Spring Boot 3.5.5 |
| APIs y comunicación | Spring MVC, Spring WebFlux, WebClient, Project Reactor, SSE |
| Seguridad | Spring Security, Spring Authorization Server, OAuth2, JWT, RSA |
| Persistencia | Spring Data JPA, PostgreSQL 15 |
| Mapeo y logging | MapStruct, Lombok, Spring AOP, SLF4J/MDC |
| Herramientas | Maven multimódulo, Docker, Docker Compose, OpenAPI/Swagger, Postman |
| Pruebas | JUnit 5, Mockito |

## Pruebas y documentación

El repositorio incluye pruebas unitarias de la lógica de negocio con JUnit 5 y Mockito:

- [`CustomerServiceImplTest`](mcs-customers/src/test/java/com/castaneda/mcs_customers/services/impl/CustomerServiceImplTest.java): búsquedas, registro, validaciones, actualización y bloqueo de clientes.
- [`ProductFinanceServiceImplTest`](mcs-finance-products/src/test/java/com/castaneda/mcs_finance_products/services/ProductFinanceServiceImplTest.java): consulta, creación y actualización de productos, incluidos los casos de producto inexistente y tipo duplicado.

La colección [`MCS - Castaneda`](MCS%20-%20Castaneda.postman_collection.json) reúne solicitudes para explorar las APIs. Los BFF y los servicios de clientes y productos incluyen documentación OpenAPI y Swagger UI en `/swagger-ui.html`.

## Ejecución local

Con Java 17, Maven y Docker Compose disponibles, desde la raíz del repositorio:

```bash
mvn clean package -DskipTests
docker compose up --build -d
```

Los Dockerfiles utilizan los JAR generados por Maven. PostgreSQL ejecuta `init.sql` al crear el volumen de datos por primera vez. Los BFF quedan disponibles en `http://localhost:8030` (web) y `http://localhost:8080` (móvil).
