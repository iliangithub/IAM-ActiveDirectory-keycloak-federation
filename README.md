
# 1.0 Objetivo

Hasta el momento en prácticas anteriores que eran:

- https://github.com/iliangithub/IAM-Keycloak-oauth2-lab

- https://github.com/iliangithub/IAM-jwt-resource-server

El proveedor de identidad (IdP) Keycloak hacía dos trabajos a la vez:
- Guardaba las cuentas y sus contraseñas.
- Autenticaba y emitía tokens.

Y se supone que el papel funcional del proveedor de identidad es:
- Autenticar.
- Afirmar quién eres. 

**Lo que no hace es guardar contraseñas, ni cuentas, ni nada.** O para ser más exactos, autenticar y afirmar quién eres **es su función esencial. Almacenar es opcional** y se delega cuando ya existe un sistema que lo hace mejor y antes.

Por lo que buscamos poco a poco introducir un entorno más realista. Actualmente se usa un concepto llamado "Directorio",  cuyo papel es precisamente el que queríamos quitarle al Keycloak:
- Almacenar.
	- Guarda quién existe.
	- Y qué atributos tiene.

Por lo tanto, la herramienta que hace de Directorio es la solución que ofrece Microsoft al respecto y prácticamente la que todos sin excepciones usan. 

- Directorio Activo o Active Directory.

Sin embargo, hay otras opciones en la industria:

- OpenLDAP.
- 389 Directory Server.
- FreeIPA que es lo equivalente a AD adaptado a Linux, con Kerberos, PKI...
- Oracle Internet Directory.

Entre otros. Insisto, el más usado de lejos es el Active Directory, que llamaremos AD.

Sabiendo esto. No nos vamos a quedar meramente en la instalación de AD y ya. El objetivo real es: _"Que una cuenta que sólo existe en AD y que Keycloak no conoce de nada (pues no almacena los usuarios), acabe decidiendo (Keycloak) si una aplicación me deja entrar o me cierra la puerta."_

Para llegar al "destino final", hay que resolver varias cosas y pasos:
- Levantar el dominio y poblarlo. Es decir, tener un controlador funcionando, con sus unidades organizativas, sus grupos y sus usuarios.
- Saber interrogar al directorio igual que lo hace una integración de verdad, con sus filtros y su sintaxis propia.
- Montar el canal cifrado, porque ninguna integración seria consulta un directorio en claro.
- Enganchar ese directorio a Keycloak sin duplicar ni una sola contraseña.
- Y conseguir que lo que el directorio sabe de una persona, su departamento y sus grupos, viaje dentro del token hasta la API y acabe convertido en un permiso o en un 403.

Y una vez montado todo, viene la segunda mitad, que es la que de verdad corresponde al puesto y no consiste en configurar nada:

- Provocar los cuatro eventos del ciclo de vida de una cuenta (alta, cambio, escritura inversa y baja) y medir qué pasa realmente y cuánto tarda en pasar.
- Extraer inventarios de tres sistemas distintos y cruzarlos para sacar lo que no cuadra: cuentas que no deberían existir, accesos que sobrevivieron a una baja y bajas que se quedaron a medias.

## 1.1 Límites, alcance y mejoras.

La sincronización hacia Entra ID y el escenario híbrido, que lo dejo para la práctica siguiente. 

Y conviene decirlo claro, esto es un laboratorio, un entorno poco realista:
- Un solo controlador.
- Contraseñas fáciles.
- Y ninguna de las medidas que tendría un dominio real.

Un entorno realista sería:
**En el propio directorio**

- Mínimo dos controladores de dominio, y en cada sede uno. Si cae el único que hay, nadie se autentica contra nada en toda la organización.
- Un tercero que haga de controlador de solo lectura (RODC) en delegaciones pequeñas o sitios sin seguridad física.
- Copias de seguridad reales del estado del sistema, no solo instantáneas de la máquina virtual, porque restaurar un controlador clonado tiene sus propios problemas.
- La papelera de Active Directory habilitada, para poder recuperar un objeto borrado por error sin tocar las copias.

**En las cuentas y las contraseñas**

- Directiva de contraseñas seria, y directivas de grano fino para colectivos distintos, porque no se trata igual a un administrador que a un becario.
- Separación de cuentas administrativas: nadie navega por internet con la cuenta que administra el dominio. El modelo que se maneja habitualmente es el de niveles, con una cuenta por nivel de privilegio.
- Segundo factor en los accesos privilegiados.
- LAPS para las contraseñas de administrador local de cada equipo, de modo que no sean todas la misma.
- Cuentas de servicio administradas por grupo (gMSA), donde el propio dominio rota la contraseña, en lugar de una cuenta con contraseña fija que acaba escrita en un fichero de configuración.

**En el cifrado y los certificados**

- Jerarquía de dos niveles en la autoridad de certificación: una raíz que se mantiene apagada y desconectada, y una subordinada emisora que es la que trabaja a diario. Lo que montas tú es una sola, instalada en el propio controlador, que es justo lo que no se hace en producción.
- Procedimiento de renovación y revocación de certificados, con sus listas publicadas y accesibles.
- Rechazo de las conexiones LDAP sin cifrar, en lugar de admitirlas como alternativa.

**En el ciclo de vida**

- Las altas no las teclea nadie a mano. Salen del sistema de recursos humanos, que es la fuente autorizada, y bajan al directorio mediante aprovisionamiento automático.
- Las bajas tienen un procedimiento definido, con sus plazos y sus responsables, y alguien comprueba que se ejecutó.
- Campañas periódicas de recertificación, con evidencia de quién revisó qué y cuándo.

**En la vigilancia**

- Los eventos del directorio van a un SIEM, con alertas sobre lo que importa: cambios en grupos privilegiados, creación de cuentas, bloqueos masivos.
- Y alguien los mira, que es la parte que más se olvida.

**En el propio Keycloak**

- Varias instancias en clúster, no un contenedor suelto.
- Base de datos externa y persistente, en lugar de la que vive dentro del contenedor.
- TLS de verdad delante, con certificado válido.
- Tiempos de vida de los tokens ajustados a la criticidad de cada aplicación.

# 2.0 Conceptos que hay que tener claros antes de empezar

Al ser el primer "approach" al AD desde una perspectiva y contexto de IAM, vamos a definir previamente los conceptos más importantes.

## 2.1 Un directorio no es una plataforma de gestión de identidades

Tal y como dijimos al principio.

Un directorio en el contexto de IAM, es la entidad que guarda:
- Quién existe (usuarios) 
- Y qué atributos tiene. 
	- Los atributos NO son permisos.
	- Los atributos son propiedades descriptivas de un objeto, en este caso el usuario:
		- Nombre.
		- Dónde encaja en la organización.
		- Cómo se le localiza.
		- Cómo se identifica, su ID.
		- De dónde es miembro, que grupo.
		- En qué estado está la cuenta.

Responde a consultas y valida credenciales. 
- No decide quién debe existir
- Ni quién debe tener acceso a qué
- Ni revisa si esos accesos siguen justificados.

**Sólo guarda datos, nada más.**

El proveedor de identidad que dijimos al principio (IdP) que en esta práctica es Keycloak, es el encargado de autenticar y emitir tokens.

Por último, una plataforma de gestión de identidades... **Que no tiene nada que ver con el proveedor de identidad (IdP) en el sentido de que resuelven problemas distintos**. Su función es:
- <b>Solicitar y aprobar accesos.</b>
	- Por lo tanto, se solicita desde la plataforma de gestión de identidades, y es ella quien la aprueba o no.
	- O mejor dicho, un empleado la solicita, la solicitud llega a la plataforma de gestión de identidades y es otro trabajador que lo aprueba o lo deniega, y queda constancia.
		- Y para ser más exactos todavía, recordemos que si solicitas permiso para acceder a un recurso u objeto, eso pasa por 3 personas (_aunque no siempre es un circuito fijo, recordemos el breakglass_).
		- La persona que te autoriza para solicitar la solicitud. Y que valida/confirma que realmente necesitas ese recurso/objeto que estás tratando de solicitar.
		- La persona que realmente te lo aprueba, el empleado/trabajador que es literalmente el dueño de ese recurso y objeto. 
		- Finalmente nosotros, el equipo de IAM, que velamos por que todo ese proceso funcione, que lo aprobado se ejecute, que no se quede nada a medias y que lo que falla se resuelva.
- <b>Aprovisionamiento y desaprovisionamiento.</b>
	- Aprovisionar es crear la cuenta de una persona en los sistemas que necesita y darle los accesos que le corresponden. Desaprovisionar es lo contrario: retirarlos.
	- O mejor dicho, la plataforma no decide por su cuenta cuándo hacerlo. Lo hace a partir de lo que diga recursos humanos, que es la fuente autorizada, es decir, quien sabe de verdad quién entra en la empresa, quién cambia de puesto y quién se va.
	    - Y para ser más exactos todavía, son tres eventos los que mueven todo esto, y en el sector se conocen como el ciclo de altas, cambios y bajas (en inglés, joiner, mover y leaver).
	    - El alta: alguien entra en la empresa, y la plataforma le crea las cuentas y los accesos de su puesto.
	    - El cambio: alguien cambia de departamento, y la plataforma le da lo nuevo y, lo que más se olvida, le quita lo anterior.
	    - La baja: alguien se va, y la plataforma le retira todo. Lo habitual es deshabilitar primero y borrar después, pasado un tiempo, para no perder la trazabilidad.
	- Todo esto se hace desde la plataforma, que escribe en cada sistema destino a través de su conector. El directorio es uno de esos destinos, el más importante, que en este caso será el Active Directory, pero no el único: también lo son el ERP, el correo o cualquier aplicación que tenga cuentas propias.
	- Y aquí está el problema que más interesa al puesto: si la baja falla en uno de esos destinos, la plataforma puede darla por hecha mientras la cuenta sigue viva en ese sistema. Es lo que se conoce como cuenta huérfana, y es justo lo que vamos a provocar y a buscar más adelante en la práctica.
- <b>Modelo de roles</b>.
	- Consiste en agrupar permisos por función de negocio, de modo que un puesto concreto traiga consigo un conjunto de accesos predefinido.
	- O mejor dicho, en lugar de que cada persona vaya pidiendo sus accesos uno a uno, se define una vez qué necesita un puesto, y quien ocupa ese puesto lo recibe todo de golpe.
	    - Y aquí conviven dos niveles que conviene no mezclar.
	    - <b>El rol de negocio:</b> lo que entiende un responsable. Por ejemplo, "analista de riesgos" o "gestor de oficina".
	    - <b>El permiso técnico:</b> lo que entiende cada sistema. Un grupo de AD, un perfil en el ERP, un rol en una aplicación concreta. En el sector se le llama entitlement.
	    - El rol de negocio no es más que un paquete de permisos técnicos repartidos por varios sistemas. El responsable aprueba "analista de riesgos" y la plataforma se encarga de traducirlo a todo lo que eso implica por debajo.
	- Hay además un tipo de acceso que no se pide nunca, porque viene con el simple hecho de entrar en la empresa: el correo, la intranet, el acceso al dominio. Son los accesos de base, y se asignan automáticamente en el alta.
	- Y el problema clásico de este modelo es la explosión de roles: si se crea un rol para cada pequeña variante de cada puesto, se acaba con más roles que personas y el modelo deja de servir para nada.
	- En el caso de AD, los roles suelen materializarse como grupos. Es lo que voy a ver en la práctica, cuando los grupos que creo en el directorio acaben viajando dentro del token y la API decida en función de ellos.
- <b>Segregación de funciones.</b>
	- Consiste en impedir que una misma persona reúna permisos que, juntos, le permitan cometer un fraude o un error grave sin que nadie más intervenga.
	- O mejor dicho, cada permiso por separado es legítimo, el problema está en la combinación. Dar de alta un proveedor está bien, ordenar un pago está bien, pero si la misma persona puede hacer las dos cosas, puede inventarse un proveedor y pagarse a sí misma.
	    - Y para ser más exactos todavía, esto se gestiona con una lista de combinaciones incompatibles, que define la propia organización según sus procesos. La plataforma no sabe por sí sola qué es peligroso, se lo tiene que decir alguien de negocio o de control interno.
	    - El control preventivo: actúa en el momento de la solicitud. Si alguien pide un acceso que choca con otro que ya tiene, la plataforma lo detecta antes de concederlo y lo bloquea o lo marca para revisión.
	    - El control detectivo: actúa después, revisando periódicamente quién tiene ya la combinación prohibida, porque muchas veces se cuela por cambios de puesto o por accesos concedidos fuera de la plataforma.
	    - Y la excepción: a veces el negocio necesita que alguien tenga la combinación, por ejemplo en una oficina pequeña con poca plantilla. Se permite, pero alguien asume el riesgo por escrito y se añade un control compensatorio, como una revisión posterior de esas operaciones.
	- Aquí se ve muy claro por qué hace falta una plataforma por encima de todo: cada sistema solo ve su parte. El ERP sabe quién paga, la aplicación de proveedores sabe quién da de alta, pero ninguno de los dos ve que es la misma persona. Solo quien tiene la foto completa de todos los accesos puede detectarlo.
	- No es un invento informático, es un principio de control interno mucho más antiguo que viene de la contabilidad. Lo que hace la plataforma es aplicarlo de forma sistemática a los permisos.
- <b>Recertificación. </b>
	- Es la revisión periódica de los permisos que ya están concedidos. Ya la expliqué con detalle en la primera práctica, así que aquí me quedo con lo que importa para entender la plataforma.
	- O mejor dicho, cada cierto tiempo la plataforma lanza una campaña, reparte a cada responsable la lista de accesos de su gente, y este tiene que confirmar uno a uno si se mantienen o se retiran.
	    - Y para ser más exactos todavía, la campaña se puede plantear desde distintos ángulos, según quién tenga el criterio para revisar.
	    - Por persona: el responsable directo revisa todo lo que tiene cada miembro de su equipo.
	    - Por aplicación: el dueño de un sistema revisa quién tiene acceso a lo suyo.
	    - Por rol: se revisa si el contenido de un rol sigue teniendo sentido, no a quién está asignado.
	- Lo que se revoca en la campaña no se queda en una casilla marcada: la plataforma lo ejecuta en los sistemas destino, por la misma vía del desaprovisionamiento.
	- Y el problema clásico es el revisor que aprueba todo sin mirar, porque tiene doscientas líneas delante y ninguna gana de leerlas. Por eso las campañas se diseñan para resaltar lo raro, como accesos que nadie usa o permisos que no encajan con el puesto, en lugar de presentarlo todo igual.
	- Conviene recordar que la recertificación es una red de seguridad, no un sustituto del proceso de bajas. Está para cazar lo que ese proceso dejó pasar.
- <b>Evidencia.</b>
	- Es el registro de todo lo anterior: quién pidió qué, quién lo aprobó, cuándo se ejecutó, quién lo revisó después y con qué resultado.
	- O mejor dicho, no basta con hacer las cosas bien, hay que poder demostrar que se hicieron. En banca, el auditor no pregunta si se revisan los accesos, pide que se le enseñe la campaña del último semestre con sus decisiones y sus fechas.
	    - Y para ser más exactos todavía, una evidencia útil tiene que cumplir varias cosas a la vez.
	    - Ser completa: tiene que enlazar la decisión con su ejecución. Una aprobación sin constancia de que el acceso se concedió de verdad, o una baja sin constancia de que se retiró en cada sistema, no demuestra nada.
	    - Ser fiable: nadie debería poder modificarla después, ni siquiera quien administra la plataforma.
	    - Conservarse: durante el tiempo que exija la normativa o la política interna, que suele ser de años.
	- No hay que confundirla con un registro de actividad cualquiera. Un log dice qué pasó en un sistema, la evidencia demuestra que un control funcionó.
	- Y en esta práctica también genero evidencia, aunque a pequeña escala: el informe de discrepancias que saco al cruzar los inventarios es exactamente el tipo de documento que se le entregaría a un auditor para demostrar que se buscaron cuentas huérfanas y qué se encontró.

Keycloak, en este laboratorio, ocupa un papel intermedio. Es un proveedor de identidad, o sea quien autentica y emite tokens, y en esta práctica lo usamos además como consumidor del directorio. No es una plataforma de gobierno, y conviene no venderlo como tal en el README.

## 2.2 Federar no es copiar

- Federar.
    - Federar es establecer una relación de confianza entre dos sistemas de identidad, de modo que uno acepte lo que el otro sabe o afirma, sin necesidad de duplicarlo.
    - O mejor dicho, en lugar de dar de alta a cada persona en Keycloak, Keycloak se fía del directorio: le pregunta quién existe, qué atributos tiene y si la contraseña es correcta.
        - Y para ser más exactos todavía, en Keycloak la palabra "federar" se usa para dos cosas distintas, y conviene no confundirlas.
        - Federación de usuarios (User federation): Keycloak se conecta a un almacén externo, como un directorio LDAP o AD, y lo usa como fuente de usuarios. Keycloak sigue mostrando su pantalla de acceso y sigue emitiendo el token. Es lo que hago en esta práctica.
        - Proveedores de identidad externos (Identity providers): Keycloak delega la autenticación entera en otro IdP, como Google o un tenant de Entra ID. Ahí la pantalla de acceso es la del otro, y Keycloak recibe una afirmación ya hecha. No es lo que hago aquí.
- Cómo valida la contraseña sin guardarla.
    - Cuando alguien inicia sesión, Keycloak recibe la contraseña, abre una conexión al directorio e intenta autenticarse con ella. En el lenguaje de LDAP eso se llama un bind. Si el bind funciona, la contraseña era buena.
    - O mejor dicho, son dos binds seguidos. Primero Keycloak se autentica con su propia cuenta de servicio, la de solo lectura, para buscar al usuario y averiguar su ruta completa en el directorio. Después lanza un segundo bind con la ruta de ese usuario y la contraseña que acaba de teclear.
    - La contraseña pasa por Keycloak, pero no se queda en él. Y el hash con el que el directorio la compara nunca sale del directorio. Por eso el canal tiene que ir cifrado: la contraseña viaja por la red en cada inicio de sesión.
- Lo que sí se copia.
    - Lo que sí puede copiar Keycloak son los atributos: nombre, correo, departamento, grupos. Depende de cómo se configure el modo de importación.
        - Con importación activada, Keycloak guarda en su base de datos un registro de cada usuario con sus atributos, y lo va actualizando cada vez que sincroniza.
        - Con importación desactivada, no guarda nada y consulta al directorio cada vez que lo necesita.
    - Y ahí empieza el problema que voy a provocar a propósito. Un usuario importado es una copia, y toda copia puede divergir del original o sobrevivirle. Si cambio el departamento en AD, Keycloak no se entera hasta la siguiente sincronización. Y si borro la cuenta en AD, el registro puede seguir en Keycloak, igual que los tokens que ya se emitieron con él.

## 2.3 El identificador tiene que ser inmutable

Esto es lógico.<br>Una persona cambia de apellido, de correo, de departamento y hasta de nombre de usuario, y tiene que seguir siendo la misma persona. <br>Si la correspondencia entre el directorio y el proveedor de identidad se ancla a un atributo que cambia, el día que alguien se case el sistema fabrica un usuario nuevo y deja el anterior sin dueño.

En Active Directory el identificador inmutable es `objectGUID`, que se asigna al crear el objeto y no cambia nunca, ni al renombrar ni al mover de unidad organizativa.

## 2.4 Active Directory

Entonces, Active Directory es el servicio de directorio de Microsoft, presente en Windows Server.<br>Recordemos que se trata de una base de datos, un fichero llamado `ntds.dit`, que vive en cada controlador de dominio.

Pues como ya dijimos tiene que haber mínimo 2 controladores de dominio, por redundancia.

Lo que se copia/replica son los cambios, objeto a objeto y atributo a atributo. Las copias de seguridad son otra cosa aparte.

Cada controlador lleva un contador interno, el USN (número de secuencia de actualización), que sube con cada cambio que se hace en él. Los controladores se preguntan entre sí "_¿qué tienes nuevo desde el último número que te vi?_", y solo se intercambian lo que ha cambiado desde entonces.

Y la granularidad es muy fina: si a un usuario le cambio el departamento, no viaja el usuario entero, viaja solo el atributo `department`. Si hubiera que ponerle una etiqueta, sería incremental a nivel de atributo.
¿Qué quiere decir esto?
- Viaja el objeto identificado por su `objectGUID`.
- Qué atributo, de nuevo identificado por un ID, un `OID`. Un número jerárquico único a nivel mundial, del estilo `1.2.840.113556.1.2.141`. Así todos los controladores saben que hablan del mismo atributo aunque cambie su nombre visible.
- **Qué versión de ese atributo.** Y aquí está lo interesante: cada atributo de cada objeto lleva sus propios **metadatos de replicación**:
	- Un número de versión, que sube cada vez que ese atributo cambia.
	- El controlador donde se originó el cambio, identificado por su `invocationID`, que es un GUID de su base de datos.
	- El USN que tenía ese controlador cuando hizo el cambio.
	- La fecha y hora del cambio.

Por lo tanto, lo que viaja en cada replicación no es el usuario entero, sino solo el atributo que ha cambiado. Y ese atributo no se identifica solo, sino por la combinación de tres cosas: el objeto al que pertenece (su `objectGUID`), qué atributo es (su identificador en el esquema) y en qué versión está, con qué controlador lo cambió y cuándo.

La única copia completa ocurre una vez: cuando se promueve un controlador nuevo, que tiene que recibir el directorio entero para arrancar.

<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>

Active Directory no trabaja solo, se apoya en tres protocolos:
- LDAP, para consultar y modificar el directorio. Es el que va a usar Keycloak.
- Kerberos, para autenticar dentro del dominio. Es lo que pasa cuando un empleado inicia sesión en su equipo.
- DNS, para que los equipos encuentren a los controladores. Sin DNS funcionando, el dominio no funciona, y por eso el controlador hace también de servidor DNS.

### 2.4.1 Jerarquía AD

Lo primero es identificar tres planos/espacios:
- Lógico.
- Físico.
- Datos.
La jerarquía empieza de arriba hacia abajo.

#### 1. Plano lógico: cómo se organiza la administración

- **Bosque.** Frontera de seguridad. Un solo esquema y una sola configuración para todo lo que contiene.
- **Árbol.** Dominios con espacio de nombres DNS contiguo.
- **Dominio.** Unidad de replicación de sus objetos y ámbito de la directiva de cuentas por defecto.
- **Contenedores y OU.**
	- Las dos sirven para agrupar objetos dentro del dominio, pero no son lo mismo.
	- Los contenedores (`CN=`) vienen creados de fábrica, como `Users`, `Computers` o `Builtin`. No admiten directivas de grupo enlazadas y no se crean desde la consola habitual, porque no están pensados para que uno organice su dominio con ellos. Son simplemente el sitio por defecto donde caen los objetos si nadie dice otra cosa.
	- Las unidades organizativas (`OU=`) son las que crea el administrador para reflejar la estructura de su organización. Admiten directivas de grupo, permiten delegar su administración en otras personas y se pueden anidar unas dentro de otras.
	- Con una excepción que despista: `Domain Controllers` parece uno más de los de fábrica, pero es una OU. Se crea sola al promover el primer controlador, y es OU precisamente para poder enlazarle la directiva que se aplica a los controladores (`Default Domain Controllers Policy`). Si fuera un contenedor, no habría forma de hacerlo.
- **Objetos hoja.** Usuarios, equipos, grupos, contactos, cuentas de servicio administradas, impresoras publicadas.

Los contenedores por defecto en la raíz del dominio, para que no te sorprendan en la primera consulta: 
- `Builtin` (grupos locales integrados, como Administradores).
- `Users`.
- `Computers`.
- `ForeignSecurityPrincipals` (representan a miembros de otros dominios de confianza).
- `Managed Service Accounts`.
- `System` (configuración interna, incluidas las directivas de grupo y las confianzas).
- `LostAndFound` (objetos huérfanos tras conflictos de replicación).
- y alguno más según la versión.
#### 2. Plano físico: dónde están las cosas

Esto es totalmente independiente del plano lógico. Un dominio puede abarcar varias sedes, y una sede puede tener controladores de varios dominios.

- **Sitio.** Un conjunto de redes bien conectadas entre sí, normalmente una sede. Determina contra qué controlador se autentica cada equipo y cómo se replica.
- **Subred.** Cada rango IP se asocia a un sitio. Así un equipo sabe en qué sitio está según su dirección.
- **Enlace de sitio.** Define la conexión entre sitios, con su coste y su calendario. Es donde viven los 180 minutos de replicación.
- **Controlador de dominio.** El servidor que aloja una copia del directorio.

En cuanto a los 180 minutos de replicación:

Imagina una empresa con dos oficinas: una en Madrid y otra en Málaga. En cada oficina hay un controlador de dominio, y los dos tienen que tener la misma información.

**Dentro de la misma oficina**, los equipos están conectados por una red rápida. Si en Madrid hay dos controladores y cambio algo en uno, el otro se entera en unos segundos. Sale barato, así que se hace al momento.

**Entre Madrid y Málaga** la conexión es más lenta y más cara. No tiene sentido mandar cada cambio en cuanto ocurre. Lo que hace AD es **acumular los cambios y mandarlos en tandas**: por defecto, una tanda cada 180 minutos, o sea, cada tres horas.

Esto significa que si a las 9:00 creo un usuario en Madrid, puede que el controlador de Málaga no lo tenga hasta las 12:00. Si ese usuario intenta iniciar sesión en Málaga a las 10:00, no existe para ese controlador.

Para que AD sepa qué oficinas hay y cómo están conectadas, se definen tres cosas:

- **Sitio**: cada oficina. Madrid es un sitio, Málaga es otro.
- **Subred**: los rangos de IP de cada oficina. Así un equipo sabe en qué sitio está según su IP.
- **Enlace de sitio**: la conexión entre Madrid y Málaga. Y en ese enlace es donde se configura cada cuánto se mandan las tandas (los 180 minutos), en qué horas se permite (por ejemplo, solo de noche) y un número llamado coste, que sirve para elegir el camino preferido si hubiera varios.

#### 3. Plano de datos: las particiones

El `ntds.dit` no es un árbol único, sino varias particiones (contextos de nombres) con ámbitos de replicación distintos:
- **Dominio.** Los objetos de ese dominio. Se replica solo entre sus controladores.
- **Configuración.** Sitios, subredes, servicios. Se replica a todo el bosque.
- **Esquema.** Las definiciones de clases y atributos. Se replica a todo el bosque.
- **Aplicación.** Particiones con replicación a medida. Las más habituales son las de DNS: `DomainDnsZones` y `ForestDnsZones`.
## 2.5 Active Directory frente a Entra ID

Ahora ¿Si AD y Entra ID o Azure AD son de Microsoft, porqué existen? ¿Se hacen competencia?

No. Resuelven problemas distintos.

### Active Directory: el directorio de la oficina

Nació en el año 2000, pensado para una empresa donde todo estaba dentro: los equipos en la red local, los servidores en el sótano, los usuarios sentados en su puesto.

- Se instala en servidores propios (en local, _on-premises_).
- Habla los protocolos de red interna: LDAP, Kerberos, NTLM.
- Gestiona equipos: los une al dominio, les aplica directivas de grupo, controla quién inicia sesión en qué ordenador.
- Tiene la jerarquía que acabas de documentar: bosques, dominios, unidades organizativas.

### Entra ID: el proveedor de identidad de la nube

Antes se llamaba **Azure Active Directory**, y Microsoft lo renombró a Entra ID en 2023, en parte para acabar con la confusión. Nació cuando las aplicaciones dejaron de estar en el sótano y pasaron a ser servicios en internet: Microsoft 365, Salesforce, cualquier SaaS.

- No se instala, es un servicio en la nube. Cada organización tiene su **tenant**.
- Habla los protocolos de la web: OAuth 2.0, OpenID Connect, SAML. Los mismos que viste con Keycloak.
- No tiene bosques, ni dominios en el sentido de AD, ni unidades organizativas, ni directivas de grupo. Su estructura es plana.
- No habla LDAP ni Kerberos de forma nativa. Una aplicación antigua que solo sepa preguntar por LDAP no puede consultarlo directamente.

Es decir, a pesar del nombre antiguo, **Entra ID nunca fue "Active Directory en la nube"**. Se parece mucho más a Keycloak que a AD: es sobre todo un proveedor de identidad, que además guarda usuarios.

### Cómo encajan

La mayoría de empresas grandes, y desde luego un banco, no pueden tirar su AD de veinte años. Tienen miles de equipos unidos al dominio y aplicaciones internas que dependen de él. Pero a la vez usan Microsoft 365 y aplicaciones en la nube que necesitan Entra ID.

La solución es el **escenario híbrido**: 
- AD sigue siendo la fuente donde se dan de alta las personas, y una herramienta de sincronización (Entra Connect Sync o Cloud Sync) copia esas identidades hacia Entra ID. 
- Así un empleado tiene una sola identidad que sirve tanto para iniciar sesión en su equipo de la oficina como para entrar en Teams.

Eso es justo lo que dejamos para la próxima práctica.

- **AD** es el directorio local, donde nace la identidad.
- **Entra ID** es directorio y proveedor de identidad en la nube, alimentado desde AD.
- **La plataforma de gestión de identidades** está por encima de los dos: decide quién debe existir y con qué accesos, y escribe en ambos.

Un matiz: Microsoft también vende funciones de gobierno dentro de Entra (Entra ID Governance), así que un mismo fabricante puede cubrir los tres papeles. Por eso, como decíamos, la pregunta útil no es qué producto es cada uno, sino qué papel cumple en cada momento.


| Aspecto | Active Directory Domain Services | Entra ID |
|---|---|---|
| Protocolo de consulta | LDAP | Microsoft Graph, API REST |
| Protocolo de autenticación | Kerberos y NTLM | OAuth 2.0, OpenID Connect, SAML |
| Estructura | Bosque, dominios, unidades organizativas | Plano, sin unidades organizativas |
| Unidad de política | Directivas de grupo sobre equipos del dominio | Acceso condicional sobre sesiones |
| Identificador inmutable | objectGUID | id del objeto |
| Ámbito natural | Red interna, ficheros, impresoras, aplicaciones heredadas | Aplicaciones de internet y servicios en la nube |

No es el mismo producto con otro nombre. Son dos directorios con modelos distintos.

# 3.0 Requisitos.



# 4.0 Procedimiento.

## 4.1 Instalar el virtualizador.

Primero voy a empezar habilitando el Hyper-V de Windows 11 Pro.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

![](img/Pasted%20image%2020261004122422.png)

Nos pedirá reinciar, reiniciamos.

![](img/Pasted%20image%2020261004122529.png)

Tras el reinicio, veremos que ya lo tenemos:

![516](img/Pasted%20image%2020261004122924.png)

## 3.2 Descargar ISO.

https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025

Mejor elegir la ISO, 64 bits, en inglés. Cuando algo falle, el mensaje de error en inglés tiene diez veces más resultados de búsqueda que su traducción. Los cmdlets de PowerShell son idénticos en los dos casos, así que no pierdes nada.

## 3.3 Creación conmutador de red.

Para nuestro caso, vamos a hacerlo vía PowerShell y no modo gráfico (`virtmgmt.msc` o simplemente Administrador de Hyper-V).

¿La razón?
- **El asistente gráfico no cubre todo.** El asistente de máquina virtual nueva te pregunta nombre, generación, memoria de inicio, red y disco, y ahí se acaba. El número de procesadores virtuales, la desactivación de los puntos de control automáticos y el orden de arranque del firmware no están en el asistente: hay que entrar después en las propiedades de la máquina y tocarlos uno a uno. O sea que en la interfaz tampoco es "siguiente, siguiente", son un asistente más tres visitas a la ventana de configuración.

- **Es el material que te pide el enunciado.** La fase 2 lo dice explícitamente para la estructura del directorio, y aplica igual aquí: un script es reproducible y una captura no. Si alguien lee tu repositorio y quiere montar el mismo laboratorio, con los comandos lo tiene en dos minutos; con capturas tiene que ir replicando clics y adivinando qué pusiste en los campos que no se ven.

- **Es como se administra de verdad.** Un Hyper-V de producción corre a menudo sobre una instalación sin escritorio, y se gestiona en remoto. La interfaz gráfica es la excepción, no la norma.

### 3.3.1 Primero mira qué adaptador vas a puentear.

Este comando es una tubería de tres pasos cuyo único objetivo es darte el nombre exacto que luego pide `-NetAdapterName`.

- **`Get-NetAdapter`** devuelve un objeto por cada tarjeta de red del equipo, físicas y virtuales, con sus propiedades: nombre, descripción, índice de interfaz, estado, dirección MAC, velocidad de enlace.

- **`Where-Object Status -eq 'Up'`** filtra y deja solo las que están operativas. Sin esto te salen también las desconectadas, las deshabilitadas y las que Windows marca como no presentes, que en un portátil suelen ser varias (Bluetooth, adaptadores virtuales de VPN, la Ethernet si no tienes cable puesto).

- Ese `Status -eq 'Up'` sin llaves es la sintaxis simplificada que existe desde PowerShell 3. La forma larga y equivalente es `Where-Object { $_.Status -eq 'Up' }`, donde `$_` representa el objeto que viene por la tubería en cada vuelta. Te conviene conocer las dos porque la larga es la que necesitas en cuanto la condición se complica.

- **`Select-Object Name, InterfaceDescription, LinkSpeed`** se queda solo con tres propiedades para que la salida sea legible. No filtra filas, recorta columnas.

```powershell
Get-NetAdapter | Where-Object Status -eq 'Up' | Select-Object Name, InterfaceDescription, LinkSpeed
```

En mi caso aparecen:

<pre>
Name                       InterfaceDescription                  LinkSpeed
----                       --------------------                  ---------
Ethernet 2                 VirtualBox Host-Only Ethernet Adapter 1 Gbps
vEthernet (Default Switch) Hyper-V Virtual Ethernet Adapter      10 Gbps
Ethernet                   Realtek Gaming GbE Family Controller  1 Gbps

</pre>

Tu tarjeta física es **`Ethernet`**, la Realtek Gaming GbE. Es la que lleva tu conexión real y la que hay que puentear.

Las otras dos descártalas:

- **`Ethernet 2`**, VirtualBox Host-Only: adaptador virtual que instaló VirtualBox para su red interna. No sale a ningún sitio.
- **`vEthernet (Default Switch)`**: el conmutador NAT que Hyper-V crea solo al instalarse. Es virtual, y además es justo el tipo de red que no nos sirve porque WSL2 no la alcanza. Ese "10 Gbps" es ficticio, es la velocidad nominal de un enlace por software.

Por cierto, no aparece ninguna Wi-Fi activa, así que estás por cable. Mejor, es el escenario más estable para el puente.

#### Creamos el conmutador con el nombre que te salga ahí:

```powershell
New-VMSwitch -Name "LabExterno" -NetAdapterName "Ethernet" -AllowManagementOS $true
```

Si solo tienes Wi-Fi, sustituye el nombre por el del adaptador inalámbrico. Funciona, pero avisa: al ejecutar esto tu conexión se corta un par de segundos mientras Windows reconfigura el adaptador. Es normal.

####  **¿Qué hace este último comando y qué es un conmutador?**

Un conmutador "switch" el que conocemos de toda la vida y funciona en capa 2, o capa 2 y 3, es una caja que literalmente reúne puertos físicos donde conectar tus dispositivos que luego se conectará a un enrutador "router".

El conmutador o switch virtual de Hyper-V y que acabamos de crear, es exactamente lo mismo, pero hecho software dentro del Windows.

Tiene puertos y reparte tráfico.

Por lo tanto, las 3 piezas son:
- La tarjeta de red de la propia máquina virtual. Como es lógico cada máquina tiene su propia tarjeta virtual.
- El conmutador virtual "Switch virtual" que repito, lo acabamos de crear (`LabExterno`) y es donde se "_enchufan_" las tarjetas de las máquinas virtuales.
- Finalmente, la tarjeta de red física de mi portátil, que conecta el switch virtual, con la red de mi casa.

<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>

Entonces. Antes de que creemos el conmutador "_switch_". El Windows usaba la tarjeta física directamente, no había ningún switch de por medio.

<pre>
Windows <----> tarjeta física de red <----> router de casa
</pre>

Cuando creas ese switch virtual. Hyper-V le asigna esa tarjeta física que antes usaba Windows al switch virtual que acabas de crear.

Como es lógico, el Windows se quedaría sin red porque estaba directamente conectado a la tarjeta física del ordenador, pero ahora ya no, la tarjeta física está conectada a un switch virtual. 

<pre>

Windows ------> ( a nada )

( a nada ) <---- switch virtual <----> tarjeta física de red <----> router de casa

</pre>

<p>Para que Windows no se quede sin internet lo enchufa al switch que acabamos de crear.</p>
Pero, para tu enchufar lo que sea un switch necesitas una tarjeta de red, cosa que Windows acaba de perder, ya que insisto, estaba directamente conectado a la tarjeta física.

Por lo tanto, se crea a Windows una tarjeta de red virtual para poder conectarse al switch virtual.

<pre>

Windows ------> Tarjeta de red virtual ---|
								 |
								 |
								 V
							switch virtual <----> tarjeta física de red<----> router de casa

</pre>

Finalmente, como vamos a crear una máquina virtual en Hyper-V, también requiere de tarjeta de red virtual.


<pre>

Windows ------> Tarjeta de red virtual ---|
								 |
								 |
								 V
							switch virtual <----> tarjeta física de red<----> router de casa

								   Ʌ
								   |
								   |
Máquina Virtual -> Tarjeta de red virtual --|
</pre>

Queriendo decir eso, que el switch se crea sólo para que el Windows no se quede sin internet ( _Al introducir en la ecuación la MV_ ) y por otro lado para que la futura máquina virtual tenga Internet.
- Y no se crea él solito, lo creamos nosotros.

La creación de tarjetas de red es también obligatorio y un requisito para conectarlo al switch, sea lo que sea.
- No nacen a la vez, la de Windows se crea "sola" al ejecuta el comando `VMSwitch` con el parámetro `-AllowManagementOS $true`. Si no la tendríamos que crear nosotros.
- Y la de la máquina virtual, pues cuando toque crear la máquina virtual.


<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>


Hyper-V tiene tres tipos de switches, y la elección determina quién puede hablar con quién:

- **Privado:** solo las máquinas virtuales entre ellas. Ni el anfitrión las ve.
- **Interno:** las máquinas virtuales y el anfitrión. Sin salida a la red física.
- **Externo:** se asocia a una tarjeta de red física y las máquinas quedan en tu red local como un equipo más.

<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>

Elegimos externo por una razón concreta de tu topología. Keycloak va a correr en Docker dentro de WSL2, y WSL2 vive detrás de su propio NAT, no es un proceso más del anfitrión a efectos de red. Desde ahí sale hacia tu red local a través del anfitrión, así que alcanza cualquier IP de tu LAN. Si el controlador está en un conmutador interno o en el conmutador predeterminado (que hace NAT), queda en una red que WSL2 no enruta, y tendrías que montar reenvío de puertos para que Keycloak llegue al 636. Con externo, el controlador tiene una IP de tu LAN y se acabó el problema.

Lo que ocurre físicamente al ejecutarlo, y por eso se corta la conexión: Hyper-V intercala un conmutador por software entre tu tarjeta física y el sistema operativo. Tu Windows deja de usar directamente la tarjeta y pasa a usar un adaptador virtual nuevo que verás como `vEthernet (LabExterno)`. La tarjeta física queda convertida en el enlace ascendente de ese conmutador.

`-AllowManagementOS $true` es justo eso: que el anfitrión conserve conectividad a través del conmutador. Si lo pones en `$false`, la tarjeta queda dedicada en exclusiva a las máquinas virtuales y tu Windows se queda sin esa interfaz. En un servidor con varias tarjetas tiene sentido; en un portátil con una, te quedas sin internet.
## 3.4 Creación máquina virtual.

a

## 3.5 Instalación.

a

# Práctica 3. Directorio corporativo real y federación en Keycloak

Active Directory sobre Windows Server, Keycloak como proveedor de identidad.
Enunciado de trabajo. Revisión del 5 de octubre de 2026.

> [!NOTE]
> Este fichero es el guion de la práctica, no el README final del repositorio. El README lo escribes tú, con tus capturas, tus errores reales y tu forma de contar las cosas. Aquí están los pasos, los comandos y los conceptos que hay que entender en cada punto.

---

## 0.0 Índice

- [1.0 Qué demuestra esta práctica](#10-qué-demuestra-esta-práctica)
- [2.0 Conceptos que hay que tener claros antes de empezar](#20-conceptos-que-hay-que-tener-claros-antes-de-empezar)
- [3.0 Arquitectura del laboratorio](#30-arquitectura-del-laboratorio)
- [4.0 Fase 0. Decidir dónde corre el controlador de dominio](#40-fase-0-decidir-dónde-corre-el-controlador-de-dominio)
- [5.0 Fase 1. Instalar y promover el controlador](#50-fase-1-instalar-y-promover-el-controlador)
- [6.0 Fase 2. Poblar el directorio con una estructura creíble](#60-fase-2-poblar-el-directorio-con-una-estructura-creíble)
- [7.0 Fase 3. Interrogar el directorio antes de tocar Keycloak](#70-fase-3-interrogar-el-directorio-antes-de-tocar-keycloak)
- [8.0 Fase 4. Cifrar la conexión antes de federar](#80-fase-4-cifrar-la-conexión-antes-de-federar)
- [9.0 Fase 5. Federar el directorio en Keycloak](#90-fase-5-federar-el-directorio-en-keycloak)
- [10.0 Fase 6. Mapear atributos y grupos](#100-fase-6-mapear-atributos-y-grupos)
- [11.0 Fase 7. Comprobar la autenticación federada](#110-fase-7-comprobar-la-autenticación-federada)
- [12.0 Fase 8. Ciclo de vida, lo que de verdad se opera](#120-fase-8-ciclo-de-vida-lo-que-de-verdad-se-opera)
- [13.0 Fase 9. Discovery, reconciliación y cuentas huérfanas](#130-fase-9-discovery-reconciliación-y-cuentas-huérfanas)
- [14.0 Fase 10. Extensiones opcionales](#140-fase-10-extensiones-opcionales)
- [15.0 Autoevaluación](#150-autoevaluación)
- [16.0 Material a documentar](#160-material-a-documentar)
- [17.0 Limitaciones del laboratorio](#170-limitaciones-del-laboratorio)
- [18.0 Fuentes consultadas](#180-fuentes-consultadas)

---

## 1.0 Qué demuestra esta práctica

En la entrevista quedó sin contestar qué relación guardan Active Directory, Entra ID y una plataforma de gestión de identidades. Esta práctica responde la mitad local de esa pregunta y, de paso, cubre la parte de descubrimiento del puesto, que no consiste en configurar nada sino en comparar inventarios de sistemas distintos y encontrar lo que no cuadra.

Se monta Active Directory de verdad, no un sustituto. Eso importa por dos razones. La primera es que el esquema de Active Directory tiene objetos y atributos propios, como `sAMAccountName`, `userPrincipalName`, `objectGUID` y `userAccountControl`, que son exactamente los que hay que mapear en cualquier integración real. La segunda es que es la única vía que permite enganchar después la sincronización hacia Entra ID, que es el escenario híbrido de la práctica siguiente.

**Tiempo realista.** Entre seis y ocho horas de trabajo efectivo, repartidas en varios días. La instalación del sistema operativo y la promoción del controlador se llevan buena parte del primer rato.

---

## 2.0 Conceptos que hay que tener claros antes de empezar

Conviene fijar el vocabulario antes de instalar nada, porque el resto de la práctica consiste en ver estas distinciones funcionando.

### 2.1 Un directorio no es una plataforma de gestión de identidades

Un directorio guarda quién existe y qué atributos tiene. Responde a consultas y valida credenciales. No decide quién debe existir, ni quién debe tener acceso a qué, ni revisa si esos accesos siguen justificados.

Una plataforma de gestión de identidades hace lo otro. Aprueba, certifica, revisa, revoca y deja rastro. Puede apoyarse en uno o en varios directorios, pero su trabajo es el gobierno, no el almacenamiento.

Keycloak, en este laboratorio, ocupa un papel intermedio. Es un proveedor de identidad, o sea quien autentica y emite tokens, y en esta práctica lo usamos además como consumidor del directorio. No es una plataforma de gobierno, y conviene no venderlo como tal en el README.

### 2.2 Federar no es copiar

Cuando Keycloak federa un directorio, no se queda con las contraseñas. Para validar una credencial abre una conexión al directorio e intenta autenticarse con ella, lo que en el lenguaje de LDAP se llama un bind. Si el bind funciona, la contraseña era buena. La credencial nunca sale del sistema que la custodia.

Lo que sí puede copiar Keycloak son los atributos, y ahí empieza el problema que vamos a provocar a propósito. Un usuario importado es una copia, y toda copia puede divergir del original o sobrevivirle.

### 2.3 El identificador tiene que ser inmutable

Una persona cambia de apellido, de correo, de departamento y hasta de nombre de usuario, y sigue siendo la misma persona. Si la correspondencia entre el directorio y el proveedor de identidad se ancla a un atributo que cambia, el día que alguien se casa el sistema fabrica un usuario nuevo y deja el anterior sin dueño.

En Active Directory el identificador estable es `objectGUID`, que se asigna al crear el objeto y no cambia nunca, ni al renombrar ni al mover de unidad organizativa.

### 2.4 Active Directory frente a Entra ID

Esta tabla merece estar en el README, porque es la respuesta corta a la pregunta de la entrevista.

| Aspecto | Active Directory Domain Services | Entra ID |
|---|---|---|
| Protocolo de consulta | LDAP | Microsoft Graph, API REST |
| Protocolo de autenticación | Kerberos y NTLM | OAuth 2.0, OpenID Connect, SAML |
| Estructura | Bosque, dominios, unidades organizativas | Plano, sin unidades organizativas |
| Unidad de política | Directivas de grupo sobre equipos del dominio | Acceso condicional sobre sesiones |
| Identificador inmutable | objectGUID | id del objeto |
| Ámbito natural | Red interna, ficheros, impresoras, aplicaciones heredadas | Aplicaciones de internet y servicios en la nube |

No es el mismo producto con otro nombre. Son dos directorios con modelos distintos.

---

## 3.0 Arquitectura del laboratorio

```
  [ Maquina virtual: Windows Server 2025 ]
  corp.lab.local
  AD DS + DNS + AD CS
  IP 192.168.56.10
        ^
        |  LDAPS 636, bind de la cuenta svc-keycloak
        |  (consulta de atributos y validacion de credenciales)
        |
  [ WSL2 Debian ]
        |
        +--- Docker: Keycloak kc-lab, realm lab-iam
        |         User federation --> el directorio
        |         Clientes spa-web y api-backend
        |
        +--- FastAPI: la API de la practica 2, puerto 8000
        |
        +--- Script de reconciliacion
                  |
                  +--- LDAP  ------> inventario del directorio
                  +--- API admin --> inventario de Keycloak
                  +--- CSV ---------> inventario de una aplicacion ficticia

  [ Postman, en Windows ]
        |
        +--- flujo de codigo de autorizacion con PKCE contra Keycloak
        +--- llamada con el token a la API de la practica 2
```

El recorrido completo que se demuestra es este. Una cuenta que vive en Active Directory se autentica contra Keycloak, Keycloak emite un token que lleva sus grupos y su departamento, y una API propia decide con ese token si devuelve datos o devuelve un 403. Tres sistemas, tres protocolos y una sola identidad.

---

## 4.0 Fase 0. Decidir dónde corre el controlador de dominio

### 4.1 Datos verificados de Windows Server 2025

- La edición de evaluación caduca a los 180 días.
- Hay que activarla por internet durante los 10 primeros días o el sistema se apaga solo.
- Se descarga del centro de evaluación de Microsoft como ISO, con versión en español, como VHD o como imagen de Azure.

### 4.2 Elegir hipervisor

Antes de descargar nada, comprobar qué edición de Windows tiene el portátil.

```powershell
systeminfo | Select-String "OS Name","OS Version","Total Physical Memory"
```

```powershell
# Alternativa, más directa
(Get-ComputerInfo).WindowsProductName
(Get-ComputerInfo).WindowsEditionId
```

| Edición | Opción recomendada | Por qué |
|---|---|---|
| Windows 11 Pro | Hyper-V | Ya viene con el sistema y comparte la plataforma de virtualización con WSL2, así que conviven sin trucos |
| Windows 11 Home | VMware Workstation Pro | Descarga sin coste para uso personal desde Broadcom, y gestiona bien la convivencia con la plataforma de máquina virtual |
| Cualquiera | VirtualBox | Gratuito, pero en un equipo con la plataforma de máquina virtual activada funciona sobre la API de Hyper-V y pierde rendimiento |

```powershell
# Ver si Hyper-V está disponible y activado
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All |
    Select-Object FeatureName, State
```

```powershell
# Activarlo, solo en Pro y desde PowerShell como administrador
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -All
```

### 4.3 Repartir la memoria

Con 16 GB en el equipo hay que ser ordenado, porque WSL2 se queda por defecto con una fracción grande de la memoria del anfitrión y luego no la devuelve del todo. Limitarlo con un fichero de configuración.

Crear o editar `C:\Users\TU_USUARIO\.wslconfig`.

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
localhostForwarding=true
```

```powershell
# Aplicar el cambio
wsl --shutdown
wsl -l -v
```

```bash
# Comprobar dentro del WSL que el límite se aplicó
free -h
nproc
```

El reparto para una sesión de trabajo queda así. Cuatro gigas para el controlador de dominio, cuatro para el WSL con Docker y Keycloak, y el resto para Windows, el navegador y Postman. Es justo pero suficiente.

### 4.4 La variante en Azure

Nada impide levantar el controlador como máquina virtual en Azure, pero entonces Keycloak debería vivir en la misma red virtual, porque publicar LDAP en internet no se hace. Esa variante encaja mejor en la práctica siguiente. Para esta, el laboratorio local evita gasto y enseña lo mismo.

---

## 5.0 Fase 1. Instalar y promover el controlador

### 5.1 Crear la máquina virtual

| Parámetro | Valor |
|---|---|
| Memoria | 4096 MB, estática mejor que dinámica |
| Procesadores virtuales | 2 |
| Disco | 60 GB de expansión dinámica |
| Generación, en Hyper-V | 2 |
| Red | Conmutador externo en modo puente, o red interna con acceso al anfitrión |
| Arranque seguro | Activado, Windows Server lo admite |

La red es lo que más condiciona. Keycloak corre dentro del WSL y tiene que poder abrir conexiones TCP contra el puerto 636 del controlador. Con el adaptador en modo puente, la máquina virtual recibe una dirección de la red de casa y todo se ve. Con una red interna hay que verificar la ruta antes de seguir.

### 5.2 Instalar el sistema

Elegir la edición **con experiencia de escritorio**, no Core. En un laboratorio las herramientas gráficas de administración del directorio ahorran mucho tiempo, y la práctica no gana nada por hacerlo a ciegas.

Al terminar, activar por internet dentro de los diez primeros días.

```powershell
# Ver el estado de la licencia y los días restantes
slmgr /dlv
```

### 5.3 Configurar la red antes de promover

Este paso parece burocrático y no lo es. Active Directory depende de DNS para localizar sus propios servicios, así que un DNS mal apuntado deja un dominio que no responde y un montón de errores que no dicen cuál es la causa.

```powershell
# Ver los adaptadores y quedarse con el índice del que toca
Get-NetAdapter | Select-Object Name, InterfaceIndex, Status, LinkSpeed
Get-NetIPConfiguration
```

```powershell
# Dirección estática. Ajusta la red a la tuya
$idx = (Get-NetAdapter | Where-Object Status -eq "Up").InterfaceIndex

New-NetIPAddress -InterfaceIndex $idx `
    -IPAddress 192.168.56.10 `
    -PrefixLength 24 `
    -DefaultGateway 192.168.56.1

# El servidor DNS tiene que apuntarse a sí mismo
Set-DnsClientServerAddress -InterfaceIndex $idx -ServerAddresses 127.0.0.1
```

```powershell
# Comprobar
Get-NetIPAddress -InterfaceIndex $idx | Select-Object IPAddress, PrefixLength
Get-DnsClientServerAddress -InterfaceIndex $idx
Test-NetConnection 8.8.8.8 -InformationLevel Quiet
```

> [!IMPORTANT]
> Mientras el rol de DNS no esté instalado, apuntarse a sí mismo deja al servidor sin resolución de nombres de internet. Es normal y se arregla en cuanto se promueve. Si necesitas internet antes de eso, pon de forma temporal el DNS del router y cámbialo justo antes de promover.

### 5.4 Renombrar el equipo

Hay que hacerlo **antes** de promover. Renombrar un controlador de dominio después es posible pero laborioso, y en un laboratorio no merece la pena.

```powershell
Rename-Computer -NewName "DC01" -Restart
```

```powershell
# Después del reinicio
hostname
```

### 5.5 Instalar el rol y promover

```powershell
# Instalar el rol con sus herramientas de administración
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

```powershell
# Comprobar que quedó instalado
Get-WindowsFeature | Where-Object Installed -eq $true | Where-Object Name -like "AD*"
```

```powershell
# Promover a controlador de un bosque nuevo
Install-ADDSForest `
    -DomainName "corp.lab.local" `
    -DomainNetbiosName "CORP" `
    -InstallDns `
    -SafeModeAdministratorPassword (Read-Host -AsSecureString "Contrasena de modo seguro") `
    -Force
```

> [!WARNING]
> He omitido a propósito los parámetros `-DomainMode` y `-ForestMode`. La documentación del cmdlet lista valores antiguos y, si se fija uno por debajo de lo que admite la versión instalada, el bosque se queda en un nivel funcional más bajo del necesario y algunas cosas no funcionan. Omitirlos deja que el instalador use el nivel que corresponde. Si quieres fijarlos, consulta primero los valores admitidos en tu versión concreta, que es la razón por la que no copio un comando de un tutorial.

La contraseña del modo seguro es la del modo de restauración de servicios de directorio. No es la del administrador del dominio. Se usa para arrancar el controlador en un modo especial cuando la base de datos del directorio está dañada, y conviene guardarla en el gestor de contraseñas y no en un post-it, porque el día que se necesita no hay otra forma de entrar.

El servidor se reinicia solo al terminar.

### 5.6 Verificar que el dominio responde

Esta es la comprobación que separa un dominio funcionando de uno que parece funcionar.

```powershell
# El dominio y el bosque existen
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode, DistinguishedName
Get-ADForest | Select-Object Name, ForestMode, GlobalCatalogs, SchemaMaster
```

```powershell
# Los registros de servicio en DNS, que es como los clientes localizan el directorio
Resolve-DnsName -Name "_ldap._tcp.corp.lab.local" -Type SRV
Resolve-DnsName -Name "_kerberos._tcp.corp.lab.local" -Type SRV
Resolve-DnsName -Name "_gc._tcp.corp.lab.local" -Type SRV
```

```powershell
# Estado de los servicios del controlador
Get-Service NTDS, DNS, Netlogon, kdc | Select-Object Name, Status, StartType
```

```powershell
# Diagnóstico completo del controlador
dcdiag /v /c /e > C:\dcdiag.txt
Select-String -Path C:\dcdiag.txt -Pattern "failed|error|advertencia|aviso" -SimpleMatch
```

```powershell
# Comprobar el canal seguro y la relación de confianza
nltest /dsgetdc:corp.lab.local
```

Esa salida de `Resolve-DnsName` con los registros de servicio es buen material para el README, porque demuestra el punto de que Active Directory se localiza por DNS y no por una dirección configurada a mano.

### 5.7 Un aviso sobre el sufijo .local

El sufijo `.local` está reservado por el estándar de resolución de nombres multicast, lo que significa que en clientes Linux con ese servicio activo la resolución de un nombre `.local` puede intentar irse por multicast en lugar de preguntar al DNS del dominio. En el WSL esto puede dar fallos intermitentes y difíciles de leer.

Dos formas de evitarlo en el laboratorio.

```bash
# Opción simple, trabajar por dirección IP en lugar de por nombre
ping -c 2 192.168.56.10
```

```bash
# Opción ordenada, fijar el nombre en el fichero de hosts del WSL
echo "192.168.56.10  dc01.corp.lab.local dc01" | sudo tee -a /etc/hosts
getent hosts dc01.corp.lab.local
```

Esto hay que documentarlo, porque además es la explicación de por qué en un entorno nuevo de verdad nadie elige `.local` para un dominio, y por qué en la práctica siguiente hace falta un dominio enrutable y verificado.

---

## 6.0 Fase 2. Poblar el directorio con una estructura creíble

La estructura es lo que convierte esto en un laboratorio útil. Un dominio con tres usuarios sueltos en el contenedor por defecto no demuestra nada.

### 6.1 El diseño

```
DC=corp,DC=lab,DC=local
  |
  +-- OU=Empleados        personal interno
  +-- OU=Externos         proveedores y contratas, con caducidad de cuenta
  +-- OU=Servicios        cuentas de servicio, ninguna persona
  +-- OU=Bajas            cuentas desactivadas pendientes de revision
  +-- OU=Grupos           grupos de seguridad
```

Grupos de seguridad de ámbito global: `Finanzas`, `TI`, `Auditoria`, `Riesgos`.

### 6.2 El script completo

Esto es mejor material para el repositorio que veinte capturas, porque un script es reproducible y una captura no. Guardarlo como `crear-estructura.ps1` en el controlador.

```powershell
<#
    crear-estructura.ps1
    Crea la estructura del laboratorio en corp.lab.local
    Ejecutar en el controlador de dominio, como administrador del dominio
#>

$dominio = "DC=corp,DC=lab,DC=local"

# --- 1. Unidades organizativas ---------------------------------------------
$unidades = @("Empleados", "Externos", "Servicios", "Bajas", "Grupos")

foreach ($u in $unidades) {
    if (-not (Get-ADOrganizationalUnit -Filter "Name -eq '$u'" -SearchBase $dominio -ErrorAction SilentlyContinue)) {
        New-ADOrganizationalUnit -Name $u -Path $dominio -ProtectedFromAccidentalDeletion $true
        Write-Host "OU creada: $u" -ForegroundColor Green
    } else {
        Write-Host "OU ya existia: $u" -ForegroundColor Yellow
    }
}

# --- 2. Grupos de seguridad -------------------------------------------------
$grupos = @("Finanzas", "TI", "Auditoria", "Riesgos")

foreach ($g in $grupos) {
    if (-not (Get-ADGroup -Filter "Name -eq '$g'" -ErrorAction SilentlyContinue)) {
        New-ADGroup -Name $g `
            -SamAccountName $g `
            -GroupCategory Security `
            -GroupScope Global `
            -Path "OU=Grupos,$dominio" `
            -Description "Grupo de laboratorio $g"
        Write-Host "Grupo creado: $g" -ForegroundColor Green
    } else {
        Write-Host "Grupo ya existia: $g" -ForegroundColor Yellow
    }
}

# --- 3. Usuarios internos ---------------------------------------------------
$internos = @(
    @{ sam = "cesar23";  nombre = "Cesar";   apellido = "Ramirez"; dep = "Auditoria"; puesto = "Auditor interno";       grupo = "Auditoria" },
    @{ sam = "mlopez";   nombre = "Marta";   apellido = "Lopez";   dep = "Finanzas";  puesto = "Analista financiero";   grupo = "Finanzas"  },
    @{ sam = "jdiaz";    nombre = "Javier";  apellido = "Diaz";    dep = "TI";        puesto = "Administrador de sistemas"; grupo = "TI"    },
    @{ sam = "arios";    nombre = "Ana";     apellido = "Rios";    dep = "Riesgos";   puesto = "Analista de riesgos";   grupo = "Riesgos"   }
)

$clave = Read-Host -AsSecureString "Contrasena inicial para los usuarios de laboratorio"

foreach ($p in $internos) {
    if (Get-ADUser -Filter "SamAccountName -eq '$($p.sam)'" -ErrorAction SilentlyContinue) {
        Write-Host "Usuario ya existia: $($p.sam)" -ForegroundColor Yellow
        continue
    }

    New-ADUser `
        -Name "$($p.nombre) $($p.apellido)" `
        -GivenName $p.nombre `
        -Surname $p.apellido `
        -SamAccountName $p.sam `
        -UserPrincipalName "$($p.sam)@corp.lab.local" `
        -DisplayName "$($p.nombre) $($p.apellido)" `
        -EmailAddress "$($p.sam)@corp.lab.local" `
        -Department $p.dep `
        -Title $p.puesto `
        -Company "Laboratorio IAM" `
        -Path "OU=Empleados,$dominio" `
        -AccountPassword $clave `
        -ChangePasswordAtLogon $false `
        -Enabled $true

    Add-ADGroupMember -Identity $p.grupo -Members $p.sam
    Write-Host "Usuario creado: $($p.sam) en $($p.dep)" -ForegroundColor Green
}

# --- 4. Usuarios externos, con caducidad de cuenta -------------------------
$externos = @(
    @{ sam = "ext.pgarcia"; nombre = "Pablo";  apellido = "Garcia";  dep = "TI";       puesto = "Consultor externo" },
    @{ sam = "ext.lsanz";   nombre = "Laura";  apellido = "Sanz";    dep = "Finanzas"; puesto = "Auditora externa"  }
)

$caducidad = (Get-Date).AddDays(30)

foreach ($p in $externos) {
    if (Get-ADUser -Filter "SamAccountName -eq '$($p.sam)'" -ErrorAction SilentlyContinue) {
        Write-Host "Usuario ya existia: $($p.sam)" -ForegroundColor Yellow
        continue
    }

    New-ADUser `
        -Name "$($p.nombre) $($p.apellido) (externo)" `
        -GivenName $p.nombre `
        -Surname $p.apellido `
        -SamAccountName $p.sam `
        -UserPrincipalName "$($p.sam)@corp.lab.local" `
        -DisplayName "$($p.nombre) $($p.apellido) (externo)" `
        -EmailAddress "$($p.sam)@corp.lab.local" `
        -Department $p.dep `
        -Title $p.puesto `
        -Company "Proveedor externo" `
        -Path "OU=Externos,$dominio" `
        -AccountPassword $clave `
        -AccountExpirationDate $caducidad `
        -Enabled $true

    Write-Host "Externo creado: $($p.sam), caduca el $($caducidad.ToString('yyyy-MM-dd'))" -ForegroundColor Green
}

# --- 5. Cuenta de servicio para Keycloak ----------------------------------
if (-not (Get-ADUser -Filter "SamAccountName -eq 'svc-keycloak'" -ErrorAction SilentlyContinue)) {

    $claveSvc = Read-Host -AsSecureString "Contrasena de la cuenta svc-keycloak"

    New-ADUser `
        -Name "svc-keycloak" `
        -SamAccountName "svc-keycloak" `
        -UserPrincipalName "svc-keycloak@corp.lab.local" `
        -DisplayName "Cuenta de servicio para la federacion de Keycloak" `
        -Description "Solo lectura. Bind LDAP desde Keycloak. No usar para nada mas." `
        -Path "OU=Servicios,$dominio" `
        -AccountPassword $claveSvc `
        -PasswordNeverExpires $true `
        -CannotChangePassword $true `
        -Enabled $true

    Write-Host "Cuenta de servicio creada: svc-keycloak" -ForegroundColor Green
}

# --- 6. Resumen -------------------------------------------------------------
Write-Host "`n--- Estructura final ---" -ForegroundColor Cyan
Get-ADOrganizationalUnit -Filter * -SearchBase $dominio |
    Select-Object Name, DistinguishedName | Format-Table -AutoSize

Get-ADUser -Filter * -SearchBase $dominio -Properties Department, Title, AccountExpirationDate |
    Select-Object SamAccountName, Name, Department, Title, Enabled, AccountExpirationDate |
    Format-Table -AutoSize

Get-ADGroup -Filter * -SearchBase "OU=Grupos,$dominio" |
    ForEach-Object {
        $miembros = (Get-ADGroupMember -Identity $_.Name | Select-Object -ExpandProperty SamAccountName) -join ", "
        [PSCustomObject]@{ Grupo = $_.Name; Miembros = $miembros }
    } | Format-Table -AutoSize
```

```powershell
# Ejecutarlo
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\crear-estructura.ps1
```

### 6.3 La cuenta de servicio, y un matiz importante

Este punto merece un párrafo propio en el README, y hay un detalle técnico que conviene contar bien porque mucha gente lo cuenta mal.

El proveedor de identidad no necesita escribir en el directorio para autenticar a nadie. Le basta leer atributos y hacer un bind con la credencial del usuario. Por tanto, darle una cuenta con privilegios amplía la superficie de ataque sin ganar nada. Es el principio de mínimo privilegio aplicado a una integración.

Ahora el matiz. En Active Directory, el grupo de usuarios autenticados tiene permiso de lectura sobre la mayoría de los atributos del directorio de forma predeterminada. Eso significa que a `svc-keycloak` **no hay que concederle** permisos de lectura, ya los tiene por ser una cuenta válida del dominio. Lo que hay que hacer es no darle nada más, y comprobarlo.

```powershell
# Ver que no pertenece a ningún grupo administrativo
Get-ADPrincipalGroupMembership svc-keycloak | Select-Object Name
```

```powershell
# Ver los permisos efectivos sobre la raíz del dominio
dsacls "DC=corp,DC=lab,DC=local" | Select-String "svc-keycloak"
```

Si esa última consulta no devuelve nada, es la respuesta correcta. La cuenta no tiene permisos explícitos, solo los heredados de usuarios autenticados, que son de lectura.

Endurecerla un poco más es buen material para el repositorio.

```powershell
# Impedir que inicie sesión de forma interactiva en estaciones de trabajo
Set-ADUser svc-keycloak -LogonWorkstations "NINGUNO"
```

```powershell
# Comprobar el estado de la cuenta
Get-ADUser svc-keycloak -Properties PasswordNeverExpires, CannotChangePassword, LogonWorkstations, MemberOf |
    Select-Object SamAccountName, Enabled, PasswordNeverExpires, CannotChangePassword, LogonWorkstations
```

> [!NOTE]
> Una contraseña que no caduca nunca es una concesión que en producción se compensa de otras formas, por ejemplo con una cuenta de servicio administrada de grupo, donde el propio directorio rota la contraseña y nunca se la comunica a un administrador. Eso se usa en la práctica siguiente con el agente de sincronización, así que aquí basta con nombrarlo como mejora pendiente.

---

## 7.0 Fase 3. Interrogar el directorio antes de tocar Keycloak

El objetivo de esta fase es entender que LDAP es un protocolo de consulta y no una aplicación. Y hacerlo antes de configurar la federación, porque así cuando la federación falle sabrás si el problema está en el directorio o en Keycloak.

### 7.1 Preparar las herramientas en el WSL

```bash
sudo apt update && sudo apt install -y ldap-utils dnsutils
ldapsearch -VV 2>&1 | head -3
```

### 7.2 Comprobar que hay ruta hasta el directorio

```bash
# Puerto LDAP en claro
nc -zv 192.168.56.10 389

# Puerto LDAPS, que todavía no funcionará
nc -zv 192.168.56.10 636

# Catálogo global
nc -zv 192.168.56.10 3268
```

```bash
# Si nc no está
sudo apt install -y netcat-openbsd
```

Si el puerto 389 no responde, el problema es de red o del cortafuegos de Windows, y no tiene sentido seguir.

```powershell
# En el controlador, comprobar que está escuchando
Get-NetTCPConnection -LocalPort 389, 636, 3268 -State Listen |
    Select-Object LocalAddress, LocalPort, State
```

### 7.3 La primera consulta

```bash
ldapsearch -x -H ldap://192.168.56.10 \
  -D "CN=svc-keycloak,OU=Servicios,DC=corp,DC=lab,DC=local" \
  -W \
  -b "DC=corp,DC=lab,DC=local" \
  "(objectClass=user)" \
  sAMAccountName userPrincipalName mail department memberOf
```

Hay que saber explicar cada parte, porque es exactamente lo que después se rellena en Keycloak.

| Parte | Qué significa |
|---|---|
| `-x` | Autenticación simple, sin SASL |
| `-H ldap://...` | El servidor y el protocolo |
| `-D` | El nombre distinguido con el que te autenticas, o sea el bind DN |
| `-W` | Pide la contraseña de forma interactiva, para no dejarla en el historial |
| `-b` | La base de búsqueda, el punto del árbol desde el que empieza |
| `"(objectClass=user)"` | El filtro, que selecciona qué objetos |
| La lista final | Los atributos que pides. Sin lista, devuelve todos |

> [!TIP]
> Nunca uses `-w contrasena` en la línea de comandos. Queda en el historial del intérprete y en la tabla de procesos, donde cualquier otro usuario de la máquina la puede leer. Usa `-W` siempre.

### 7.4 Ejercicios sobre la misma consulta

```bash
# Guardar las credenciales en variables para no repetirlas
BINDDN="CN=svc-keycloak,OU=Servicios,DC=corp,DC=lab,DC=local"
BASE="DC=corp,DC=lab,DC=local"
SRV="ldap://192.168.56.10"
```

**Alcance de la búsqueda.** Tres valores y tres resultados distintos.

```bash
# base, solo el objeto indicado en -b
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -s base "(objectClass=*)" dn

# one, solo los hijos directos
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -s one "(objectClass=*)" dn

# sub, todo el subárbol, que es el valor por defecto
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -s sub "(objectClass=user)" dn
```

**Buscar solo personas y no cuentas de equipo.** Esto sorprende la primera vez, porque en Active Directory los equipos también son de clase `user`.

```bash
# Devuelve usuarios y equipos
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(objectClass=user)" sAMAccountName | grep sAMAccountName

# Solo personas
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(objectClass=user))" sAMAccountName department
```

Ese `objectCategory=person` es el filtro que se usa en la vida real, y el `-LLL` quita los comentarios y la información de versión para que la salida sea legible.

**Filtrar por pertenencia a grupo.**

```bash
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(objectClass=user)(memberOf=CN=Auditoria,OU=Grupos,DC=corp,DC=lab,DC=local))" \
  sAMAccountName mail
```

**Filtrar por grupos anidados.** Active Directory tiene una regla de coincidencia especial para recorrer la cadena de pertenencia hacia arriba.

```bash
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(memberOf:1.2.840.113556.1.4.1941:=CN=Auditoria,OU=Grupos,DC=corp,DC=lab,DC=local))" \
  sAMAccountName
```

**Las cuentas desactivadas.** Este es el filtro que hay que documentar, porque es el que sirve para auditar. El estado de una cuenta no es un campo de verdadero o falso, es un bit dentro del entero `userAccountControl`.

```bash
# Cuentas desactivadas, el bit 2
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" \
  sAMAccountName userAccountControl
```

```bash
# Cuentas activas, negando el mismo filtro
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(objectClass=user)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))" \
  sAMAccountName
```

```bash
# Cuentas con contraseña que no caduca, el bit 65536
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(userAccountControl:1.2.840.113556.1.4.803:=65536))" \
  sAMAccountName description
```

Esa cadena de números es un identificador de regla de coincidencia. La que acaba en `803` significa que todos los bits indicados deben estar presentes. Hay otra que acaba en `804` y significa que basta con que esté alguno. Explicar esto en el README demuestra que entendiste el mecanismo en lugar de copiar un filtro.

Tabla de los bits que más se usan.

| Valor | Significado |
|---|---|
| 2 | Cuenta deshabilitada |
| 16 | Cuenta bloqueada |
| 32 | No se requiere contraseña |
| 64 | El usuario no puede cambiar la contraseña |
| 65536 | La contraseña no caduca nunca |
| 8388608 | La contraseña ha caducado |
| 4194304 | La cuenta es sensible y no se puede delegar |

**Cuentas caducadas.** Útil para los externos.

```bash
# accountExpires almacena el tiempo en intervalos de 100 nanosegundos desde 1601
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(accountExpires>=1)(accountExpires<=9223372036854775806))" \
  sAMAccountName accountExpires
```

Ese formato de fecha merece una línea en el README, porque es un clásico de los directorios y la razón de que las comparaciones de fechas en LDAP salgan mal cuando se hacen a la ligera.

**Los atributos reales de un usuario.** Antes de mapear nada, hay que ver qué hay.

```bash
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(sAMAccountName=cesar23)" "*" | head -60
```

```bash
# Y el identificador inmutable, que viene en binario y por eso sale codificado
ldapsearch -x -H $SRV -D "$BINDDN" -W -b "$BASE" -LLL \
  "(sAMAccountName=cesar23)" objectGUID objectSid distinguishedName whenCreated whenChanged
```

Anotar los nombres exactos de los atributos. Son los que se escriben en la fase 5.

### 7.5 Lo que acabas de aprender y conviene escribir

- LDAP no es una aplicación, es un protocolo de consulta sobre un árbol.
- La identidad con la que consultas y la identidad que buscas son cosas distintas.
- Un filtro mal hecho devuelve equipos cuando creías pedir personas.
- El estado de una cuenta vive dentro de un entero de bits, no en un campo booleano.
- Los atributos binarios, como el identificador único, no se leen a simple vista.

---

## 8.0 Fase 4. Cifrar la conexión antes de federar

LDAP en claro transporta la contraseña del bind y la del usuario que se autentica. En un laboratorio se puede empezar en claro para comprobar que todo funciona, pero dejarlo así sería un hallazgo de auditoría, no una configuración.

Lo interesante de documentar no es el procedimiento, es el motivo. Sin cifrado, alguien con acceso a la misma red lee las credenciales de la cuenta de servicio, y con ellas puede enumerar el directorio entero. Y como se vio en la fase anterior, enumerar un directorio da nombres, correos, departamentos, pertenencias a grupos y estado de las cuentas, que es el material con el que se prepara un ataque dirigido.

### 8.1 Demostrarlo antes de arreglarlo

Esto es opcional y es la mejor captura del README si lo haces. Con el tráfico en claro, un analizador de red muestra la contraseña del bind.

```bash
sudo apt install -y tcpdump
```

```bash
# En una terminal, capturar el tráfico LDAP
sudo tcpdump -i any -nn -A 'tcp port 389' -w ldap-claro.pcap
```

```bash
# En otra, lanzar una consulta
ldapsearch -x -H ldap://192.168.56.10 -D "$BINDDN" -W -b "$BASE" -s base "(objectClass=*)" dn
```

```bash
# Después, revisar la captura buscando la operación de bind
sudo tcpdump -r ldap-claro.pcap -A | grep -a -A3 "svc-keycloak" | head -20
```

> [!WARNING]
> Haz esto solo en tu laboratorio y con tus propias cuentas. Y cuando lo documentes, no publiques la captura ni la contraseña, describe el hallazgo y borra el fichero.

```bash
rm -f ldap-claro.pcap
```

### 8.2 Requisitos verificados del certificado

Para que un controlador de dominio acepte conexiones en el puerto 636, su certificado debe cumplir lo siguiente.

- Uso mejorado de clave con **autenticación de servidor**, identificador `1.3.6.1.5.5.7.3.1`.
- El nombre de dominio completo del controlador en el campo de nombre común o como entrada DNS en el nombre alternativo del firmante.
- La clave privada presente en el almacén del equipo local y correctamente asociada al certificado, sin protección fuerte de clave privada activada.
- Emitido por una autoridad en la que confíen tanto el controlador como los clientes.
- El certificado puede estar en el almacén personal del equipo o en el almacén NTDS. El segundo es preferible, porque el directorio lo consulta primero y permite recargarlo sin reiniciar.

### 8.3 Camino recomendado, una autoridad de certificación en el dominio

Es el camino que se parece a la realidad y además deja el certificado renovándose solo.

```powershell
# Instalar el rol
Install-WindowsFeature -Name AD-Certificate -IncludeManagementTools
```

```powershell
# Configurar una autoridad raíz de empresa
Install-AdcsCertificationAuthority `
    -CAType EnterpriseRootCA `
    -CACommonName "Laboratorio-IAM-CA" `
    -KeyLength 2048 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years `
    -ValidityPeriodUnits 5 `
    -Force
```

```powershell
# Comprobar que el servicio está en marcha
Get-Service CertSvc | Select-Object Name, Status
certutil -cainfo
```

Con una autoridad de empresa, la plantilla de controlador de dominio permite la inscripción automática, y el certificado llega con el uso de clave y el nombre correctos. Puede tardar un poco, o se puede forzar.

```powershell
# Forzar la inscripción automática
certutil -pulse
gpupdate /force
```

```powershell
# Ver si el controlador ya tiene su certificado
Get-ChildItem Cert:\LocalMachine\My |
    Select-Object Subject, NotAfter, @{ Name = "EKU"; Expression = { ($_.EnhancedKeyUsageList | ForEach-Object { $_.FriendlyName }) -join ", " } },
        @{ Name = "SAN"; Expression = { ($_.Extensions | Where-Object { $_.Oid.FriendlyName -eq "Subject Alternative Name" }).Format($false) } } |
    Format-List
```

Si no aparece, se solicita a mano.

```powershell
certlm.msc
```

Expandir **Personal**, clic derecho en **Certificados**, **Todas las tareas**, **Solicitar nuevo certificado**, directiva de inscripción de Active Directory, plantilla **Controlador de dominio**, inscribir.

Reiniciar el controlador después de la inscripción.

```powershell
Restart-Computer
```

### 8.4 Comprobar LDAPS desde el propio servidor

```powershell
ldp.exe
```

En la herramienta, menú **Conexión**, **Conectar**, servidor `dc01.corp.lab.local`, puerto `636`, marcar la casilla de SSL. Si aparece la información del árbol en el panel derecho, LDAPS funciona.

```powershell
# Alternativa por línea de comandos
Test-NetConnection -ComputerName dc01.corp.lab.local -Port 636
```

### 8.5 Exportar el certificado de la autoridad y confiar en él desde el WSL

Keycloak y `ldapsearch` tienen que confiar en quien firmó el certificado del controlador.

```powershell
# Exportar el certificado de la autoridad raíz en formato base 64
certutil -ca.cert C:\lab\ca-laboratorio.cer
```

```powershell
# Comprobar que es legible
Get-Content C:\lab\ca-laboratorio.cer -TotalCount 3
```

Pasar el fichero al WSL. Si el controlador es una máquina virtual, lo práctico es una carpeta compartida del hipervisor, o habilitar OpenSSH en el servidor y traerlo con `scp`.

```powershell
# Habilitar el servidor SSH en Windows Server, si hace falta
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName "OpenSSH Server" -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

```bash
# Desde el WSL
mkdir -p ~/lab/certs
scp Administrador@192.168.56.10:C:/lab/ca-laboratorio.cer ~/lab/certs/
```

```bash
# Convertir a PEM si hiciera falta y revisar el contenido
cd ~/lab/certs
openssl x509 -in ca-laboratorio.cer -text -noout | head -20

# Si el fichero viniera en DER en lugar de base 64
openssl x509 -inform DER -in ca-laboratorio.cer -out ca-laboratorio.pem
```

```bash
# Confiar en ella a nivel de sistema, en el WSL
sudo cp ca-laboratorio.pem /usr/local/share/ca-certificates/ca-laboratorio.crt
sudo update-ca-certificates
```

### 8.6 Probar LDAPS desde el WSL

```bash
# Ver el certificado que presenta el controlador
echo | openssl s_client -connect 192.168.56.10:636 -showcerts 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

```bash
# Consulta por LDAPS
ldapsearch -x -H ldaps://dc01.corp.lab.local \
  -D "$BINDDN" -W \
  -b "$BASE" -LLL \
  "(sAMAccountName=cesar23)" sAMAccountName mail department
```

Si falla por validación del certificado, el diagnóstico se hace en dos pasos. Primero comprobar que el problema es el certificado y no otra cosa, y después arreglar la confianza en lugar de desactivar la comprobación.

```bash
# SOLO para diagnosticar, nunca como configuración definitiva
LDAPTLS_REQCERT=never ldapsearch -x -H ldaps://192.168.56.10 \
  -D "$BINDDN" -W -b "$BASE" -s base "(objectClass=*)" dn
```

```bash
# Forma correcta, indicando explícitamente la autoridad de confianza
LDAPTLS_CACERT=$HOME/lab/certs/ca-laboratorio.pem \
ldapsearch -x -H ldaps://dc01.corp.lab.local \
  -D "$BINDDN" -W -b "$BASE" -LLL "(sAMAccountName=cesar23)" sAMAccountName
```

> [!IMPORTANT]
> El nombre con el que te conectas tiene que coincidir con el del certificado. Si el certificado dice `dc01.corp.lab.local` y tú conectas a `192.168.56.10`, la validación falla aunque todo lo demás esté bien. De ahí la entrada en `/etc/hosts` de la fase 1. Esto es un error clásico y vale la pena contarlo en el README, porque explica por qué en la configuración de Keycloak se pone un nombre y no una dirección.

### 8.7 Que Keycloak confíe en la autoridad

El contenedor de Keycloak tiene su propio almacén de confianza y no ve el del WSL. Keycloak carga de forma automática los certificados que encuentra en su directorio de almacenes de confianza, así que basta montar el fichero ahí.

```bash
# Parar y recrear el contenedor montando el certificado
docker stop kc-lab
docker rm kc-lab

docker run -d \
  --name kc-lab \
  -p 8080:8080 \
  -v ~/lab/certs/ca-laboratorio.pem:/opt/keycloak/conf/truststores/ca-laboratorio.pem:ro \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:26.7.0 \
  start-dev
```

> [!WARNING]
> Si recreas el contenedor pierdes el realm `lab-iam` con todo el trabajo de las prácticas anteriores, porque en modo de desarrollo los datos viven dentro del contenedor. **Exporta antes** el realm desde la consola de administración, en los ajustes del realm, con la acción de exportación parcial, y vuelve a importarlo después. La exportación parcial no incluye usuarios, lo cual en este caso da igual porque los usuarios van a venir del directorio.

```bash
# Comprobar que el certificado está dentro y que Keycloak arrancó
docker exec kc-lab ls -l /opt/keycloak/conf/truststores/
docker logs kc-lab 2>&1 | tail -20
```

```bash
# Comprobar desde dentro del contenedor que alcanza el puerto 636
docker exec kc-lab bash -c "timeout 3 bash -c '</dev/tcp/192.168.56.10/636' && echo abierto || echo cerrado"
```

Si da cerrado, el problema es de red entre el contenedor y la máquina virtual, y hay que resolverlo antes de configurar la federación. Según el hipervisor, puede hacer falta añadir una regla al cortafuegos de Windows o cambiar el modo del adaptador de red.

```powershell
# En el controlador, permitir LDAPS de forma explícita
New-NetFirewallRule -DisplayName "LDAPS laboratorio" `
    -Direction Inbound -Protocol TCP -LocalPort 636 -Action Allow
```

---

## 9.0 Fase 5. Federar el directorio en Keycloak

### 9.1 Dónde se configura

En el realm `lab-iam`, sección **User federation**, añadir un proveedor **ldap**.

### 9.2 Los campos, con lo que va en cada uno y por qué

Los rótulos pueden variar algo entre versiones, así que verifícalos en tu interfaz.

| Campo | Valor | Por qué |
|---|---|---|
| Vendor | Active Directory | Precarga los nombres de atributo propios del esquema de Active Directory |
| Connection URL | `ldaps://dc01.corp.lab.local:636` | Nombre, no dirección, para que la validación del certificado funcione |
| Enable StartTLS | Desactivado | Es la alternativa a LDAPS sobre el puerto 389. Se usa una o la otra, no las dos |
| Bind type | simple | Autenticación con un nombre distinguido y una contraseña |
| Bind DN | `CN=svc-keycloak,OU=Servicios,DC=corp,DC=lab,DC=local` | La cuenta de servicio de solo lectura |
| Bind credentials | la contraseña de esa cuenta | Se guarda cifrada en la base de datos de Keycloak |
| Edit mode | READ_ONLY para empezar | Decide la dirección del flujo de datos. Ver más abajo |
| Users DN | `DC=corp,DC=lab,DC=local` | La rama desde la que buscar. Se puede acotar a una unidad concreta |
| Username LDAP attribute | `sAMAccountName` | El nombre con el que la gente se identifica en el dominio |
| RDN LDAP attribute | `cn` | El atributo que forma el nombre relativo del objeto |
| UUID LDAP attribute | `objectGUID` | El identificador inmutable. Es lo más importante de toda la pantalla |
| User object classes | `person, organizationalPerson, user` | Qué objetos se consideran usuarios |
| User LDAP filter | `(objectCategory=person)` | Filtro adicional, para no traerse cuentas de equipo |
| Search scope | Subtree | Buscar en todo el subárbol |
| Import users | Activado | Crea una copia local en Keycloak. Ver el apartado 9.4 |
| Sync Registrations | Desactivado | Si los usuarios creados en Keycloak se escriben en el directorio |
| Periodic full sync | Activado, por ejemplo cada 3600 segundos | Sincronización completa periódica |
| Periodic changed users sync | Activado, por ejemplo cada 300 segundos | Solo los modificados desde la última vez |
| Trust Email | A criterio | Si se da por verificado el correo que viene del directorio |
| Connection pooling | Activado | Reutiliza conexiones, importa en directorios grandes |

### 9.3 El modo de edición, que es lo que de verdad hay que entender

| Modo | Comportamiento |
|---|---|
| READ_ONLY | Keycloak lee del directorio y nunca escribe en él. Los atributos no se pueden modificar desde Keycloak |
| WRITABLE | Lo que se cambia en Keycloak se propaga al directorio |
| UNSYNCED | Los usuarios se importan y después viven en Keycloak por su cuenta, sin devolver nada al origen |

Elegir mal aquí produce divergencia silenciosa entre dos almacenes que todo el mundo da por sincronizados. El modo sin sincronización es el más traicionero, porque funciona perfectamente y va separando las dos copias día a día sin avisar a nadie. Esto es el origen de una parte apreciable de los incidentes de identidad híbrida, y conviene decirlo con esas palabras en el README.

### 9.4 Importar o no importar

La casilla de importación merece explicación propia, porque es la que decide si existe una copia.

Con la importación activada, Keycloak crea un usuario local la primera vez que lo ve y lo mantiene al día con las sincronizaciones. Gana en rendimiento, permite listar usuarios sin consultar el directorio y hace que todo funcione aunque el directorio esté momentáneamente inaccesible. A cambio, crea una copia que puede divergir y, sobre todo, que puede sobrevivir al original. Eso es precisamente lo que se va a provocar en la fase 9.

Con la importación desactivada, Keycloak consulta el directorio cada vez. No hay copia y no hay huérfanas, pero cada operación depende de que el directorio responda.

Para esta práctica conviene dejarla activada, porque el objetivo incluye fabricar una cuenta huérfana a mano y verla.

### 9.5 Por qué objectGUID y no el nombre de usuario

Un usuario puede cambiar de apellido, de correo y de nombre de cuenta, y sigue siendo el mismo sujeto. Si la federación se ancla a un atributo mutable, el día que alguien se casa Keycloak ve un usuario nuevo, lo crea, y deja el anterior sin pareja. Resultado, dos identidades para una persona y una de ellas sin dueño.

`objectGUID` se asigna al crear el objeto y no cambia ni al renombrar ni al mover de unidad organizativa. Comprobarlo es un experimento de un minuto y vale una captura.

```powershell
# Antes
Get-ADUser cesar23 -Properties objectGUID | Select-Object SamAccountName, objectGUID

# Mover y renombrar
Move-ADObject -Identity (Get-ADUser cesar23).DistinguishedName -TargetPath "OU=Externos,DC=corp,DC=lab,DC=local"
Set-ADUser cesar23 -Surname "Ramirez Soto"
Rename-ADObject -Identity (Get-ADUser cesar23).DistinguishedName -NewName "Cesar Ramirez Soto"

# Después
Get-ADUser cesar23 -Properties objectGUID, DistinguishedName |
    Select-Object SamAccountName, objectGUID, DistinguishedName
```

El identificador es el mismo y el nombre distinguido ha cambiado. Ahí está la demostración.

```powershell
# Dejarlo como estaba
Move-ADObject -Identity (Get-ADUser cesar23).DistinguishedName -TargetPath "OU=Empleados,DC=corp,DC=lab,DC=local"
Set-ADUser cesar23 -Surname "Ramirez"
Rename-ADObject -Identity (Get-ADUser cesar23).DistinguishedName -NewName "Cesar Ramirez"
```

### 9.6 Probar la conexión y sincronizar

En la pantalla del proveedor hay botones para comprobar la conexión y la autenticación. Usarlos antes de guardar.

Si la conexión falla, el orden de diagnóstico es este, de abajo hacia arriba.

1. ¿Llega el contenedor al puerto 636? Ya se comprobó en el apartado 8.7.
2. ¿Confía Keycloak en la autoridad? Revisar el registro del contenedor buscando errores de enlace de certificado.
3. ¿El nombre coincide con el del certificado?
4. ¿El nombre distinguido del bind está bien escrito, con las comas y sin espacios de más?
5. ¿La contraseña es la correcta y la cuenta está habilitada?

```bash
# Errores de la federación en el registro de Keycloak
docker logs kc-lab 2>&1 | grep -i -E "ldap|truststore|certificate|PKIX" | tail -30
```

Cuando funcione, en la pestaña **Action** del proveedor están las sincronizaciones. Lanzar primero la completa.

```bash
# Comprobar el resultado por la API de administración
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master --user admin --password admin

docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam --fields username,email,enabled,federationLink
```

### 9.7 Ver qué guardó Keycloak de cada usuario

Esto es importante para la fase de reconciliación, porque aquí aparece la clave de correlación.

```bash
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam \
  -q username=cesar23 | head -40
```

En la salida hay que localizar los atributos que Keycloak añade a los usuarios federados, entre ellos el identificador del proveedor y el que guarda el identificador único del directorio. Anota los nombres exactos que veas, porque son los que va a usar el script de la fase 9 y pueden variar entre versiones.

---

## 10.0 Fase 6. Mapear atributos y grupos

Un atributo del directorio no llega al token por arte de magia. Hace falta un mapeador en la federación para traerlo hasta Keycloak, y otro mapeador de protocolo para meterlo en el token. Son dos pasos distintos y confundirlos es el error más común de esta fase.

### 10.1 Revisar lo que Keycloak crea de oficio

En el proveedor LDAP, pestaña **Mappers**. Vienen creados varios, para el nombre, el apellido, el correo, el nombre de usuario y el nombre distinguido. Mirarlos y entender qué hace cada uno.

### 10.2 Añadir un mapeador para el departamento

**Add mapper**, tipo `user-attribute-ldap-mapper`.

| Campo | Valor |
|---|---|
| Name | `department` |
| User Model Attribute | `department` |
| LDAP Attribute | `department` |
| Read Only | Activado |
| Always Read Value From LDAP | Activado |
| Is Mandatory In LDAP | Desactivado |

La opción de leer siempre el valor del directorio merece una línea. Con ella activada, Keycloak no se fía de su copia y consulta el origen, lo que reduce la divergencia a cambio de más consultas.

Sincronizar y comprobar.

```bash
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=cesar23 | grep -A5 attributes
```

### 10.3 Añadir el mapeador de grupos

**Add mapper**, tipo `group-ldap-mapper`.

| Campo | Valor | Comentario |
|---|---|---|
| Name | `grupos-ad` | |
| LDAP Groups DN | `OU=Grupos,DC=corp,DC=lab,DC=local` | La rama donde viven los grupos |
| Group Name LDAP Attribute | `cn` | |
| Group Object Classes | `group` | En Active Directory es `group`, en OpenLDAP sería `groupOfNames` |
| Membership LDAP Attribute | `member` | |
| Membership Attribute Type | DN | La pertenencia se expresa con nombres distinguidos |
| Membership User LDAP Attribute | `sAMAccountName` | |
| Mode | READ_ONLY | |
| User Groups Retrieve Strategy | LOAD_GROUPS_BY_MEMBER_ATTRIBUTE | Ver el apartado siguiente |
| Preserve Group Inheritance | Activado | Mantiene la jerarquía de grupos anidados |
| Drop non-existing groups during sync | A criterio | Si los grupos borrados en el origen desaparecen de Keycloak |

### 10.4 Los dos lados del mismo hecho

La pertenencia a un grupo se puede leer desde dos sitios opuestos. Desde el grupo, mirando su atributo de miembros. O desde el usuario, mirando su atributo de grupos a los que pertenece.

| Estrategia | Cómo consulta | Cuándo conviene |
|---|---|---|
| Por el atributo de miembros del grupo | Lee los grupos y mira quién está dentro | Pocos grupos con muchos miembros |
| Por el atributo del usuario | Lee el usuario y mira a qué grupos pertenece | Muchos grupos, consulta de un solo usuario |
| Recursiva por el atributo del usuario | Igual pero recorriendo anidamientos | Jerarquías profundas, con coste añadido |

En un directorio de ocho usuarios da igual. En uno de cincuenta mil, la elección equivocada convierte un inicio de sesión en una consulta que tarda segundos. Documentar el razonamiento vale más que el valor elegido.

### 10.5 Sincronizar los grupos y comprobar

En la pestaña del mapeador hay una acción para sincronizar los grupos del directorio hacia Keycloak.

```bash
# Los grupos deberían aparecer en el realm
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get groups -r lab-iam --fields name,path
```

```bash
# Y la pertenencia de un usuario concreto
ID=$(docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=cesar23 --fields id --format csv --noquotes)
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get "users/$ID/groups" -r lab-iam --fields name,path
```

### 10.6 Llevar el departamento y los grupos hasta el token

Esto no es parte de la federación, es configuración del cliente o del ámbito, igual que en las dos prácticas anteriores.

En el cliente `spa-web`, o mejor en un ámbito de cliente dedicado, añadir dos mapeadores de protocolo.

**Para el departamento**, tipo **User Attribute**.

| Campo | Valor |
|---|---|
| Name | `department` |
| User Attribute | `department` |
| Token Claim Name | `department` |
| Claim JSON Type | String |
| Add to access token | Activado |
| Add to ID token | A criterio |

**Para los grupos**, tipo **Group Membership**.

| Campo | Valor |
|---|---|
| Name | `grupos` |
| Token Claim Name | `groups` |
| Full group path | Desactivado, para que salga el nombre y no la ruta |
| Add to access token | Activado |

Si después de esto el token no lleva los campos, el problema está aquí y no en la federación. Ese fallo forma parte del ejercicio y conviene documentarlo tal como ocurrió.

---

## 11.0 Fase 7. Comprobar la autenticación federada

### 11.1 Iniciar sesión con un usuario del directorio

Reutilizar el cliente `spa-web` y Postman, con el flujo de código de autorización y PKCE de la práctica 1. El usuario es uno del directorio, por ejemplo `cesar23`, y la contraseña es la del dominio.

Si entra, acabas de comprobar algo importante. Ese usuario no tiene contraseña almacenada en Keycloak. Keycloak abrió una conexión al directorio e intentó un bind con esa credencial, y el directorio dijo que era correcta.

### 11.2 Decodificar el token y buscar los campos

```bash
# Con el access token en la variable AT
echo $AT | cut -d. -f2 | base64 -d 2>/dev/null | jq
```

```bash
# Si falla por el relleno de base64
python3 -c "
import sys, base64, json
p = '$AT'.split('.')[1]
print(json.dumps(json.loads(base64.urlsafe_b64decode(p + '=' * (-len(p) % 4))), indent=2, ensure_ascii=False))
"
```

Hay que localizar `preferred_username`, `department` y `groups`. Si no están, revisar los mapeadores de protocolo del apartado 10.6.

### 11.3 Cerrar el círculo con la API de la práctica 2

Esta es la parte más satisfactoria del laboratorio. Una cuenta que vive en Active Directory va a decidir si una API escrita por ti devuelve datos o devuelve un 403.

```bash
cd ~/iam-api
source .venv/bin/activate
uvicorn main:app --reload --port 8000
```

```bash
# Con el token del usuario federado
curl -s http://localhost:8000/yo -H "Authorization: Bearer $AT" | jq
```

```bash
# Y el endpoint que exige rol
curl -s -i http://localhost:8000/informes -H "Authorization: Bearer $AT"
```

El rol `auditor` de la práctica 2 era un rol de realm creado a mano en Keycloak. Ahora tienes dos formas de conceder autorización, y conviene probar las dos y explicar la diferencia.

La primera es seguir usando roles de realm asignados en Keycloak. La segunda es que la pertenencia venga del directorio, a través del grupo `Auditoria`, y que la API lea el campo de grupos en lugar del de roles.

```python
# Variante de la función de autorización que lee grupos en lugar de roles
def exigir_grupo(datos, grupo):
    grupos = datos.get("groups", [])
    if grupo not in grupos:
        raise HTTPException(status_code=403, detail=f"Hace falta pertenecer a '{grupo}'")
```

```python
@app.get("/informes")
def informes(authorization: str = Header(default=None)):
    datos = validar_token(authorization)
    exigir_grupo(datos, "Auditoria")
    return {
        "informes": ["cierre-mensual", "accesos-privilegiados"],
        "solicitado_por": datos.get("preferred_username"),
        "departamento": datos.get("department"),
    }
```

Aquí hay un párrafo de peso para el README. Con la primera variante, el acceso se gobierna en Keycloak y el directorio solo autentica. Con la segunda, el acceso se gobierna en el directorio, donde ya existe un proceso para dar y quitar pertenencias, y Keycloak solo transporta el dato. La segunda es la que escala en una organización, y es la razón por la que en las empresas los permisos se modelan como grupos del directorio.

### 11.4 Ver la marca de federación

En **Users**, abrir `cesar23`. Keycloak lo marca como federado e indica de qué proveedor viene. Esa captura resume la fase.

```bash
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=cesar23 \
  --fields username,federationLink,enabled,email
```

### 11.5 Comprobar que no hay credencial local

```bash
# El usuario federado no debería tener credenciales propias de tipo contraseña
ID=$(docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=cesar23 --fields id --format csv --noquotes)
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get "users/$ID/credentials" -r lab-iam
```

Punto para la entrevista. Federar significa preguntar a otro, no copiar. El proveedor de identidad no es el almacén de la credencial.

---

## 12.0 Fase 8. Ciclo de vida, lo que de verdad se opera

Estos cuatro escenarios son el corazón de la práctica. Hay que documentar el comportamiento **observado**, con el tiempo real medido, no el comportamiento esperado.

### 12.1 Alta

```powershell
New-ADUser -Name "Lucia Fernandez" `
    -GivenName "Lucia" -Surname "Fernandez" `
    -SamAccountName "luciaf" `
    -UserPrincipalName "luciaf@corp.lab.local" `
    -DisplayName "Lucia Fernandez" `
    -EmailAddress "luciaf@corp.lab.local" `
    -Department "Riesgos" `
    -Title "Analista de riesgos" `
    -Path "OU=Empleados,DC=corp,DC=lab,DC=local" `
    -AccountPassword (Read-Host -AsSecureString "Contrasena") `
    -Enabled $true

Add-ADGroupMember -Identity "Riesgos" -Members luciaf
```

```bash
# ¿Aparece ya en Keycloak sin sincronizar?
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=luciaf
```

Probar a iniciar sesión con esa cuenta antes de lanzar ninguna sincronización. El resultado depende de si Keycloak resuelve en el momento de la consulta o solo trabaja con lo que importó, y ahí está la lección. Anotar qué pasó y cuánto tardó.

### 12.2 Cambio

```powershell
Set-ADUser luciaf -Department "Auditoria" -Title "Auditora interna"
Add-ADGroupMember -Identity "Auditoria" -Members luciaf
Remove-ADGroupMember -Identity "Riesgos" -Members luciaf -Confirm:$false
```

```powershell
# Confirmar en el origen
Get-ADUser luciaf -Properties Department, Title | Select-Object SamAccountName, Department, Title
Get-ADPrincipalGroupMembership luciaf | Select-Object Name
```

Comprobar cuándo se refleja. En el siguiente inicio de sesión, en la siguiente sincronización programada, o nunca, según cómo estén configurados los mapeadores. Volver a pedir un token y mirar si el campo del departamento cambió.

```bash
# Forzar la sincronización de los modificados desde la API de administración
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get components -r lab-iam \
  --fields id,name,providerId | grep -B2 -A2 ldap
```

El identificador del componente que devuelve esa consulta es el del proveedor LDAP, y con él se puede lanzar la sincronización sin pasar por la interfaz. Verifica la ruta exacta en la documentación de tu versión antes de ejecutarlo, porque la API de sincronización ha cambiado entre versiones.

### 12.3 Escritura inversa

Con el modo de edición en solo lectura, intentar cambiar un atributo desde Keycloak. Anotar el mensaje de error exacto, porque eso es lo que un operador vería.

Después cambiar el modo a escribible, repetirlo y verificar en el directorio que el cambio llegó.

```bash
# Cambiar el apellido desde Keycloak
ID=$(docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=luciaf --fields id --format csv --noquotes)
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh update "users/$ID" -r lab-iam -s 'lastName=Fernandez Soto'
```

```powershell
# Comprobar en el origen si el cambio llegó
Get-ADUser luciaf -Properties sn, DisplayName | Select-Object SamAccountName, sn, DisplayName
```

```bash
# Y por LDAP, que es la comprobación independiente
ldapsearch -x -H ldaps://dc01.corp.lab.local -D "$BINDDN" -W -b "$BASE" -LLL \
  "(sAMAccountName=luciaf)" sn displayName whenChanged
```

Volver a dejarlo en solo lectura al terminar, y explicar en el README por qué ese es el valor por defecto razonable.

### 12.4 Baja

Hay que distinguir con claridad desactivar de borrar.

```powershell
Disable-ADAccount -Identity luciaf
Get-ADUser luciaf -Properties Enabled, userAccountControl |
    Select-Object SamAccountName, Enabled, userAccountControl
```

```bash
# ¿Qué ve Keycloak?
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=luciaf --fields username,enabled
```

Probar tres cosas distintas, porque dan tres respuestas distintas.

1. **Un inicio de sesión nuevo.** Debe fallar, porque el bind contra el directorio ya no funciona.
2. **Un token ya emitido.** Probar la API de la práctica 2 con un token que se obtuvo antes de la baja. Esta parte sorprende, y es exactamente lo que ya viste en la práctica 1. Desactivar la cuenta en el origen no invalida por sí mismo un token que ya se emitió. El acceso se corta cuando ese token caduca o cuando algo fuerza una reevaluación.
3. **El refresco del token.** Intentar canjear el refresh token. Aquí sí debería fallar, porque el canje pasa por Keycloak.

```bash
# Token antiguo contra la API
curl -s -i http://localhost:8000/yo -H "Authorization: Bearer $AT_ANTIGUO"

# Intentar refrescar
curl -s -X POST "http://localhost:8080/realms/lab-iam/protocol/openid-connect/token" \
  -d "grant_type=refresh_token" \
  -d "client_id=spa-web" \
  -d "refresh_token=$RT" | jq
```

Esa secuencia de tres pruebas, con sus salidas reales, es el mejor contenido de la fase. Y la conclusión que hay que escribir es incómoda y verdadera. Entre la baja en el directorio y el corte efectivo del acceso hay una ventana que depende de la vida del token, y reducirla a cero exige mecanismos adicionales como la introspección en cada petición o la evaluación continua de acceso.

Y la diferencia entre desactivar y borrar.

```powershell
# Reactivar y mover a la unidad de bajas, que es lo que se hace de verdad
Enable-ADAccount -Identity luciaf
Disable-ADAccount -Identity luciaf
Move-ADObject -Identity (Get-ADUser luciaf).DistinguishedName `
    -TargetPath "OU=Bajas,DC=corp,DC=lab,DC=local"
Set-ADUser luciaf -Description "Baja el $(Get-Date -Format 'yyyy-MM-dd'). Pendiente de revision de accesos."
```

En entornos regulados casi nunca se borra. La cuenta desactivada conserva el identificador, el historial y la trazabilidad, que es lo que permite responder meses después a qué accesos tenía una persona en una fecha concreta. Un borrado destruye esa cadena, y además deja huérfanos los objetos que apuntaban a ese identificador.

---

## 13.0 Fase 9. Discovery, reconciliación y cuentas huérfanas

Esta es la fase que da nombre al puesto y la que más peso debe tener en el README.

### 13.1 Fabricar una cuenta huérfana a mano

```powershell
# Borrar un usuario del directorio, sin desactivarlo antes
Remove-ADUser -Identity luciaf -Confirm:$false
```

```powershell
# Comprobar que ya no está
Get-ADUser -Filter "SamAccountName -eq 'luciaf'" -ErrorAction SilentlyContinue
```

```bash
# Y mirar qué pasó con la copia importada en Keycloak
docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam -q username=luciaf
```

Si el usuario sobrevive en Keycloak, acabas de fabricar una cuenta huérfana con tus manos. Si además conserva roles o pertenencias a grupos, acabas de fabricar un acceso que ya nadie gobierna desde el origen. Esa captura es el centro del README.

Lanzar una sincronización completa y volver a comprobar, porque el resultado depende de la configuración.

### 13.2 Sacar los tres inventarios

**Inventario 1, el directorio.**

```powershell
New-Item -ItemType Directory -Path C:\inventarios -Force | Out-Null

Get-ADUser -Filter * -SearchBase "DC=corp,DC=lab,DC=local" `
    -Properties SamAccountName, UserPrincipalName, mail, Department, Title, Enabled, objectGUID, DistinguishedName, whenChanged, AccountExpirationDate |
    Where-Object { $_.DistinguishedName -notlike "*OU=Servicios*" } |
    Select-Object `
        SamAccountName,
        UserPrincipalName,
        mail,
        Department,
        Title,
        Enabled,
        DistinguishedName,
        whenChanged,
        AccountExpirationDate,
        @{ Name = "ObjectGuid"; Expression = { $_.objectGUID.ToString() } } |
    Export-Csv -Path C:\inventarios\ad.csv -NoTypeInformation -Encoding UTF8

Write-Host "Exportados $((Import-Csv C:\inventarios\ad.csv).Count) usuarios del directorio"
```

**Inventario 2, Keycloak.**

```bash
mkdir -p ~/inventarios

docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master --user admin --password admin

docker exec -i kc-lab /opt/keycloak/bin/kcadm.sh get users -r lab-iam \
  -q max=1000 > ~/inventarios/keycloak.json

jq 'length' ~/inventarios/keycloak.json
jq '.[0]' ~/inventarios/keycloak.json
```

Esa última consulta es importante. Hay que mirar un usuario completo para identificar con qué nombre guarda Keycloak el identificador del directorio, porque es la clave de correlación y puede cambiar entre versiones.

```bash
# Buscar los atributos que delatan el origen
jq '[.[] | {username, federationLink, enabled, attributes}] | .[0:3]' ~/inventarios/keycloak.json
```

**Inventario 3, una aplicación de negocio ficticia.**

Con dos fuentes se comparan listas. Con tres aparece el problema real, que es decidir cuál manda cuando no coinciden. Crear a mano un CSV que simule el inventario de una aplicación que alguien exportó para la auditoría, con un par de incoherencias metidas a propósito.

```bash
cat > ~/inventarios/app.csv <<'EOF'
usuario,correo,departamento,activo,ultimo_acceso
cesar23,cesar23@corp.lab.local,Auditoria,true,2026-09-28
mlopez,mlopez@corp.lab.local,Finanzas,true,2026-09-30
jdiaz,jdiaz@corp.lab.local,TI,true,2026-10-01
luciaf,luciaf@corp.lab.local,Riesgos,true,2026-08-15
antiguo.empleado,antiguo@corp.lab.local,Finanzas,true,2025-03-12
ext.pgarcia,ext.pgarcia@corp.lab.local,Sistemas,true,2026-09-15
EOF

cat ~/inventarios/app.csv
```

Las incoherencias plantadas son tres. `luciaf` ya no existe en el directorio. `antiguo.empleado` no existió nunca en ningún sitio y representa una cuenta creada a mano hace año y medio. Y `ext.pgarcia` tiene un departamento distinto del que figura en el origen.

**Pasar los ficheros de Windows al WSL.**

```bash
# Si el controlador es una máquina virtual, por SSH
scp Administrador@192.168.56.10:C:/inventarios/ad.csv ~/inventarios/
ls -l ~/inventarios/
```

### 13.3 El script de reconciliación

Guardar como `reconciliar.py`.

```python
#!/usr/bin/env python3
"""
Reconcilia tres inventarios de identidades.

  1. Active Directory, exportado con PowerShell a ad.csv
  2. Keycloak, exportado con kcadm a keycloak.json
  3. Una aplicacion de negocio, en app.csv

La salida son las tres preguntas de cualquier revision de accesos.
Que existe en el origen y no en el destino, que existe en el destino
y no en el origen, y que existe en los dos con datos distintos.
"""

import csv
import json
import sys
from pathlib import Path

BASE = Path.home() / "inventarios"

# El nombre del atributo donde Keycloak guarda el identificador del directorio.
# Hay que confirmarlo mirando un usuario real, porque cambia entre versiones.
ATRIBUTO_ID_LDAP = "LDAP_ID"
ATRIBUTO_DN_LDAP = "LDAP_ENTRY_DN"


def cargar_csv(nombre):
    ruta = BASE / nombre
    if not ruta.exists():
        sys.exit(f"Falta el fichero {ruta}")
    with ruta.open(encoding="utf-8-sig", newline="") as f:
        return list(csv.DictReader(f))


def cargar_json(nombre):
    ruta = BASE / nombre
    if not ruta.exists():
        sys.exit(f"Falta el fichero {ruta}")
    with ruta.open(encoding="utf-8") as f:
        return json.load(f)


def primer_valor(usuario_kc, clave):
    """Los atributos de Keycloak llegan como listas."""
    valor = (usuario_kc.get("attributes") or {}).get(clave)
    if isinstance(valor, list) and valor:
        return valor[0]
    return valor


def booleano(texto):
    return str(texto).strip().lower() in ("true", "1", "si", "yes")


# --------------------------------------------------------------------------
# Carga
# --------------------------------------------------------------------------
ad = cargar_csv("ad.csv")
kc = cargar_json("keycloak.json")
app = cargar_csv("app.csv")

# Indices del directorio, por identificador inmutable y por nombre de cuenta
ad_por_guid = {f["ObjectGuid"].lower(): f for f in ad if f.get("ObjectGuid")}
ad_por_sam = {f["SamAccountName"].lower(): f for f in ad if f.get("SamAccountName")}

# Indices de Keycloak
kc_por_guid = {}
kc_sin_origen = []

for u in kc:
    guid = primer_valor(u, ATRIBUTO_ID_LDAP)
    if guid:
        kc_por_guid[str(guid).lower()] = u
    else:
        kc_sin_origen.append(u)

kc_por_usuario = {u["username"].lower(): u for u in kc}
app_por_usuario = {f["usuario"].lower(): f for f in app}

ancho = 74
print("=" * ancho)
print("INFORME DE RECONCILIACION DE IDENTIDADES")
print("=" * ancho)
print(f"Directorio            {len(ad)} cuentas")
print(f"Keycloak              {len(kc)} cuentas, {len(kc_por_guid)} federadas")
print(f"Aplicacion            {len(app)} cuentas")

# --------------------------------------------------------------------------
# Tramo 1. Directorio frente a Keycloak, cruzando por el identificador inmutable
# --------------------------------------------------------------------------
print()
print("=" * ancho)
print("TRAMO 1. Directorio frente a Keycloak")
print("=" * ancho)

solo_ad = set(ad_por_guid) - set(kc_por_guid)
solo_kc = set(kc_por_guid) - set(ad_por_guid)

print(f"\n[A] En el directorio y no federadas en Keycloak ({len(solo_ad)})")
if not solo_ad:
    print("    ninguna")
for g in sorted(solo_ad):
    f = ad_por_guid[g]
    print(f"    - {f['SamAccountName']:<16} {f.get('Department','') or '-':<12} "
          f"habilitada={f.get('Enabled')}")
    print(f"      {f.get('DistinguishedName','')}")

print(f"\n[B] Federadas en Keycloak y sin origen en el directorio ({len(solo_kc)})  HUERFANAS")
if not solo_kc:
    print("    ninguna")
for g in sorted(solo_kc):
    u = kc_por_guid[g]
    print(f"    - {u['username']:<16} habilitada={u.get('enabled')} "
          f"correo={u.get('email','-')}")
    print(f"      origen declarado: {primer_valor(u, ATRIBUTO_DN_LDAP)}")

print(f"\n[C] Cuentas locales de Keycloak, sin relacion con el directorio ({len(kc_sin_origen)})")
if not kc_sin_origen:
    print("    ninguna")
for u in kc_sin_origen:
    print(f"    - {u['username']:<16} habilitada={u.get('enabled')}   "
          f"creada directamente en Keycloak")

print("\n[D] Presentes en los dos con datos divergentes")
divergencias = 0
for guid, f_ad in ad_por_guid.items():
    u_kc = kc_por_guid.get(guid)
    if not u_kc:
        continue

    dep_ad = (f_ad.get("Department") or "").strip()
    dep_kc = (primer_valor(u_kc, "department") or "").strip()
    if dep_ad != dep_kc:
        divergencias += 1
        print(f"    - {f_ad['SamAccountName']}: departamento "
              f"'{dep_ad}' en el directorio, '{dep_kc}' en Keycloak")

    activo_ad = booleano(f_ad.get("Enabled"))
    activo_kc = bool(u_kc.get("enabled"))
    if activo_ad != activo_kc:
        divergencias += 1
        marca = "  ACCESO RESIDUAL" if activo_kc and not activo_ad else ""
        print(f"    - {f_ad['SamAccountName']}: habilitada={activo_ad} en el directorio, "
              f"habilitada={activo_kc} en Keycloak{marca}")

    correo_ad = (f_ad.get("mail") or "").strip().lower()
    correo_kc = (u_kc.get("email") or "").strip().lower()
    if correo_ad and correo_kc and correo_ad != correo_kc:
        divergencias += 1
        print(f"    - {f_ad['SamAccountName']}: correo '{correo_ad}' frente a '{correo_kc}'")

if divergencias == 0:
    print("    ninguna")

# --------------------------------------------------------------------------
# Tramo 2. Directorio frente a la aplicacion de negocio
# --------------------------------------------------------------------------
print()
print("=" * ancho)
print("TRAMO 2. Directorio frente a la aplicacion de negocio")
print("=" * ancho)

solo_app = set(app_por_usuario) - set(ad_por_sam)
sin_acceso = set(ad_por_sam) - set(app_por_usuario)

print(f"\n[E] En la aplicacion y sin cuenta en el directorio ({len(solo_app)})  HUERFANAS")
if not solo_app:
    print("    ninguna")
for s in sorted(solo_app):
    f = app_por_usuario[s]
    print(f"    - {f['usuario']:<20} ultimo acceso {f.get('ultimo_acceso','-')}  "
          f"activo={f.get('activo')}")

print(f"\n[F] En el directorio y sin acceso a la aplicacion ({len(sin_acceso)})")
print("    (informativo, no todo el mundo necesita todas las aplicaciones)")
for s in sorted(sin_acceso):
    print(f"    - {ad_por_sam[s]['SamAccountName']}")

print("\n[G] Activos en la aplicacion y deshabilitados en el directorio")
residuales = 0
for s, f_app in app_por_usuario.items():
    f_ad = ad_por_sam.get(s)
    if not f_ad:
        continue
    if booleano(f_app.get("activo")) and not booleano(f_ad.get("Enabled")):
        residuales += 1
        print(f"    - {f_app['usuario']}  ACCESO RESIDUAL, la persona esta de baja")
if residuales == 0:
    print("    ninguno")

print("\n[H] Divergencia de departamento entre el directorio y la aplicacion")
difs = 0
for s, f_app in app_por_usuario.items():
    f_ad = ad_por_sam.get(s)
    if not f_ad:
        continue
    dep_ad = (f_ad.get("Department") or "").strip()
    dep_app = (f_app.get("departamento") or "").strip()
    if dep_ad and dep_app and dep_ad != dep_app:
        difs += 1
        print(f"    - {f_app['usuario']}: '{dep_ad}' en el directorio, "
              f"'{dep_app}' en la aplicacion")
if difs == 0:
    print("    ninguna")

# --------------------------------------------------------------------------
# Resumen
# --------------------------------------------------------------------------
print()
print("=" * ancho)
print("RESUMEN DE HALLAZGOS")
print("=" * ancho)
print(f"Huerfanas en Keycloak                      {len(solo_kc)}")
print(f"Cuentas locales sin relacion con el origen {len(kc_sin_origen)}")
print(f"Huerfanas en la aplicacion                 {len(solo_app)}")
print(f"Accesos residuales detectados              {residuales}")
print(f"Divergencias de atributos                  {divergencias + difs}")
```

```bash
python3 reconciliar.py | tee reconciliacion.txt
```

### 13.4 La clave de correlación, y por qué no es el correo

Este apartado es corto y pesa mucho en una entrevista.

El script cruza el directorio con Keycloak por el identificador inmutable, no por el nombre de usuario. Y cruza el directorio con la aplicación por el nombre de usuario solo porque la aplicación ficticia no guarda nada mejor, que es exactamente el problema del mundo real.

Por qué no vale el correo. Cambia cuando alguien se casa o cuando la empresa cambia de dominio. Se reutiliza cuando alguien se va y entra otro con el mismo nombre. Y se comparte en buzones funcionales, donde una dirección no corresponde a una persona.

Por qué no vale el nombre de usuario. También cambia con los apellidos, y en organizaciones con fusiones detrás puede haber colisiones.

Qué pasa cuando el destino no guarda un identificador estable. Entonces la reconciliación es aproximada y hay que documentarlo como limitación, porque un informe que presenta coincidencias aproximadas como certezas es peor que no tener informe. Y la recomendación técnica que sale de ahí es que cualquier integración nueva debe guardar el identificador del origen desde el primer día.

### 13.5 Ampliar el inventario con lo que de verdad se audita

Un inventario de cuentas es el mínimo. Un informe de descubrimiento decente añade estas cosas, y tenerlas en cuenta demuestra que entiendes el trabajo.

```powershell
# Cuentas con contraseña que no caduca
Get-ADUser -Filter 'PasswordNeverExpires -eq $true' -Properties PasswordNeverExpires, DistinguishedName |
    Select-Object SamAccountName, DistinguishedName

# Cuentas que no han iniciado sesión en mucho tiempo
$limite = (Get-Date).AddDays(-90)
Get-ADUser -Filter * -Properties LastLogonDate |
    Where-Object { $_.LastLogonDate -lt $limite -or $null -eq $_.LastLogonDate } |
    Select-Object SamAccountName, LastLogonDate, Enabled

# Miembros de los grupos privilegiados
foreach ($g in @("Domain Admins", "Enterprise Admins", "Schema Admins", "Administrators")) {
    Write-Host "`n--- $g ---"
    Get-ADGroupMember -Identity $g -ErrorAction SilentlyContinue |
        Select-Object SamAccountName, objectClass
}

# Cuentas sin departamento, que suelen ser las que nadie reclama
Get-ADUser -Filter * -Properties Department |
    Where-Object { -not $_.Department } |
    Select-Object SamAccountName, DistinguishedName
```

Cada una de esas consultas responde a una pregunta que alguien hace en una auditoría de verdad, y la lista entera es buen cierre para el README.

---

## 14.0 Fase 10. Extensiones opcionales

### 14.1 Kerberos y SPNEGO

Configurar el inicio de sesión integrado para que un equipo unido al dominio no pida credenciales. Demuestra por qué Active Directory es un mundo de Kerberos y Entra ID no lo es.

Requiere crear una cuenta de servicio con un nombre principal de servicio, generar un fichero de claves y montarlo en el contenedor de Keycloak.

```powershell
# Crear el nombre principal de servicio y el fichero de claves
setspn -S HTTP/keycloak.corp.lab.local CORP\svc-kerberos

ktpass -princ HTTP/keycloak.corp.lab.local@CORP.LAB.LOCAL `
    -mapuser CORP\svc-kerberos `
    -pass * `
    -crypto AES256-SHA1 `
    -ptype KRB5_NT_PRINCIPAL `
    -out C:\lab\keycloak.keytab
```

```powershell
# Verificar
setspn -L CORP\svc-kerberos
```

Después, en el proveedor LDAP de Keycloak, activar la integración de Kerberos indicando el reino, el nombre principal del servidor y la ruta del fichero de claves dentro del contenedor.

Esta extensión es la más laboriosa de todas y la más vistosa. Si el tiempo aprieta, documéntala como trabajo pendiente en lugar de dejarla a medias.

### 14.2 Directiva de contraseñas del dominio

```powershell
# Ver la directiva actual
Get-ADDefaultDomainPasswordPolicy
```

```powershell
# Endurecerla
Set-ADDefaultDomainPasswordPolicy -Identity corp.lab.local `
    -MinPasswordLength 14 `
    -PasswordHistoryCount 24 `
    -ComplexityEnabled $true `
    -LockoutThreshold 5 `
    -LockoutDuration 00:15:00 `
    -LockoutObservationWindow 00:15:00
```

El experimento interesante es comprobar quién manda. Intentar establecer en Keycloak una contraseña que incumpla la directiva del dominio y ver qué pasa. Con el modo de solo lectura, Keycloak no debería ni intentarlo. Con el modo escribible, el rechazo tiene que venir del directorio, y ese mensaje de error es buen material.

```powershell
# Probar el bloqueo de cuenta con intentos fallidos
Get-ADUser cesar23 -Properties LockedOut, badPwdCount, lastBadPasswordAttempt |
    Select-Object SamAccountName, LockedOut, badPwdCount, lastBadPasswordAttempt
```

```powershell
# Desbloquear
Unlock-ADAccount -Identity cesar23
```

Y una pregunta que conviene dejar contestada en el README. Si Keycloak intenta un bind con una contraseña equivocada varias veces, ¿se bloquea la cuenta en el directorio? La respuesta importa, porque significa que un formulario de inicio de sesión expuesto en internet puede bloquear cuentas del directorio corporativo, y eso es una denegación de servicio sobre las personas.

### 14.3 Caducidad de los externos

```powershell
# Forzar la caducidad de una cuenta externa poniéndola en el pasado
Set-ADUser ext.pgarcia -AccountExpirationDate (Get-Date).AddDays(-1)
Get-ADUser ext.pgarcia -Properties AccountExpirationDate |
    Select-Object SamAccountName, AccountExpirationDate, Enabled
```

Observar el efecto sobre el inicio de sesión. Y fijarse en un detalle que confunde a mucha gente. Una cuenta caducada no aparece como deshabilitada, su valor de estado sigue diciendo que está habilitada. Son dos conceptos distintos, y un inventario que solo mire el estado de habilitación pasa por alto a los externos caducados.

```bash
# Cómo se ven por LDAP las caducadas frente a las deshabilitadas
ldapsearch -x -H ldaps://dc01.corp.lab.local -D "$BINDDN" -W -b "$BASE" -LLL \
  "(&(objectCategory=person)(accountExpires>=1)(accountExpires<=9223372036854775806))" \
  sAMAccountName accountExpires userAccountControl
```

Ese contraste merece una línea en el informe de descubrimiento, porque es una vía real por la que un acceso sigue contado como válido cuando ya no lo es.

```powershell
# Restaurar
Set-ADUser ext.pgarcia -AccountExpirationDate (Get-Date).AddDays(30)
```

---

## 15.0 Autoevaluación

Sin mirar apuntes.

1. Diferencia entre un directorio y una plataforma de gestión de identidades.
2. Diferencia entre federar y aprovisionar, con un ejemplo de cada uno.
3. Por qué Entra ID no es Active Directory en la nube. Al menos tres diferencias técnicas.
4. Qué protocolo usa cada uno para autenticar y cuál para consultar.
5. Qué significan el nombre distinguido del bind, la base de búsqueda, el filtro y el alcance en una consulta LDAP.
6. Por qué un filtro por clase de objeto devuelve equipos además de personas, y cómo se arregla.
7. Dónde vive el estado de una cuenta en Active Directory y cómo se consulta.
8. Diferencia entre una cuenta deshabilitada y una cuenta caducada, y por qué importa en un inventario.
9. Qué consecuencia operativa tiene elegir mal el modo de edición de la federación.
10. Por qué el bind de la integración se hace con una cuenta sin permisos de escritura, y por qué en Active Directory no hace falta concederle permisos de lectura.
11. Por qué el identificador de correlación tiene que ser inmutable, y qué pasa el día que no lo es.
12. Qué requisitos debe cumplir el certificado de un controlador de dominio para que acepte LDAPS.
13. Qué ocurre con un token ya emitido cuando se desactiva la cuenta en el directorio, y cómo se reduce esa ventana.
14. Cómo detectarías cuentas huérfanas con tres sistemas destino distintos, y qué usarías como clave de cruce.
15. Por qué en entornos regulados casi nunca se borra una cuenta.

---

## 16.0 Material a documentar

- Diagrama con el directorio, Keycloak, Postman y la API de la práctica 2, indicando la dirección de cada flujo y el protocolo de cada flecha.
- Tabla comparativa de Active Directory frente a Entra ID.
- El script de PowerShell que crea la estructura, con la explicación del diseño de unidades organizativas.
- El apartado de la cuenta de servicio, con el matiz de que la lectura ya viene dada y lo que se evita es concederle más.
- Las consultas LDAP, con especial atención al filtro de bits sobre el estado de la cuenta y al formato de fechas.
- La captura, si la haces, del tráfico en claro y el motivo para cifrar.
- Los requisitos del certificado del controlador y cómo se verificó LDAPS desde los dos lados.
- La configuración del proveedor LDAP campo por campo, con la justificación de cada valor y no solo la captura.
- La prueba del identificador inmutable, moviendo y renombrando al usuario.
- La diferencia entre el mapeador de federación y el mapeador de protocolo, que es el error más común de la práctica.
- Los dos modelos de autorización en la API, por rol de Keycloak y por grupo del directorio, con el razonamiento de cuál escala.
- El resultado real de los cuatro experimentos de ciclo de vida, con los tiempos medidos.
- La secuencia de tres pruebas sobre la baja, o sea inicio de sesión nuevo, token antiguo y refresco.
- El script de reconciliación y su salida, con las huérfanas y los accesos residuales detectados.
- El apartado sobre la clave de correlación y por qué no vale el correo.
- Las consultas de descubrimiento adicionales, como las contraseñas que no caducan y los grupos privilegiados.
- Los errores que te encontraste de verdad, con el mensaje literal y cómo los resolviste.

---

## 17.0 Limitaciones del laboratorio

- Un solo controlador de dominio, sin segundo sitio, sin replicación y sin copias de seguridad del directorio.
- Un solo dominio y un solo bosque, así que no hay confianzas entre dominios ni los problemas que eso trae.
- Keycloak corre en modo de desarrollo, con la base de datos dentro del contenedor y sin persistencia real.
- La autoridad de certificación es del propio dominio y se usa para un único fin, sin separación entre raíz y emisora, sin lista de revocación publicada en condiciones y sin protección de la clave.
- El dominio usa el sufijo `.local`, que está reservado para resolución multicast y no es enrutable, con los problemas descritos en la fase 1. En un entorno nuevo de verdad nadie elegiría ese sufijo.
- Las contraseñas de las cuentas de laboratorio son débiles y compartidas.
- La cuenta de servicio tiene contraseña que no caduca. La alternativa correcta es una cuenta de servicio administrada de grupo, que se usa en la práctica siguiente.
- El inventario de la aplicación de negocio es un CSV escrito a mano, no una consulta a un sistema real.
- No hay registro centralizado ni correlación de eventos, así que la trazabilidad depende de mirar cada sistema por separado.
- Los datos son inventados. En un entorno real todo lo que aquí se exporta a CSV sería información personal sujeta a protección de datos, y un fichero de inventario en el escritorio sería un incidente.

---

## 18.0 Fuentes consultadas

Microsoft Learn, Install-ADDSForest.
https://learn.microsoft.com/en-us/powershell/module/addsdeployment/install-addsforest

Microsoft Learn, configurar certificados para LDAP sobre SSL en Active Directory.
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/configure-ldap-signing-certificates

Microsoft Evaluation Center, Windows Server 2025.
https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025

Microsoft Learn, herramientas de sincronización con Entra ID.
https://learn.microsoft.com/en-us/entra/identity/hybrid/sync-tools

Microsoft Learn, condiciones de la prueba de Entra ID P1 y P2.
https://learn.microsoft.com/en-us/answers/questions/2151340/activate-p1-or-p2-trial-licenses

Keycloak, guía de administración del servidor.
https://www.keycloak.org/docs/latest/server_admin/index.html

---

## 19.0 Lo que queda para la práctica siguiente

El escenario híbrido completo. Un inquilino de Entra ID, la sincronización desde este mismo Active Directory con Entra Cloud Sync y el aprovisionamiento por SCIM hacia una aplicación. Está desarrollado en su propio enunciado, y arranca justo donde termina este.

