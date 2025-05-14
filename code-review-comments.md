# Code Review - Servicio de Gestión de Calidad de Anuncios

## Estructura Actual del Proyecto

El proyecto sigue una estructura que combina elementos de arquitectura hexagonal y Domain-Driven Design (DDD), dividido en tres capas principales:

- **domain**: Contiene las entidades y reglas de negocio (Ad, Picture, Typology, etc.)
- **application**: Contiene la lógica de aplicación (AdsService, AdsServiceImpl)
- **infrastructure**: Contiene la comunicación con el exterior, dividida en:
  - **api**: Controladores REST y DTOs (AdsController, PublicAd, QualityAd)
  - **persistence**: Implementación de repositorios (InMemoryPersistence)

## Recomendaciones de Mejora

### 1. Arquitectura y Estructura

#### 1.1. Separación de responsabilidades

- **AdRepository#1-13**: El repositorio no debería estar en la capa de dominio, solo deberia estar la interfaz(port) y la implementación(adapter) debería estar en la capa de infraestructura.

- **AdsService#10-12**: La clase de servicio tiene demasiadas responsabilidades, incluyendo la lógica de negocio para calcular puntuaciones y la transformación de entidades a DTOs. Deberían separarse estas responsabilidades.

- **AdsServiceImpl#14-139**: La clase de servicio tiene demasiadas responsabilidades, incluyendo la lógica de negocio para calcular puntuaciones y la transformación de entidades a DTOs. Deberían separarse estas responsabilidades.

#### 1.2. Cumplimiento con Arquitectura Hexagonal

- La arquitectura actual mezcla conceptos hexagonales con una estructura más tradicional en capas. Para una arquitectura hexagonal pura, deberíamos:
  - Separar claramente los puertos (interfaces) de los adaptadores (implementaciones)
  - Crear puertos secundarios para la persistencia
  - Orientar el dominio al núcleo de la aplicación

### 2. Código y Buenas Prácticas

#### 2.1. Clases de Dominio

- **Ad#55-117**: Las clases de dominio están usando getters/setters, debería usar la librería de lombok y usarla tambien para los métodos equals, hashCode y toString

- **Ad#50**: El método `isComplete()` tiene una lógica compleja con múltiples condiciones anidadas. Debería refactorizarse para mayor claridad y mantenibilidad.


#### 2.2. Lógica de Negocio

- **AdsServiceImpl#68-119**: La lógica para calcular la puntuación es compleja y difícil de mantener. Debería extraerse a clases separadas siguiendo el patrón Strategy o Chain of Responsibility.

- **AdsServiceImpl#93-103**: La lógica para puntuar según el número de palabras tiene condiciones anidadas que podrían simplificarse.

- **AdsServiceImpl#110-114**: La búsqueda de palabras clave es case-sensitive y no normaliza acentos, lo que podría causar problemas con textos reales.

- **AdsServiceImpl#119**: El puntaje de completitud sobrescribe la puntuación total en lugar de sumarla, lo que parece un error según los requisitos.

#### 2.3. Patrones y Principios

- **AdsServiceImpl#24-35 y #42-57**: Hay duplicación de código en la transformación de entidades a DTOs. Debería extraerse a un mapper o utilizar una biblioteca como MapStruct.

- No se están utilizando Value Objects para conceptos de dominio como Score.

- No se están aplicando patrones como Factory, Builder o Repository de forma coherente.

### 3. Testing

- **AdsServiceImplTest**: Los tests son muy limitados y solo verifican que se llamen ciertos métodos, no la lógica real de cálculo de puntuaciones.

- Faltan tests unitarios para la lógica de dominio.

- No hay tests para los controladores REST ni para los repositorios.

#### 3.1 Tests de Integración

- No existe ningún test de integración que verifique el flujo completo desde la API REST hasta la capa de persistencia.
  
- Deberían implementarse tests de integración usando `@SpringBootTest` para probar el comportamiento completo de los endpoints:
  - Test para verificar el cálculo de puntuaciones
  - Test para verificar la obtención de anuncios públicos
  - Test para verificar la obtención de anuncios de calidad

- Los tests de integración deberían configurar un conjunto de datos de prueba que permitan validar todos los casos de uso descritos en los requisitos, como distintos tipos de anuncios, puntuaciones por debajo/encima del umbral, etc.

### 4. Performance y Escalabilidad

- **InMemoryPersistence**: La implementación en memoria no es adecuada para un entorno de producción. Debería haber una implementación alternativa con una base de datos real.

- No hay manejo de concurrencia ni transacciones, lo que podría causar problemas en un entorno multi-usuario.

### 5. API REST y Documentation

- **AdsController**: No hay validación de entrada ni manejo adecuado de errores.

- Falta documentación de la API (Swagger/OpenAPI) para facilitar su uso por otros desarrolladores.

- El endpoint `calculateScore` devuelve `202 Accepted` pero no ofrece un mecanismo para verificar el estado de la operación.

- **AdsController#27-31**: El método `calculateScore()` está mal implementado porque:
  - Usa `@GetMapping` cuando debería ser `@PostMapping` o `@PutMapping`, ya que modifica el estado del servidor (calcula y guarda puntuaciones).
  - El verbo HTTP GET está diseñado solo para operaciones de lectura que no modifican recursos.
  - Devuelve `ResponseEntity<Void>` sin proporcionar al cliente información sobre el resultado de la operación.
  - Si el cálculo es potencialmente costoso (muchos anuncios), podría bloquear el hilo de la petición durante mucho tiempo, siendo mejor una implementación asíncrona.

#### 5.1 Manejo de Excepciones

- No existe un mecanismo centralizado para el manejo de excepciones en la aplicación. Es necesario implementar un `@ControllerAdvice` para gestionar las excepciones de forma consistente.

- Debería crearse una estructura de excepciones que incluya:
  - Excepciones de dominio específicas (ej: `InvalidAdException`, `ScoreCalculationException`)
  - Excepciones técnicas (ej: `PersistenceException`, `RepositoryException`)
  - Un `GlobalExceptionHandler` que gestione todas las excepciones y devuelva respuestas HTTP apropiadas

- El `@ControllerAdvice` debería manejar al menos:
  - Excepciones 400 (Bad Request) para errores de validación
  - Excepciones 404 (Not Found) para recursos no encontrados
  - Excepciones 500 (Internal Server Error) para errores inesperados
  - Logging adecuado de las excepciones para facilitar la depuración

### 6. Configuración y Propiedades

- **application.yml**: El archivo está vacío. Deberían definirse propiedades para diferentes entornos (dev, test, prod).

### 7. Dependencias y Actualización

- El proyecto usa Spring Boot 2.1.3.RELEASE, que es una versión antigua (de febrero 2019) con posibles vulnerabilidades. Debería actualizarse a la versión más reciente (Spring Boot 3.x).

- La versión de Java es 1.8, que ya no recibe actualizaciones de soporte público. Debería actualizarse a Java 17 o 21 (LTS).

- Faltan dependencias importantes:
  - `spring-boot-starter-validation` para validación de entrada en controladores
  - `springdoc-openapi-ui` (o `springfox-swagger2`) para documentación de la API
  - `spring-boot-starter-test` para pruebas completas con Spring Boot
  - `spring-boot-starter-actuator` para monitorización y métricas
  - `spring-boot-devtools` para desarrollo más eficiente

- No hay dependencias para persistencia, lo que confirma que solo se usa memoria. Deberían añadirse:
  - `spring-boot-starter-data-jpa` para persistencia con JPA
  - Una base de datos embebida como H2 para desarrollo y pruebas

### 8. Mejoras Específicas Recomendadas

1. **Refactorizar la lógica de cálculo de puntuaciones**:
   - Crear clases separadas para cada regla de puntuación
   - Aplicar el patrón Chain of Responsibility o Strategy

2. **Mejorar el modelo de dominio**:   
   - Eliminar getters/setters y usar Lombok

3. **Separar el mapeo de entidades y DTOs**:
   - Crear clases Mapper dedicadas o usar MapStruct

4. **Mejorar los tests**:
   - Añadir tests unitarios para cada regla de puntuación
   - Añadir tests de integración para el flujo completo
   - Añadir tests para los controladores REST

5. **Mejorar la API REST**:
   - Añadir validación de entrada
   - Implementar manejo adecuado de errores con `@ControllerAdvice`
   - Documentar la API con Swagger/OpenAPI
   - Corregir el método `calculateScore()` para usar el verbo HTTP adecuado y proporcionar respuestas significativas

6. **Implementar persistencia adecuada**:
   - Añadir una implementación con JPA/Hibernate
   - Configurar transacciones y manejo de concurrencia

7. **Mejorar la configuración**:
   - Definir propiedades para diferentes entornos
   - Añadir logging adecuado

8. **Actualizar dependencias**:
   - Actualizar a Spring Boot 3.x
   - Actualizar a Java 17 o 21
   - Añadir las dependencias faltantes mencionadas

9. **Cumplir con la arquitectura hexagonal**:
   - Separar puertos y adaptadores correctamente
   - Orientar el dominio al núcleo de la aplicación

Estos cambios permitirían tener una aplicación más mantenible, testeable y escalable que cumpla con los principios de DDD y arquitectura hexagonal. 