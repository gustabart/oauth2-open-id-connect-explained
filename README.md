# OAuth2 y Open ID Connect

## Intención:

Este documento busca describir de forma clara y precisa que es y como funciona OAuth2 y su extensión Open ID Connect. Pero de ninguna manera pretende ser una descipción pormenorizada y completa de los mismos. Para ello se debe recurrir a las especificaciones oficiales y/o demás referencias que se anexan al final de este documento.

## Tabla de Contenidos

## Introducción:

El framework OAuth2 define un mecanismo seguro mediante el cual una aplicaciòn cliente (Client) puede acceder a recursos que un usuario final (Resource Owner o End-User) posee en otro sistema (Resource Server). Decimos que es seguro porque es el usuario final quien autoriza a la aplicaciòn cliente a acceder a sus recursos alojados en el resource server. 

Mas adelante veremos que existen varias flujos de interacción pero el mas completo y comunmente usado es el Authorization Code.  
Bajo este flujo, el usuario final autoriza a la aplicaciòn cliente a acceder a sus recursos usando como intermediario a un servidor de autorización (Authorization Server) en el cual confía. Es decir, el usuario presenta sus credenciales al servidor de autorizaciòn y este otorga un token de acceso (Access Token) a la aplicación cliente que le servirá para acceder a los recursos del usuario alojados el servidor de recursos (que podrìa ser o no el mismo servidor de autorización).  

La ventaja de usar un servidor de autorizaciòn es que el usuario final en ningún momento necista brindar sus credenciales a la aplicación cliente.

El escenario descrito hasta aquì lo podemos instanciar de la siguiente manera:
- **Client:** Birthday Greeting - Una aplicaciòn web que manda saludos de cumpleaños a tus contactos de facebook.
- **Resource Server:** Facebook.
- **Authorization Server:** Facebook.
- **Resource Owner:** Bruno - Usuario final que quiera usar los servicios de Birthday Greeting.

En este escenario, cuando Bruno quiere acceder a 'Birthday Greeting', esta lo redirije mediante el user-agent a Facebook para que Bruno se autentique y preste consetimiento para que 'Birthday Greeting' acceda a sus contactos. Acto seguido Facebook intercambia mensajes con 'Birthday Greeting' para finalmente reponderle el access token.

Para que la comunicación entre el servidor de autenticación (Facebook) y el cliente (Birthday Greeting) sea posible y segura es necesario que antes (de alguna forma no definida en la especificación) el cliente se registre en el servidor de autorización obteniendo un id (client-id) y una contraseña 
(client-secret) del mismo. El cliente podrá obtener el token de acceso sólo presentando el client-id y client-secret que le fueron asignados.    

Este token de accesso es la llave que usará 'Birthday Greeting' para pedirle a Facebook los contactos de Bruno y enviarles un saludo de compleaños si corresponde.
Y ese es en principio el espíritu del token de acceso, un simple string que autoriza al cliente, pero que no contiene ninguna información acerca de la identidad del usuario (de Bruno en este caso) ni de ninguna otra informaciòn asociada al mismo. Aunque nada impida que sí la tenga de algún modo (la especificación es abierta en este sentido).

Esto signfica que OAuth2 es un protocolo de Autorización y NO de Autenticación.

**¿ Y que hay de Open ID Connect ?**

Muchas veces la aplicación Cliente necesita autenticar al usuario final en su sistema, es decir, necesita concer su identidad y mantener una sesión con el mismo
(típicamenete mandando una cookie al user-agent que sirva de session id). Se quiere que el usuario final opere sobre la apliación cliente como un usuario propio ella, pero sin que esta tenga que manejar su autenticación, ni nada de lo que eso conlleva como formularios de registro, de login, almacenar contraseñas y demás.

En este escenario tambièn se puede aplicar OAuth2 ya que justamente permite al cliente DELEGAR el login en otro sistema. Pero falta algo, ahora el Cliente no sólo necesita un token de acceso sino que además necesita conocer la identidad del usuario como ser su nombre, login, email, roles y demás datos.

Es aquí donde aparece Open ID Connect como un protocolo que extiende a OAuth2. Básicamente adiciona dos cosas.
 - ID Token: Es un token JWT enviado por el servidor de autorización (Facebook) junto con el Access Token que sirve para identificar al usuario.
 - UserInfo Endpoint: Es un endpoint que brinda información adicional del usuario como email, roles y demás en un documento JSON.

## Flujos:

Llamamos flujo al modelo de interacciones entre los distintos actores y dispositivos utilizados para llevar a cabo la autorización (y autenticación en el caso de Open ID Connect). OAuth2 define cuatro posibles flujos: 

- **Authorization Code**
- **Implicit**
- **Resource Owner password credentials**
- **Client Credentials**

En este documento nos centraremos fundamentalmente en Authorization Code, ya que es el mas completo, seguro, y comunmente usado.  
Los otros flujos suelen ser derivaciones del primero y sólo dedicaremos un parrafo en sus respectivas secciones. 

## Authorization Code Flow

Este flujo está optimizado para clientes confidenciales. Es decir, clientes que corren en un backend server y por lo tanto pueden asegurar la confidencialidad de claves. Este flujo se basa en redirecciones, por lo tanto el cliente debe ser capaz de interactuar con el user-agent del usuario enviandole respuestas con código de estado 302 y aceptando requests (vía redirecciones) del servidor de autorizacón.

[TODO - GRAFICO]

1- El usuario hace un request al cliente a través de un user-agent. 

   ```http
   GET /api/greeting HTTP/1.1
   Host: www.birthday_greeting.com 
   ```
   
   En verdad la especificaciòn no dice nada sobre este request, lo incluimos simplemente para que quede claro como se inicia el flujo.
 
2- El cliente encuentra que el usuario no tiene sesión y entonces redirije a la uri authorization de facebook mediante el user-agent.
    Genera entonces una resuesta de esta forma:

   HTTP/1.1 302 Found
   Location: https://www.facebook.com/v2.8/dialog/oauth?
					response_type=code
					&client_id=greeting-id
					&state=xyz
					&redirect_uri=https://www.birthday.greeting.com/login/oauth2/code/daut 
   
   El parametro redirect_uri funciona como un callback que el servidor de autorización usará en el paso siguiente.
   
   Además la respuesta suele incluir el Set-Cookie header para trazar la sesión en futuros requests (esto esta fuera del alcance del protocolo):
   
   Set-Cookie: JSESSIONID=DDE82AA1997A6179EA8F566D202D8C8E; Path=/; HttpOnly
 
3- El servidor de autenticación recibe el request de autorización y presenta una pàgina de login al usuario.
   El usuario se autentica con exito y otorga consentimiento para que el cliente acceda a sus recursos (o un subconjunto de ellos).

4- El servidor de autorización responde con una redirección mediante el user-agent.
	La url de redirección es el redirect_uri suminstrado por el cliente en el paso anterior. Además anexa a la misma un còdigo de autorización:	
	
     HTTP/1.1 302 Found
     Location: https://www.birthday.greeting.com/login/oauth2/code/daut?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz	

5- Con el código recibido en el paso anterior, el cliente solicita ahora un access token al servidor de autorización mediante una petición de tipo POST.

     POST /v2.8/oauth/token HTTP/1.1
     Host: https://www.facebook.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded

     grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
     &redirect_uri=https://www.birthday.greeting.com/login/oauth2/code/daut

     En el Authorization header se deben indicar un token que representa el client-id y client-secret codificados en base64.	 

6- El servidor de autorización verifica que todos los datos de la petición sean válidos y responde con un access token.
    Opcionalmente también puede responder un refresh token:
	 
	 HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache

     {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
	   "id_token": "JWT Token si además el servidor de autenticación es Open ID Connect"
     }

7- (Open ID Connect) El cliente hace un request al UserInfo Endpoint del servidor de recursos.
    Como se trata de una solcitud de recurso el cliente debe indicar el access token en la petición:
	
   GET /me?fields=id,name,email HTTP/1.1
   Host: https://graph.facebook.com 	
   Authorization: Bearer czZCaGRSa3F0MzpnWDFmQmF0M2JW
	
8- (Open ID Connect) El servidor de autorizacion responde en un documento JSON la información de usuario:

  HTTP/1.1 200 OK
  Content-Type: application/json

  {
   "sub": "248289761001",
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "preferred_username": "j.doe",
   "email": "janedoe@example.com",
   "picture": "http://example.com/janedoe/me.jpg"
  }
    
Cumplido este flujo de autorizacion (y autenticación si hablamos de Open ID Connect), la aplicación cliente ya posee todo lo necesario (el access token)
para obener recursos del resource server de forma segura. 

GRAFICO

Ahora cada vez que el usuario hace un request al cliente desde el User-Agent, este encuentra una session asociada (tipicamente por una cookie que envía el user-agent)
y de ser necesario puede hacer requests al servidor de recursos incluyendo siempre el access token. El Resource Server antes atender el pedido debe verificar la validez del access token; 
como lleva a cabo esta verificación esta afuera del alcance de la especificaciòn.

### Back Channel y Front Channel

## Implicit

Hay situaciones donde el cliente sólo funciona en el user-agent (es decir, no tiene posibilidad de usar el back channel). Hablamos bàsicamente de aplicaciones HTML y Javascript puras 
(o derivados como Typescript, Angular, React, etc.). Este flujo entonces omite el intercambio de còdigos; el cliente envìa su client-id y secret en el primer request al servidor de autorización
y este responde directamente con el access token (previo consentimiento del usuario final por supuesto).

## Resource Owner password credentials

Este flujo puede darse en situaciones donde el resource owner tiene plena confianza en el cliente y entonces le brinda sus credenciales.
Podrìa ser por ejemplo el sistema operativo de un dispositivo móbil. El cliente entonces interactùa directamente en el back channel con el servidor de autorización 
para obtener el access token. Su uso es desaconsejado. 

## Client Credentials
 
Como en el caso anterior la interacción ocurre sólo en el back channel. Tiene sentido en situaciones de comunicación maquina a maquina. 
En este flujo el cliente simple envìa sus credenciales al cliente y este responde el access token. No participa o no tiene sentido la figura del resource owner.  

## Access Token

Un Access Token es la credencial que le permite al cliente acceder a recursos protegidos. Consiste en un string que usualmente no es legible para el cliente.
Cualquier estructura, formato o contenido adicional que se le quiera dar al access token esta fuera del alcance de la especificacion de OAuth2.
Muchos sistemas que implementaron OAuth2 usaron el access token además para transmitir información del usuario y sus roles o capacidades.
Con el objetivo de ordenar las cosas y cumplir con esta necesidad es que aparece Open ID Connect. Dicha extensiòn propone un nuevo token 
estandarizado denominado ID Token. 

Mas información sobre el Access Token aquí.

## ID Token

El ID Token es representado como un JSON Web Token (JWT) y contiene de manera estandarizada información que identifica al usuario.
Tipicamente un ID Token tiene esta estructura:

(Heders: Info sobre la segurizaciòn del token)
.
  {
   "iss": "https://www.facebook.com",
   "sub": "24400320",
   "aud": "s6BhdRkqt3",
   "nonce": "n-0S6_WzA2Mj",
   "exp": 1311281970,
   "iat": 1311280970,
   "auth_time": 1311280969,
   "acr": "urn:mace:incommon:iap:silver"
  }
.
(Signature)

Mas precisiones sobre el ID Token aquí.

## OAuth2 para SSO

[TODO]

## Miscelanias:
- OAuth2 esta definido unicamente para HTTP.
[TODO]

## Referencias
[TODO]





