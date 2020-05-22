# Implementa autenticación en tus aplicaciones con el estándar JSON Web Token

¿Sabías que, autenticación y autorización son dos términos distintos?
* **Autenticación**: Es el proceso en el cual verificamos la identidad de un usuario. Es decir, comprobamos que ese usuario existe y es válido.

* **Autorización**: Es la acción de otorgar permisos, a los usuarios, para interactuar en las distintas funcionalidades de nuestra aplicación.

Ahora que ya sabes esto, te invito a continuar leyendo este post, para que aprendas cómo autenticar a tus usuarios con el estándar **JSON Web Token**.

A continuación hablaremos de qué es una sesión, sus distintos tipos y cómo podemos aplicar la autenticación para estas sesiones con JSON Web Token, en aplicaciones web. Estos conceptos no están atados a un lenguaje de programación en específico, sino que son aplicables en cualquiera de ellos.

## JSON Web Token
Es un estándar de internet que nos permite comunicarnos entre servicios web de una manera más segura. También, es un mecanismo de autenticación sin estado, lo que conocemos como "stateless". Éste tiene una anatomía que se compone de tres partes:

**Header**: El cual consta de la propiedad `alg`, que indica el algoritmo usado en la firma y la propiedad `typ`, que define el tipo de token, en nuestro caso `JWT`.
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
* Creador `iss`, identifica a quién creó el JWT.
* Tiempo de expiración `exp`, del JWT para verificar si está vencido y obligar al usuario a volver a autenticarse.
* Creado `iat`, indica cuando fue creado el JWT.

**Signature**: Es la firma del código JWT y está compuesta por el Header, más el Payload codificado. La firma del JWT se genera usando los campos anteriores, en base64 y una key secreta (que solo se sepa en los servidores que creen o usen el JWT) para usar un algoritmo de encriptación. La forma de hacerlo (usando pseudo código), sería la siguiente:

```
key =  'secret'
unsignedToken = base64Encode(header) + '.' + base64Encode(payload)
signature = SHA256(key, unsignedToken)
token = unsignedToken + '.' + signature
```

## Autenticación tradicional vs. JWT
* Tradicional: Al abrir el navegador se crea una sesión con un ID de la misma, el cual es almacenado en las cookies. A partir de ese momento, en todos los request, se envía la información almacenada.

<br>

* JWT: Al procesar la autenticación se firma un JWT, el cual genera un token que es enviado al cliente para ser almacenado en memoria o en las cookies y ser utilizado en cada request.

## Sesiones en la web
Una sesión es un intercambio de información semipermanente, también conocido como diálogo, entre dos o más dispositivos de comunicación. En términos generales, una sesión es una manera de preservar un estado deseado.

* **Sesión del lado del servidor**
Suele ser una pieza de información, que se guarda en memoria, o en una base de datos. Permitiendo hacerle seguimiento a la información de autenticación, con el fin de identificar al usuario y determinar cuál es su estado. Mantener la sesión de esta manera en el lado del servidor es lo que se considera *"statefull"*, es decir, que maneja un estado.

* **Sesión del lado del cliente**
En su base tiene la misma estructura que la sesión del lado del servidor, pero ésta es "stateless". Es decir, que no maneja estado y se comporta de manera distinta:
    + Cuando el usuario hace "login" agregamos una bandera con un token, para indicar que está "logueado".
    + En cualquier punto de la aplicación verificamos la expiración del token. Si el token expira, cambiamos la bandera para indicar que no está "logueado".
    + Esto se suele chequear cuando la ruta cambia. Si el token expiró, redireccionamos a la ruta de "login" y actualizamos el estado como "logout".

Entendiendo qué es un JSON Web Token, su anatomía y cómo utilizarlo en una sesión, tanto del lado del cliente como del servidor. Ahora te mostraré:

## Implementación de autenticación en NodeJS, Express y Mongoose.

* Lo primero que necesitamos es, crear un archivo con las variables de entorno que usaremos para la conexión a la base de datos y el secreto que nos servirá al momento de firmar el JWT.
```
// .env

// CONFIG
PORT=
NODE_ENV=

// MONGO
DB_USER=
DB_PASSWORD=
DB_HOST=
DB_NAME=

// AUTH
AUTH_JWT_SECRET
```

* Luego, para facilitar el manejo de las variables de entorno, creamos un archivo config. Éste paso es opcional, pero es una buena practica para mantener la modularidad en tu código.

```
// config.js

// Requerimos la librería 'dotenv' para traer las variables de entorno.
require("dotenv").config();

const config = {
  dev: process.env.NODE_ENV !== "production",
  port: process.env.PORT || 8001,
  dbUser: process.env.DB_USER,
  dbPassword: process.env.DB_PASSWORD,
  dbHost: process.env.DB_HOST,
  dbName: process.env.DB_NAME,
  authJwtSecret: process.env.AUTH_JWT_SECRET
};

module.exports = { config };
```

* Ahora, creamos dos archivos, para el manejo de usuarios. Uno para la conexión a la base de datos (en este caso yo utilice una base de datos Mongo, pero puedes utilizar la base de datos de tu preferencia) y otro para definir el modelo de schema del usuario.
```
// mongo.js

const mongoose = require("mongoose");
// Importo el archivo config para utilizar las variables de entorno.
const { config } = require('./config');

// Asigno las variables de entorno codificadas para un mejor manejo al momento de hacer el stringConnection.
const USER = encodeURIComponent(config.dbUser);
const PASSWORD = encodeURIComponent(config.dbPassword);
const DB_NAME = config.dbName;
const DB_HOST = config.dbHost;

// Asigno el stringConnection.
const MONGO_URI = `mongodb+srv://${USER}:${PASSWORD}@${DB_HOST}/${DB_NAME}?retryWrites=true&w=majority`;

// Establezco la conexión con la base de datos.
mongoose
  .connect(MONGO_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true
  })
  .then(db => console.log("Database is connected"))
  .catch(err => console.log(err));
 ```
```
// user.js

const { Schema, model } = require('mongoose');

// Schema del usuario con todos sus atributos.
const userSchema = new Schema({
    id: String,
	avatar: String,
	age: Number,
	email: String,
	name: String,
	role: 'admin' | 'user',
    surname: String,
    password: String
});

// Función básica, con un condicional, para validar la contraseña.
userSchema.methods.validatePassword = async function(password) {
    return (password === this.password);
}

module.exports = model('users', userSchema);
```

* Creamos un archivo con el enrutador para la autenticación de usuarios, con todas sus validaciones, el cual luego sera implementado al momento de exponer el servicio.

```
// auth.js

const express = require("express");
const jwt = require("jsonwebtoken");
const { config } = require("./config");

// Importamos el schema del usuario para utilizar sus funcionalidades.
const User = require("./User");

// El servicio recibirá una app de Express como parámetro.
const authApi = (app) => {

  // Instanciamos un enrutador de Express
  const router = express.Router();

  // Definimos la base del endpoint del servicio y que enrutador usara.
  app.use("/api/v0", router);

  // Definimos el endpoint que sera utilizado para autenticarse.
  router.post("/authenticate", async (req, res) => {

    // Extraemos el email y el password que viene en el body del request.
    const { email, password } = req.body;

    // Buscamos el usuario en la base de datos con el email recibido.
    const user = await User.findOne({ email: email });

    // Validamos que el usuario exista, caso contrario se envía un mensaje de 'usuario inexistente'.
    if (!user) {
      return res.status(404).send("The email doesn't exists");
    }

    // Verificamos el password del usuario.
    const validated = await user.validatePassword(password);

    // Validamos que el password sea correcta, caso contrario se envía un mensaje de 'usuario no autorizado'.
    if (!validated) {
      return res.status(401).json({ auth: false, token: null });
    }

    // Si todas las validaciones anteriores salieron bien, firmo el JWT con el ID del usuario, el secret de la aplicación y la cantidad de tiempo en segundos en la que expira el token.
    const token = jwt.sign({ id: user._id }, config.authJwtSecret, {
      expiresIn: 60 * 60 * 3
    });

    // Enviamos una respuesta de autenticación exitosa, acompañada del token.
    res.status(200).json({ auth: true, token });
  });
}

module.exports = authApi;
```

* Por último, y no menos importante, creamos el archivo index donde exponemos el servicio.

```
// index.js

const express = require("express");
const cors = require("cors");
const { config } = require("./config");
const authApi = require("./auth");

// Instancia de la conexión a la base de datos.
require("./mongo");

// Instancia un servidor de Express.
const app = express();

// Habilitamos el uso de JSON y CORS en todo el servicio.
app.use(express.json());
app.use(cors());

// Agregamos el enrutador al servicio.
authApi(app);

// Exponemos el servicio en el puerto configurado
app.listen(config.port, () => {
  console.log(`Listening http://localhost:${config.port}`);
});
```

Listo, con esos pasos ya tenemos una implementacion de autenticacion con JWT. Ahora, quiero invitarte a seguir creciendo en tu carrera profesional, tomando el [Curso de Autenticación con Passport.js](https://platzi.com/cursos/passport/), donde aprenderas a generar estrategias de autenticación Sign-In y Sign-Out usando Passport.js; Agregar autenticación con Facebook, Twitter y Google a tus desarrollos; y gestionar de manera sencilla los procesos de éxito y falla en la autenticación de tu aplicación.
