# ice-chat-project

# Chat por consola con ZeroC Ice

Taller de Computación en Internet I (09810 - TIC) - Universidad Icesi (2026-2)
Hecho por: Valentina Tangarife Rincón

## De qué se trata

En clase veníamos trabajando con sockets, donde uno mismo tiene que armar los mensajes, convertirlos a bytes, preocuparse por el orden de los bytes y manejar los hilos. En este taller hicimos un chat parecido, pero usando ZeroC Ice, que es un middleware que se encarga de toda esa parte. Ice usa el modelo RPC (Remote Procedure Call): el cliente llama los métodos del servidor como si fueran locales (por ejemplo `chatPrx.login("Alice")`) e Ice los manda por la red.

## Cómo funciona

Lo primero es el archivo `Chat.ice`, que es el contrato y está escrito en Slice. Slice no es un lenguaje de programación, solo sirve para describir qué puede pedirle el cliente al servidor. Ahí se define:

- `ChatMessage`: un mensaje con id, remitente, texto y hora
- `ChatException`: el error que manda el servidor cuando algo no se puede hacer, por ejemplo si el nickname ya existe
- `ChatRoom`: los métodos del servidor, que son `login`, `postMessage`, `getPendingMessages`, `getOnlineUsers` y `logout`

Los métodos que solo consultan están marcados como `idempotent`, para que Ice los pueda reintentar si falla la red sin causar problemas.

Con `slice2java` ese archivo se convierte en clases Java (`ChatRoom`, `ChatRoomPrx`, `ChatMessage` y `ChatException`). Gradle lo hace solo antes de compilar, con la tarea `compileSlice`. A partir de ahí, todo el servidor y el cliente son Java normal.

En el servidor, `ChatRoomI` tiene la lógica del chat: guarda los usuarios conectados y el historial de mensajes. Como Ice puede atender a varios clientes al mismo tiempo, usé estructuras seguras para hilos (`ConcurrentHashMap`, `CopyOnWriteArrayList` y `AtomicLong`) y `synchronized` en `login` y `logout`, para que dos personas no entren con el mismo nombre al mismo tiempo. `ServerMain` arranca Ice, abre el puerto 10000 y registra el servicio con el nombre `ChatService`.

El cliente (`ClientMain`) se conecta con el proxy `ChatService:default -h 127.0.0.1 -p 10000` y usa `checkedCast` para confirmar que del otro lado sí hay un `ChatRoom`. Después pide el nickname y arranca un hilo en segundo plano que cada 500 ms le pregunta al servidor si hay mensajes nuevos. Así uno puede seguir escribiendo mientras llegan los mensajes de los demás.

## Estructura del proyecto

```
ice-chat-project/
├── settings.gradle            # Declara los tres módulos
├── build.gradle               # Configuración común: Java 17 y librería de Ice
├── common/
│   ├── build.gradle           # Tarea compileSlice (ejecuta slice2java)
│   └── src/main/slice/
│       └── Chat.ice           # Contrato del servicio
├── server/
│   ├── build.gradle
│   └── src/main/java/chat/server/
│       ├── ChatRoomI.java     # Lógica del chat
│       └── ServerMain.java    # Arranque del servidor
└── client/
    ├── build.gradle           # Permite leer el teclado con gradle run
    └── src/main/java/chat/client/
        └── ClientMain.java    # Cliente por consola
```

## Qué se necesita

- Java 17 o superior (yo usé Java 25)
- Gradle (yo usé la 9.8.0)
- ZeroC Ice 3.7, para tener `slice2java`

En Mac se instala con:

```
brew install gradle
brew install zeroc-ice/tap/ice@3.7
```

Para comprobar que quedó bien, `slice2java -v` debe mostrar una versión 3.7.

Ojo: si se instala solo con `brew install ice`, queda la versión 3.8, que no trae slice2java. A mí me pasó y tuve que instalar la 3.7.

## Cómo correrlo

Primero compilar desde la carpeta del proyecto:

```
gradle build
```

Después abrir tres terminales. En la primera se corre el servidor, que debe mostrar `SERVIDOR ZEROC ICE INICIADO EXITOSAMENTE`:

```
gradle :server:run --console=plain
```

Y en las otras dos, un cliente en cada una:

```
gradle :client:run --console=plain
```

Cada cliente pide un nickname (yo usé Alice y bob). Para enviar un mensaje solo se escribe y se da Enter. También están los comandos `/users` para ver quién está conectado y `/exit` para salir.

## Pruebas

Probé cinco cosas:

- que los mensajes lleguen entre los clientes con el nombre y la hora
- que `/users` muestre a todos los conectados
- que no deje entrar dos veces con el mismo nombre: el servidor manda la `ChatException` y el cliente vuelve a pedir el nickname sin cerrarse
- que avise a los demás cuando alguien sale con `/exit`
- qué pasa si se apaga el servidor: el cliente muestra la excepción `com.zeroc.Ice.ConnectionRefusedException` y se cierra sin romperse

## Cambios que le hice a la guía

- Cambié la forma de escribir las tareas en los `build.gradle` (`tasks.register` y `tasks.named` en vez de `task` y `run { }`), porque la guía usa una sintaxis vieja y yo tengo Gradle 9.
- En la tarea `compileSlice` puse la ruta completa de `slice2java`, para que también funcione cuando se compila desde IntelliJ.
- Modifiqué el cliente para que, cuando se caiga el servidor, muestre el nombre de la excepción de Ice, porque antes solo mostraba el error de Java.
