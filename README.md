# Post-contenido — Unidad 5: Integración en Aplicaciones Web
## Sistema de Reserva de Laboratorios (Parte 1: API REST)

## Descripción
Repositorio del post-contenido de la Unidad 5 de Patrones de Diseño de Software. Proyecto Spring Boot (`reservas-labs-api`) para la gestión de reservas de laboratorios de cómputo universitarios, implementando una arquitectura en capas desacoplada (Entity, Repository, Service, Controller) sobre una base de datos en memoria H2.

## Parte 1 — Repository, Service y Controller REST
LaboratorioRepository y ReservaRepository extienden JpaRepository; ReservaRepository agrega una consulta JPQL propia para detectar solapamientos de horario en el motor de base de datos. ReservaService concentra las reglas de negocio institucionales (solapamiento, horario de atención de 07:00 a 21:00, duración entre 30 minutos y 3 horas, y rechazo de cancelaciones extemporáneas). ReservaController y LaboratorioController exponen las rutas REST bajo `/api/reservas` y `/api/laboratorios`. Ver paquetes `model/`, `repository/`, `service/`, `exception/` y `controller/`.

## Cómo ejecutar
```bash
$ mvn clean package
$ mvn spring-boot:run
```

- **API REST Reservas:** http://localhost:8080/api/reservas
- **API REST Laboratorios:** http://localhost:8080/api/laboratorios
- **Consola H2:** http://localhost:8080/h2-console  
  - *JDBC URL:* `jdbc:h2:mem:reservas_labs_db`  
  - *Usuario:* `sa` | *Contraseña:* (vacía)

## Decisiones de diseño

### Punto de decisión 1 — Ubicación de la validación de solapamiento
El dilema de diseño consistió en determinar si la validación de solapamiento de horarios debía procesarse por completo en la capa Service, extrayendo todas las reservas del laboratorio a memoria RAM para evaluarlas mediante Java puro, o si debía fundamentarse en una consulta específica del Repository que realice el filtrado en el motor de base de datos. La alternativa de filtrar en memoria se descartó categóricamente debido a que, a medida que el historial de reservas crece en producción, transferir tablas masivas por la red e instanciar colecciones enteras degrada el rendimiento de forma crítica y genera un cuello de botella innecesario.

Por este motivo, se adoptó una solución colaborativa entre capas. La interfaz `ReservaRepository` resuelve de manera eficiente la consulta JPQL `buscarSolapamientos`, delegando el filtrado temporal al motor SQL mediante índices relacionales. Sin embargo, el Repository se limita exclusivamente a responder la pregunta de acceso a datos sobre qué registros colisionan en ese rango horario. La decisión final de negocio (evaluar si la lista resultante contiene elementos y detonar la excepción `ReservaConflictException` con un mensaje institucional claro) reside de manera exclusiva en `ReservaService`.

Si la capa de presentación o los controladores llamaran directamente a la consulta `buscarSolapamientos()` del Repository eludiendo al Service, se rompería el principio de separación de preocupaciones. En dicho escenario adverso, el controlador quedaría fuertemente acoplado a la infraestructura de persistencia y asumiría lógica de dominio que no le corresponde, lo cual impediría reutilizar la validación en otras interfaces cliente y obligaría a replicar las comprobaciones de estado manualmente.

### Punto de decisión 2 — Reglas con y sin apoyo del Repository
El segundo dilema radicó en establecer el límite arquitectónico entre las reglas de dominio que exigen conectarse a la base de datos y aquellas que deben resolverse de forma autónoma dentro del Service. El criterio rector adoptado determina que una regla de negocio requiere el apoyo del Repository únicamente cuando su comprobación depende del estado histórico o concurrente de otras entidades del sistema, como sucede con la detección de solapamientos entre solicitudes ajenas.

Por el contrario, la validación del horario de atención de la universidad (entre las 07:00 y las 21:00) y de la duración permitida por sesión (mínimo 30 minutos y máximo 3 horas) depende exclusivamente de los atributos propios de la reserva que se pretende crear. Debido a que estos datos vienen dados en el objeto entrante, la verificación se encapsuló en el método privado `validarHorarioYDuracion` dentro de `ReservaService`, operando con las clases estándar `Duration` y `LocalTime` de Java puro.

Ejecutar esta verificación en memoria antes de cualquier interacción con la base de datos evita llamadas de entrada y salida (I/O) superfluas, rechaza de forma inmediata solicitudes con parámetros inconsistentes y reduce la latencia global del sistema, asegurando que el acceso a datos solo se efectúe cuando la entidad satisface todas sus invariantes intrínsecas.

### Nota sobre LaboratorioController
Dentro de la arquitectura implementada, la clase `LaboratorioController` constituye la única excepción intencional a la directriz de diseño que establece que un controlador nunca debe acceder directamente al repositorio. El catálogo de laboratorios opera como una administración básica de consulta y registro que carece de invariantes de negocio o validaciones de dominio complejas.

Bajo estas condiciones, introducir una clase intermedia como `LaboratorioService` cuyos métodos únicamente delegaran llamadas hacia `LaboratorioRepository` incurriría de forma directa en el antipatrón de Service anémico. Dicho antipatrón incrementa la complejidad accidental del proyecto y genera código redundante sin aportar valor funcional. Por lo tanto, la capa de servicio se incorpora exclusivamente cuando existe lógica de negocio sustancial que justifique su presencia, como sucede en `ReservaService`, evitando seguir esquemas de capas de manera dogmática o puramente mecánica.

## Evidencias de Ejecución (Capturas de Pantalla)

### Parte 1: Endpoints REST y Persistencia en H2
- **Creación de Reserva Exitosa (HTTP 201):**  
  ![Reserva Creada 201](screenshots/p1/captura1_reserva_201.png)

- **Conflicto de Solapamiento Detectado (HTTP 409):**  
  ![Conflicto de Solapamiento 409](screenshots/p1/captura2_solapamiento_409.png)

- **Rechazo por Horario de Atención Fuera de Rango:**  
  ![Horario Inválido](screenshots/p1/captura3_horario_invalido.png)

- **Registros Almacenados en Consola H2:**  
  ![Consola H2](screenshots/p1/captura4_h2_reservas.png)

## Herramientas utilizadas
- Java 17, Spring Boot 3.2, Spring Data JPA, H2 Database
- Apache Maven, cURL / PowerShell, Git, GitHub

## Conclusiones
El desarrollo de esta primera parte permitió consolidar una arquitectura en capas limpia, comprobando que la separación estricta entre presentación, servicio y persistencia previene el acoplamiento y facilita la prueba independiente de componentes. La distinción entre validaciones delegadas al motor SQL y validaciones autónomas en memoria optimiza el uso de recursos y garantiza la integridad de los datos institucionales.