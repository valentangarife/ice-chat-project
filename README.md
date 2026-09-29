# ice-chat-project

# Chat por consola con ZeroC Ice

Taller de Computación en Internet I - Universidad Icesi (2026-2)
Hecho por: Valentina Tangarife Rincón

## De qué se trata

En clase veníamos trabajando con sockets, donde uno mismo tiene que armar los mensajes, convertirlos a bytes y manejar los hilos. En este taller hicimos un chat parecido, pero usando ZeroC Ice, que es un middleware que se encarga de toda esa parte. Con Ice el cliente llama los métodos del servidor como si fueran locales (por ejemplo `chatPrx.login("Alice")`) y Ice los manda por la red.

Lo primero es el archivo `Chat.ice`, que es el contrato: ahí se dice qué métodos tiene el servidor (login, enviar mensaje, consultar mensajes, ver usuarios y salir) y qué datos se envían. Con `slice2java` ese archivo se convierte en clases Java, y a partir de ahí todo el servidor y el cliente son Java normal.

El proyecto está dividido en tres módulos de Gradle:

- `common`: el contrato `Chat.ice` y el código que genera slice2java
- `server`: `ChatRoomI`, que tiene la lógica del chat, y `ServerMain`, que arranca el servidor en el puerto 10000
- `client`: `ClientMain`, que pide el nickname, lee lo que uno escribe y tiene un hilo que cada 500 ms pregunta si hay mensajes nuevos

## Qué se necesita

- Java 17 o superior
- Gradle
- ZeroC Ice 3.7 (para tener `slice2java`)

En Mac se instala con:

```
brew install gradle
brew install zeroc-ice/tap/ice@3.7
```

Ojo: si se instala solo `brew install ice`, queda la versión 3.8, que no trae slice2java. A mí me pasó y tuve que instalar la 3.7.

## Cómo correrlo

Primero compilar desde la carpeta del proyecto:

```
gradle build
```

Después abrir tres terminales. En la primera se corre el servidor:

```
gradle :server:run --console=plain
```

Y en las otras dos, un cliente en cada una:

```
gradle :client:run --console=plain
```

Cada cliente pide un nickname (yo usé Alice y bob). Para enviar un mensaje solo se escribe y se da Enter. También están los comandos `/users` para ver quién está conectado y `/exit` para salir.

## Pruebas

Probé que los mensajes lleguen entre los clientes, el comando `/users`, que no deje entrar dos veces con el mismo nombre, que avise cuando alguien sale y qué pasa si se apaga el servidor. En ese último caso el cliente muestra la excepción `com.zeroc.Ice.ConnectionRefusedException` y se cierra sin romperse.

## Cambios que le hice a la guía

- Cambié la forma de escribir las tareas en los `build.gradle` porque la guía usa una sintaxis vieja y yo tengo Gradle 9.
- Modifiqué el cliente para que cuando se caiga el servidor muestre el nombre de la excepción de Ice, porque antes solo mostraba el error de Java.
