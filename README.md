# Bimestral
Examen Bimestral

```scala
import doobie._
import doobie.implicits._
import cats.effect._
import cats.implicits._
import scala.io.Source
import java.sql.Date
import doobie.implicits.javasql._

object InyeccionChampions extends IOApp.Simple {


  case class Partido(
                      id: Int,
                      equipoLocal: String,
                      equipoVisitante: String,
                      fecha: Date,
                      estadio: String,
                      golesLocal: Int,
                      golesVisitante: Int,
                      jugado: Boolean
                    )


  val xa = Transactor.fromDriverManager[IO](
    driver = "com.mysql.cj.jdbc.Driver",
    url = "jdbc:mysql://localhost:3306/champions_league",
    user = "root",
    password = "suco2018",
    logHandler = None
  )


  def insertPartido(p: Partido): ConnectionIO[Int] =
    sql"""
      INSERT INTO partidos_champions (id, equipo_local, equipo_visitante, fecha_partido, estadio, goles_local, goles_visitante, partido_jugado)
      VALUES (${p.id}, ${p.equipoLocal}, ${p.equipoVisitante}, ${p.fecha}, ${p.estadio}, ${p.golesLocal}, ${p.golesVisitante}, ${p.jugado})
    """.update.run

  // Función para limpiar la tabla antes de la carga
  def limpiarTabla(): ConnectionIO[Int] =
    sql"TRUNCATE TABLE partidos_champions".update.run

  // Función para listar los datos insertados y verificar
  def listAllPartidos(): ConnectionIO[List[Partido]] =
    sql"SELECT id, equipo_local, equipo_visitante, fecha_partido, estadio, goles_local, goles_visitante, partido_jugado FROM partidos_champions"
      .query[Partido]
      .to[List]

  override def run: IO[Unit] = {

    val path = "C:\\Programacion Funcional y Reactiva\\Bimestral\\src\\main\\resources\\data\\futbol.csv"

    val cargarDatos = IO.blocking {
      val source = Source.fromFile(path)
      val lineas = source.getLines().drop(1).toList // Saltamos la cabecera
      source.close()
      lineas
    }

    val programa = for {
      lineas <- cargarDatos


      partidos = lineas.map { linea =>
        val cols = linea.split(",")
        Partido(
          cols(0).trim.toInt,
          cols(1).trim,
          cols(2).trim,
          Date.valueOf(cols(3).trim), // Convierte "YYYY-MM-DD" a java.sql.Date
          cols(4).trim,
          cols(5).trim.toInt,
          cols(6).trim.toInt,
          cols(7).trim.toLowerCase.toBoolean
        )
      }


      _ <- limpiarTabla().transact(xa)
      _ <- partidos.traverse(p => insertPartido(p)).transact(xa)


      resultado <- listAllPartidos().transact(xa)

      _ <- IO.println(s"Se han insertado ${resultado.length} registros exitosamente.")
      _ <- IO(resultado.take(5).foreach(println)) // Imprime los primeros 5 para verificar
    } yield ()

    programa
  }
}
```

Script

```sql
CREATE DATABASE IF NOT EXISTS futbol_db;
USE futbol_db;

CREATE TABLE partidos (
    id INT PRIMARY KEY,
    equipo_local VARCHAR(100) NOT NULL,
    equipo_visitante VARCHAR(100) NOT NULL,
    fecha_partido DATE NOT NULL,
    estadio VARCHAR(150) NOT NULL,
    goles_local INT NOT NULL,
    goles_visitante INT NOT NULL,
    partido_jugado BOOLEAN NOT NULL
);
select  * from partidos;
```


### Capturas

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/9ca7c410-5531-4d17-85bd-d52398a38480" />


<img width="1362" height="584" alt="image" src="https://github.com/user-attachments/assets/b39bfb12-9f80-4aa2-b81f-d4ac85bd054b" />
