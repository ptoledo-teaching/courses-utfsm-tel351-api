# API - Lambda Functions y API Gateway

Este laboratorio se centra en la ejecución de funciones mediante AWS Lambda y en su publicación mediante Lambda Function URL y Amazon API Gateway. La actividad implementa un sistema de diagnóstico para el Halcón Milenario y comunicaciones de la Holonet para observar configuración, eventos, routes, integrations y registros de ejecución.

## Resultados esperados

Al completar el laboratorio podrá:

1. Crear una Lambda function desde cero y probar el código inicial generado por AWS
2. Publicar una function mediante una Lambda Function URL
3. Modificar la function inicial para implementar y probar una aplicación en Python
4. Utilizar variables de entorno para modificar su comportamiento sin cambiar el código
5. Crear y probar una Lambda function escrita en Node.js
6. Crear una HTTP API con routes e integrations diferentes
7. Provocar una falla controlada e inspeccionarla mediante Amazon CloudWatch Logs
8. Reconocer los datos HTTP que un navegador incorpora al `event`
9. Comparar eventos generados por distintas formas de invocación
10. Eliminar todos los recursos utilizados

## Preparación previa

Antes del laboratorio:

- Confirme que su cuenta AWS personal se encuentra operativa
- Ingrese a AWS Management Console mediante la identidad administrativa de uso regular
- Confirme que puede acceder a AWS Lambda, Amazon API Gateway y Amazon CloudWatch

## Actividad

### 1. Establecer la Región principal

1. En la esquina superior derecha seleccione **South America (São Paulo)**
2. Confirme que la Región visible sea **`sa-east-1`**

> Todos los recursos del laboratorio se crean en `sa-east-1`. No cambie la Región durante la actividad.

### 2. Reconocer el ciclo básico de una Lambda function

#### 2.1 Crear una function desde cero

1. En el buscador superior busque y abra **Lambda**
2. En **Functions**, seleccione **Create function**
3. Seleccione **Author from scratch**
4. Configure:

   | Campo | Valor |
   | --- | --- |
   | Function name | `tel351-hyperdrive` |
   | Runtime | Python 3.14 |

5. Mantenga las demás opciones con sus valores por omisión
6. Seleccione **Create function**
7. Espere hasta que la consola confirme que la function fue creada

La opción **Author from scratch** crea la Lambda function con un archivo y un handler iniciales. En esta primera actividad no edite el código generado ni cambie la configuración del runtime o del handler.

#### 2.2 Ejecutar un test desde la consola Lambda

1. En la pestaña **Code**, observe el código generado por AWS sin modificarlo
2. Seleccione **Test → Create new event**
3. Use como **Event name**: `hello-console` y tipo `Synchronous`
4. Use el siguiente **Event JSON**:

   ```json
   {}
   ```

5. Guarde y luego ejecute el test con el botón **Invoke**
6. Confirme que la invocación termina correctamente (se despliegará automaticamente una pestaña output)
7. Del el resultado y reconozca:
   - El valor retornado por el handler
   - El status de la ejecución
   - La duración de ejecución, invocación y cobro
   - La memoria configurada y la memoria utilizada
   - El identificador de la solicitud

El test invoca directamente la function y entrega el JSON configurado al handler como `event`. El código inicial no utiliza su contenido, por lo que un objeto vacío es suficiente.

#### 2.3 Crear y probar una Lambda Function URL

1. Abra **Configuration → Function URL**
2. Seleccione **Create function URL**
3. En **Auth type**, seleccione **NONE**
4. Seleccione **Save**
5. Copie la **Function URL** y ábrala en una pestaña nueva del navegador
6. Confirme que obtiene la respuesta definida por el código inicial

> `NONE` permite invocar la function sin autenticación. Cualquier persona que conozca la URL podrá utilizarla hasta que sea eliminada.

La misma function fue invocada primero desde la consola Lambda y luego mediante un endpoint HTTP público. Ninguna de estas acciones requirió editar o desplegar código.

### 3. Implementar el diagnóstico del hiperpropulsor

La function `tel351-hyperdrive` ya puede invocarse mediante la consola y su Function URL. En esta etapa se modificará su configuración y se reemplazará el código inicial para implementar un diagnóstico sencillo.

#### 3.1 Configurar las variables de entorno

1. Abra **Configuration → Environment variables**
2. Seleccione **Edit**
3. Agregue:

   | Key | Value |
   | --- | --- |
   | `SHIP_NAME` | `Millennium Falcon` |
   | `MAX_TEMPERATURE` | `85` |

4. Seleccione **Save**

Estas variables corresponden a configuración, la function leerá sus valores durante cada invocación.

#### 3.2 Implementar la function

1. Abra la pestaña **Code**
2. En **Code source**, abra `lambda_function.py`
3. Reemplace su contenido por:

   ```python
   import json
   import os


   def lambda_handler(event, context):
       parameters = event.get("queryStringParameters") or {}
       raw_temperature = parameters.get("temperature")

       ship_name = os.environ["SHIP_NAME"]
       max_temperature = float(os.environ["MAX_TEMPERATURE"])

       print(f"Received temperature={raw_temperature} for ship={ship_name}")

       temperature = float(raw_temperature)
       status = "OPERATIONAL" if temperature <= max_temperature else "CRITICAL"

       body = {
           "ship": ship_name,
           "system": "Hyperdrive",
           "temperature": temperature,
           "maximumTemperature": max_temperature,
           "status": status,
           "requestId": context.aws_request_id,
       }

       return {
           "statusCode": 200,
           "headers": {"content-type": "application/json"},
           "body": json.dumps(body),
       }
   ```

4. Seleccione **Deploy** y espere la confirmación

La function espera un `event` con `queryStringParameters`, como los que producen una Function URL y una HTTP API. `context` entrega metadatos propios de la invocación y no datos enviados por el cliente.

#### 3.3 Probar desde la consola Lambda

1. Seleccione **Test → Create new event**
2. Use como **Event name**: `hyperdrive-operational`
3. En **Event JSON**, escriba:

   ```json
   {
     "queryStringParameters": {
       "temperature": "72"
     }
   }
   ```

4. Guarde y ejecute el test
5. Confirme que la invocación termina correctamente
6. Abra el resultado y compruebe:
   - `statusCode` es `200`
   - El `body` identifica la nave y el sistema
   - `temperature` es `72.0`
   - `maximumTemperature` es `85.0`
   - `status` es `OPERATIONAL`
   - Existe un `requestId`
7. En el menú de **Test events** edite el test `hyperdrive-operational`, cambie la temperatura del test a `95` y ejecútelo nuevamente
8. Confirme que la invocación continúa siendo exitosa, pero `status` cambia a `CRITICAL`

#### 3.4 Modificar la configuración

1. Regrese a **Configuration → Environment variables → Edit**
2. Cambie `MAX_TEMPERATURE` a `100`
3. Guarde la configuración
4. Ejecute nuevamente el test con temperatura `95`
5. Confirme que `maximumTemperature` es `100.0` y `status` vuelve a ser `OPERATIONAL`
6. Restablezca `MAX_TEMPERATURE` a `85`

El comportamiento cambió sin modificar ni desplegar nuevamente el código.

### 4. Crear el inspector de transmisiones Holonet

#### 4.1 Crear la Lambda function en Node.js

1. Regrese a **Lambda → Functions**
2. Seleccione **Create function → Author from scratch**
3. Configure:

   | Campo | Valor |
   | --- | --- |
   | Function name | `tel351-holonet` |
   | Runtime | Node.js 24.x |

4. Mantenga las demás opciones con sus valores por omisión
5. Seleccione **Create function**

#### 4.2 Implementar la function

1. Abra la pestaña **Code**
2. En **Code source**, abra `index.mjs`
3. Reemplace su contenido por:

   ```javascript
   export const handler = async (event, context) => {
     console.log(JSON.stringify({
       requestId: context.awsRequestId,
       routeKey: event.routeKey,
       rawPath: event.rawPath,
     }));

     const body = {
       message: "Holonet transmission received",
       event,
       context: {
         awsRequestId: context.awsRequestId,
         functionName: context.functionName,
         remainingTimeInMillis: context.getRemainingTimeInMillis(),
       },
     };

     return {
       statusCode: 200,
       headers: { "content-type": "application/json" },
       body: JSON.stringify(body, null, 2),
     };
   };
   ```

4. Seleccione **Deploy** y espere la confirmación

#### 4.3 Probar desde la consola Lambda

1. Seleccione **Test → Create new event**
2. Use como **Event name**: `holonet-console`
3. En **Event JSON**, escriba:

   ```json
   {
     "source": "lambda-console",
     "transmission": {
       "sender": "Leia",
       "destination": "Yavin-4",
       "priority": "high"
     }
   }
   ```

4. Guarde y ejecute el test
5. Confirme que el `body` contiene exactamente el `event` proporcionado
6. Observe que este evento no contiene método, path, headers ni otros datos HTTP

La consola Lambda entrega directamente el JSON configurado como evento de prueba. Un endpoint HTTP debe construir un evento a partir de una solicitud real.

### 5. Publicar las functions mediante API Gateway

#### 5.1 Crear una HTTP API vacía

1. En el buscador superior busque y abra **API Gateway**
2. En **APIs**, seleccione **Create an API**
3. En **HTTP API**, seleccione **Build**
4. Use como **API name**: `tel351-rebel-api`
5. En **IP address type**, seleccione **IPv4** y continue con next
6. Avance sin crear routes; se configurarán después
7. Mantenga el stage `$default` con **Auto-deploy** habilitado
8. Revise y cree la API

El stage `$default` permite utilizar la invoke URL sin agregar el nombre de un stage al path. Auto-deploy publica los cambios posteriores sin crear manualmente otro deployment.

#### 5.2 Crear las routes

1. Dentro de `tel351-rebel-api`, abra **Routes**
2. Seleccione **Create**
3. Configure:

   | Method | Path |
   | --- | --- |
   | GET | `/hyperdrive` |

4. Cree la route
5. Repita el procedimiento para:

   | Method | Path |
   | --- | --- |
   | ANY | `/holonet` |

La route `GET /hyperdrive` acepta solamente ese método. `ANY /holonet` acepta cualquier método que no tenga otra route más específica para el mismo path.

#### 5.3 Crear y asociar las integrations

1. Abra **Integrations**
2. Seleccione **Manage integrations → Create**
3. Configure la primera integration:

   | Campo | Valor |
   | --- | --- |
   | Attach this integration to a route | `GET /hyperdrive` |
   | Integration type | Lambda function |
   | Lambda function | `tel351-hyperdrive` |

4. Cree la integration
5. Repita el procedimiento para:

   | Campo | Valor |
   | --- | --- |
   | Attach this integration to a route | `ANY /holonet` |
   | Integration type | Lambda function |
   | Lambda function | `tel351-holonet` |

6. En **Routes** podrá verificar que cada route ahora tiene asociada la correspondiente Lambda Function

#### 5.4 Probar el diagnóstico del hiperpropulsor

1. Abra **Stages → `$default`** o la vista general de la API
2. Copie la **Invoke URL** y ábrala en el navegador. Debería obtener un mensaje `Not Found`
3. Edite la url para agregar el parámetro **temperature** con valor 72

   ```text
   <invoke-url>/hyperdrive?temperature=72
   ```

4. Confirme la respuesta que debe indicar `status: OPERATIONAL`
5. Cambie la temperatura a `95`
6. Confirme `status: CRITICAL`

Ambas respuestas deben mantener status HTTP `200`: el segundo resultado representa una condición crítica de la nave, no una falla al ejecutar la function.

### 6. Inspeccionar un error mediante CloudWatch Logs

#### 6.1 Provocar una falla de ejecución

1. Abra en el navegador y vuelva a ejecutar:

   ```text
   <invoke-url>/hyperdrive?temperature=unknown
   ```

2. Confirme que la respuesta ya no corresponde a un diagnóstico `OPERATIONAL` o `CRITICAL`
3. Observe que API Gateway devuelve un error HTTP de la familia `5xx`

#### 6.2 Localizar el error

1. Abra **Lambda → Functions → `tel351-hyperdrive`**
2. Abra **Monitor**
3. Seleccione **View CloudWatch logs**
4. En el log group `/aws/lambda/tel351-hyperdrive`, abra el log stream más reciente
5. Localice la invocación que contiene `Received temperature=unknown`
6. Identifique:
   - El `RequestId` de la invocación
   - El tipo y mensaje de la excepción
   - El stack trace que apunta a la conversión de la temperatura
   - Los registros `START`, `END` y `REPORT`
   - La duración y memoria informadas en `REPORT`

El error HTTP solamente indica que la integración no pudo producir una respuesta válida. CloudWatch Logs entrega el detalle necesario para localizar la causa dentro de la function.

> No es necesario solucionar el error, esta exploración de CloudWatch es para identificar donde se puede obtener la información sobre las ejecuciones de las funciones lambda

### 7. Inspeccionar el evento recibido desde API Gateway

#### 7.1 Probar la transmisión Holonet

1. Abra en el navegador:

   ```text
   <invoke-url>/holonet?sender=Leia&destination=Yavin-4&priority=high
   ```

2. Puede confirmar que la función llamada es `tel351-holonet` por el mensaje `Holonet transmission received`
3. Esta función entrega al usuario el evento completo que está siendo recibido por la función lambda:
   - `version` es `2.0`
   - `routeKey` identifica `ANY /holonet`
   - `rawPath` es `/holonet`
   - `rawQueryString` contiene los query parameters enviados
   - `queryStringParameters` contiene `sender`, `destination` y `priority`
   - `requestContext.stage` es `$default`
   - `requestContext.http.method` es `GET`
   - `requestContext.domainName` utiliza `execute-api`
4. Dentro de `headers`, busque valores agregados por su navegador, como `accept`, `accept-language` y `user-agent`

#### 7.2 Comparar los eventos

Compare las dos invocaciones realizadas sobre `tel351-holonet`:

- La consola Lambda entregó solamente el JSON definido como test event
- API Gateway generó un evento HTTP asociado a la route `ANY /holonet`
- El método, dominio, route y path fueron incorporados por API Gateway a partir de la solicitud HTTP
- Los headers y datos del navegador dependen del cliente que realizó la solicitud y no fueron definidos en el test event

### 8. Comprobación final

Antes de iniciar la limpieza confirme:

- `tel351-hyperdrive` pudo invocarse mediante un test y su Function URL antes de modificar el código inicial
- `tel351-hyperdrive` responde según `MAX_TEMPERATURE`
- `tel351-holonet` retorna el `event` y un resumen de `context`
- La Function URL permite invocar directamente una Lambda function desde el navegador
- `GET /hyperdrive` y `ANY /holonet` se encuentran conectadas a integrations diferentes
- La invoke URL de API Gateway permite utilizar ambas routes
- Puede reconocer diferencias entre el test event de la consola Lambda y el evento HTTP generado por API Gateway
- La falla por temperatura inválida aparece en CloudWatch Logs

### 9. Limpieza posterior

Realice la limpieza después de completar todas las comprobaciones, fuera del bloque de clase si es necesario.

#### 9.1 API Gateway

1. Abra **API Gateway → APIs**
2. Seleccione `tel351-rebel-api`
3. Use **Actions → Delete** y confirme

#### 9.2 Lambda Function URL y functions

1. Abra **Lambda → Functions → `tel351-hyperdrive`**
2. En **Configuration → Function URL**, seleccione **Delete** y confirme
3. Antes de eliminar las functions, abra **Configuration → Permissions** en cada una y anote el nombre de su execution role
4. Regrese a **Functions**
5. Seleccione `tel351-hyperdrive` y `tel351-holonet`
6. Use **Actions → Delete** y confirme

#### 9.3 CloudWatch Logs

1. Abra **CloudWatch → Log groups**
2. Seleccione:
   - `/aws/lambda/tel351-hyperdrive`
   - `/aws/lambda/tel351-holonet`
3. Use **Actions → Delete log group(s)** y confirme

#### 9.4 Execution roles

1. Abra **IAM → Roles**
2. Busque los execution roles anotados de las funciones
3. Seleccione los roles
4. Seleccione **Delete** y confirme

#### 9.5 Verificación final de limpieza

Confirme:

- `tel351-rebel-api` no aparece en API Gateway
- No existe la Function URL
- Ambas Lambda functions fueron eliminadas
- Los dos log groups fueron eliminados
- No permanecen los dos execution roles creados para el laboratorio
