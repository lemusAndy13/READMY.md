# Bienestar Estudiantil · Análisis de Emociones por Foto

Aplicación que analiza una foto facial para identificar la emoción dominante con AWS Rekognition y devuelve recomendaciones de bienestar desde una base de datos local (SQLite). Incluye interfaz web simple (`index.html`), función backend en AWS Lambda (`lambda_fuction.py`) y script para generar la base de datos (`crear_bd.py`).


## 1) Resumen funcional
- El usuario inicia sesión o se registra.
- Acepta consentimiento de análisis.
- Sube una imagen o usa la cámara del dispositivo.
- El backend (Lambda) analiza la emoción mediante Rekognition.
- Se consultan consejos en `SQLite` según la emoción detectada y se responde con 2–3 técnicas rápidas.


## 2) Componentes y arquitectura
- Frontend: `index.html` (HTML + Tailwind desde CDN, JS vanilla). Configura la URL de API en `API_URL`.
- Backend: `lambda_fuction.py` (handler `lambda_handler`), expuesto via API Gateway (método POST y OPTIONS).
- Base de datos: `consejos.db` (SQLite) con tablas `Usuarios` y `Consejos`. Se genera con `crear_bd.py`.

### Arquitectura Tecnológica

  | Capa | Tecnología | Versión | Justificación |
  |------|------------|---------|---------------|
  | Frontend | HTML5/CSS3/JS | - | Compatibilidad universal |
  | Backend | Python/Flask | 3.8+ | Desarrollo rápido |
  | Procesamiento | AWS Lambda | - | Escalabilidad automática |
  | IA | AWS Rekognition | - | Precisión sin entrenamiento |
  | Base de Datos | SQLite | 3.x | Simplicidad para MVP |

Flujo lógico (alto nivel):
1. Frontend envía `POST` a API Gateway → Lambda.
2. Lambda decodifica imagen (base64), llama a `Rekognition:DetectFaces`, determina emoción.
3. Lambda busca 3 consejos aleatorios por emoción en `SQLite` y responde JSON.
4. Frontend muestra emoción, confianza y las técnicas.
   
   ```mermaid
      graph TD
          A[CLIENTE<br>Browser] --> B[API GATEWAY<br>AWS]
          B --> C[LAMBDA<br>Python]
          C --> D[REKOGNITION<br>AWS]
          D --> E[SQLITE<br>Local]
      
          style A fill:#e1f5fe,stroke:#01579b,stroke-width:2px
          style B fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
          style C fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
          style D fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
          style E fill:#fce4ec,stroke:#c2185b,stroke-width:2px

## 3) Estructura del proyecto
```
.
├─ index.html              # Interfaz web (login/registro, consentimiento, captura/subida, resultados)
├─ lambda_fuction.py       # Backend AWS Lambda (analiza y consulta BD)
├─ crear_bd.py             # Genera/limpia y puebla la BD SQLite
└─ consejos.db             # Base de datos SQLite con consejos y un usuario de prueba
```


## 4) Base de datos
La base se genera con `crear_bd.py`. Este script:
- Elimina el archivo `consejos.db` previo si existe.
- Crea tablas:
  - `Usuarios(carnet TEXT PRIMARY KEY, password TEXT NOT NULL)`
  - `Consejos(id INTEGER PK AUTOINCREMENT, emocion TEXT NOT NULL, categoria TEXT, titulo TEXT NOT NULL, texto TEXT NOT NULL, duracion TEXT)`
- Inserta un usuario de prueba (`2023001` / `upana123`).
- Inserta consejos para emociones: FEAR, CALM, ANGRY, SAD, HAPPY, SURPRISED, NEUTRAL.

Comando para regenerar la BD en local (Windows PowerShell):
```powershell
cd "C:\Users\andrs\OneDrive\Escritorio\Lambda"
py -m venv .venv
.\.venv\Scripts\Activate.ps1
py crear_bd.py
```

Notas:
- Para producción en Lambda, el archivo `consejos.db` se empaqueta junto con el código y se copia a `/tmp/consejos.db` al inicio de la ejecución (Lambda solo permite escritura en `/tmp`).

### 4.1) ¿Por qué SQLite aquí? Comparativa rápida
- SQLite
  - Pros: cero gestión; sin red; rápido para lecturas; ideal para demos/embebido; portabilidad.
  - Contras: concurrencia de escritura limitada; no distribuida; gestión de backups manual.
- DynamoDB (alternativa NoSQL serverless)
  - Pros: escalado masivo, alta disponibilidad, latencia baja, gestión cero.
  - Contras: modelo de datos distinto (tablas/particiones), consultas por emoción requieren claves o índices; costo por RCU/WCU.
- RDS/Aurora (alternativa SQL gestionada)
  - Pros: SQL completo; transaccional; conexiones desde múltiples Lambdas.
  - Contras: costo base; gestión de conexiones (pooling) en serverless; red y latencia mayores.

Cuándo migrar: si necesitas multiusuario en producción, auditoría, o más escrituras/consultas complejas, considera DynamoDB (claves bien elegidas) o Postgres en RDS/Aurora.


## 5) API del backend (AWS Lambda + API Gateway)
Handler: `lambda_handler(event, context)` dentro de `lambda_fuction.py`.

- CORS: responde con `Access-Control-Allow-Origin: *`, `Content-Type`, y métodos `POST, OPTIONS`.
- Rutas: expuesto como único endpoint POST (y OPTIONS para preflight) mediante API Gateway.

### 5.1 OPTIONS (preflight)
- Petición: `OPTIONS /`
- Respuesta: `200` vacía con headers CORS.

### 5.2 POST — Registro de usuario
- Body JSON:
```json
{ "accion": "registro", "carnet": "2023xxx", "password": "tuPass" }
```
- Respuestas:
  - `200`: `{"autorizado": true, "mensaje": "Usuario creado"}`
  - `409`: `{"autorizado": false, "mensaje": "El usuario ya existe"}`
  - `500`: `{"autorizado": false, "mensaje": "Error al guardar"}`

### 5.3 POST — Login
- Body JSON:
```json
{ "accion": "login", "carnet": "2023xxx", "password": "tuPass" }
```
- Respuestas:
  - `200`: `{"autorizado": true, "mensaje": "Bienvenido"}`
  - `401`: `{"autorizado": false, "mensaje": "Credenciales incorrectas"}`

### 5.4 POST — Análisis de emoción (por defecto)
- Si `accion` no es `registro` ni `login`, el backend trata la solicitud como análisis.
- Body JSON (imagen en base64; data URL o solo base64):
```json
{ "image": "data:image/jpeg;base64,..." }
```
- Respuestas:
  - `200` con contenido:
```json
{
  "emocion": "ALEGRÍA | TRISTEZA | ENOJO | CALMA | SORPRESA | ANSIEDAD | ...",
  "confianza": "94%",
  "consejos": [
    {"titulo":"Respiración 4-7-8","texto":"Inhala...","duracion":"2 min"},
    {"titulo":"5-4-3-2-1","texto":"Identifica...","duracion":"3 min"}
  ]
}
```
  - `200` con lista vacía si no hay rostros (`"emocion": "No detectada"`)
  - `400` si falta `image`
  - `500` error interno

Mapeo de emociones (Lambda traduce de Rekognition):
```
FEAR→ANSIEDAD, CALM→CALMA, ANGRY→ENOJO, SAD→TRISTEZA, HAPPY→ALEGRÍA, SURPRISED→SORPRESA
```

Permisos IAM requeridos para la función:
- `rekognition:DetectFaces`
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

### 5.5) ¿Por qué AWS Rekognition? Alternativas y consideraciones
- AWS Rekognition
  - Pros: servicio gestionado, detección de emociones lista sin entrenar, integración nativa con AWS, buen SLA.
  - Contras: dependencia de proveedor; costo por uso; precisión sujeta a sesgos del modelo.
- Alternativas:
  - Google Cloud Vision / Azure Face API: capacidades similares; elegir según tu nube/herramientas actuales.
  - Modelos propios (OpenVINO/ONNX/TensorRT): control total y privacidad, pero requieren entrenamiento, MLOps y hosting.

Privacidad y ética: informa uso, no almacenes imágenes, cumple regulaciones locales (consentimiento, protección de datos).

### 5.6) Transporte de imágenes: base64 vs multipart/form‑data
- Base64 (actual)
  - Pros: fácil de enviar en JSON; simple en fetch; sin manejo de límites multi‑parte.
  - Contras: 33% de overhead en tamaño; payloads más grandes; límites de API Gateway se alcanzan antes.
- multipart/form‑data
  - Pros: menos overhead; mejor para archivos grandes.
  - Contras: manejo más complejo en Lambda (parsers), CORS y configuración adicional.

Recomendación: mantener base64 en POC; para producción con imágenes grandes, migrar a multipart o subir la imagen a S3 y pasar solo la URL/presigned URL.


## 6) Frontend (`index.html`)
- UI con pasos: Login/Registro → Consentimiento → Selección (Cámara/Subir) → Captura/Upload → Resultados.
- Configura la constante `API_URL` en el bloque `<script>`:
  ```js
  const API_URL = "https://<tu-api-id>.execute-api.<region>.amazonaws.com/<stage>/<ruta>";
  ```
- Eventos principales:
  - Registro/Login: `accion: 'registro' | 'login'` con `carnet` y `password`.
  - Cámara: usa `getUserMedia` para previsualizar y capturar imagen (canvas → base64).
  - Subida: arrastrar/soltar o seleccionar archivo; previsualiza y guarda base64.
  - Envío: `fetch(API_URL, { method: 'POST', body: JSON.stringify({ image }) })`.
  - Renderiza resultados con emoción, confianza y consejos.

### 6.2) Decisiones de frontend y comparativas
- Tailwind CSS por CDN
  - Pros: rapidez de prototipo, diseño consistente, sin build ni toolchain.
  - Contras: tamaño del CSS base; estilos en clases largas; para producción conviene purgar CSS.
  - Alternativas: Bootstrap (componentes listos, pero más opinión); CSS puro (más trabajo), frameworks (React+Next) si el proyecto crece.
- getUserMedia (cámara) vs solo subir archivo
  - getUserMedia: experiencia fluida y captura directa; requiere permisos del navegador y HTTPS.
  - Subir archivo: más simple y compatible; flujo menos inmediato.
- fetch nativo vs Axios
  - fetch: estándar, sin dependencias.
  - Axios: interceptores, timeouts por defecto y utilidades; añade peso.
- UI estática (un archivo) vs framework SPA
  - Estática: menor complejidad y costo; ideal para POC.
  - SPA (React/Vue): escalabilidad en estado/rutas, pero requiere build y hosting adicional.


## 6.1) Diagrama de flujo (alto nivel)

![Flujo alto nivel](assets/flujo-alto-nivel.png)

Generar/actualizar la imagen (Windows PowerShell, sin instalar globalmente):
```powershell
cd "C:\Users\andrs\OneDrive\Escritorio\Lambda"
npx -y @mermaid-js/mermaid-cli -i diagrams/flujo-alto-nivel.mmd -o assets/flujo-alto-nivel.png
# (Opcional) SVG de alta calidad:
npx -y @mermaid-js/mermaid-cli -i diagrams/flujo-alto-nivel.mmd -o assets/flujo-alto-nivel.svg
```


## 7) Despliegue en AWS Lambda
1. Crear un rol IAM con permisos de Rekognition y CloudWatch Logs.
2. Empaquetar en un ZIP: `lambda_fuction.py` y `consejos.db` en la raíz del paquete.
3. Crear función Lambda (runtime Python 3.x). Handler: `lambda_fuction.lambda_handler`.
4. Subir ZIP e indicar el handler anterior.
5. Crear API Gateway HTTP/REST, integrar con Lambda. Habilitar CORS (`POST, OPTIONS`).
6. Actualizar `API_URL` en `index.html` con la URL del API Gateway.

Notas:
- El código copia `consejos.db` desde el paquete a `/tmp/` en la primera invocación (para poder escribir si se crea nuevo usuario). `/tmp` está limitado a 512 MB.
- El nombre del archivo es `lambda_fuction.py` (tal cual). Si cambias el nombre a `lambda_function.py`, recuerda actualizar el handler en Lambda.

### 7.1) ¿Por qué Lambda + API Gateway frente a otras opciones?
- Frente a EC2/ECS: sin servidores que administrar, escalado y facturación por invocación; a cambio, límites de tiempo/espacio y patrón stateless.
- Frente a CloudFront Functions/Edge: Lambda aquí es más adecuada para CPU y dependencias (boto3) y tamaño de payload.
- Frente a Amplify/Backend completo: menor curva y coste inicial; si crece, considera arquitecturas modulares (S3+CloudFront para frontend, Lambda/API Gateway, DynamoDB).


## 8) Ejecución y pruebas locales (opcional)
- Para regenerar BD: ver sección 4.
- Para probar Rekognition en local, necesitas credenciales AWS configuradas (`aws configure`) y `boto3`. Un ejemplo mínimo:
```python
import boto3, base64
rekognition = boto3.client('rekognition', region_name='us-east-2')
with open('foto.jpg','rb') as f:
    img = f.read()
res = rekognition.detect_faces(Image={'Bytes': img}, Attributes=['ALL'])
print(res['FaceDetails'][0]['Emotions'])
```


## 9) Seguridad, privacidad y consideraciones
- Privacidad: el frontend declara y sigue un flujo de consentimiento; las imágenes se envían para análisis y no se guardan en el backend.
- CORS abierto (`*`) para facilitar pruebas; en producción, restringir orígenes.
- Credenciales: actualmente las contraseñas se almacenan en texto plano en `Usuarios` (solo para demo). Recomendado:
  - Hash de contraseñas (bcrypt/argon2) y sal.
  - Tokens (JWT) o sesiones si la app escala.
- Validación: reforzar validaciones de inputs y límites de tamaño de imagen.
- Observabilidad: agregar logs estructurados y métricas si se despliega a producción.

### 9.1) Manejo de contraseñas y autenticación (mejor práctica)
- Hashing recomendado: Argon2 (memoria‑duro) o bcrypt (ampliamente adoptado). Evitar texto plano.
- Sal aleatoria por usuario y factor de costo adecuado.
- Tokens de sesión: JWT con vencimiento corto y refresh; o cookies con SameSite/HttpOnly/Secure.
- Rate limiting y captcha para endpoints de autenticación si se expone públicamente.


## 10) Requisitos
- Python 3.10+ (para ejecutar `crear_bd.py` en local).
- AWS Account, IAM Role para Rekognition (en despliegue).
- Navegador moderno con acceso a cámara (si se usa modo cámara).


## 11) Roadmap (mejoras futuras)
- Hash de contraseñas y recuperación segura.
- Filtrado avanzado de consejos por categoría/tiempo/usuario.
- Internacionalización de UI y mensajes.
- Pruebas unitarias del handler y UI.
- Auditoría y métricas (CloudWatch, X-Ray).


## 12) Licencia
Sin licencia explícita. Define una según tus necesidades (por ejemplo, MIT).


## 13) Glosario de componentes y tecnologías

- Componentes del proyecto
  - `index.html` (Frontend): Interfaz web donde el usuario inicia sesión/registro, otorga consentimiento, elige cámara o subir imagen y ve resultados.
  - `lambda_fuction.py` (Backend): Función AWS Lambda que recibe la imagen, usa Rekognition para detectar emoción y consulta SQLite para devolver consejos.
  - `crear_bd.py` (Generador de BD): Script que crea y puebla `consejos.db` con tablas `Usuarios` y `Consejos`.
  - `consejos.db` (SQLite): Base de datos empaquetada con la Lambda; en ejecución se copia a `/tmp/consejos.db`.

- Frontend (`index.html`)
  - Tailwind CSS (CDN): Framework CSS utilitario para estilizar la UI sin proceso de build.
  - Google Fonts y Material Symbols: Tipografía y set de íconos para una UI clara y accesible.
  - `API_URL`: Constante con la URL de tu API Gateway; el frontend usa esta dirección para las peticiones.
  - Login/Registro: Envía `POST` con `{ accion: 'login'|'registro', carnet, password }`.
  - Consentimiento: Paso informativo que explica el uso de la cámara y privacidad.
  - Cámara (`navigator.mediaDevices.getUserMedia`): Accede a cámara, previsualiza, captura en `canvas` y convierte a base64.
  - Subida de imagen (`FileReader.readAsDataURL`): Lee un archivo elegido/arrastrado, lo muestra y lo guarda en base64.
  - Envío (`fetch`): Envía `{ image: <base64> }` por `POST` a la API y luego renderiza emoción + consejos.
  - Manejo de estados en UI: Muestra/oculta pasos, loaders y mensajes de error/éxito.
  - CORS: Permite llamadas desde el navegador; la Lambda responde con headers CORS.

- Backend (`lambda_fuction.py`)
  - Handler `lambda_handler`: Punto de entrada de la función.
  - CORS y `OPTIONS`: Responde a preflight con `200` y headers CORS para permitir llamadas desde el navegador.
  - Copia de BD a `/tmp`: Lambda solo permite escritura en `/tmp`; se copia `consejos.db` del paquete a `/tmp/consejos.db` si no existe.
  - Acción `registro`: Inserta un usuario nuevo en `Usuarios`; devuelve 200/409/500 según el caso.
  - Acción `login`: Verifica credenciales en `Usuarios`; devuelve 200/401.
  - Análisis (por defecto):
    - Decodifica base64 de la imagen.
    - Llama a `rekognition.detect_faces(..., Attributes=['ALL'])`.
    - Obtiene emoción dominante y confianza.
    - Consulta 3 consejos aleatorios de `Consejos` por emoción.
    - Mapea emociones a español y responde JSON.
  - Errores: Retorna `500` con mensaje genérico si algo falla.

- Base de datos (`crear_bd.py` y `consejos.db`)
  - Tablas:
    - `Usuarios(carnet, password)`: Para login/registro (demo: password en texto plano).
    - `Consejos(id, emocion, categoria, titulo, texto, duracion)`: Técnicas por emoción.
  - Seed:
    - Usuario de prueba `2023001 / upana123`.
    - Consejos para FEAR, CALM, ANGRY, SAD, HAPPY, SURPRISED, NEUTRAL.
  - Regeneración: El script borra `consejos.db` si existe, crea tablas y vuelve a insertar datos.

- Servicios y librerías AWS
  - AWS Lambda: Ejecuta el backend sin servidor. Handler: `lambda_fuction.lambda_handler`.
  - API Gateway: Endpoint HTTP que recibe `POST/OPTIONS` y reenvía a Lambda.
  - AWS Rekognition: `DetectFaces` para obtener emociones y su confianza.
  - `boto3`: SDK de AWS en Python para invocar Rekognition.

- APIs del navegador usadas
  - `navigator.mediaDevices.getUserMedia`: Acceso a cámara (requiere permiso del usuario).
  - `HTMLCanvasElement.toDataURL`: Convierte un frame capturado a imagen base64.
  - `FileReader.readAsDataURL`: Convierte archivos locales a base64.
  - `fetch`: Peticiones HTTP al backend desde el navegador.

- Rutas y variables clave
  - `API_URL`: Debe apuntar a tu API Gateway desplegada (ver sección 6).
  - `DB_PATH = '/tmp/consejos.db'`: Ruta de ejecución para la BD en Lambda.
  - `LAMBDA_TASK_ROOT`: Directorio del paquete desplegado en Lambda; de ahí se copia la BD a `/tmp`.

- Seguridad y privacidad
  - CORS: Abierto a `*` para pruebas; en producción, restringir orígenes permitidos.
  - Contraseñas: Actualmente en texto plano por ser demo; en producción usar hash (bcrypt/argon2) y sal.
  - Privacidad de imágenes: No se almacenan; se procesan para análisis y se descartan.
  - Límites de Lambda: `/tmp` limitado (~512 MB); API Gateway tiene límites de tamaño de payload.

- Decisiones de diseño (por qué estas tecnologías)
  - SQLite: Simple y portable para una demo; suficiente para consultas rápidas por emoción.
  - Rekognition: Servicio gestionado y preciso para emociones sin entrenar modelos propios.
  - `/tmp` en Lambda: Único lugar escribible, necesario para permitir nuevos registros de usuario.
  - Tailwind por CDN: Velocidad de prototipo sin pipeline de build.
  - Base64 en requests: Simplifica el envío de imágenes por JSON sin multipart/form-data.
  - `ORDER BY RANDOM() LIMIT 3`: Ofrece variedad de consejos en cada respuesta para la misma emoción.



