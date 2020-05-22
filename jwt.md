# Aplica autenticación en tus aplicaciones con el estándar JSON Web Token

¿Sabías qué autenticación y autorización son dos términos distintos?
* **Autenticación** es el proceso en el cual verificamos la identidad de un usuario, es decir, comprobamos qué ese usuario existe y es válido.

<br>

* **Autorización** es la acción de otorgar permisos a los usuarios para interactuar en las distintas funcionalidades de nuestra aplicación.

Ahora qué ya sabes esto, te invito a continuar leyendo este post, para qué aprendas cómo autenticar a tus usuarios con el estándar **JSON Web Token**.

A continuación hablaremos de que es una sesión, sus distintos tipos y como podemos aplicar la autenticación para estas sesiones con JSON Web Token en aplicaciones web. Estos conceptos no están atados a un lenguaje de programación en especifico, sino que son aplicables en cualquiera de ellos.

## JSON Web Token
Es un estándar de internet qué nos permite comunicarnos entre servicios web de una manera más segura, también, es un mecanismo de autenticación sin estado, lo qué conocemos como ‘stateless’. Esté tiene una anatomía qué se compone de tres partes:

**Header**: El cual consta de la propiedad `alg` que indica el algoritmo usado para en la firma y la propiedad `typ` que define el tipo de token, en nuestro caso `JWT`.
```
{
  "alg": "HS256",
  "typ": "JWT"
}
```
**Payload**: Es donde guardamos toda la información de nuestros usuarios, aunque hay una serie de nombres de propiedades definidos en el estándar, podemos utilizar cualquier propiedad que nos sea de utilidad, por ejemplo:
```
{
  "id": "1",
  "username": "massimokris"
}
```

Algunas propiedades estándar:
* Creador `iss` - Identifica a quien creo el JWT
* Tiempo de expiración `exp` - del JWT para verificar si esta vencido y obligar al usuario a volver a autenticarse.
* Creado `iat` - Indica cuando fue creado el JWT.

**Signature**: Es la firma del código JWT y está compuesta por el Header más el Payload codificado. La firma del JWT se genera usando los campos anteriores en base64 y una key secreta (que solo se sepa en los servidores que creen o usen el JWT) para usar un algoritmo de encriptación. La forma de hacerlo sería la siguiente (usando pseudo código):

```
key =  'secret'
unsignedToken = base64Encode(header) + '.' + base64Encode(payload)
signature = SHA256(key, unsignedToken)
token = unsignedToken + '.' + signature
```

## Autenticación tradicional vs JWT
* Tradicional: Al abrir un navegador se crea una sesión con un ID de la misma, el cual es almacenado en las cookies y a partir de ese momento, en todos los request se envía la información almacenada.

<br>

* JWT: Al procesar la autenticación se firma un JWT, el cual genera un token que es enviado al cliente y este debe ser almacenado en memoria o en las cookies para ser utilizado en cada request.

## Sesiones en la web
Una sesión es un intercambio de información semipermanente, también conocido como diálogo, entre dos o más dispositivos de comunicación. En términos generales una sesión es una manera de preservar un estado deseado.

* ##### Sesión del lado del servidor
Suele ser una pieza de información que se guarda en memoria, o en una base de datos y esta permite hacerle seguimiento a la información de autenticación, con el fin de identificar al usuario y determinar cuál es el estado. Mantener la sesión de esta manera en el lado del servidor es lo que se considera *“statefull”*, es decir que maneja un estado.

* ##### Sesión del lado del cliente
En su base tiene la misma estructura que la sesión del lado del servidor, pero esta es "stateless", es decir que no maneja estado y se comporta de manera distinta:

   + Cuando el usuario hace “login” agregamos una bandera con un token para indicar que está "logueado".
   + En cualquier punto de la aplicación verificamos la expiración del token, si el token expira, cambiamos la bandera para indicar que no está "logueado".
   + Esto se suele chequear cuando la ruta cambia. Si el token expiró, redireccionamos a la ruta de “login” y actualizamos el estado como “logout”.

Entendiendo que es un JSON Web Token, su anatomía y como utilizarlo en una sesión tanto del lado del cliente como del servidor, te mostrare un ejemplo de implementación con JavaScript.
