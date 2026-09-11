Ejercicio 1: Rutas y verbos HTTP

Qué se viola:

Recursos como sustantivos, no verbos en la URL: /obtenerTodasLasTareas, /crearNuevaTarea meten el verbo en la ruta, cuando en REST el verbo ya lo da el método HTTP. Es redundante y poco idiomático.
Uso incorrecto del verbo HTTP para la acción: modificar una tarea con POST en vez de PUT/PATCH, y eliminar con GET (¡grave! GET debe ser una operación segura y sin efectos secundarios; usarlo para borrar rompe esa garantía, y además navegadores/proxies pueden cachear o prefetchear GETs, borrando cosas sin que el usuario lo pida).

Reescritura orientada a recursos:

GET    /api/v1/tareas
POST   /api/v1/tareas
PUT    /api/v1/tareas/:id   (o PATCH si es parcial)
DELETE /api/v1/tareas/:id

La idea central: la URL identifica el recurso (/tareas, /tareas/:id), y el método HTTP identifica la acción sobre ese recurso.

Ejercicio 2: as no valida nada

as CrearTareaDto es una aserción de tipo: le dice al compilador "confiá en mí, esto es un CrearTareaDto". Pero TypeScript se borra completamente en tiempo de ejecución (se compila a JS plano) — no genera ningún chequeo real. Es solo azúcar para el editor/compilador.

Riesgos concretos:

Con {}: datos.titulo es undefined. Si el service intenta datos.titulo.trim(), explota con TypeError: Cannot read properties of undefined.
Con {"titulo": 123}: titulo es un número, no string. Si más adelante se guarda en la base o se usa en lógica que asume string (.toLowerCase(), concatenación, etc.), falla o genera datos corruptos silenciosamente.

La conclusión pedagógica: los tipos de TypeScript son un contrato en tiempo de compilación, no una validación en runtime. Para runtime hace falta validación explícita (a mano, como el ejercicio 3, o con una librería como Zod).

Ejercicio 3: Validación manual

1) ¿Por qué 422 y no 400?

400 Bad Request = el cuerpo está mal formado sintácticamente (no es JSON válido, o falta la estructura básica).
422 Unprocessable Content = el JSON está bien formado sintácticamente, pero semánticamente no cumple las reglas de negocio (título muy corto, prioridad no permitida). El servidor entendió perfectamente lo que le mandaron, pero no lo puede procesar.

Es una distinción fina, pero importante: sintaxis vs. semántica/reglas de negocio.

2) {"titulo": " AB ", "prioridad": "alta"}

objeto.titulo.trim() → "AB", longitud 2.
La condición es < 3, y 2 es menor que 3 → falla la validación, lanza AppError(422, "INVALID_TITLE", ...).

Ojo con el detalle: la validación de longitud se hace después de hacer .trim(), así que espacios en los extremos no cuentan para el mínimo. Eso es correcto y evita que alguien "trampee" con " " + letras de relleno solo de espacios.

Ejercicio 4: PUT vs PATCH

1) PUT /tareas/5 con solo {"completada": true}

Según la especificación HTTP, PUT significa reemplazo completo del recurso. El código hace:

js
const { titulo, prioridad, completada } = req.body;
service.reemplazar(id, { titulo, prioridad, completada });

Como titulo y prioridad no vienen en el body, quedan undefined. El resultado: la tarea pierde su título y prioridad (se sobrescriben con undefined o valores vacíos). Esto es el problema clásico de usar PUT quirúrgicamente — si el cliente no manda el objeto completo, "borra" sin querer los campos faltantes.

2) Idempotencia

Idempotente = ejecutar la operación 1 vez o N veces produce el mismo estado final en el servidor. PUT es idempotente: reemplazar el recurso por el mismo valor 5 veces deja el recurso igual que reemplazarlo 1 vez.
POST no es idempotente: cada llamada típicamente crea un recurso nuevo (si hacés POST /tareas 3 veces con el mismo body, tenés 3 tareas nuevas, no 1).
PATCH, en general, tampoco se garantiza idempotente (depende de la operación: "sumar 1" no lo es; "setear campo a X" sí lo es).
Ejercicio 5: Parseo de parámetros

1) GET /tareas/abc

Number("abc") → NaN. Entonces:

Number.isInteger(NaN) → false → la condición !Number.isInteger(id) es true → se lanza AppError(400, "INVALID_ID", ...).
La API responde 400 Bad Request con ese mensaje de error (gracias al middleware centralizado, que veremos en el ejercicio 6).

2) ¿Por qué validar entero positivo > 0?

Porque los IDs en este dominio son autoincrementales positivos. Validar antes de tocar la capa de servicio:

Evita propagar NaN o negativos a consultas/lógica de negocio, donde podrían causar comportamientos raros (ej. buscar índice -1 en un array, o comparaciones que siempre den false/true inesperadamente).
Es "fail fast": cortás el error lo antes posible, con un mensaje claro, en vez de que explote más abajo con un error críptico.
Ejercicio 6: Middleware de errores centralizado

1) ¿Por qué filtrar instanceof AppError?

Porque AppError representa errores esperados y controlados (validación, recurso no encontrado, etc.) que ya tienen su status, code y message pensados para mostrarse al cliente tal cual. Cualquier otro error (una excepción no prevista, un fallo de librería, un bug) no es seguro exponerlo directamente — hay que tratarlo como error genérico 500.

2) ¿Por qué no exponer stack trace / detalles técnicos?

Seguridad: el stack trace puede revelar rutas del servidor, estructura interna del código, versión de librerías, y en el caso de un fallo de BD, hasta strings de conexión o nombres de tablas — información valiosa para un atacante.
UX/contrato: el cliente no necesita (ni debería) parsear detalles internos; solo necesita saber que algo salió mal en el servidor.
La traza sí se loggea internamente (console.error(error)) para que el equipo de desarrollo pueda diagnosticar, pero nunca sale en la respuesta JSON.
Ejercicio 7: Acoplamiento vs. Inyección de Dependencias

1) El problema

El controlador crea sus propias dependencias (new TareasRepository(), new TareasService(...)) en vez de recibirlas. Esto es acoplamiento fuerte (tight coupling):

No podés testear el controlador de forma aislada, porque siempre va a instanciar el repositorio real (que quizás toca una base de datos real).
Si cambia la implementación del repositorio (por ejemplo, de memoria a PostgreSQL), hay que tocar el controlador.
Viola el Principio de Inversión de Dependencias (la "D" de SOLID): las clases de alto nivel no deberían depender de implementaciones concretas de bajo nivel.

2) Refactor con Inyección de Dependencias

typescript
export class TareasController {
    constructor(private readonly service: TareasService) {}

    public obtenerTodas = (_req: Request, res: Response) => {
        const data = this.service.obtenerTodas();
        res.status(200).json({ data });
    }
}

// Composición externa (ej. en un archivo de "bootstrap" o "container")
const repository = new TareasRepository();
const service = new TareasService(repository);
const controller = new TareasController(service);

Ahora el controlador recibe el service ya armado, no lo crea. Esto facilita el testing porque en un test unitario podés inyectar un mock/stub del service:

typescript
const mockService = { obtenerTodas: jest.fn().mockReturnValue([...]) } as any;
const controller = new TareasController(mockService);
// probás el controller sin tocar la lógica real ni la BD
Ejercicio 8: Paginación

1) ?page=3&limit=5

start = (3 - 1) * 5 = 10
end   = 10 + 5 = 15

Entonces slice(10, 15) → devuelve los elementos con índice 10 al 14 (5 elementos, la "página 3" de a 5).

2) ?page=-2&limit=5000

Math.max(Number(-2) || 1, 1) → Number(-2) es -2 (valor truthy, no cae en el || 1), entonces Math.max(-2, 1) → 1. El page negativo queda forzado al mínimo válido (1).
Para limit: Math.max(5000, 1) → 5000, luego Math.min(5000, 100) → 100. El límite queda "clampeado" (acotado) a un máximo razonable (100), evitando que alguien pida 5000 registros de una y sobrecargue el servidor.

Esto es una técnica común: sanitizar/acotar inputs del cliente en vez de confiar ciegamente en ellos.

Ejercicio 9: Arquitectura por capas violada

1) Responsabilidades mezcladas

En un solo bloque de router se está haciendo:

Acceso a datos directo (baseDeDatosMemoria.find(...)) → esto es trabajo del Repository.
Lógica de negocio (tarea.vistas = (tarea.vistas || 0) + 1) → esto es trabajo del Service.
Manejo de la petición/respuesta HTTP → lo único que sí le corresponde al Router/Controller.

Se están saltando por completo las capas de Controller (delgado, orquesta), Service (reglas de negocio) y Repository (acceso a datos abstraído).

2) Responsabilidad única de Router y Controller

Router: mapear URL + método HTTP → función controladora. Nada de lógica.
Controller: traducir la petición HTTP (leer req.params, req.body, req.query) a una llamada al Service, y traducir el resultado del Service a una respuesta HTTP (status code, formato JSON). No debe conocer detalles de cómo se guardan los datos ni contener reglas de negocio.
Ejercicio 10: OpenAPI

1) {"titulo": "A", "prioridad": "urgente"}

Según el schema:

titulo: "A" viola minLength: 3 (tiene 1 carácter).
prioridad: "urgente" no está en el enum: [baja, media, alta].

Ambas violaciones son de validación semántica sobre un JSON bien formado → según el contrato, la API debe responder 422 ("Datos inválidos"), tal como está documentado en el spec.

2) OpenAPI vs. Swagger

OpenAPI es la especificación (el estándar, el formato/estructura del documento que describe una API — rutas, schemas, respuestas, etc.). Actualmente mantenida por la OpenAPI Initiative.
Swagger era el nombre original de ese ecosistema de herramientas (Swagger UI, Swagger Editor, Swagger Codegen) antes de que la especificación se donara y renombrara a "OpenAPI Specification" (a partir de la versión 3.0). Hoy "Swagger" sobrevive como marca de las herramientas (Swagger UI para visualizar, etc.), no como el nombre del estándar en sí.
Ejercicio 11: Cambios incompatibles (breaking changes)

1) ¿Por qué rompe a los clientes?

Los clientes (frontend, mobile) ya tienen código que asume la estructura original: response.data.id, response.data.prioridad. Si el backend cambia a response.data.metadata.id y response.data.nivelPrioridad:

El código cliente que lee response.data.id ahora obtiene undefined.
El código que lee response.data.prioridad también obtiene undefined.

Esto rompe funcionalidad sin que el cliente haya cambiado nada — es un breaking change de contrato, no aditivo (agregar un campo nuevo no rompe nada; renombrar o mover uno existente, sí).

2) Cómo versionar correctamente

En vez de modificar /api/v1/... in place, se crea una nueva versión de la ruta:

/api/v2/tareas/:id

Así los clientes viejos siguen usando /api/v1/... (que no cambia) mientras los nuevos o los que migren usan /api/v2/.... Da tiempo a deprecar v1 gradualmente en vez de romper todo de golpe.

Ejercicio 12: Header Location y envoltorio data

1) Propósito del header Location

Es parte de la semántica estándar de HTTP para respuestas 201 Created: le indica al cliente dónde (qué URL) puede encontrar/consultar el recurso recién creado. En este caso, /api/v1/tareas/{id} — el cliente podría hacer un GET a esa URL después para volver a consultar el recurso.

2) ¿Por qué envolver en { data: tarea }?

Ventajas prácticas:

Permite extender la respuesta en el futuro sin romper el contrato (agregar meta, links, pagination, etc. junto a data, sin mezclarlo con las propiedades del recurso).
Da consistencia a todas las respuestas de la API (todas tienen data, o error en caso de fallo — ver ejercicio 6), facilitando que el cliente parsee de forma uniforme.
Evita ambigüedad entre "propiedades del recurso" y "metadatos de la respuesta".
Ejercicio 13: Inmutabilidad en el repositorio

1) Riesgo de la Opción A

return this.tareas devuelve la misma referencia al array interno. Cualquier código externo que reciba ese array puede mutarlo directamente: repo.obtenerTodasA().pop() elimina realmente el último elemento del array interno del repositorio, sin pasar por ningún método de negocio ni validación. Esto rompe el encapsulamiento: el estado interno de la clase queda expuesto y modificable desde afuera sin control.

2) Cómo lo soluciona la Opción B

return [...this.tareas] crea una copia superficial (shallow copy) del array. Quien reciba el resultado puede hacer .pop(), .push(), etc. sobre esa copia sin afectar en absoluto el array interno real. El estado del repositorio queda protegido; solo se puede modificar a través de los métodos que la clase expone explícitamente (como crear, actualizar, etc.).

(Nota: es copia superficial, así que si los objetos Tarea dentro del array se mutaran directamente —tareas[0].titulo = "x"— eso sí afectaría el original, porque las referencias a los objetos internos siguen siendo las mismas. Para inmutabilidad total haría falta clonar también los objetos.)

Ejercicio 14: cURL

1) -i

Le dice a curl que muestre los headers de la respuesta HTTP además del body (por defecto solo muestra el body). Sirve para ver el código de estado (HTTP/1.1 201 Created), Content-Type, Location, etc.

2) Sin Content-Type: application/json

express.json() es un middleware que solo parsea el body como JSON si el header Content-Type indica application/json. Si ese header falta (o dice otra cosa), Express no interpreta el cuerpo como JSON: req.body queda como objeto vacío {} (asumiendo que no hay otro middleware de parseo que lo capture). Esto dispararía las validaciones del ejercicio 3 (título ausente → 422), aunque el cliente sí haya mandado datos en el string del -d.

Ejercicio 15: DTOs vs. Entidad completa

1) ¿Por qué id, completada, fechaCreacion no están en el DTO de entrada?

CrearTareaDto representa lo que el cliente debe/puede enviar al crear una tarea. Esos tres campos son:

Generados por el sistema (id, fechaCreacion) — el cliente no tiene forma de saber qué ID le va a tocar, ni tiene por qué mandar una fecha (el servidor sabe mejor cuál es el momento real de creación).
Un valor inicial de negocio (completada: false por defecto — una tarea recién creada nunca empieza completada).

Separar el DTO de entrada de la entidad completa evita que el cliente controle campos que no le corresponden.

2) ¿Dónde se generan?

En la capa de Service (a veces Repository, dependiendo de si el ID lo genera la lógica de negocio o la base de datos/motor de persistencia — por ejemplo, un autoincremental de SQL lo generaría el Repository/BD). Pero nunca deben venir del cliente HTTP, porque:

El id debe ser único y consistente con el sistema de persistencia (evitar colisiones, IDs duplicados o inventados).
La fechaCreacion debe reflejar el momento real del servidor, no lo que el cliente diga (que podría mentir o tener el reloj mal configurado) — es una cuestión de integridad de los datos.

¿Querés que armemos un quiz interactivo con estos temas para repasar, o preferís que profundicemos en algún ejercicio puntual (por ejemplo, DI con un ejemplo más completo, o validación con Zod en vez de a mano)?
