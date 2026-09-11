# Frontera entre equipos

> **Para quien lee (persona o IA):** reglas para cuando tu código es una pieza de un sistema
> que otro equipo termina (un front, un cliente, otro servicio). Cada una lleva 💡 la lección
> que la originó. Si sos una IA y te mandaron acá: verificá cuáles de estas están cubiertas
> en el repo donde estás trabajando y decí cuáles faltan. Origen: un back consumido por un
> front de otro equipo, 2026.

**La idea de fondo:** en la segunda mitad de ese proyecto, las fallas más dañinas para el
usuario **no eran del back**: venían del otro lado del contrato. Eso se diseña desde el día
uno o se descubre después de semanas de arreglar el lado equivocado.

## 1. Loggear lo que llega, no solo lo que se responde

Guardar el payload **recibido** en cada request (o un resumen fiel: qué campos vinieron, con
qué tamaño, vacíos o no).

💡 Se midió una falla que parecía propia; el log del payload mostró que 2 casos eran nuestros
y 22 del cliente, que en una ruta mandaba un campo vacío. Sin ese log, la falla se arreglaba
del lado equivocado.

## 2. Una clave de correlación compartida, desde el primer día

Un identificador de request que viaje en ambas direcciones y quede en los logs de los dos
lados.

💡 Costó semanas ponerlo después; cuesta una línea ponerlo antes. Al cruzarlo apareció el
cuello real: no era técnico, era la tasa de feedback de los usuarios (3%).

## 3. El contrato es un archivo versionado, no un chat

Rutas, campos, formatos, quién manda qué, qué pasa si falta. En el repo, con fecha.

💡 Cada acuerdo verbal se rompió sin que nadie lo notara. Lo que estaba escrito se pudo
señalar; lo que no, se re-discutió.

## 4. Una prueba obligatoria del otro lado por cada ruta

Pedirle al otro equipo una prueba automática que falle si una ruta deja de mandar lo pactado.
Tu código no puede compensar lo que no recibe.

💡 La ruta que mandaba el campo vacío era nueva; nadie la probó contra el contrato.

## 5. Un solo punto de conversión de formato

Si un identificador tiene forma "cruda" de un lado y normalizada del otro, la conversión
vive en **un** lugar, documentado. Nunca "arreglar" el formato en más de un sitio.

💡 Un carácter especial en una URL se codificaba en un lado y se perdía en otro; se
"arregló" tres veces en tres lugares antes de unificarlo.

## 6. Conocer el visor antes de diseñar la salida

Qué renderiza y qué no (tablas, negritas, enlaces), y una prueba que lo verifique.

💡 El visor no soportaba tablas; las respuestas con tabla se veían como texto roto durante
semanas hasta que un usuario lo dijo.

## 7. El timeout del otro lado es tu límite real

Lo que tarde más que el timeout del cliente **se ve como caída**, aunque termine bien. Lo que
puede tardar más se pre-calienta o se responde en dos pasos; no se optimiza el segundo 61.

💡 "No responde" era un timeout de 60 s del front sobre un proceso que terminaba a los 70.

## 8. Qué señal del usuario sirve

- Feedback negativo **con texto**: la única señal diagnóstica.
- Feedback positivo genérico: no es calidad, es "sonó bien". No usarlo como métrica.
- La tasa de feedback es el techo de todo análisis por usuario: antes de pedir más
  telemetría, pedir más feedback.

💡 Los pulgares arriba en resúmenes no correlacionaban con nada verificable.

## 9. Recursos compartidos se negocian, no se asumen

Si el otro equipo usa la misma GPU, base o cola, su pico es tu timeout. Cualquier medición
"tuya" incluye la de ellos.

💡 El verificador del cliente usaba el mismo generador; el consumo de GPU nunca fue solo
nuestro.
