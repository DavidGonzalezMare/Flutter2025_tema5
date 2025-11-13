# 4. El Widget StreamBuilder


En apartados anteriores hemos visto como el widget *FutureBuilder* nos permite la generación de interfaces en Flutter a partir de datos asíncronos (*Future*).

El giny `StreamBuilder` será muy parecido al `FutureBuilder`, con la diferencia de que este nos permitirá trabajar en tiempo real con datos asíncronos mediante `Streams`. 

<br>

## StreamBuilder

La gestión de *Streams* en Flutter se realiza mediante *StreamBuilder*. Estos widgets nos permiten redibujar un widget cada vez que ocurre un nuevo evento en el Stream. 

Si hacemos uso del *Snippet* `StreamBldr`, veremos que la forma más común de crear un `StreamBuilder` toma la siguiente forma:

```dart
StreamBuilder(
  stream: stream,
  initialData: initialData,
  builder: (BuildContext context, AsyncSnapshot snapshot) {
    return Container(
      child: child,
    );
  },
),
```
Este constructor recibe un *Stream*, con el origen de los datos, unos valores iniciales (`initialData`), y un método `builder` para construir la interfaz. Este `builder` recibirá un `BuildContext`, para saber la ubicación en el árbol de widgets, y un `AsyncSnapshot`, con la información sobre los acontecimientos que van sucediendo en el `Stream`. 

En este snapshot recordamos que teníamos propiedades como `hasData`, para saber si el *Stream* ha devuelto datos, has sido necesario, para saber si contiene algún error, `conectationState`, para ver el estado del *Stream*, así como fecha y error, con la información que contiene el evento o posibles errores.

<br>

## Ejemplo: Un reloj digital

A modo de ejemplo, vamos a crear una aplicación que muestre la hora del sistema en tiempo real haciendo uso de un Stream que emite la hora a cada segundo.

### Generación del Stream

En primer lugar, haremos una función generadora que nos devuelva un Stream que contendrá un texto. 

Esta función tendrá el siguiente código:

```dart
Stream<String> obtenerHoraStream() async* {
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    DateTime horaActual = DateTime.now();
    String hora = horaActual.hour.toString().padLeft(2, '0');
    String minutos = horaActual.minute.toString().padLeft(2, '0');
    String segundos = horaActual.second.toString().padLeft(2, '0');
    yield "$hora:$minutos:$segundos";
  }
}
```

Como vemos, se trata de un bucle infinito, que en cada segundo obtiene la hora y genera un mensaje de texto con la hora el formato *HH:MM:SS*. Observad que estamos utilizando un nuevo método `padLeft`, que añade tantos caracteres del tipo indicado (en este caso un `'0'`) a la izquierda para que la cadena tenga el número de caracteres indicados como primer argumento (en este caso, 2).

Otras formas de expresar la función

Vamos a ver un par más de formas de definir esta función generadora, que nos servirán para descubrir algunos aspectos interesantes de Dart.

- **A. Expresar la función como un `getter`**: Aunque esta función no pertenece a una clase, podemos expresarla como si se tratara de un getter, de manera que se podría utilizar como una variable o propiedad. Para ello haríamos uso de la palabra clave `get` antes del nombre, y eliminaríamos los argumentos:
  
```dart
Stream<String> get obtenerHoraStream2 async* {
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    DateTime horaActual = DateTime.now();
    String hora = horaActual.hour.toString().padLeft(2, '0');
    String minutos = horaActual.minute.toString().padLeft(2, '0');
    String segundos = horaActual.second.toString().padLeft(2, '0');
    yield "$hora:$minutos:$segundos";
  }
}
```

- **B. Expresar la función de forma recursiva con `yield`**: La orden `yield` se utiliza con el fin de volcar sobre el Stream el resultado de una llamada a una función que devuelva un *Stream*. Así pues, se puede utilizar para expresar recursivamente este tipo de funciones. Por ejemplo:
  
```dart
// Ejemplo recursivo. Observemos que no existe el bucle, 
// sino una llamada recursiva a la misma función.

Stream<String> obtenerHoraStream3() async* {
  await Future.delayed(const Duration(seconds: 1));
  DateTime horaActual = DateTime.now();
  String hora = horaActual.hour.toString().padLeft(2, '0');
  String minutos = horaActual.minute.toString().padLeft(2, '0');
  String segundos = horaActual.second.toString().padLeft(2, '0');
  yield "$hora:$minutos:$segundos";

  yield* obtenerHoraStream3();
}
```

### Uso del Stream en un StreamBuilder

Una vez tenemos definida la función que nos devuelva el Stream, vamos a aplicar el widget `StreamBuilder`, que estará vinculado a este *Stream* y se redibujará cada vez que haya un cambio.

```dart
class WidgetReloj extends StatelessWidget {
  const WidgetReloj({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: StreamBuilder(
          stream: obtenerHoraStream(),
          builder: (context, AsyncSnapshot<String> snapshot) {
            // Para ver el estado de la conexión
            // debugPrint(snapshot.connectionState.toString());
            return Center(
              child: Text(
                snapshot.data ?? "",
                style: Theme.of(context).textTheme.displayLarge,
              ),
            );
          }),
    );
  }
}
```

Como veis, el código es prácticamente igual que para el `FutureBuilder`. Definimos la propiedad `stream` con la función que nos proporciona éste (`obtenerHoraStream`), y la propiedad `builder`, con el código para regenerar cada vez el widget. Si aquí observamos el estado de conexión del `AsyncSnapshot`, veremos ahora que muestra el estado `ConnectionState.active`, indicando que la conexión al Stream sigue activa.

<br>

## Ejemplo 2. Un cronómetro

Vamos a avanzar un poco más, y a crear ahora un cronómetro, el cual, además de llevar el recuento del tiempo permitirá algunas funcionalidades, como pausar y retomar el tiempo o resetearlo.

![Cronometro1](./images/imagen04.png)

<br>

![Cronometro2](./images/imagen05.png)

 
Como vemos, aparecen dos botones. El primero servirá para iniciar o pausar el tiempo, y el segundo para ponerlo a 0 siempre que no esté en marcha.

Para ello, generaremos dos clases:

- Una clase `Cronometro`, donde encapsularemos toda esta funcionalidad y que entre sus propiedades contendrá un par de Streams sobre los que se emitirán eventos hacia la interfaz de usuario,
  
- Una clase `Crono` que será un widget sin estado, que se construirá mediante varios `StreamBuilder`, que estarán escuchando los Streams de la clase `Cronometro`.
- 
En primer lugar, vemos la clase Cronometro.

La  clase *Cronometro*

Esta clase encapsulará la lógica de la aplicación y tendrá las siguientes propiedades:

- `late Duration tiempo`: Que será el valor en sí del cronómetro. Se define como `late` y se inicializará en el constructor.
  
- `late final StreamController<String> _tiempoController`: Un controlador para el `StreamController` sobre el que se emitirán los cambios en el tiempo transcurrido. También se inicializará en el constructor.
  
- `Timer? _timer`: La clase `Timer` ofrece temporizadores que permiten lanzar eventos de forma periódica. Este _timer reemplazará el bucle infinito que utilizábamos en el reloj para actualizar el tiempo, ya que, además, nos permitirá detenerlo y retomarlo.
  
- `late bool isRunning`: Contiene un valor lógico que indica si el crono está en marcha. Se inicializará en el constructor.
  
- `late final StreamController<bool> _statusController`: Un controlador para el Stream Sobre el que se emitirá el valor de la propiedad lógica `isRunning` después de cada cambio en la misma.
  
Vemos también los diferentes métodos que implementa esta clase. A continuación haremos una breve descripción de cada uno, y mostraremos el código comentado.

- El constructor Cronometro(). Inicializará las propiedades y emitirá por el Stream los valores iniciales.
  
```dart
  // Constructor
  Cronometro() {
    /* Inicialització */

    // Temporitzador a 0
    tiempo = const Duration(seconds: 0);

    // Inicializarmos los controladore de Stream
    _tiempoController = StreamController<String>();
    _statusController = StreamController<bool>();

    // Inicialment no estarà en marxa
    isRunning = false;

    // Emetem per l'Stream els valors inicials
    _tiempoController.add(tiempoToString(tiempo));
    _statusController.add(isRunning);
  }
```

- Método `tiempoToString(Duration tiempo)`, será un método de utilidad que convertirá un valor de tipo `Duration` en un `String` con formato Minutos:Segundos:Décimas (MM:SS:DD). Para ello, obtiene el resto (*remainder*) de la división del tiempo, tanto en minutos, como segundos (SS) y décimas (DD) entre 60, añadiendo un 0 a la izquierda si hace falta para completar los 20 minutos:
  
```dart
  String tiempoToString(Duration tiempo) {
    // Este método convierte el tiempo proporcionado en el 
    // formato MM:SS:DD (Minutos, Segundos, Decimas)
    String minutos = tiempo.inMinutes.remainder(60).toString().padLeft(2, "0");
    String segundos = tiempo.inSeconds.remainder(60).toString().padLeft(2, "0");
    String decimas =
        ((tiempo.inMilliseconds) * 100).remainder(60).toString().padLeft(2, "0");
    return "$minutos:$segundos:$decimas";
  }

```

- `void inicia()`, para poner en marcha el cronómetro a través del temporizador. Este método comprobará inicialmente que no estuviera previamente en marcha con el fin de no incorporar múltiples temporizadores. Observe que se hace uso del método `Timer.periodic` para establecer un temporizador que periódicamente, cada 100 ms ejecutará una función de *callback*. Esta función, añadirá una duración de 100ms al tiempo transcurrido, y emitirá un evento con este valor, convertido en `String` en el *Stream*. Además, cuando el flag `isRunning` cambie de valor, también se emitirá el valor del mismo:
  
```dart
  void inicia() {
    // Método para iniciar el crono

    // Comprueba que no esturviera previamente en marcha
    // (Si no añadiría distintos temporizadores)
    if (!(isRunning)) {
      // Activa el flag isRunning
      isRunning = true;
      // y emite este estado por el Stream
      _statusController.add(isRunning);

      // Activa el temporitzador (_timer), que cada 100 milisegons
      // añade una duración de 100 milisegundos al tiempo transcurrido

      _timer = Timer.periodic(const Duration(milliseconds: 100), (timer) {
        tiempo += const Duration(milliseconds: 100);
        // Afegim el temps a l'Stream
        _tiempoController.add(tiempoToString(tiempo));
      });
    }
  }
```

- `void para()`, para detener el temporizador, y por lo tanto parar el tiempo del cronómetro. Esto lo hará cancelando el `_timer`, de manera que ya no ejecute el callback cada 100 ms. Además, marcará el flag `isRunning` a `false`, para saber que se puede volver a retomar posteriormente, y emitirá este valor por el *Stream* correspondiente:
  
```dart
  void para() {
    // Simplemente, cancelamos el temporizador para que
    // no se actualize el tiempo
    _timer?.cancel();

    // MArcamos el flag como que no está en marcha
    isRunning = false;
    // y emitimos este estado por el Stream correspondiente
    _statusController.add(isRunning); 
  }
```

- `void reinicia()`, para resetear a 00:00:00 el valor del contador. Observe que debemos emitir este valor a 0 por el Stream para que se reflejen los cambios en la interfaz.
  
```dart
  void reinicia() {
    // Para reiniciar el contador, lo ponemos a 0
    tiempo = const Duration(seconds: 0);
    // Y emitimos este valor por el Strema a través 
    // del controlador
    _tiempoController.add(tiempoToString(tiempo));
  }
```

- `Stream<String> get obtenerTiempoStream`, será un método de acceso (*get*) que devolverá el *Stream* asociado al controlador `_tiempoController`, y a través del cual recibirá los eventos la misma interfaz. Observe que este mecanismo hará el papel de la función generadora que utilizábamos para obtener la hora, ya que proporciona el *Stream* que necesitará el *StreamBuilder* en la interfaz de usuario.

```dart  
  Stream<String> get obtenerTiempoStream => _tiempoController.stream;
```

- `Stream<bool> get ObtenerStatusStream`, será otro método de acceso (get) que devuelve el Stream asociado al controlador `_statusController`, sobre el que se emiten los cambios al valor de `isRunning`:
  
```dart
  Stream<bool> get obtenerStatusStream => _statusController.stream;
```
- `void dispose()`, que cerraría el Stream cuando deje de utilizarse el widget.

```dart
  void dispose() {
    _tiempoController.close();
  }
```

### Generando la interfaz de usuario

El widget principal para la pantalla del cronómetro será un widget sin estado, que definiremos en la clase **Crono**. Este widget mostrará el tiempo transcurrido, y un par de botones para controlarlo. 

**El widget personalizado BotonCrono**

Antes, sin embargo, de abordar esta clase principal, echamos un vistazo a los botones que controlan el estado del crono. Estos botones serán de tipo `ElevatedButton` personalizados para que tengan un aspecto redondo, como si se tratara de un *Floating Action Button*. Para ello, crearemos una nueva clase `BotonCrono`. El constructor de esta clase requerirá que le proporcionemos un icono y una función de callback que será invocada cuando se pulse el botón:

```dart
class BotonCrono extends StatelessWidget {
  const BotonCrono({super.key, required this.onPressed, required this.icon});
  final dynamic onPressed;
  final IconData icon;

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: 70,
      height: 70,
      child: ElevatedButton(
        style: ElevatedButton.styleFrom(
            shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(100),
        )),
        onPressed: onPressed,
        child: Icon(icon),
      ),
    );
  }
}
```

**La clase Crono**

Pasamos ya a la clase `Crono` como tal. Como hemos dicho, ésta se define como un widget sin estado, que contiene una propiedad final (`miCronometro`) que será un objeto de tipo `Cronometro`.

```dart
class Crono extends StatelessWidget {
  Crono({super.key});

  // Esta contendrá una propiedad de la clase Cronometro
  final Cronometro miCronometro = Cronometro();

...}
```

Ahora, el método build de la pantalla nos devolverá una estructura de aplicación Material (`Scaffold`), cuyo cuerpo será un `StreamBuilder`:

```dart
class Crono extends StatelessWidget {
  Crono({super.key});

  ...

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: StreamBuilder(
        ...
        );
  }
```

Este `StreamBuilder` deberá construirse en función del `Stream` Sobre el que se emite información sobre el tiempo transcurrido en la clase `Cronometro`, y que obtendremos a través del método `miCronometro.obtenerTiempoStream`:

```dart
StreamBuilder(
  stream: miCronometro.obtenerTiempoStream,
  builder: (context, AsyncSnapshot<String> snapshot) {
    //...
  })
```

Cuando se emiten datos sobre el Stream, se reconstruirá el widget, mediante el correspondiente `builder`. En este caso, constará principalmente de un texto con el valor que se ha emitido en el `Stream`, organizado en forma de columna dentro de un `ScrollView` y dentro de un `Center`:

```dart
StreamBuilder(
  stream: miCronometro.obtenerTiempoStream,
  builder: (context, AsyncSnapshot<String> snapshot) {
    return Center(
      child: SingleChildScrollView(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              snapshot.data ?? "",
              style: Theme.of(context).textTheme.displayLarge,
            ),
            // Aquí irán los botones para  gestionar 
            // el cronómetro (Inicia/Pausa + Reset)
          ],
        ),
      ),
    );
})
```

En la misma columna, bajo el texto habrá que mostrar los dos botones de inicio/pausa y restauración de los valores. Como estos botones se mostrarán y tendrán un comportamiento según esté en marcha o no el cronómetro, los implementaremos también mediante un `StreamBuilder`, que dependerá del `Stream` con el estado del cronómetro, y que obtenemos con miCronometro.

```dart
StreamBuilder<bool>(
  stream: miCronometro.obtenerStatusStream,
  builder: (context, snapshot) {
    bool runningStatus = snapshot.data ?? false;
    return createButtonRow(runningStatus);
  })
```

Como vemos, este `StreamBuilder` dependerá del estado del cronómetro, que será un valor lógico que indicará si el cronómetro está corriendo o no. Cada vez que se reciba un `snapshot` con este valor, se redibujará el widget. Para ello haremos uso del método auxiliar `createButtonRow`, al que le proporcionaremos el valor del snapshot y nos devolverá una fila con los dos botones, con el aspecto y el comportamiento esperados. Su código será el siguiente:

```dart
  Widget createButtonRow(bool isRunning) {
    if (!isRunning) {
      return Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          BotonCrono(
            icon: Icons.play_arrow,
            onPressed: () => miCronometro.inicia(),
          ),
          BotonCrono(
            icon: Icons.undo,
            onPressed: () => miCronometro.reinicia(),
          ),
        ],
      );
    } else {
      return Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,
        children: [
          BotonCrono(icon: Icons.pause, onPressed: () => miCronometro.para()),
          const BotonCrono(icon: Icons.undo, onPressed: null),
        ],
      );
    }
  }
```

<script src="https://gist.github.com/DavidGonzalezMare/9ccfc0735843340b76386ba57b6727bc.js"></script>

[https://dartpad.dev/embed-flutter.html?id=9ccfc0735843340b76386ba57b6727bc](https://dartpad.dev/embed-flutter.html?id=9ccfc0735843340b76386ba57b6727bc)
