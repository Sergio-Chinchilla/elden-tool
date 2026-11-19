# Elden Tool

Guía interactiva y gratuita para **Elden Ring** que te dice qué hacer a continuación según *tu* partida: qué zonas explorar, qué bosses te tocan, qué tienes que hacer (y evitar) para conseguir el final que quieres, y cómo montar tu build.

> Proyecto de fans, de código abierto y sin ánimo de lucro. No está afiliado a FromSoftware ni a Bandai Namco. No usa imágenes, iconos, mapas, música ni otros recursos del juego.

**Versión 1.x:** juego base. **Versión 2.0:** *Shadow of the Erdtree* (DLC).

---

## ¿Qué puedes hacer con Elden Tool?

### 1. Registrar tu progreso

Marca lo que ya has hecho en tu partida y la guía se adapta:

- **Bosses** derrotados (principales y opcionales).
- **Eventos principales** de la historia (por ejemplo, quemar el Árbol Áureo en la Forja de los Gigantes).
- **Objetos clave** que abren o cierran caminos: Grandes Runas, Medallón de Rold, mitades del Medallón de Dectus y del de Haligtree, Anillo de Luna Oscura, etc.
- **Estado de los NPC**: vivo, muerto o misión fallida, y en qué paso de su misión estás.
- **Coleccionables**: Semillas Doradas y Lágrimas Sagradas que ya has recogido, con dónde encontrar las siguientes.
- Tu **nivel** y tus **atributos**.

No hace falta marcarlo todo: si indicas que has matado a Maliketh, la guía deduce que el Árbol ya está quemado y todo lo que eso implica.

### 2. Ruta sugerida

Una propuesta de **qué zonas explorar a continuación**, ordenadas según tu nivel y lo que tienes desbloqueado. Cada zona muestra:

- Nivel recomendado.
- Bosses que hay en ella y cuáles te faltan.
- Coleccionables y objetos importantes pendientes.
- Requisitos para acceder (por ejemplo, "necesitas dos Grandes Runas").

### 3. Próximos bosses

Lista de **los siguientes bosses a los que puedes enfrentarte**, con filtros por:

- Obligatorio u opcional.
- Región o zona.
- Rango de nivel recomendado.
- Accesible ahora / bloqueado / ya no accesible.

### 4. Avisos de bloqueo

Si algo **ya no es accesible** por cómo has avanzado, la guía lo dice claramente y explica por qué. Ejemplo: has matado a Maliketh e intentas explorar la Capital Real de Leyndell → *"Ya no puedes: la capital quedó en cenizas al quemar el Árbol Áureo."*

También hay una lista de **cosas que se pierden** si no las haces a tiempo, para que puedas revisarla antes de un punto de no retorno.

### 5. Guía de finales

Elige el **final que quieres** y la guía te muestra:

- ✅ **Lo que SÍ tienes que hacer**, empezando por los pasos más próximos.
- ⛔ **Lo que tienes que EVITAR** para no bloquearlo (matar a cierto NPC, avanzar más allá de cierto punto antes de completar una misión…), destacando los riesgos más cercanos.
- 🔒 Si el final **ya está bloqueado** porque has avanzado demasiado por otro camino, te lo indica y explica qué lo bloqueó.

Finales del juego base: Era de la Fractura, Era de las Estrellas, Era del Orden Perfecto, Era del Ocaso, Bendición de la Desesperación y Señor de la Llama Frenética.

### 6. Constructor de builds

- Catálogo de **armas, escudos, hechizos, encantamientos y cenizas de guerra**.
- Cada uno tiene **etiquetas** para filtrar:
  - *Mecánicas*: Rayo, Fuego, Magia, Sagrado, Sangrado, Congelación, Veneno, Podredumbre, Sueño, Locura, Muerte…
  - *Temáticas*: por ejemplo, el Chapitel de Helphen como "Muerte" (llama fantasmal) aunque su efecto sea congelación.
  - *De tipo y atributo*: espada recta, catalizador, FUE, DES, INT, FE, ARC…
- Combina etiquetas para builds **mixtas** (por ejemplo, Muerte + Hielo) y añade lo que quieras a tu build.
- Para cada arma o hechizo:
  - **Cómo conseguirlo**, con enlace a un vídeo y a la wiki.
  - **Atributos necesarios**.
  - **Escalado** (letras por atributo, y cómo cambia con cada afinidad).
- La build calcula las **stats mínimas** para usar todo lo que has elegido.
- **Builds predefinidas** para empezar (por ejemplo, "Rayo de fe", "Sangrado DES/ARC", "Hechicero de hielo").
- Comparte tu build con un enlace.

### 7. Guardar y cargar tu partida

- Tu progreso se **guarda automáticamente en tu navegador**.
- Puedes **descargarlo como fichero JSON** y **volver a cargarlo** en otro dispositivo o navegador.
- Más adelante: cuentas de usuario para sincronizar el progreso.

### 8. Otras características

- **Modo sin spoilers**: oculta nombres de bosses, zonas y eventos que todavía no has alcanzado.
- **Español** por defecto, con los nombres oficiales de la traducción del juego. **Inglés** como alternativa.
- Diseño pensado para el **móvil**, para consultarlo mientras juegas.

---

## Hoja de ruta

| Versión | Contenido |
| --- | --- |
| 1.0 | Progreso, ruta sugerida, próximos bosses, avisos de bloqueo, guía de finales, catálogo con etiquetas, constructor de builds, guardar/cargar JSON |
| 1.x | Calculadora de daño (AR) completa, más builds predefinidas, inglés |
| 2.0 | *Shadow of the Erdtree*: nuevas zonas, bosses, misiones, armas y la Bendición del Scaduárbol |
| 3.0 | Cuentas de usuario y sincronización |

## Usarlo

La web se publica en GitHub Pages: *(enlace pendiente)*.

## Contribuir

Las contribuciones son bienvenidas, sobre todo para revisar datos (requisitos, bloqueos, pasos de misiones) y añadir enlaces a vídeos. Los detalles técnicos, el modelo de datos y cómo ejecutar el proyecto están en [DOCUMENTATION.md](DOCUMENTATION.md).

## Licencia y créditos

- **Código:** [MIT](LICENSE).
- **Datos curados:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.es).
- Datos numéricos de objetos obtenidos con [erdb](https://github.com/EldenRingDatabase/erdb) (MIT).
- Referencia de la comunidad: [Elden Ring Wiki en wiki.gg](https://eldenring.wiki.gg/) (CC BY-SA 4.0).

*ELDEN RING* es una marca de FromSoftware, Inc. / Bandai Namco Entertainment Inc. Este proyecto no está afiliado a ninguna de ellas ni cuenta con su respaldo.
