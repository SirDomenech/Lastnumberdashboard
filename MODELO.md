# El modelo — Índice de impactos ACB 26/27

Documento de referencia sobre cómo se calculan el rating de jugador, el rating de
equipo y los arquetipos que alimentan el dashboard. Es una traducción a lenguaje
llano de la lógica que vive en `backend/pipeline/29_valencia_dashboard_data.py`
(el script de producción vigente) — cualquier cambio de metodología debe
actualizarse primero ahí y luego reflejarse aquí.

Aviso editorial (el mismo que lleva el pie del dashboard): los pesos y bonus de
este modelo son decisiones editoriales de Last Number Data / N3XT Sports, no
coeficientes ajustados estadísticamente de forma exhaustiva. Se revisan en cada
corte de la temporada.

## 1. De dónde sale el rating de un jugador (0-100)

El cálculo tiene tres capas, en este orden:

1. **Producción propia (`rating_base`)** — percentil de producción individual
   de la 25/26 (o de la liga de origen, con traducción de nivel si no jugó en
   ACB), sin ningún ajuste de contexto. Es la métrica que se usa tal cual para
   valorar bajas (`baja_mas_sensible`).
2. **Ajustes de sistema y competición** — encima de `rating_base` se aplican,
   cuando corresponde: (a) un ajuste de sistema/entrenador de destino cuando
   hay evidencia suficiente (p. ej. el descuento a las 5 últimas jornadas de
   Real Madrid 25/26, ya con el título garantizado y el nivel competitivo más
   bajo); (b) un plus de Euroliga (`euroliga_ajuste`, guardado también como
   `rating_pre_euroliga`/`euroliga_nota`) para plantillas que juegan más
   competición de alto nivel; (c) penalización de competencia posicional
   cuando hay saturación de perfiles en el mismo puesto dentro del mismo
   equipo.
3. **Ecosistema (Q_liga / R_equipo / T_equipo)** — la capa final, pedida
   explícitamente por Miguel (24/09/2026): el rating de un jugador no debe
   medir solo "qué tan bueno es en aislado", sino combinar calidad individual
   con el contexto de equipo. Se calculan tres percentiles 0-100 y se
   combinan con pesos fijos:

   | Componente | Qué mide | Peso |
   |---|---|---|
   | `Q_liga` | Percentil de calidad **dentro de su posición** (Base/Alero/Pívot), entre los 18 equipos | 50% |
   | `R_equipo` | Percentil de su puesto en el ranking interno de su propio equipo (100 = el mejor de su plantilla) | 20% |
   | `T_equipo` | Percentil del equipo en el ranking de NetRtg proyectado 26/27 (100 = mejor equipo proyectado) | 30% |

   Los pesos (50/20/30) se confirmaron con Miguel tras revisar una simulación
   con jugadores reales de los tres escenarios extremos. Cita suya sobre la
   idea: *"si tú de mí esperas que sea el mejor del equipo, el mejor de la
   liga y que el equipo sea el mejor, deberías ponerme un 99. Pero si esperas
   de mí que sea top 3 de la liga, el mejor de mi equipo y el equipo top 10,
   puedes ponerme un 90."*

   Leyma Coruña y Monbus Obradoiro (ascendidos de Primera FEB) no tienen
   NetRtg real 25/26, así que no tienen `T_equipo` calculable: se les asigna
   un percentil neutro de 50 (ni castigo ni beneficio) con los mismos pesos
   50/20/30 que el resto de la liga, para que un ascendido no pueda escalar
   al top de la liga solo por calidad individual sin ranking probado en ACB.

   El resultado (`rating_0_100_raw`) se pasa por una compresión que evita que
   la escala se dispare por encima de 100 o por debajo de un suelo de 40
   puntos (ver "La escala tiene un suelo de 40" en el propio dashboard).

### El modelo de nivel de equipo (`T_equipo`)

Se probó si el rating medio de un roster (percentil ponderado por uso/AST%)
predice el NetRtg real del equipo, y no correlaciona (R²=0.00) — lógico,
porque el uso ofensivo es un recurso de suma cero dentro de un equipo, así que
promediarlo no puede reflejar la calidad colectiva. En su lugar se usa la
**suma** del Game Score por partido (GmSc_PJ, métrica aditiva de Hollinger) del
roster cualificado, que sí predice razonablemente el NetRtg real
(**R²=0.68, n=18, p<0.0001**). Ese es el modelo de regresión que proyecta el
NetRtg 26/27 de cada equipo y, con él, `T_equipo`.

### Ajustes manuales explícitos

Un puñado de jugadores tiene un ajuste manual documentado en el código,
siempre a pedido explícito de Miguel y siempre con la razón anotada junto al
override:

- **Suelo de rating** (`RATING_FLOOR_OVERRIDE`): un jugador no puede quedar
  por debajo de otro de referencia mientras se espera un dato más fiable
  (ej. Max Shulga vs. Mo Gueye).
- **Ajuste de bajas** (`BAJA_RATING_OVERRIDE`): cuando el box-score de ACB no
  capta bien a un jugador que no jugó con galones de titular pero es una
  referencia clara (ej. Mario Hezonja, MVP ACB 25/26 pero con producción
  "solo" de 75.4 en bruto por jugar en una rotación muy cargada; Jean
  Montero, producción excelente por minuto pero pocos minutos en Valencia).
- **Etiqueta "referencia"** (parte del fichero de arquetipos revisado por
  Miguel): un jugador etiquetado como referencia de su equipo se garantiza
  top-3 de rating dentro de su plantilla, con un ajuste que evita que dos
  "referencia" del mismo equipo se empujen entre sí sin converger.

Todos estos ajustes son trazables: cada jugador afectado lleva una nota
(`rating_floor_note`, `rating_override_note`, etc.) en `roster_dashboard_data.json`
explicando qué se cambió y por qué.

## 2. Los 6 atributos del radar hexagonal (por jugador)

Reutilizan las mismas referencias percentiles del rating, recombinadas para
lectura visual rápida — no son métricas nuevas:

| Atributo | Cómo se calcula |
|---|---|
| Anotación | media( percentil PPG, percentil eFG% ) — volumen + eficiencia |
| Creación de juego | media( percentil AST%, percentil APG ) |
| Cuidado de balón | percentil TOV% (invertido — menos pérdidas = mejor) |
| Defensa | media( percentil tapones por partido, percentil robos por partido ) |
| Rebote | percentil TRB% |
| Impacto | percentil de +/- por 40 minutos, dentro de su posición (sin dato para altas sin box-score en ACB) |

## 3. El rating de equipo (0-100) y sus 6 ejes

Pedido explícito de Miguel: sustituye el antiguo bonus/penalización continuo
de hasta ±8 puntos por NetRtg de equipo. Combina 6 atributos de plantilla,
cada uno la media ponderada por minutos esperados de los jugadores del
roster (los jugadores con más peso real cuentan más que uno de rotación
corta):

**Anotación · Cuidado de balón · Rebote · Defensa · Consistencia · Creación de juego**

- Los primeros 5 reutilizan los mismos atributos por jugador de la sección
  anterior, agregados a nivel de plantilla.
- **Consistencia** es la única excepción: mide la fiabilidad de la
  proyección de cada jugador (100 = máxima certeza), penalizada por señales
  de riesgo — alta desde fuera de ACB, muestra pequeña de partidos,
  lesión, debutante — y agregada igual que los demás ejes. No es varianza
  partido a partido, es "cuánto nos fiamos de este roster para 26/27".

El rating de equipo es la media simple de los 6 (primer corte, sin ponderar
ninguno por encima de otro — ajustable si Miguel quiere pesos distintos tras
ver el resultado en la práctica).

El ranking de equipos que se ve en el dashboard usa el **percentil** de cada
eje entre los 18 equipos (no el 0-100 crudo, que es una suma sin techo fijo),
tanto para el gráfico hexagonal como para identificar las "virtudes y
defectos" — los 2 ejes donde cada equipo está mejor/peor situado frente al
resto de la liga.

## 4. Roles y arquetipos

Los roles de cada jugador (Floor General, Primary Ball Handler, Scorer, Stretch
Big, etc.) parten de una clasificación automática por altura + perfil
estadístico (`12_classify.py`), con overrides manuales de posición para casos
donde el criterio puramente estadístico se equivoca (ej. un pívot con perfil
atípico). Miguel revisó y corrigió a mano el arquetipo de 223/224 jugadores del
roster 26/27 (fichero `Arquetipos_26-27_revision_Miguel.csv`); donde dio una
etiqueta válida, esta sustituye al rol calculado por el modelo. Esa misma
revisión trae las "etiquetas especiales" (líder, jugador referencia, etc.) que
alimentan ajustes de rating explícitos (ver sección 1) o marcas informativas en
el dashboard.

## 5. Posición (Base / Alero / Pívot)

Cada jugador tiene un `pos_group` (Guard/Forward/Center en los datos, mostrado
en español en el dashboard como Base/Alero/Pívot), derivado de la misma
clasificación de `12_classify.py`, con overrides manuales puntuales por
criterio real de baloncesto cuando la altura/posición registrada no refleja
cómo juega realmente el jugador. El filtro de posición del ranking de
jugadores y el comparador de atributos (que solo compara jugadores del mismo
puesto) usan este mismo campo.

## 6. Qué falta / próximos pasos

Ver la pestaña "Evolución" del dashboard para la hoja de ruta de versiones del
modelo (v1 preseason → v5 tras la J34). Pendiente explícito, documentado en el
propio pipeline: los jugadores que continúan en la plantilla ascendida de
Leyma Coruña / Monbus Obradoiro (procedentes de Primera FEB) no están cubiertos
todavía — exigiría investigar su posición/altura y una referencia de nivel
propia de Primera FEB.

## 7. Dónde mirar en el código

Todo lo de arriba vive comentado in situ en
`backend/pipeline/29_valencia_dashboard_data.py`, sección por sección (busca
los bloques `# ====...====`). Los scripts `01_...` a `28_...` son iteraciones
anteriores del modelo, ya superadas — se conservan como historial de cómo se
llegó hasta aquí, pero no hace falta ejecutarlos para reproducir el dashboard
actual. Ver `README.md` para el orden de ejecución y cómo regenerar los datos.
