# LearnDash Video Auditor

> Análisis del plugin a partir del código fuente (3 clases PHP, 3 módulos JS,
> 2 hojas CSS y 1 vista).
> Versión analizada: **2.8.3** — 16 de octubre de 2026.
> Ubicación: `wp-content/plugins/learndash-video-auditor/`

---

## 1. Ficha técnica

| Dato | Valor |
|---|---|
| **Nombre** | LearnDash Video Auditor |
| **Versión** | 2.8.3 |
| **Autor** | Codex |
| **Text domain** | `ld-video-auditor` |
| **Requisitos** | WordPress 5.8+, PHP 7.4+, LearnDash (`sfwd-lms`) |
| **Dependencia opcional** | Extensión PHP `ZipArchive` (solo para exportar XLSX) |
| **Ubicación en el menú** | LearnDash LMS → Auditor de videos |
| **Capacidad requerida** | `manage_options` |
| **Tamaño** | ~3.150 líneas |

## 2. Qué resuelve

En una instalación de LearnDash con muchos cursos, el video de cada lección
puede estar guardado en **cinco sitios distintos**: en `post_content`, en el
campo *Video Progression* de LearnDash (`_sfwd-lessons`), dentro de metadatos
serializados de otros plugins o page builders, como iframe pegado a mano, o
como shortcode. No hay forma nativa de responder "¿qué lecciones tienen video y
cuál?" sin abrirlas una por una.

El plugin hace tres cosas sobre ese problema:

1. **Audita** — escanea un curso completo y lista, lección por lección, qué
   video tiene y de dónde salió.
2. **Edita** — permite cambiar o quitar el video desde la propia tabla, sin
   entrar a la lección.
3. **Controla la reproducción** — para videos de Bunny Stream, firma los embeds
   con token y bloquea "Marcar completado" hasta que el alumno termine de ver el
   video.

## 3. Arquitectura

Tres clases, sin capa de abstracción intermedia. El bootstrap las carga en orden
y arranca un único singleton en `plugins_loaded`:

| Clase | Responsabilidad |
|---|---|
| `LDVA_Scanner` | Detección y normalización. Recorre cursos, lecciones y fuentes de contenido; extrae videos; genera la *clave canónica* de cada uno. Sin estado ni efectos secundarios. |
| `LDVA_Admin` | Todo lo demás: menú, assets, 6 endpoints AJAX, exportación, firma de tokens, filtros de `the_content` y el gate de completado. Singleton. |
| `LDVA_Exporter` | Filtrado y generación de CSV y XLSX. Escribe el XLSX a mano (ZIP + XML OOXML), sin librerías. |

**Frontend**: JavaScript puro, sin jQuery. `admin.js` para el panel y
`bunny-progression.js` para la lección, este último apoyado en el `playerjs`
oficial de Bunny cargado desde su CDN.

## 4. El concepto central: la clave canónica

Casi todo el plugin gira alrededor de `LDVA_Scanner::video_key()`, que reduce
cualquier referencia a un video a una cadena del tipo `youtube:dqw4w9wgxcq`,
`vimeo:123456789` o `bunny stream:4321/a1b2c3…`.

Esto resuelve que un mismo video aparezca en formas distintas.
`youtu.be/ABC`, `youtube.com/watch?v=ABC` y un
`<iframe src="youtube.com/embed/ABC">` producen la misma clave, así que:

- **No se duplican filas**: las tres formas se agrupan en un solo registro que
  conserva la URL original *y* el código embebido.
- **La edición acierta aunque el HTML haya cambiado**: si el iframe original
  tiene otros atributos o entidades, `replace_video_by_id()` lo localiza por el
  ID y no por coincidencia literal.
- **El borrado es completo**: barre `post_content`, la configuración de
  LearnDash, las metas propias del plugin y cualquier otra meta —incluso
  serializada o anidada en objetos— que contenga ese ID.

Se escoge además la URL "preferida" para mostrar: `vimeo.com` antes que
`player.vimeo.com`, `youtu.be` o `watch?v=` antes que `embed/`.

## 5. Detección

`scan_lesson()` reúne como fuentes el `post_content` **más todos los metadatos
de la lección**, aplanando arrays y objetos serializados recursivamente. Sobre
cada fuente busca iframes y URLs sueltas. Si hay un shortcode guardado, también
lo **ejecuta** (`do_shortcode`) y escanea su salida.

Plataformas reconocidas por host:

| Plataforma | Hosts |
|---|---|
| YouTube | `youtube.com`, `youtu.be`, `youtube-nocookie.com` |
| Vimeo | `vimeo.com` (y subdominios) |
| Bunny Stream | `player.mediadelivery.net`, `iframe.mediadelivery.net`, `dash.bunny.net` |

Para enumerar las lecciones de un curso usa una cascada de tres estrategias:
`learndash_course_get_steps_by_type()`, luego `learndash_get_course_steps()`, y
como último recurso un barrido de todas las lecciones comprobando pertenencia
por `course_id`, por `_sfwd-lessons` o por `learndash_get_course_id()`. Incluye
borradores, privadas y pendientes, no solo publicadas.

## 6. Pantalla de administración

**Filtros**: curso (dispara el escaneo AJAX), buscador libre, plataforma y
estado (Encontrado / No encontrado). Las lecciones sin video aparecen
igualmente, marcadas como "No encontrado" — que es justamente lo que se suele
estar buscando.

**Tabla**: curso, lección, plataforma, URL original, HTML embebido, shortcode,
estado y un menú de Acciones con *Ver lección*, *Añadir/Editar video* y
*Remover video*. Las celdas largas se truncan con botón de copiado.

**Exportación**, tres variantes:

- CSV del curso actual, respetando los filtros aplicados.
- Excel del curso actual, igual.
- **Excel de todos los cursos** — recorre el sitio entero y exporta solo las
  lecciones que sí tienen video, con 4 columnas (curso, lección, plataforma,
  enlace).

El XLSX se arma manualmente como ZIP de XML: trae fila de encabezado en negrita,
panel congelado y autofiltro.

La pantalla **oculta los avisos de otros plugins**
(`remove_all_actions('admin_notices')`) solo en esta página, vía
`current_screen`.

## 7. Edición y borrado

El modal acepta tres entradas —URL, HTML embebido y shortcode— con precedencia:
shortcode > iframe > URL.

Al guardar, el plugin decide **dónde** escribir según dónde estaba el video:

- Si lo encuentra en `post_content`, lo reemplaza ahí mismo (por coincidencia
  literal o, si falla, por ID).
- Si el video que se edita es el de *Video Progression* de LearnDash, actualiza
  `_sfwd-lessons` y llama a `learndash_update_setting()`.
- Si no está en ninguno de los dos, lo guarda en sus propias metas
  (`_ldva_video_url`, `_ldva_video_embed`, `_ldva_video_shortcode`), que luego
  se inyectan al final de la lección vía `the_content`.

**Regla especial para Bunny** (desde la 2.8.0): cualquier video de Bunny se
guarda **siempre** como shortcode `[bunnycdn_stream_video]`, nunca como iframe
suelto. Si no logra derivar biblioteca e ID válidos, devuelve error en vez de
guardar un iframe sin firmar. El motivo es que solo el shortcode recalcula el
token en cada visita; un iframe estático quedaría firmado con una fecha fija y
expiraría.

Todo lo que entra pasa por `wp_kses()` con una lista blanca estricta de
atributos de `<iframe>` y `<div>`.

## 8. Bunny Stream: firma por token

Si una biblioteca de Bunny tiene activado *Embed View Token Authentication*, los
videos no reproducen sin firmar la URL. El plugin añade una sección donde se
guarda, **por biblioteca**, la clave de seguridad y un TTL (con casilla "No
expira" que en realidad fija 10 años, porque Bunny siempre exige fecha de
vencimiento).

La firma es `SHA256(clave + video_id + expires)` y se aplica en dos momentos:

1. **Al renderizar el shortcode** — el plugin registra `bunnycdn_stream_video`
   solo si no existe ya, como respaldo del plugin oficial de BunnyCDN.
2. **En una pasada final sobre el HTML ya ensamblado** (`resign_bunny_embeds`,
   en `the_content` prioridad 30) — reescribe *cualquier* URL de embed de Bunny
   que aparezca, venga de donde venga: del contenido, de una meta, de versiones
   antiguas del plugin o del shortcode. Descarta el `token`/`expires` previo y
   vuelve a firmar con la hora de la visita.

El atributo `tokenAuth="false"` desactiva la firma para un video puntual.

## 9. Control de progresión ("no avanzas sin ver el video")

Es la función más delicada, con **doble validación**:

**En el navegador** (`bunny-progression.js`): detecta los iframes de Bunny, se
engancha al evento `ended` del `playerjs` oficial, deshabilita el botón de
"Marcar completado" y muestra un aviso bajo el video. Al terminar cada video lo
reporta por AJAX. El estado inicial viene del servidor, no de `localStorage`.

**En el servidor** (`gate_mark_complete`, filtro
`learndash_process_mark_complete`): devuelve `false` si falta algún video por
ver, de modo que **no se puede eludir desde el navegador**. El endpoint que
registra el avance valida nonce, sesión, y que la clave reportada pertenezca
realmente a esa lección — no se pueden forjar claves.

Detalles de diseño que importan:

- `bunny_video_keys()` solo cuenta los videos **efectivamente visibles** en la
  lección. Sobre-incluir bloquearía la lección de forma permanente.
- Los administradores omiten la barrera.
- Se puede desactivar con
  `add_filter('ldva_enforce_bunny_progression', '__return_false')`.
- El avance se guarda por usuario y lección en `_ldva_bunny_watched_{lesson_id}`.

## 10. Datos que almacena

| Clave | Dónde | Contenido |
|---|---|---|
| `_ldva_video_url` | postmeta | URL gestionada por el plugin |
| `_ldva_video_embed` | postmeta | Iframe gestionado por el plugin |
| `_ldva_video_shortcode` | postmeta | Shortcode gestionado por el plugin |
| `_ldva_bunny_watched_{id}` | usermeta | Videos Bunny ya vistos por ese usuario |
| `ldva_bunny_tokens` | option | Claves de token por biblioteca (`autoload` desactivado) |

`uninstall.php` borra las cinco.

## 11. Seguridad

- Los 6 endpoints AJAX de administración exigen `manage_options` + nonce
  `ldva_nonce`; la exportación usa `check_admin_referer` con su propio nonce.
- El endpoint de progresión es el único accesible a usuarios normales, con nonce
  propio (`ldva_progression`) y validación de pertenencia del video a la
  lección.
- `wp_kses()` con lista blanca en todo HTML entrante.
- **Protección contra inyección de fórmulas** en CSV: prefija con comilla simple
  los valores que empiecen por `=`, `+`, `-`, `@`, tab o CR.
- Las claves de token se guardan con `autoload = false`.

## 12. Historial de correcciones críticas

El changelog registra tres fallos graves corregidos en versiones consecutivas,
todos en la misma zona del código, que vale la pena conocer si se toca:

- **2.8.1** — `preg_replace_callback()` devuelve `null` cuando PCRE agota
  `pcre.backtrack_limit` (habitual en lecciones hechas con Elementor, por el
  tamaño del contenido). Como el método estaba tipado para devolver `string`,
  provocaba un error fatal en *cualquier* lección con video de Bunny. Ahora, si
  el refirmado falla, la lección se muestra sin refirmar ese video en vez de
  romperse.
- **2.8.2** — `gate_mark_complete()` exigía 4 parámetros, pero según la ruta
  interna LearnDash dispara el filtro con solo 3. En PHP 8+ eso era un
  `ArgumentCountError` fatal justo al pulsar "Marcar como completado".
- **2.8.3** — ese mismo método asumía que el tercer parámetro llegaba como ID
  numérico; cuando LearnDash lo entrega como `WP_User`, el cast daba `1` y el
  gate evaluaba el progreso del usuario 1 en lugar del real, bloqueando la
  lección en silencio.

## 13. Limitaciones conocidas

- **Elementor** — la firma de tokens opera sobre `the_content`. Una lección
  armada íntegramente en Elementor guarda su contenido en `_elementor_data`, así
  que un iframe allí puede no pasar por ese filtro y quedar sin firmar. Está
  documentado explícitamente en el changelog.
- **Escaneo sin caché** — cada selección de curso vuelve a escanear todas sus
  lecciones y a ejecutar sus shortcodes. El Excel global repite eso para el
  sitio entero en una sola petición; en instalaciones grandes puede acercarse al
  tiempo límite de ejecución.
- **XLSX sin estilos** — el generador propio produce un archivo funcional
  (negrita, panel congelado, autofiltro) pero sin colores ni formato, a
  diferencia del que usa el plugin "Administrador de Usuarios (WMS Latam)".
- **`playerjs` desde CDN externo** — sin conexión a
  `assets.mediadelivery.net`, no hay señal de finalización y el botón queda
  bloqueado; es la decisión deliberada (el servidor también lo exige), pero
  conviene saberlo.
