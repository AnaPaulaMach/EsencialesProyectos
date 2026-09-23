# Frontera entre equipos

**Alcance:** cuando tu código es una pieza de un sistema que otro equipo termina (un front,
un cliente, otro servicio).

**La idea de fondo:** cuando el sistema madura, muchas de las fallas más dañinas para el
usuario **no son del back**: vienen del otro lado del contrato. Eso se diseña desde el día
uno o se descubre después de semanas de arreglar el lado equivocado.

## 1. Loggear lo que llega, no solo lo que se responde

Guardar el payload **recibido** en cada request (o un resumen fiel: qué campos vinieron, con
qué tamaño, vacíos o no).

💡 Una falla que parece propia puede ser un campo que el cliente manda vacío en una ruta. Sin
el log de lo recibido, se arregla del lado equivocado.

## 2. Una clave de correlación compartida, desde el primer día

Un identificador de request que viaje en ambas direcciones y quede en los logs de los dos
lados.

💡 Ponerlo después cuesta semanas; antes, una línea. Y cruzar los dos lados suele mostrar
cuellos que ninguno ve solo (por ejemplo, que casi nadie deja feedback).

## 3. El contrato es un archivo versionado, no un chat

Rutas, campos, formatos, quién manda qué, qué pasa si falta. En el repo, con fecha.

## 4. Una prueba obligatoria del otro lado por cada ruta

Pedirle al otro equipo una prueba automática que falle si una ruta deja de mandar lo pactado.
Tu código no puede compensar lo que no recibe.

## 5. Un solo punto de conversión de formato

Si un identificador tiene forma "cruda" de un lado y normalizada del otro, la conversión
vive en **un** lugar, documentado. Nunca "arreglar" el formato en más de un sitio.

💡 Un carácter especial en una URL (`#`, `%`) que se codifica en un lado y se pierde en otro
termina "arreglado" en varios lugares que se contradicen.

## 6. Conocer el visor antes de diseñar la salida

Qué renderiza y qué no (tablas, negritas, enlaces), y una prueba que lo verifique.

💡 Si el visor no soporta tablas, una respuesta con tabla se ve como texto roto, y nadie del
equipo lo nota: lo nota el usuario.

## 7. El timeout del otro lado es tu límite real

Lo que tarde más que el timeout del cliente **se ve como caída**, aunque termine bien. Lo que
puede tardar más se pre-calienta o se responde en dos pasos; no se optimiza el segundo 61.

💡 "No responde" suele ser un timeout del cliente sobre un proceso que termina bien, unos
segundos tarde.

## 8. Qué señal del usuario sirve

- Feedback negativo **con texto**: la única señal diagnóstica.
- Feedback positivo genérico: no es calidad, es "sonó bien". No usarlo como métrica.
- La tasa de feedback es el techo de todo análisis por usuario: antes de pedir más
  telemetría, pedir más feedback.

## 9. Recursos compartidos se negocian, no se asumen

Si el otro equipo usa la misma GPU, base o cola, su pico es tu timeout. Cualquier medición
"tuya" incluye la de ellos.

## 10. Tu log solo ve lo que el otro dejó pasar

Si el otro lado resuelve o reescribe parte de los casos antes de mandarlos, tu log no tiene el
**denominador**. La tasa real se mide sobre el log del otro lado (o cruzando los dos por la
clave del §2).

💡 Si el front transforma parte de los casos antes de enviarlos, tu log nunca los ve y la tasa
que medís es de otra población.

## 11. Mantené tu red aunque el otro frene hoy

Que el otro equipo filtre un caso malo **hoy** no te exime de la guarda propia: su filtro
cambia con su próximo deploy y vos no te enterás. La tuya es barata si es angosta y medida.

💡 Mientras el filtro del otro lado no exista (o cambie), la guarda propia es lo único que frena
esos casos.
