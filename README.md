- Resolver de forma grupal cada ejercicio.
- Escribir las soluciones en 2 issues separados; cuyos títulos deberán nomenclarse como “2025-MA-GRUPO-XX Ejercicio X”
- Escribir en el issue los nombres de los participantes


# Ejercicio 1 

Nos llamaron para mantener el sistema del minimercado Cilindro, un mercado familiar que cuenta con una única sucursal por el momento. Una de las funcionalidades más importantes del sistema es poder calcular, al final de cada mes, el monto total vendido, ya que esto ayuda a los propietarios a tomar decisiones financieras.

El modelo de datos del sistema es el siguiente:

<img width="658" height="636" alt="image" src="https://github.com/user-attachments/assets/690b1338-df2b-4f98-8843-f68146663343" />


El sistema muestra en una pantalla el monto total recaudado en cada mes desde que se instaló el sistema hasta la fecha. Para eso, ejecuta la siguiente consulta cada vez que alguien quiere acceder a los resultados:


```SQL
 SELECT 
  MONTH(v.fecha_hora) AS mes, 
  YEAR(v.fecha_hora) AS anio, 
  SUM(p.precio * iv.unidades_vendidas) AS total_x_mes
FROM venta v
INNER JOIN item_venta iv ON iv.venta_id = v.id
INNER JOIN producto p ON p.id = iv.producto_id
GROUP BY mes, anio;

```

Durante los primeros meses post instalación, el sistema funciona correctamente. Sin embargo, en un mes en particular se decidió reducir el precio de algunos productos por una promoción durante los dos últimos días del mes. Al calcular el monto total vendido de ese mes, los propietarios notaron que el monto era sorpresivamente bajo y muy distinto al que esperaban.

Luego, al revisar el monto de meses anteriores, afirmaron que estos también eran distintos a los que habían visto la primera vez que se calcularon. Debido a esta sospecha, los propietarios compararán venta por venta contra su libro contable para verificar si hubo un error del sistema.


#### ¿Qué cree que puede estar sucediendo? ¿Tienen razón los propietarios o el sistema funciona como debe?

#### En caso de haber un problema, enuncie de cuál se trata y explique detalladamente cómo lo solucionaría.


------



# Ejercicio 2

Nos llamaron para realizar una reversión de un famoso juego online, pero que no sea *pay to win*.

**DDSI Battle Arena** es un juego de estrategia en tiempo real donde los jugadores compiten en batallas utilizando mazos de cartas, las cuales pueden ser tropas, hechizos o estructuras en un campo de batalla. El objetivo principal es destruir las torres del oponente y ganar trofeos, que representan el progreso competitivo del jugador.

Nuestro equipo detectó las siguientes entidades clave:
- **Jugador**: Cada usuario del juego tiene un nombre, nivel, cantidad de trofeos y puede pertenecer a un clan, una agrupación social que permite cooperar y competir colectivamente.
- **Clan**: Agrupa jugadores y les permite participar en guerras de clanes y otros eventos. Se almacena su nombre, nivel, entre otros.
- **Carta**: Representa cada una de las unidades, hechizos o estructuras que los jugadores pueden usar. Las cartas tienen tipos de rareza y demás datos como el daño por segundo, etc.
- **Mazo**: Cada jugador puede armar hasta 4 mazos con hasta 8 cartas para usarlos en batalla.
- **Partida**: Registra un enfrentamiento entre jugadores, incluyendo información como los mazos utilizados, el/los ganadores, el modo de juego y la duración.

#### Arquitectura y funcionalidades iniciales

El sistema tendrá una arquitectura distribuida. Nuestro equipo se encargará de un servicio que gestiona a los jugadores, clanes y mazos, además de recopilar información de las partidas ya jugadas. Las partidas en tiempo real se gestionarán en otro servicio, el cual nos enviará una solicitud POST al finalizar la partida con los datos correspondientes.

#### Modelo de datos

Para una primera versión, se plantea el siguiente modelo de datos:

<img width="855" height="742" alt="image" src="https://github.com/user-attachments/assets/28b961a5-9b87-4cd4-be5c-cc0d3ea4e764" />

Como estamos en una primera versión se tomaron las siguientes **decisiones** con el fin de no complejizar el modelo en esta etapa y sacar algo funcional al mercado rápidamente:

- Un jugador puede usar únicamente uno de sus mazos al jugar una partida.
- Un mazo pertenece únicamente a un jugador. Si otro jugador tiene un mazo con las mismas cartas, se representará como dos filas distintas en la tabla.
- Si un jugador quiere editar un mazo, se creará un nuevo mazo y se marcará como activo, mientras que el mazo editado se desactiva.

#### Leaderboards

Se deberán implementar distintos leaderboards visibles dentro del juego, con actualización frecuente para reflejar métricas de desempeño de jugadores, clanes, cartas y modos de juego:

Los leaderboards solicitados son:

- Top 10 cartas más jugadas de la semana: ranking que muestre las cartas con mayor número de partidas jugadas en la semana en curso.

- Top 5 cartas con mayor cantidad de victorias de la semana: ranking que destaque las cartas con mayor número de victorias en la semana.

- Modo de juego más jugado del mes: leaderboard que muestre el modo de juego con mayor número de partidas en el mes actual.

- Top 50 clanes con más trofeos ganados en el mes: ranking acumulado por clan, sumando los trofeos ganados por todos los integrantes durante el mes.

- Top 200 jugadores con más victorias en el mes: leaderboard global que muestre los jugadores con mayor número de victorias acumuladas en el mes.

Estos leaderboards deben ser accesibles desde la interfaz del juego, con actualizaciones regulares que reflejen la evolución del juego en tiempo casi real.

En un principio, se decidió resolver los leaderboards ejecutando queries agregadas SQL en un CronTask cada 5 minutos, guardando los resultados en una tabla. Por ejemplo, la query para los jugadores con más victorias:

```SQL 
-- Parámetros
-- @mes = número de mes (1 a 12)
-- @anio = año (ej: 2025)
-- @limite = cantidad de jugadores a mostrar

SELECT 
    j.id AS jugador_id,
    j.nombre AS jugador,
    COUNT(*) AS victorias
FROM mazo_partida mp
INNER JOIN partida_jugada pj ON mp.partida_id = pj.id
INNER JOIN mazo m ON mp.mazo_id = m.id
INNER JOIN jugador j ON m.jugador_id = j.id
WHERE mp.ganador = TRUE
  AND MONTH(pj.fecha_hora) = @mes
  AND YEAR(pj.fecha_hora) = @anio
GROUP BY j.id, j.nombre
ORDER BY victorias DESC
LIMIT @limite;
``` 
Al cabo de un tiempo, el juego se vuelve un éxito y el número de jugadores aumenta considerablemente. Los cálculos de los rankings empiezan a tardar demasiado, haciendo que el servicio se **caiga por falta de memoria**.

#### Teniendo en cuenta que el proyecto se encuentra excedido de presupuesto y no se autoriza la adquisición de nuevos recursos (hardware, licencias ni contratación adicional):
#### 1) Elabore una propuesta para resolver el problema identificado utilizando exclusivamente la infraestructura y herramientas actuales.
#### 2) Explique detalladamente qué deberían tener en cuenta el resto de desarrolladores para que su cambio funcione correctamente.
