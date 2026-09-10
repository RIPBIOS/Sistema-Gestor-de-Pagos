# Residencias Mejorado — README_REFACTORIZACION

## 1. Problemas del proyecto original
Duplicación masiva (~30 CRUD: CRUDSIC/CRUDIAM/CRUDTICS/... × regular/IR/ATR idénticos salvo código de carrera), SQL Injection en `login.php` (`"... WHERE matricula='$matricula'"`), CSS duplicado y colores hardcodeados, reportes dispersos (`generar_*_pdf.php` ×7, solo filtro carrera), escaneo PDF con lógica 1-pág vs 2-pág separada e inconsistente (Tesseract v2 vs v4, ZXing a medias), `conexion.php` con `die()` exponiendo errores, mezcla latin1/utf8mb4, passwords en texto plano `char(16)`.

## 2. Problemas de crudsic.php
Mezclaba POST/GET, SQL, HTML, JS (PDF.js+Tesseract inline ~300 líneas), modales y notificaciones en 551 líneas. La versión mejorada lo divide en: `controllers/EstudianteControlador.php` + `models/*` + `services/*` + `crud_estudiantes.php` (vista) + `assets/js/escaneo-linea-pago.js`.

## 3-4. Archivos creados / reorganizados
`config/database.php`, `config/constantes.php`, `models/EstudianteModelo.php`, `models/LineaPagoModelo.php`, `models/NotificacionModelo.php`, `models/ReporteModelo.php`, `controllers/EstudianteControlador.php`, `controllers/ReporteControlador.php`, `services/Seguridad.php`, `services/Registro.php`, `crud_estudiantes.php` (unificado), `views/reinscripciones/atrasados.php` (unifica todos los *ATR), `views/reportes/index.php`, `assets/css/estilos.css`, `assets/js/escaneo-linea-pago.js`, `index.php`, `login.php` (seguro), `database_mejoras.sql`. Se reutilizan `vendor/` y `fpdf/` originales. No se tocó `C:\xampp\htdocs\Residencias`.

## 5. Funcionalidades conservadas
Guardar línea 27 dígitos con verificación de duplicados en 7 tablas, listar con PDF (`JOIN pdf_files WHERE subidopdf=1`), excluir matrícula recién procesada, notificaciones (escaneo_fallido/linea_registrada/proceso_incompleto/personalizado/pago_confirmado) con borrado transaccional de PDF, reactivación +3 días, reportes PDF/Excel, login por matrícula.

## 6-7. Rendimiento / Seguridad
Paginación SQL (`LIMIT/OFFSET`), filtros en SQL (no en PHP), early-exit del OCR tras hallar el código, límite a páginas 1-2, preparadas en todo input `$_GET/$_POST`, normalización `lineaPago()`, `nombreArchivoSeguro()` + validación MIME real, `htmlspecialchars` en vistas, log técnico en `logs/error.log` con mensaje genérico al usuario, `session_regenerate_id`.

## 8-9. CRUD reutilizable y filtros
`crud_estudiantes.php?carrera=SIC&estatus=Si` — `carrera` validada contra `CARRERAS_VALIDAS`, `estatus` Si/No. Mismo archivo sirve SIC/TICS/IAMB/ICIV/IEME/IGEE/IIND/IQU/IMAT/IMCT/LADM. Filtros: carrera, estatus, matrícula (LIKE), paginación. Atrasados: mismos + periodo vía `fecha_permitida`.

## 10. Escaneo PDF
Tecnologías conservadas: PDF.js 3.11 + Tesseract.js v4 (las que funcionaban). Flujo: cargar PDF → render págs 1-2 (scale 2) → recorte (x4%,y25%,w47%,h5%) → binarización umbral 140 → escalado ×2 → OCR whitelist `0-9` psm 6 → normalizar (quita espacios: `970000 213034...` → `970000213034...`) → validar `/^\d{27}$/` → autollenar `#lineap`. Informa página donde se halló o "no encontrado". Sin falsos positivos (regex estricta).

## 11-13. Reportes (PDF/Excel)
`views/reportes/index.php`: módulos `estudiantes|regulares|irregulares|inscripciones|examenes|cuota|constancias|biblioteca`; filtros carrera/estatus/fecha inicio-fin/búsqueda; previsualización con conteo (100 filas muestra); descarga PDF vía FPDF y Excel vía PhpSpreadsheet (ya en `vendor/`).

## 14. Reinscripciones
`views/reinscripciones/atrasados.php` unifica todos los `CRUD*ATR.php` (misma query: `subido=FALSE AND subidopdf=0 AND (fecha_permitida IS NULL OR < NOW())`) + filtros carrera/estatus/matrícula + reactivar +3 días.

## 15-16. Ejecución / Dependencias
XAMPP (Apache+PHP 8.x+MariaDB). Importar `BASES DE DATOS/*.sql` originales (mydb, crud_php). Copiar carpeta a `htdocs/Residencias_Mejorado`, abrir `http://localhost/Residencias_Mejorado/`. Deps: FPDF (incluido), PhpSpreadsheet/phpmailer/pdfparser (en `vendor/`), CDN pdf.js + tesseract (requiere internet). Sin frameworks nuevos.

## 18. Apartados por rol (admin / alumno)
- ADMIN (`views/admin/panel.php`, requiere `$_SESSION['admin']`, equivale a `MENUCRUD.html`): sidebar con Reinscripciones Regulares/Irregulares por carrera (apuntan a `crud_estudiantes.php?carrera=XXX&estatus=Si|No`), Atrasadas unificado + por carrera (`views/reinscripciones/atrasados.php?carrera=XXX`) y Generación de reportes por módulo (`views/reportes/index.php?modulo=...`). Contenido en iframe como el original. `crud_estudiantes.php`, `atrasados.php` y `reportes/index.php` exigen sesión admin.
- ALUMNO (`views/alumno/panel.php`, requiere `$_SESSION['matricula']`, equivale a `SubidaArchivos.html`): sidebar con DATOS GENERALES (`datos_generales.php`, refactor de `estudiantess.php`: solo su matrícula, fecha_permitida, estado de trámites), Mis Notificaciones (`mis_notificaciones.php`, refactor de `notificaciones.php`) y grupos de trámites con modal AJAX -> `subir_archivo.php?tipo=`.
- Subida unificada (`models/ArchivoModelo.php` + `views/alumno/subir_archivo.php`): reemplaza los ~20 `upload_*.php`. Mapa real verificado por grep: reinscripcion->subidopdf, CEN/CEC/CSS/CRHA, ECCH/ECCHS/MB, CSEA/CSM, CECI/DEA/DEIM/DEMB/EP/EG/EHAI, MIC/MICA/CRS. Conserva reglas: 1 solo PDF de reinscripción, bloqueo por `fecha_permitida`, no sobreescritura, MIME PDF real.
- Login dual (`login.php?rol=admin|alumno` + `services/Auth.php`): admin usa credenciales originales admin/admin123 (se corrigió bug `OR`->`AND` de `validate.php`); alumno usa `usuarios` con preparada. `logout.php` cierra ambos.

## 19. Correcciones solicitadas (reportes idénticos, sesión, panel, layout)
- Reportes idénticos al original: `ReporteControlador` replica byte por byte el layout de `generar_*_pdf.php` (PDF FPDF L/A4, título Arial B 12 centrado, columnas 20/80/60/40/80 h=10 con borde; Excel título A1:E1 bold 14 centrado, headers bold/center/borde fino/relleno DDDDDD, datos desde fila 3, autosize A-E, mismos nombres `{tabla}_{fecha}.pdf/.xlsx`, misma consulta `SELECT * ... ORDER BY Carrera, Nombre`, mismo form POST `carreras[] + formato` con Bootstrap original). Se conserva incluso el nombre de archivo de cuota (`biblioteca_...`) como en `generarcuota_pdf.php`. Solo cambia que el módulo se elige con `?modulo=` en vez de 7 archivos.
- Sesión y flechas del navegador: `Auth::evitarCache()` (no-store/no-cache/Expires 0) en login, panels, CRUD, atrasados, reportes y alumno; `logout.php` destruye sesión + cookie; `login.php` redirige si ya hay sesión; `pageshow` recarga si viene de caché, así atrás no muestra pantallas con sesión cerrada sino que rebota a login.
- Botón Panel admin anidado: enlaces a panel/salir usan `target="_top"` (y `top.location.href` en botones), así dentro del iframe no se carga el panel dentro de sí mismo.
- CRUD fijo a pantalla completa: panels admin/alumno con `body overflow:hidden`, topbar fija 60px, `.layout{position:fixed;top:60px;bottom:0}`, sidebar fija con scroll propio e iframe `width/height 100%`, sin doble scroll: el CRUD ocupa todo el espacio de navegación.

## 20. CRUD unificado de trámites + login único
- Análisis previo: los CRUD de Biblioteca (ECCH/ECCHS/MB), Cuota (CSEA/CSM), Exámenes (CECI/DEA/DEIM/DEMB/EP/EG/EHAI) e Inscripciones (CIC) son el mismo flujo con distinta tabla `crud_php`, columna `subidoXXX` y título h2; Constancias no tiene CRUD admin en el original (títulos inferidos de `upload_constancias*.php`, documentado en `TRAMITES_CATALOGO`); `CRUDinscripciones1/CRUDModulos*` son legacy (guardan en `regulares`) y se unifican al flujo vigente `inscripciones/CIC`.
- Nuevo `crud_tramites.php?tramite=XXX` (17 trámites en `TRAMITES_CATALOGO`): replica listado `e.{col}=1 AND p.{col}=1`, guardado con `ON DUPLICATE KEY`, verificación UNION 7 tablas, notificaciones con sufijo original ("(Pago de biblioteca/examenes/cuota/constancias/inscripciones)"), borrado transaccional por columna y escaneo 27 dígitos. Conserva filtros carrera + estatus + matrícula.
- Reinscripciones (`crud_estudiantes.php?carrera=XXX&estatus=Si|No`): se eliminó el desplegable de estatus (era redundante dentro de cada sección Regular/Irregular); el estatus queda fijo por URL y se muestra como insignia + campo oculto. Se conserva filtro por carrera y matrícula.
- Login único: `index.php` redirige a `login.php` (un solo formulario usuario/matrícula + contraseña, sin pestañas admin/alumno; detecta admin primero y luego alumno). Recuperación conservada: `recuperar.php` (matrícula → confirmar identidad → inserta `solicitudes`, refactor con preparadas de `recuperar.php` + `cambiar_contraseña.php`) + `solicitar_recuperacion.php` (endpoint JSON original) + enlace "¿Olvidaste tu contraseña?" en el login.

## 21. Apartado SuperAdmin (SA) en el login único
- Acceso: mismo `login.php` con credenciales originales `SA`/`Superadmin` (antes validadas en JS de `index2.html`, ahora en servidor) → `views/superadmin/panel.php`. `index.php` también redirige sesión SA.
- Apartados idénticos al original, solo con topbar del CRUD de pagos: `estudiantes.php` (← `SUPADMINESTU.php`: alta con matrícula auto, editar, eliminar, listado), `fechas.php` (← `SUPADMINFECH.php`: fecha permitida por tipo/carrera), `passwords.php` (← `crud_password.php`: cambio + aviso por correo, misma config), `respaldo.php` + `generar_respaldo.php` (respaldo BD por fechas/completo). Solo se adaptó el `require` de conexión, la guardia `requerirSuperadmin()` y la topbar; `estiloSA.css` se copió tal cual. Ningún otro apartado fue modificado.

## 22. Fixes SuperAdmin (modal editar + paginación)
- Editar bloqueado (pantalla gris): `assets/css/estilos.css` define `.modal{z-index:50}` y se cargaba después de Bootstrap, cuyo `.modal-backdrop` usa z-index 1050; el fondo gris quedaba ENCIMA del modal (visible pero no clicable). Fix: en los 4 apartados SA se quitó el link a estilos.css y la topbar usa clase propia `.sa-topbar` con el mismo diseño verde, cero conflictos con Bootstrap. Ningún otro estilo/función cambió.
- Lista con paginación (`views/superadmin/estudiantes.php`): paginación en cliente, 15 por página, con Anterior/Siguiente, "Página X de Y" y "Mostrando A–B de N", integrada con la búsqueda en tiempo real existente (filtra y pagina sobre el resultado). Crear/editar/eliminar/buscar funcionan igual; diseño intacto.

## 23. Conexión de formularioins.html con mydb
- Faltantes detectados: `validar_estudiante.php` no existía (validación AJAX daba 404 y el botón Guardar quedaba deshabilitado) y `procesar_formularioinsc.php` pedía `database.php` inexistente (fatal al ejecutar). Fix sin alterar nada existente: se creó `validar_estudiante.php` (copia exacta del original, solo include adaptado) y `database.php` (shim hacia `config/database.php`, misma BD `mydb`). Formulario, procesador, correos y diseño intactos.

## 24. Comparador + CRUD único de reinscripciones con filtros arriba
- Comparador (`views/comparador/`): copia exacta de `comp23.html` (comparación PDF/Excel en cliente, exportación, notificaciones por coincidencias) + `send_notification_comp.php` (solo include adaptado a mydb + respuesta JSON si no hay sesión admin) + `estiloCOMP.css` e imágenes copiadas tal cual. Botón agregado en el panel admin. Sin cambios de elementos.
- Reinscripciones: lateral reducido a un solo CRUD (Regulares / Irregulares / Atrasadas, sin listas desplegables por carrera); la carrera se elige con los filtros existentes dentro de la página. Filtros movidos por encima del formulario en `crud_estudiantes.php` y `atrasados.php` (solo reorden de tarjetas, funciones intactas).

## 25. Filtros en trámites y reportes
- Inscripciones (`crud_tramites.php?tramite=CIC`): sin filtro de estatus (solo carrera + matrícula); el backend también ignora `?estatus=` en ese módulo. Resto de trámites conserva estatus. Filtros por encima del formulario en todos los CRUD de trámites.
- Reportes: agregado filtro de estatus (Todos/Regular/Irregular, mismo estilo Bootstrap) que filtra por `mydb.estudiantes.regular` vía subconsulta por matrícula; carrera sigue por checkboxes. Sin filtro seleccionado la salida es idéntica a la original (mismo layout, nombre de archivo y consulta).

## 17. Pendientes
Aplicar `database_mejoras.sql` en ventana de mantenimiento; migrar passwords a hash; eliminar columna inexistente `archivo_subido` en SubirPDF.php original; unificar `upload_*.php` restantes; agregar CSRF tokens; tests E2E con PDFs reales de 1/2 páginas.
