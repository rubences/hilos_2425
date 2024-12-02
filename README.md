# Trabajo con hilos paso a paso: Trabajando con Executors Services

Es un concepto a priori sencillo. A menudo a los desarrolladores de software se nos presenta el problema de, más allá de conseguir que una aplicación funcione correctamente, que lo haga de manera más rápida para satisfacer los requisitos del cliente. En su día ese fue mi caso, y quiero compartir ciertas nociones básicas sobre la concurrencia en Java, de una manera práctica y basada en ejemplos, que pueda servir como guía inicial.

Para introducirnos en el tema conviene diferenciar los conceptos de concurrencia y paralelismo. Concurrencia se da cuando dos o más tareas se desarrollan en el mismo intervalo de tiempo, pero que no necesariamente están progresando en el mismo instante. Es un concepto más general que el paralelismo, el cual consiste en llevar a cabo multitareas en el mismo instante literalmente.

Existen dos conceptos básicos asociados a la concurrencia:

Proceso: es un programa en ejecución. Tiene su propio espacio de memoria, enlaces a recursos, I/O... Los procesos están aislados entre sí.
Hilo: es un camino de ejecución dentro de un proceso. Cada proceso tiene al menos un hilo, llamado hilo principal. Los hilos comparten los recursos del proceso, incluida la memoria, por lo que pueden comunicarse entre sí. Cada hilo tiene su propia callstack.
Los agentes que ejecutan dichos procesos e hilos son los procesadores (CPU).

Una vez que hemos decidido optimizar nuestro programa mediante concurrencia, y aunque no es estrictamente obligatorio conocer el nivel de paralelismo máximo que ofrece la máquina para la cual desarrollamos, es muy importante para hacer previsiones de optimización o para contrastar los resultados en pruebas.

Cada placa base de una computadora dispone de uno o varios sockets donde insertar procesadores. El número de “cores” o procesadores físicos puede ser uno o varios. Si nuestro equipo tiene un socket y n cores, significaría que se podrían ejecutarse literalmente n procesos realmente en paralelo. Más adelante explicaremos un matiz a esta afirmación. Por otro lado, existe una tecnología que permite que un procesador físico se convierta en dos procesadores lógicos. A efectos prácticos el sistema operativo cree que dispone el doble de procesadores. A nivel de rendimiento esta tecnología que “engaña” al SO no iguala la potencia real de dos procesadores físicos.

coresInfo

 

Como caso práctico, se puede ver en la información del equipo que dispone de un socket, que contiene 4 cores y éstos duplican su número a 8 procesadores lógicos. El nivel de paralelismo es de 8 (1*4*2).

Retomando la distinción entre los conceptos de concurrencia y paralelismo, en una computadora con un solo core podría llevarse a cabo el primero, pero solo en una multicore el segundo.

 

Una vez conocido dicho nivel de paralelismo, podríamos cuestionarnos cómo un sistema operativo normalmente tiene numerosos procesos ejecutándose a la vez. El SO se encarga de administrar los tiempos de ejecución de esos procesos en los CPUs que tiene a su disposición. Aunque puede parecer contradictorio, las computadoras pueden ejecutar muchas más tareas que la cantidad de procesadores disponibles. En realidad, no se ejecutan en el mismo instante físico. Las tareas se ejecutan de forma intercalada, pero el sistema operativo cambia entre las tareas con tanta frecuencia que a los usuarios les parece que se están ejecutando en el mismo instante físico.

 
Tras esta breve introducción teórica, pasemos al código…

¿Cómo se crea un hilo y se ejecuta una tarea? Para crear un hilo existen dos opciones:

Creando una clase que extienda a ‘Thread’ y sobrescribiendo el método ‘run()’. El contenido de ese método será la tarea que se realice cuando una instancia de esa clase ejecute el método ‘start()’.
Creando un objeto que implemente la interfaz ‘Runnable’ y pasándoselo al constructor de ‘Thread’. ‘Runnable’ es una interfaz funcional que define un único método ‘run()’ que no devuelve nada. Esta manera es más flexible y menos acoplada, será la que usemos en esta guía.
En el siguiente ejemplo se crea un hilo al que se le pasa una tarea y, posteriormente, se ejecuta:

package com.softtek;

import java.time.Duration;
import java.time.Instant;

public class ThreadRunnable {
	
	private static final Instant INICIO = Instant.now(); 

	public static void main(String[] args) {
		
		Runnable tarea = () -> {
			try {
				Log("Empieza la tarea");
				Thread.sleep(5000);
				Log("Termina la tarea");
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		};
		
		Thread hilo = new Thread(tarea);
		
		hilo.start();
		
		try {
			Log("Se empieza a esperar al hilo");
			hilo.join(3000);
			Log("Se termina de esperar al hilo");
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ThreadRunnable Output
	PT0.062S [main] Se empieza a esperar al hilo
	PT0.062S [Thread-0] Empieza la tarea
	PT3.125S [main] Se termina de esperar al hilo
	PT5.128S [Thread-0] Termina la tarea
*/
Una vez que el hilo se lanza, la ejecución de la tarea se lleva a cabo en el ‘Thread-0’, mientras el código sigue en el hilo principal ‘main’. Para simular una tarea real (que puede ser un cálculo pesado, una consulta de base de datos, etc…) se define un ‘sleep()’ de 5 segundos. Para que el principal espere (quede bloqueado) hasta que el hilo termine su tarea se usa el método ‘join()’. Este método tiene una variante a la cual se le puede asignar un timeout y, si sobrepasa ese tiempo, deja de esperar. En este ejemplo se han puesto 3 segundos.

Al analizar el resultado obtenido puede ser llamativo que la traza ‘Se empieza a esperar al hilo’ sea anterior a la ‘Empieza la tarea’. Esto se debe a que una vez lanzado, el hilo se desacopla en paralelo, continuando el código por el camino principal. No se podría prever con certeza cuál de las dos trazas se pintará antes.

Se mencionan escuetamente los métodos 'wait()', que deja bloqueado al hilo que lo llama, y el método 'notify()', que desbloquea a los hilos bloqueados por 'wait()', debido a su relación con la concurrencia. Ambos son de bajo nivel y existen abstracciones más fáciles de usar.

En este punto ya sabemos cómo crear un hilo y lanzar una tarea que corra sobre el mismo. La creación y eliminación de hilos tiene cierto coste computacional, siendo compleja la administración por parte del usuario. Para dar solución a este inconveniente y añadir numerosas funcionalidades, la API de Java, en su paquete ‘java.util.concurrent’, nos ofrece un framework que pasamos a explicar a continuación.

 

Executor es un framework para la ejecución de tareas de manera concurrente. Desacopla la creación y manejo de los hilos de las tareas:

ExecutorFramework

 

Se encarga de crear un pool de hilos que llevarán a cabo las tareas. Desde fuera simplemente enviamos las tareas que se deseen realizar y éstas se añaden a una cola. El Executor se encarga de pasarlas a ejecución, gestionando la utilización de los hilos disponibles. Desde el lado de las tareas no hay que preocuparse de la creación de los mismos, de si están libres u ocupados, o de cerrarlos. 

 

A modo esquemático, y sin entrar en profundidad, la API de Java ofrece la siguiente implementación del framework:

InterfacesExecutor

 

Executor: una sencilla interfaz con el método ‘execute(Runnable)’. Es la definición mínima del framewok, se encarga de ejecutar una tarea.
ExecutorService: añade funcionalidades para el manejo del ciclo de vida de las tareas.
ScheduledExecutorService: añade funcionalidades para la programación de las tareas.
 

Se explicará el uso de este framework, las principales implementaciones y sus métodos con una serie de ejercicios.

En este primer ejemplo vamos a crear un Executor y utilizarlo para que nos ejecute una tarea Runnable.

package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ExecutorServiceRunnable {
	
	private static final Instant INICIO = Instant.now(); 
	
	public static void main(String[] args) {
		
		ExecutorService executor = Executors.newSingleThreadExecutor();
		
		Runnable tarea= ()->{
			Log("Inicio de la tarea");
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			Log("Finaliza la tarea");
		};
		
		executor.submit(tarea);
		executor.shutdown();
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ExecutorServiceRunnable Output
	PT0.078S [pool-1-thread-1] Inicio de la tarea
	PT1.125S [pool-1-thread-1] Finaliza la tarea
*/
Hemos utilizado la implementación de ExecutorService de un solo hilo ayudándonos de la clase auxiliar Executors. Esta clase es una factoría de executors y tiene métodos de utilidad.

Es importante apagar el executor una vez que ya no se vaya a usar más, sino se quedará activo esperando indefinidamente consumiendo recursos.

 

¿Y si quisiéramos que la tarea devolviese un resultado? La solución es la interfaz Callable<V>. Esta interfaz funcional define el método 'call()' que devuelve un resultado V. El funcionamiento es análogo a un Runnable pero cuando la tarea finaliza devuelve el resultado. Cuando se le envía una tarea de este tipo al Executor, éste devuelve un objeto Future<V>, el cual representa un enlace a la tarea, mediante el cual puedes saber si la tarea ha terminado, obtener su resultado o cancelarla.

package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

public class ExecutorServiceCallable {
	
	private static final Instant INICIO = Instant.now(); 
	
	public static void main(String[] args) throws InterruptedException, ExecutionException {
		
		ExecutorService executor = Executors.newSingleThreadExecutor();
	
		Callable<String> tarea= ()->{
			Log("Inicio de la tarea");
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			Log("Finaliza la tarea");
			return "Resultado de la tarea";
		};
		
		Future<String> future = executor.submit(tarea);
		
		Log(future.isDone());
		String resultado = future.get();
		Log(future.isDone());

		Log(resultado);
		executor.shutdown();
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ExecutorServiceCallable Output
	PT0.068S [pool-1-thread-1] Inicio de la tarea
	PT0.068S [main] false
	PT2.125S [pool-1-thread-1] Finaliza la tarea
	PT2.125S [main] true
	PT2.127S [main] Resultado de la tarea
*/
En el código de ejemplo podemos ver que para obtener el resultado llamamos al método ‘get()’. Es importante tener en cuenta que este método bloquea el hilo principal hasta que la tarea termine o, en otras palabras, las líneas de código posteriores no seguirán su ejecución hasta una vez obtenido el resultado.

 

Para apagar un Executor existen dos formas. Haciendo una llamada a ‘shutdown()’, el executor deja de aceptar nuevas tareas y cuando las finaliza se cierra, es decir, espera a que las tareas actualmente en ejecución o en la cola se realicen. Mediante ‘shutdownNow()’ se paran de manera abrupta, finalizando en ese mismo momento el executor. En la siguiente pieza de código veremos un método asociado a ‘shutdown()’, el awaitTermination().

package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ExecutorServiceAwaitTermination {
	
	private static final Instant INICIO = Instant.now(); 
	
	public static void main(String[] args) throws InterruptedException {
		
		ExecutorService executor = Executors.newSingleThreadExecutor();
		
		Runnable tarea= ()->{
			Log("Inicio de la tarea");
			try {
				TimeUnit.SECONDS.sleep(5);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			Log("Finaliza la tarea");
		};
		
		executor.submit(tarea);
		
		executor.shutdown();
		executor.awaitTermination(2, TimeUnit.SECONDS);
		
		Log("El hilo principal continúa...");
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ExecutorServiceAwaitTermination Output
	--Sin awaitTermination
	PT0.078S [pool-1-thread-1] Inicio de la tarea
	PT0.078S [main] El hilo principal continúa...
	PT5.126S [pool-1-thread-1] Finaliza la tarea
	
	--Con awaitTermination: 2seg
	PT0.068S [pool-1-thread-1] Inicio de la tarea
	PT2.075S [main] El hilo principal continúa...
	PT5.102S [pool-1-thread-1] Finaliza la tarea
	
	--Con awaitTermination: 7seg
	PT0.072S [pool-1-thread-1] Inicio de la tarea
	PT5.151S [pool-1-thread-1] Finaliza la tarea
	PT5.151S [main] El hilo principal continúa...
*/
Es importante tener en cuenta que haciendo ‘shutdown()’ no se bloquea el código, las tareas que hayan quedado pendientes en los respectivos hilos terminarán su ejecución de manera asíncrona, pero el hilo principal del executor por su parte también continuará. ¿Y si se desease bloquear el hilo principal hasta que esas tareas terminasen? La api ofrece para ello el método awaitTermination(timeout), el cual bloquea el código hasta que las tareas terminen o se acabe el tiempo de timeout declarado, lo primero que ocurra de ambas.

 

Hasta ahora hemos visto como utilizar el framework para tareas individuales, pero también ofrece la posibilidad de gestionar grupos de tareas:

invokeAll: se envía una colección de tareas al executor y éste devuelve una lista con los correspondientes futures. Esta llamada es bloqueante, es decir, se espera a que todas las tareas estén completadas (ya sea de manera exitosa o devolviendo excepción).
invokeAny: se envía una colección de tareas al executor y éste devuelve la primera tarea completada de manera exitosa, si es que la hubiera. Si ninguna terminara correctamente se devolvería una ExecutionException. En el momento que el executor valida un resultado exitoso, cancela el resto de tareas en ejecución. Este método también es bloqueante.
package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class ExecutorServiceInvokeAllAny {
	
	private static final Instant INICIO = Instant.now();
	private static int contadorTareas = 1;

	public static void main(String[] args) throws InterruptedException, ExecutionException {
		
		List<Callable<String>> tareas= 
				Stream.generate(ExecutorServiceInvokeAllAny::getTareaSleepUnSegundo)
					.limit(5)
					.collect(Collectors.toList());
		
		ExecutorService executor = Executors.newFixedThreadPool(2);
		
		List<Future<String>> futures = executor.invokeAll(tareas);
		for(Future<String> future : futures) {
			String resultado= future.get();
			Log(resultado);
		}
		
		Log("El hilo principal continúa...");
		
		String resultadoAny = executor.invokeAny(tareas);
		Log(resultadoAny);
		
		Log("El hilo principal continúa...");
		executor.shutdown();
	}
	
	private static Callable<String> getTareaSleepUnSegundo() {
		int numeroTarea = contadorTareas++;
		
		return ()->{
			Log("Inicio de la tarea " + numeroTarea);
			try {
				TimeUnit.SECONDS.sleep(1);
				Log("Finaliza la tarea " + numeroTarea);
				return "resultado de la tarea " + numeroTarea;
			} catch (InterruptedException e) {
				Log("sleep ha sido interrumpido en tarea " + numeroTarea);
				return null;
			}
		};
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ExecutorServiceInvokeAllAny Output
	--invokeAll
	PT0.084S [pool-1-thread-2] Inicio de la tarea 2
	PT0.084S [pool-1-thread-1] Inicio de la tarea 1
	PT1.116S [pool-1-thread-2] Finaliza la tarea 2
	PT1.116S [pool-1-thread-1] Finaliza la tarea 1
	PT1.116S [pool-1-thread-2] Inicio de la tarea 3
	PT1.116S [pool-1-thread-1] Inicio de la tarea 4
	PT2.12S [pool-1-thread-1] Finaliza la tarea 4
	PT2.12S [pool-1-thread-2] Finaliza la tarea 3
	PT2.12S [pool-1-thread-1] Inicio de la tarea 5
	PT3.123S [pool-1-thread-1] Finaliza la tarea 5
	PT3.123S [main] resultado de la tarea 1
	PT3.125S [main] resultado de la tarea 2
	PT3.126S [main] resultado de la tarea 3
	PT3.126S [main] resultado de la tarea 4
	PT3.126S [main] resultado de la tarea 5
	PT3.127S [main] El hilo principal continúa...
	--invokeAny
	PT3.127S [pool-1-thread-2] Inicio de la tarea 1
	PT3.127S [pool-1-thread-1] Inicio de la tarea 2
	PT4.128S [pool-1-thread-1] Finaliza la tarea 2
	PT4.128S [pool-1-thread-2] Finaliza la tarea 1
	PT4.128S [pool-1-thread-1] Inicio de la tarea 3
	PT4.128S [main] resultado de la tarea 2
	PT4.129S [pool-1-thread-1] sleep ha sido interrumpido en tarea 3
	PT4.129S [main] El hilo principal continúa...
*/
En las trazas del ejercicio puede verse que ambas tareas bloquean el código del hilo principal. También puede observarse que invokeAny ejecuta tareas, cuando valida que tiene una completada exitosa cancela el resto que aún están ejecutándose.

 

Hasta ahora hemos enviado nuestras tareas al Executor dejando en sus manos el paso a ejecución, pero podríamos necesitar programar en el tiempo cuándo realizar la tarea. La solución a este problema es la interfaz ScheduleExecutorService, la cual añade las siguientes funcionalidades:

schedule: empieza a ejecutar la tarea tras un tiempo determinado.
scheduleAtFixedRate: lanza secuencialmente la misma tarea cada intervalo de tiempo definido. Es independiente de lo que la tarea tarde en acabarse. También se puede definir un delay inicial.
scheduleWithFixedDelay: lanza secuencialmente la misma tarea cada intervalo de tiempo definido una vez acabada la instancia previa. Espera a que una tarea se acabe para comenzar el intervalo de espera e iniciar la siguiente. También se puede definir un delay inicial.
package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledExecutorServiceExample {
	
	private static final Instant INICIO = Instant.now(); 
	
	public static void main(String[] args) {
		
		ScheduledExecutorService scheduledExecutor = Executors.newScheduledThreadPool(1);
		
		Runnable tarea= ()->{
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			Log("Ejecución de tarea");
		};
		
		scheduledExecutor.schedule(tarea, 3, TimeUnit.SECONDS);
		scheduledExecutor.scheduleAtFixedRate(tarea, 2, 1, TimeUnit.SECONDS);
		scheduledExecutor.scheduleWithFixedDelay(tarea, 2, 1, TimeUnit.SECONDS);
		
		try {
			TimeUnit.SECONDS.sleep(10);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		scheduledExecutor.shutdown();
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s [%s] %s", 
			Duration.between(INICIO, Instant.now()), Thread.currentThread().getName(), mensaje.toString()));
	}
}

/* ScheduledExecutorServiceExample Output
	--schedule(tarea, 3, TimeUnit.SECONDS);
	PT4.094S [pool-1-thread-1] Ejecución de tarea
	
	--scheduleAtFixedRate(tarea, 2, 1, TimeUnit.SECONDS);
	PT3.094S [pool-1-thread-1] Ejecución de tarea
	PT4.128S [pool-1-thread-1] Ejecución de tarea
	PT5.132S [pool-1-thread-1] Ejecución de tarea
	PT6.136S [pool-1-thread-1] Ejecución de tarea
	PT7.138S [pool-1-thread-1] Ejecución de tarea
	PT8.142S [pool-1-thread-1] Ejecución de tarea
	PT9.144S [pool-1-thread-1] Ejecución de tarea
	PT10.147S [pool-1-thread-1] Ejecución de tarea
	
	--scheduleWithFixedDelay(tarea, 2, 1, TimeUnit.SECONDS);
	PT3.094S [pool-1-thread-1] Ejecución de tarea
	PT5.146S [pool-1-thread-1] Ejecución de tarea
	PT7.165S [pool-1-thread-1] Ejecución de tarea
	PT9.183S [pool-1-thread-1] Ejecución de tarea
*/
 
En el ejemplo lanzamos las tres funcionalidades por separado, las salidas de consola corresponden a cada una individualmente (es decir, se comentan las demás en cada caso). Como el segundo y tercer caso serían de ejecución cíclica infinita, hemos definido un tope temporal al Executor de 10 segundos.

 

Con estos ejemplos hemos visto el funcionamiento de los Executors.

 

Ayudándonos de la clase de utilidad Executors, hemos creado una serie de tipos de Executors y se presenta la duda ¿Cuál elegir para mi programa? Para ello vamos a explicar las peculiaridades de cada uno de ellos en la siguiente lista, la cual explica el nivel de multithreading y a continuación indica (entre paréntesis) la interfaz que implementa:

newCachedThreadPool(): variable, crea tantos hilos nuevos como sea necesario, reutilizando los ya creados a medida que quedan libres y destruyendo los que están un tiempo sin actividad (ThreadPoolExecutor).
newFixedThreadPool(n): fijado por el usuario (ThreadPoolExecutor).
newSingleThreadExecutor(): fijado a 1 (ThreadPoolExecutor).
newWorkStealingPool(): según el número de procesadores lógicos disponibles (ForkJoinPool).
newScheduledThreadPool(n): fijado por el usuario (ScheduledThreadPoolExecutor).
commonPool(): según el número de procesadores lógicos disponibles menos 1 (ForkJoinPool).

En el siguiente código vamos a lanzar a los distintos Executors un grupo de 40 tareas consistentes en esperar 1 segundo.

package com.softtek;

import java.time.Duration;
import java.time.Instant;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class ExecutorServicesNumberOfThreads { 
	
	public static void main(String[] args) throws InterruptedException {

		Log(Runtime.getRuntime().availableProcessors());
		
		Log(ForkJoinPool.commonPool().getParallelism());
		
		List<ExecutorService> executorServices = Arrays.asList( 
			Executors.newCachedThreadPool(),
			Executors.newFixedThreadPool(3),
			Executors.newSingleThreadExecutor(),
			Executors.newWorkStealingPool(),
			Executors.newScheduledThreadPool(5),
			ForkJoinPool.commonPool());
		
		List<Callable<Object>> tareas= 
			Stream.generate(ExecutorServicesNumberOfThreads::getTareaSleepUnSegundo)
				.limit(40)
				.collect(Collectors.toList());
		
		for(ExecutorService executorService: executorServices) {
			Instant inicio= Instant.now();
			executorService.invokeAll(tareas);
			Log(Duration.between(inicio, Instant.now()));
		}
		
		executorServices.stream().forEach(ExecutorService::shutdown);
	}
	
	private static Callable<Object> getTareaSleepUnSegundo() {
		return Executors.callable(()->{ 
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});
	}
	
	private static void Log(Object mensaje) {
		System.out.println(String.format("%s", mensaje.toString()));
	}
}

/* ExecutorServicesNumberOfThreads Output
	8
	PT1.008S
	PT14.004S
	PT40.011S
	PT5.004S
	PT8.003S
	PT6.005S
*/
Desde código podemos conocer el número de procesadores lógicos disponibles mediante "Runtime.getRuntime().availableProcessors()". Los resultados que vemos en consola constatan el nivel de multihilo. Vemos cómo para el mismo grupo de tareas los tiempos son sensiblemente distintos. Será labor del desarrollador elegir uno u otro buscando un equilibrio entre tiempos, consumos de memoria, escalabilidad, etc…

