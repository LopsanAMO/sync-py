
##QUICKSTART SAT

### Requerimientos

1. Manejo de shell/bash
2. Algunas credenciales de acceso al SAT (RFC y CIEC)
3. [Python 2.7.6](https://www.python.org/downloads/)
4. Tener instado[Python Install Package pip](https://pip.pypa.io/en/stable/installing/)
5. Tener instalada la librería requests de python (pip install requests)


### Introducción

A lo largo de este tutorial te enseñaremos como consumir API SYNC por medio del SDK de python. Al terminar este tutorial habrás podido crear nuevos usuarios en Paybook, sincronizar algunas instituciones de estos usuarios y visualizar las transacciones sincronizadas.

La documentación completa del SDK la puedes consultar [aquí](https://github.com/Paybook/sync-py/blob/master/readme.md) 

##En la consola:

####1. Instalamos el SDK de Paybook y dependencias:

Para consumir el API de Paybook desde python lo primero que tenemos que hacer es instalar la libreria de paybook haciendo uso del paquete de instalaciones de python:

```
$ sudo pip install paybook==1.5
```

Este tutorial está basado en el script [quickstart.py](https://github.com/Paybook/sync-py/blob/master/quickstart.py). Es recomendable descargar el archivo, configurar los valores YOUR_API_KEY, CIEC y RFC y ejecutarlo:

```
$ python quickstart_sat.py
```

Una vez que has ejecutado el archivo podemos continuar analizando el código.

####2. Iniciamos pyhton:
Una vez instalada la librería de python (SDK) ejecutamos el interprete de python desde la consola (puedes también crear un archivo con extensión .py con todos los comandos y ejecutarlo al final): 

```
$ python
>>>
```

####3. Importamos paybook
El primer paso es importar el SDK y algunas dependencias:

```python
import time
import sys
import paybook.sdk as paybook_sdk
```

####4. Configuramos el SDK
Una vez importado el SDK tenemos que configurarlo, para esto únicamente se necesita tu API KEY de Paybook.

```python
paybook_sdk.Paybook(YOUR_API_KEY)
```

####5. Creamos un usuario:
Una vez configurado el SDK, el primer paso será crear un usuario, este usuario será, por ejemplo, aquel del cual queremos obtener sus cuentas bancarias, o bien, sus facturas del SAT.

**Importante**: todo usuario estará ligado al API KEY con el que configuraste el SDK (paso 4)

```python
user = paybook_sdk.User(name='MY_USER')
```

####6. Consultamos los usuarios ligados a nuestra API KEY:
Para verificar que el usuario creado en el paso 5 se haya creado corréctamente podemos consultar la lista de usuarios ligados a nuestra API KEY.

```python
my_users = paybook_sdk.User.get()
	for user in my_users:
	    print user.name
```

####7. Creamos una nueva sesión:
Para sincronizar las facturas del SAT, o bien, las cuentas de una institución bancaria de un usuario primero tenemos que crear una sesión, la sesión estará ligada al usuario y tiene un proceso de expiración de 5 minutos después de que ésta ha estado inactiva. Para crear una sesión:

```python
session = paybook_sdk.Session(user)
```

####8. Podemos validar la sesión creada:
De manera opcional podemos validar la sesión, es decir, checar que no haya expirado.

```python
session_verified = session.verify()
print 'Session verified: ' + str(session_verified)
```

####9. Consultamos el catálogo de instituciones que podemos sincronizar y extraemos el SAT:
Paybook tiene un catálogo de instituciones que podemos sincronizar por usuario:

![Instituciones](https://github.com/Paybook/sync-py/blob/master/sites.png "Instituciones")

A continuación consultaremos este catálogo y seleccionaremos el sitio del SAT para sincronizar las facturas del usuario que hemos creado en el paso 5:

```python
sat_site = None
sites = paybook_sdk.Catalogues.get_sites(session=session)
for site in sites:
	print site.name.encode('utf-8')
	if site.name == 'CIEC':
	    sat_site = site
print 'SAT site: ' + sat_site.id_site + ' ' + sat_site.id_site
```

####10. Configuramos nuestras credenciales del SAT:
Una vez que hemos obtenido el sitio del SAT del catálogo de institiciones, configuraremos las credenciales de nuestro usuario (estas credenciales son las que el usuario utiliza para acceder al portal del SAT).

```python
credentials_data = {
	'rfc' : 'RFC',
	'password' : 'CIEC'
}
sat_credentials = paybook_sdk.Credentials(session=session,id_site=sat_site.id_site,credentials=credentials_data)
print sat_credentials.username
```

####11. Checamos el estatus de sincronización de las credenciales creadas y esperamos a que la sincronización finalice:
Cada vez que registamos unas credenciales Paybook levanta un Job que se encargará de validar esas credenciales y posteriormente sincronizar las transacciones. Este Job se puede representar como una maquina de estados:

![Job Estatus](https://github.com/Paybook/sync-py/blob/master/normal.png "Job Estatus")

Una vez registradas las credenciales se obtiene el primer estado (Código 100), posteriormente una vez que el Job ha empezado se obtiene el segundo estado (Código 101). Después de aquí, en caso de que las credenciales sean válidas, prosiguen los estados 202, 201 o 200. Estos indican que la sincronización está pendiente (sigue en proceso, código 201), que no se encontraron transacciones (código 202), o bien, la sincronización ha terminado (código 200). El SDK proporciona un método para consultar el estado actual del Job, el cual se puede consultar constantemente hasta que se obtenga el estado requerido por el usuario, para este ejemplo especifico consultamos el estatus hasta que se obtenga un código 200, es decir, que la sincronización haya terminado:

```python
sat_sync_completed = False
while not sat_sync_completed: 
	print 'Polling ... '
	time.sleep(5)
	sat_status = sat_credentials.get_status(session=session)
	for status in sat_status:
		code = status['code']
		if code == 200:
			sat_sync_completed = True
```

####12. Consultamos las facturas sincronizadas:
Una vez que ya hemos checado el estatus de la sincronización y hemos verificado que ha terminado (código 200) podemos consultar las facturas sincronizadas:
```python
sat_transactions = paybook_sdk.Transaction.get(session=session)
print 'Facturas del SAT: ' + str(len(sat_transactions))
```

####13. Consultamos la información de archivos adjuntos:
Podemos también consultar los archivos adjuntos a estas facturas, recordemos que por cada factura el SAT tiene una archivo XML y un archivo PDF por cada factura.
```python
attachments = paybook_sdk.Attachment.get(session=session)
print 'Archivos XML/PDF del SAT: ' + str(len(attachments))
```

####14. Obtenemos el XML y PDF de alguna factura:
Podemos descargar estos archivos:
```python
if len(attachments) > 0:
	i = 0
	for attachment in attachments:
		i+=1
		id_attachment = attachment.url[1:]
		print id_attachment
		attachment_content = paybook_sdk.Attachment.get(session=session,id_attachment=id_attachment)
		print 'Attachment ' + str(i) + ':'
		print str(attachment_content)
		if i == 2:
			break	
```

¡Felicidades! has terminado con este tutorial. 

### Siguientes Pasos

- Revisar el tutorial de como sincronizar una institución bancaria con credenciales simples (usuario y contraseña) [clic aquí](https://github.com/Paybook/sync-py/blob/master/quickstart_normal_bank.md)

- Revisar el tutorial de como sincronizar una institución bancaria con token [clic aquí](https://github.com/Paybook/sync-py/blob/master/quickstart_token_bank.md)

- Puedes consultar y analizar la documentación completa del SDK [aquí](https://github.com/Paybook/sync-py/blob/master/readme.md)

- Puedes consultar y analizar la documentación del API REST [aquí](https://www.paybook.com/sync/docs#api-Overview)

- Acceder a nuestro proyecto en Github y checar todos los recursos que Paybook tiene para ti [clic aquí](https://github.com/Paybook)

























