# Artisan Console

- [Introducción](#introduction)
    - [Tinker (REPL)](#tinker)
- [Comandos de escritura](#writing-commands)
    - [Generación de comandos](#generating-commands)
    - [Estructura de mando](#command-structure)
    - [Comandos de cierre](#closure-commands)
    - [Comandos aislables](#isolatable-commands)
- [Definición de expectativas de entrada](#defining-input-expectations)
    - [Argumentos](#arguments)
    - [Opciones](#options)
    - [Matrices de entrada](#input-arrays)
    - [Descripciones de entrada](#input-descriptions)
- [E/S de comando](#command-io)
    - [Recuperando entrada](#retrieving-input)
    - [Solicitud de entrada](#prompting-for-input)
    - [Salida de escritura](#writing-output)
- [Registro de comandos](#registering-commands)
- [Ejecución programática de comandos](#programmatically-executing-commands)
    - [Llamar comandos desde otros comandos](#calling-commands-from-other-commands)
- [Manejo de señales](#signal-handling)
- [Personalización de talones](#stub-customization)
- [Eventos](#events)

<a name="introduction"></a>
## Introducción

Artisan es la interfaz de línea de comandos incluida con Laravel. Artisan existe en la raíz de su aplicación como el script `artisan` y proporciona una serie de comandos útiles que pueden ayudarlo mientras construye su aplicación. Para ver una lista de todos los comandos Artisan disponibles, puede usar el comando `list`:

```shell
php artisan list
```

Cada comando también incluye una pantalla de "ayuda" que muestra y describe los argumentos y opciones disponibles del comando. Para ver una pantalla de ayuda, anteponga `help` al nombre del comando:

```shell
php artisan help migrate
```

<a name="laravel-sail"></a>
#### Laravel Sail

Si está usando [Laravel Sail](/docs/{{version}}/sail) como su entorno de desarrollo local, recuerde usar la línea de comando `sail` para invocar los comandos de Artisan. Sail ejecutará sus comandos Artisan dentro de los contenedores Docker de su aplicación:

```shell
./vendor/bin/sail artisan list
```

<a name="tinker"></a>
### Tinker (REPL)

Laravel Tinker es un potente REPL para el Framework de Laravel, impulsado por el paquete [PsySH](https://github.com/bobthecow/psysh).

<a name="installation"></a>
#### Instalación

Todas las aplicaciones de Laravel incluyen Tinker por defecto. Sin embargo, puede instalar Tinker usando Composer si lo ha eliminado previamente de su aplicación:

```shell
composer require laravel/tinker
```

> **Note**  
> ¿Busca una interfaz de usuario gráfica para interactuar con su aplicación Laravel? ¡Echa un vistazo a [Tinkerwell](https://tinkerwell.app)!

<a name="usage"></a>
#### Uso

Tinker le permite interactuar con toda su aplicación Laravel en la línea de comando, incluidos sus modelos, trabajos, eventos y más de Eloquent. Para ingresar al entorno Tinker, ejecute el comando `tinker` Artisan:

```shell
php artisan tinker
```

Puedes publicar el archivo de configuración de Tinker usando el comando `vendor:publish`:

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> **Warning**  
> La función auxiliar `dispatch` y el método `dispatch` en la clase `Dispatchable` dependen de la recolección de elementos no utilizados para colocar el trabajo en la cola. Por lo tanto, al usar tinker, debe usar `Bus::dispatch` o `Queue::push` para enviar trabajos.

<a name="command-allow-list"></a>
#### Lista de comandos permitidos

Tinker utiliza una lista de "permisos" para determinar qué comandos de Artisan pueden ejecutarse dentro de su shell. De forma predeterminada, puede ejecutar los comandos `clear-compiled`, `down`, `env`, `inspire`, `migrate`, `optimize` y `up`. Si desea permitir más comandos, puede agregarlos a la matriz `commands` en su archivo de configuración `tinker.php`:

    'commands' => [
        // App\Console\Commands\ExampleCommand::class,
    ],

<a name="classes-that-should-not-be-aliased"></a>
#### Classes That Should Not Be Aliased

Por lo general, Tinker crea automáticamente un alias de clases a medida que interactúa con ellas en Tinker. Sin embargo, es posible que desee nunca poner alias a algunas clases. Puede lograr esto enumerando las clases en la matriz `dont_alias` de su archivo de configuración `tinker.php`:

    'dont_alias' => [
        App\Models\User::class,
    ],

<a name="writing-commands"></a>
## Comandos de escritura

Además de los comandos proporcionados con Artisan, puede crear sus propios comandos personalizados. Los comandos generalmente se almacenan en el directorio `app/Console/Commands`; sin embargo, usted es libre de elegir su propia ubicación de almacenamiento siempre que Composer pueda cargar sus comandos.

<a name="generating-commands"></a>
### Generación de comandos

Para crear un nuevo comando, puede usar el comando Artisan `make:command`. Este comando creará una nueva clase de comando en el directorio `app/Console/Commands`. No se preocupe si este directorio no existe en su aplicación; se creará la primera vez que ejecute el comando `make:command` de Artisan:

```shell
php artisan make:command SendEmails
```

<a name="command-structure"></a>
### Estructura de mando

Después de generar su comando, debe definir los valores apropiados para las propiedades `signature` y `description` de la clase. Estas propiedades se usarán cuando muestre su comando en la pantalla `list`. La propiedad `signature` también le permite definir [las expectativas de entrada de su comando](#defining-input-expectations). Se llamará al método `handle` cuando se ejecute su comando. Puede colocar su lógica de comando en este método.

Echemos un vistazo a un comando de ejemplo. Tenga en cuenta que podemos solicitar cualquier dependencia que necesitemos a través del método `handle` del comando. El Laravel [contenedor de servicios](/docs/{{version}}/container) inyectará automáticamente todas las dependencias que tienen sugerencias de tipo en la firma de este método:

    <?php

    namespace App\Console\Commands;

    use App\Models\User;
    use App\Support\DripEmailer;
    use Illuminate\Console\Command;

    class SendEmails extends Command
    {
        /**
         * The name and signature of the console command.
         *
         * @var string
         */
        protected $signature = 'mail:send {user}';

        /**
         * The console command description.
         *
         * @var string
         */
        protected $description = 'Send a marketing email to a user';

        /**
         * Execute the console command.
         */
        public function handle(DripEmailer $drip): void
        {
            $drip->send(User::find($this->argument('user')));
        }
    }

> **Note**  
> Para una mayor reutilización del código, es una buena práctica mantener los comandos de la consola livianos y dejar que se remitan a los servicios de la aplicación para realizar sus tareas. En el ejemplo anterior, tenga en cuenta que inyectamos una clase de servicio para hacer el "trabajo pesado" de enviar los correos electrónicos.

<a name="closure-commands"></a>
### Comandos de cierre

Los comandos basados ​​en cierre proporcionan una alternativa a la definición de comandos de consola como clases. De la misma manera que los cierres de rutas son una alternativa a los controladores, piense en los cierres de comandos como una alternativa a las clases de comandos. Dentro del método `commands` de su archivo `app/Console/Kernel.php`, Laravel carga el archivo `routes/console.php`:

    /**
     * Registre los comandos basados ​​en cierre para la aplicación.
     */
    protected function commands(): void
    {
        require base_path('routes/console.php');
    }

Aunque este archivo no define rutas HTTP, define puntos de entrada (rutas) basados ​​en la consola en su aplicación. Dentro de este archivo, puede definir todos sus comandos de consola basados ​​en el cierre usando el método `Artisan::command`. El método `command` acepta dos argumentos: la [firma del comando](#defining-input-expectations)y un cierre que recibe los argumentos y opciones del comando:

    Artisan::command('mail:send {user}', function (string $user) {
        $this->info("Enviando correo electrónico a: {$user}!");
    });

El cierre está vinculado a la instancia de comando subyacente, por lo que tiene acceso completo a todos los métodos auxiliares a los que normalmente podría acceder en una clase de comando completa.

<a name="type-hinting-dependencies"></a>
#### Dependencias de sugerencia de tipo

Además de recibir los argumentos y las opciones de su comando, los cierres de comandos también pueden indicar dependencias adicionales que le gustaría resolver fuera del [contenedor de servicio](/docs/{{version}}/container):

    use App\Models\User;
    use App\Support\DripEmailer;

    Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
        $drip->send(User::find($user));
    });

<a name="closure-command-descriptions"></a>
#### Descripciones de los comandos de cierre

Al definir un comando basado en el cierre, puede usar el `purpose` para agregar una descripción al comando. Esta descripción se mostrará cuando ejecute el comando `php artisan list` o `php artisan help`:

    Artisan::command('mail:send {user}', function (string $user) {
        // ...
    })->purpose('Enviar un correo electrónico de marketing a un usuario');

<a name="isolatable-commands"></a>
### Comandos aislables

> **Warning**
> Para utilizar esta característica, su aplicación debe estar usando el controlador de caché `memcached`, `redis`, `dynamodb`, `database`, `file`, o `array` como el controlador de caché predeterminado de su aplicación. Además, todos los servidores deben comunicarse con el mismo servidor de caché central.

En ocasiones, es posible que desee asegurarse de que solo se pueda ejecutar una instancia de un comando a la vez. Para lograr esto, puede implementar la interfaz `Illuminate\Contracts\Console\Isolatable` en tu clase de comando:

    <?php

    namespace App\Console\Commands;

    use Illuminate\Console\Command;
    use Illuminate\Contracts\Console\Isolatable;

    class SendEmails extends Command implements Isolatable
    {
        // ...
    }

Cuando un comando se marca como`Isolatable`, Laravel agregará automáticamente un `--isolated` opción al comando. Cuando se invoca el comando con esa opción, Laravel se asegurará de que no se estén ejecutando otras instancias de ese comando. Laravel logra esto al intentar adquirir un bloqueo atómico utilizando el controlador de caché predeterminado de su aplicación. Si se están ejecutando otras instancias del comando, el comando no se ejecutará; sin embargo, el comando seguirá saliendo con un código de estado de salida exitoso:

```shell
php artisan mail:send 1 --isolated
```

Si desea especificar el código de estado de salida que debe devolver el comando si no se puede ejecutar, puede proporcionar el código de estado deseado a través de la opción`isolated`:

```shell
php artisan mail:send 1 --isolated=12
```

<a name="lock-expiration-time"></a>
#### Tiempo de caducidad del bloqueo

De forma predeterminada, los bloqueos de aislamiento caducan una vez finalizado el comando. O, si el comando se interrumpe y no puede finalizar, el bloqueo caducará después de una hora. Sin embargo, puede ajustar el tiempo de caducidad del bloqueo definiendo un método `isolationLockExpiresAt` a tu comando:

```php
use DateTimeInterface;
use DateInterval;

/**
 * Determine cuándo caduca un bloqueo de aislamiento para el comando.
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->addMinutes(5);
}
```

<a name="defining-input-expectations"></a>
## Definición de expectativas de entrada

Al escribir comandos de consola, es común recopilar información del usuario a través de argumentos u opciones. Laravel hace que sea muy conveniente definir la entrada que espera del usuario que usa la propiedad `signature` en tus comandos. La propiedad `signature` le permite definir el nombre, los argumentos y las opciones para el comando en una sintaxis única, expresiva y similar a una ruta.

<a name="arguments"></a>
### Argumentos

Todos los argumentos y opciones proporcionados por el usuario están entre llaves. En el siguiente ejemplo, el comando define un argumento requerido: `user`:

    /**
     * El nombre y la firma del comando de la consola.
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

También puede hacer que los argumentos sean opcionales o definir valores predeterminados para los argumentos:

    // Argumento opcional...
    'mail:send {user?}'

    // Argumento opcional con valor predeterminado...
    'mail:send {user=foo}'

<a name="options"></a>
### Opciones

Las opciones, como los argumentos, son otra forma de entrada del usuario. Las opciones van precedidas de dos guiones (`--`) cuando se proporcionan a través de la línea de comandos. Hay dos tipos de opciones: las que reciben un valor y las que no. Las opciones que no reciben un valor sirven como un "cambio" booleano. Veamos un ejemplo de este tipo de opción:

    /**
     * El nombre y la firma del comando de la consola.
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--queue}';

En este ejemplo, el interruptor `--queue`se puede especificar al llamar al comando Artisan. Si el interruptor `--queue` se ha definido, el valor de la opción será `true`. De lo contrario, el valor será `false`:

```shell
php artisan mail:send 1 --queue
```

<a name="options-with-values"></a>
#### Opciones con valores

A continuación, echemos un vistazo a una opción que espera un valor. Si el usuario debe especificar un valor para una opción, debe agregar el sufijo del nombre de la opción con un signo `=`:

    /**
     * El nombre y la firma del comando de la consola.
     *
     * @var string
     */
    protected $signature = 'mail:send {user} {--queue=}';

En este ejemplo, el usuario puede pasar un valor para la opción así. Si no se especifica la opción al invocar el comando, su valor será `null`:

```shell
php artisan mail:send 1 --queue=default
```

Puede asignar valores predeterminados a las opciones especificando el valor predeterminado después del nombre de la opción. Si el usuario no pasa ningún valor de opción, se utilizará el valor predeterminado:

    'mail:send {user} {--queue=default}'

<a name="option-shortcuts"></a>
#### Atajos de opciones

Para asignar un atajo al definir una opción, puede especificarlo antes del nombre de la opción y usar el carácter `|` como delimitador para separar el atajo del nombre completo de la opción:

    'mail:send {user} {--Q|queue}'

Al invocar el comando en su terminal, los accesos directos de opciones deben tener un solo guión como prefijo y no se debe incluir ningún carácter `=` al especificar un valor para la opción:la opción:

```shell
php artisan mail:send 1 -Qdefault
```

<a name="input-arrays"></a>
### Matrices de entrada

Si desea definir argumentos u opciones para esperar múltiples valores de entrada, puede usar el carácter `*`. Primero, echemos un vistazo a un ejemplo que especifica tal argumento:

    'mail:send {user*}'

Al llamar a este método, los argumentos `user` se pueden pasar en orden a la línea de comando. Por ejemplo, el siguiente comando establecerá el valor de `user` en una matriz con `1` y `2` como sus valores:

```shell
php artisan mail:send 1 2
```

Este carácter `*` se puede combinar con una definición de argumento opcional para permitir cero o más instancias de un argumento:

    'mail:send {user?*}'

<a name="option-arrays"></a>
#### Matrices de opciones

Al definir una opción que espera múltiples valores de entrada, cada valor de opción pasado al comando debe tener como prefijo el nombre de la opción:

    'mail:send {--id=*}'

Tal comando puede invocarse pasando múltiples argumentos `--id`:

```shell
php artisan mail:send --id=1 --id=2
```

<a name="input-descriptions"></a>
### Descripciones de entrada

Puede asignar descripciones a argumentos de entrada y opciones separando el nombre del argumento de la descripción mediante dos puntos. Si necesita un poco más de espacio para definir su comando, siéntase libre de distribuir la definición en varias líneas:

    /**
     * El nombre y la firma del comando de la consola.
     *
     * @var string
     */
    protected $signature = 'mail:send
                            {user : The ID of the user}
                            {--queue : Whether the job should be queued}';

<a name="command-io"></a>
## E/S de comando

<a name="retrieving-input"></a>
### Recuperando entrada

Mientras se ejecuta su comando, es probable que necesite acceder a los valores de los argumentos y opciones aceptados por su comando. Para ello, puede utilizar los métodos `argument` y `option`. Si un argumento u opción no existe, se devolverá `null`:

    /**
     * Ejecute el comando de la consola.
     */
    public function handle(): void
    {
        $userId = $this->argument('user');
    }

Si necesita recuperar todos los argumentos como un `array`, llame al método `arguments`:

    $arguments = $this->arguments();

Las opciones se pueden recuperar tan fácilmente como los argumentos utilizando el método `option`. Para recuperar todas las opciones como una matriz, llama al método `options`:

    // Recuperar una opción específica...
    $queueName = $this->option('queue');

    // Recuperar todas las opciones como una matriz...
    $options = $this->options();

<a name="prompting-for-input"></a>
### Solicitud de entrada

Además de mostrar la salida, también puede pedirle al usuario que proporcione información durante la ejecución de su comando. El método `ask` le pedirá al usuario la pregunta dada, aceptará su entrada y luego devolverá la entrada del usuario a su comando:

    /**
     * Ejecute el comando de la consola.
     */
    public function handle(): void
    {
        $name = $this->ask('What is your name?');

        // ...
    }

El método `secret` es similar a `ask`, pero la entrada del usuario no será visible para ellos mientras escriben en la consola. Este método es útil cuando se solicita información confidencial, como contraseñas:

    $password = $this->secret('¿Cual es la contraseña?');

<a name="asking-for-confirmation"></a>
#### Pidiendo Confirmación

Si necesita pedirle al usuario una simple confirmación de "sí o no", puede usar el método `confirm`. De forma predeterminada, este método devolverá `false`. Sin embargo, si el usuario ingresa `y` o `yes` en respuesta al aviso, el método devolverá `true`.

    if ($this->confirm('¿Desea continuar?')) {
        // ...
    }

Si es necesario, puede especificar que la solicitud de confirmación devuelva `true` de forma predeterminada al pasar `true` como segundo argumento del método `confirm`:

    if ($this->confirm('¿Desea continuar?', true)) {
        // ...
    }

<a name="auto-completion"></a>
#### Autocompletar

El método `anticipate` se puede utilizar para proporcionar autocompletado para posibles opciones. El usuario aún puede proporcionar cualquier respuesta, independientemente de las sugerencias de autocompletado:

    $name = $this->anticipate('¿Cómo te llamas?', ['Juan', 'Daniel']);

Alternativamente, puede pasar un cierre como segundo argumento al método `anticipate`. El cierre se llamará cada vez que el usuario escriba un carácter de entrada. El cierre debe aceptar un parámetro de cadena que contenga la entrada del usuario hasta el momento y devolver una serie de opciones para completar automáticamente:

    $name = $this->anticipate('¿Cuál es su dirección?', function (string $input) {
        // Devolver opciones de autocompletado...
    });

<a name="multiple-choice-questions"></a>
#### Preguntas de respuestas múltiples

Si necesita darle al usuario un conjunto predefinido de opciones al hacer una pregunta, puede usar el método `choice`. Puede establecer el índice de matriz del valor predeterminado que se devolverá si no se elige ninguna opción pasando el índice como el tercer argumento del método:

    $name = $this->choice(
        '¿Cómo te llamas?',
        ['Juan', 'Daniel'],
        $defaultIndex
    );

Además, el método `choice` acepta argumentos opcionales cuarto y quinto para determinar el número máximo de intentos para seleccionar una respuesta válida y si se permiten selecciones múltiples:

    $name = $this->choice(
        '¿Cómo te llamas?',
        ['Juan', 'Daniel'],
        $defaultIndex,
        $maxAttempts = null,
        $allowMultipleSelections = false
    );

<a name="writing-output"></a>
### Salida de escritura

Para enviar resultados a la consola, puede utilizar los métodos `line`, `info`, `comment`, `question`, `warn` y `error`. Cada uno de estos métodos utilizará colores ANSI apropiados para su propósito. Por ejemplo, mostremos información general al usuario. Normalmente, el método `info` se mostrará en la consola como texto de color verde:

    /**
     * Ejecute el comando de la consola.
     */
    public function handle(): void
    {
        // ...

        $this->info('¡El comando fue exitoso!');
    }

Para mostrar un mensaje de error, utilice el método `error`. El texto del mensaje de error generalmente se muestra en rojo:

    $this->error('Something went wrong!');

Puede utilizar el método `line` para mostrar texto sin formato y sin color:

    $this->line('Display this on the screen');

Puede usar el método `newLine` para mostrar una línea en blanco:

    // Escribe una sola línea en blanco...
    $this->newLine();

    // Escribe tres líneas en blanco...
    $this->newLine(3);

<a name="tables"></a>
#### Mesas

El método `table` facilita el formateo correcto de múltiples filas/columnas de datos. Todo lo que necesita hacer es proporcionar los nombres de las columnas y los datos de la tabla y Laravel
calcule automáticamente el ancho y la altura apropiados de la mesa para usted:

    use App\Models\User;

    $this->table(
        ['Nombre', 'Correo electrónico'],
        User::all(['name', 'email'])->toArray()
    );

<a name="progress-bars"></a>
#### Barras de progreso

Para tareas de ejecución prolongada, puede ser útil mostrar una barra de progreso que informe a los usuarios qué tan completa está la tarea. Usando el método `withProgressBar`, Laravel mostrará una barra de progreso y avanzará su progreso para cada iteración sobre un valor iterable dado:

    use App\Models\User;

    $users = $this->withProgressBar(User::all(), function (User $user) {
        $this->performTask($user);
    });

A veces, es posible que necesite más control manual sobre cómo avanza una barra de progreso. Primero, defina el número total de pasos que recorrerá el proceso. Luego, avance la barra de progreso después de procesar cada elemento:

    $users = App\Models\User::all();

    $bar = $this->output->createProgressBar(count($users));

    $bar->start();

    foreach ($users as $user) {
        $this->performTask($user);

        $bar->advance();
    }

    $bar->finish();

> **Note**  
> Para obtener opciones más avanzadas, consulta la [documentación del componente Barra de progreso de Symfony](https://symfony.com/doc/current/components/console/helpers/progressbar.html).

<a name="registering-commands"></a>
## Registro de comandos

Todos los comandos de su consola están registrados dentro de la clase `App\Console\Kernel` de su aplicación, que es el "núcleo de la consola" de su aplicación. Dentro del método `commands` de esta clase, verá una llamada al método `load` del núcleo. El método `load` escaneará el directorio `app/Console/Commands` y registrará automáticamente cada comando que contiene con Artisan. Incluso puede realizar llamadas adicionales al método `load` para escanear otros directorios en busca de comandos de Artisan:

    /**
     * Registre los comandos para la aplicación.
     */
    protected function commands(): void
    {
        $this->load(__DIR__.'/Commands');
        $this->load(__DIR__.'/../Domain/Orders/Commands');

        // ...
    }

Si es necesario, puede registrar comandos manualmente agregando el nombre de clase del comando a una propiedad `$commands` dentro de su clase `App\Console\Kernel`. Si esta propiedad aún no está definida en su kernel, debe definirla manualmente. Cuando se inicia Artisan, todos los comandos enumerados en esta propiedad serán resueltos por el [contenedor de servicio](/docs/{{version}}/container) y registrado con Artisan:

    protected $commands = [
        Commands\SendEmails::class
    ];

<a name="programmatically-executing-commands"></a>
## Ejecución programática de comandos

A veces, es posible que desee ejecutar un comando de Artisan fuera de la CLI. Por ejemplo, es posible que desee ejecutar un comando de Artisan desde una ruta o un controlador. Puede usar el método `call` en la fachada `Artisan` para lograr esto. El método `call` acepta el nombre de la firma del comando o el nombre de la clase como primer argumento, y una matriz de parámetros del comando como segundo argumento. El código de salida será devuelto:

    use Illuminate\Support\Facades\Artisan;

    Route::post('/user/{user}/mail', function (string $user) {
        $exitCode = Artisan::call('mail:send', [
            'user' => $user, '--queue' => 'default'
        ]);

        // ...
    });

Alternativamente, puede pasar el comando completo de Artisan al método `call` como una cadena:

    Artisan::call('mail:send 1 --queue=default');

<a name="passing-array-values"></a>
#### Pasar valores de matriz

Si su comando define una opción que acepta una matriz, puede pasar una matriz de valores a esa opción:

    use Illuminate\Support\Facades\Artisan;

    Route::post('/mail', function () {
        $exitCode = Artisan::call('mail:send', [
            '--id' => [5, 13]
        ]);
    });

<a name="passing-boolean-values"></a>
#### Pasar valores booleanos

Si necesita especificar el valor de una opción que no acepta valores de cadena, como el indicador `--force` en el comando `migrate:refresh`, debe pasar `true` o `false` como el valor de la opción:

    $exitCode = Artisan::call('migrate:refresh', [
        '--force' => true,
    ]);

<a name="queueing-artisan-commands"></a>
#### Comandos artesanales en cola

Usando el método `queue` en la fachada de `Artisan`, puede incluso poner en cola los comandos de Artisan para que sean procesados ​​en segundo plano por sus [trabajadores de la cola](/docs/{{version}}/queues). Antes de usar este método, asegúrese de haber configurado su cola y de estar ejecutando un oyente de cola:

    use Illuminate\Support\Facades\Artisan;

    Route::post('/user/{user}/mail', function (string $user) {
        Artisan::queue('mail:send', [
            'user' => $user, '--queue' => 'default'
        ]);

        // ...
    });

Usando los métodos `onConnection` y `onQueue`, puede especificar la conexión o la cola a la que se debe enviar el comando Artisan:

    Artisan::queue('mail:send', [
        'user' => 1, '--queue' => 'default'
    ])->onConnection('redis')->onQueue('commands');

<a name="calling-commands-from-other-commands"></a>
### Llamar comandos desde otros comandos

En ocasiones, es posible que desee llamar a otros comandos desde un comando Artisan existente. Puede hacerlo utilizando el método `call`. Este método `call` acepta el nombre del comando y una serie de argumentos/opciones del comando:

    /**
     * Ejecute el comando de la consola.
     */
    public function handle(): void
    {
        $this->call('mail:send', [
            'user' => 1, '--queue' => 'default'
        ]);

        // ...
    }

Si desea llamar a otro comando de la consola y suprimir toda su salida, puede usar el método `callSilently`. El método `callSilently` tiene la misma firma que el método `call`:

    $this->callSilently('mail:send', [
        'user' => 1, '--queue' => 'default'
    ]);

<a name="signal-handling"></a>
## Manejo de señales

Como sabrá, los sistemas operativos permiten que se envíen señales a los procesos en ejecución. Por ejemplo, la señal `SIGTERM` es cómo los sistemas operativos le piden a un programa que termine. Si desea escuchar señales en los comandos de su consola Artisan y ejecutar código cuando ocurran, puede usar el método `trap`:

    /**
     * Ejecute el comando de la consola.
     */
    public function handle(): void
    {
        $this->trap(SIGTERM, fn () => $this->shouldKeepRunning = false);

        while ($this->shouldKeepRunning) {
            // ...
        }
    }

Para escuchar varias señales a la vez, puede proporcionar una serie de señales al método `trap`:

    $this->trap([SIGTERM, SIGQUIT], function (int $signal) {
        $this->shouldKeepRunning = false;

        dump($signal); // SIGTERM / SIGQUIT
    });

<a name="stub-customization"></a>
## Personalización de talones

Los comandos `make` de la consola Artisan se utilizan para crear una variedad de clases, como controladores, trabajos, migraciones y pruebas. Estas clases se generan utilizando archivos "stub" que se completan con valores basados ​​en su entrada. Sin embargo, es posible que desee realizar pequeños cambios en los archivos generados por Artisan. Para lograr esto, puede usar el comando `stub:publish` para publicar los stubs más comunes en su aplicación para que pueda personalizarlos:

```shell
php artisan stub:publish
```

Los stubs publicados se ubicarán dentro de un directorio `stubs` en la raíz de su aplicación. Cualquier cambio que realice en estos stubs se reflejará cuando genere sus clases correspondientes utilizando los comandos `make` de Artisan.

<a name="events"></a>
## Eventos

Artisan envía tres eventos cuando ejecuta comandos: `Illuminate\Console\Events\ArtisanStarting`, `Illuminate\Console\Events\CommandStarting`, y `Illuminate\Console\Events\CommandFinished`. El evento `ArtisanStarting` se envía inmediatamente cuando Artisan comienza a ejecutarse. A continuación, el evento `CommandStarting` se envía inmediatamente antes de que se ejecute un comando. Finalmente, el evento `CommandFinished` se envía una vez que un comando termina de ejecutarse.
