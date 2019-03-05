## Kotlin Coroutines

La programación asíncrona o no bloqueante es la nueva realidad. Ya sea que estemos creando aplicaciones de servidor, de escritorio o móviles, es importante que proporcionemos una experiencia que no solo sea fluida desde la perspectiva del usuario, sino que también sea escalable cuando sea necesario.

[Overview](https://kotlinlang.org/docs/reference/coroutines-overview.html)

En esta demo se uso el patron de arquitectura MVP y las siguientes bibliotecas:

 - [Coroutines](https://github.com/Kotlin/kotlinx.coroutines)
 - [Retrofit](https://github.com/square/retrofit)
 - [Gson Converter](https://github.com/square/retrofit/tree/master/retrofit-converters/gson#gson-converter)
 - [Kotlin Coroutines Adapter](https://github.com/JakeWharton/retrofit2-kotlin-coroutines-adapter)
 - [Gson](https://github.com/google/gson)
 - [Koin](https://github.com/InsertKoinIO/koin)

> No es necesario usar Koin para la implementacion de Kotlin Coroutines, pero se recomienda su uso.

 
## Implementación

Agregamos las dependencias a nuestro archivo build.gradle.

```javascript
dependencies {
	// Coroutines
	implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.0.1'  
	implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.0.1"
	// Retrofit
	implementation 'com.squareup.retrofit2:retrofit:2.5.0'
	// Gson Converter
	implementation 'com.squareup.retrofit2:converter-gson:2.5.0'  
	// Kotlin Coroutines Adapter
	implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter:0.9.2'
	// Koin 
	implementation 'org.koin:koin-androidx-scope:1.0.2'
}
```
Creamos nuestro modelo User:
```kotlin
class User (
	var id : Int,  
	var lastname : String,  
	var name : String  
)
```
Luego creamos nuestra interfaz Service donde agregaremos los servicios que vayamos a consumir, pero en el tipo de retorno de los métodos los asignaremos de esta manera:

```kotlin
Deferred<Response<List<User>>>
```
Donde nuestra interfaz deberia verse asi:
```kotlin
interface Service {  
  
    @GET("data.json")  
    fun getUsers(): Deferred<Response<List<User>>>  
}
```
Luego creamos nuestro **objeto** RestApi, donde crearemos una instancia de Retrofit, pero ademas de crearlo como normalmente lo creamos agregamos la siguiente linea de código:

```kotlin
.addCallAdapterFactory(CoroutineCallAdapterFactory())
```
Esta linea funcionara como un adaptador al instanciar Retrofit, donde nuestro objeto RestApi debería verse así:
```kotlin
object RestApi {  
  
    private lateinit var retrofit: Retrofit  
    private val okHttpClient = OkHttpClient  
        .Builder()  
        .readTimeout(10, TimeUnit.SECONDS)  
        .writeTimeout(10, TimeUnit.SECONDS)  
        .connectTimeout(10, TimeUnit.SECONDS)  
        .build()  
  
    fun instanceClient(): Service {  
        retrofit = Retrofit.Builder()  
            .baseUrl(BuildConfig.BASE_URL)  
            .client(okHttpClient)  
            .addConverterFactory(GsonConverterFactory.create())  
            .addCallAdapterFactory(CoroutineCallAdapterFactory())  
            .build()  
        return retrofit.create(Service::class.java)  
    }  
}
```
Al usar el patrón de arquitectura MVP, primero creamos nuestro controlador que debería verse así:
```kotlin
interface MainController {  
    interface View {  
        fun onLoadUserSuccessful(list : List<User>)
        fun onLoadUserFailed(message : String)  
    }  
  
    interface Presenter {  
        suspend fun loadUsers()  
    }  
}
```
Si observan al metodo **loadUsers** le agregue el prefijo **suspend**.
Una función de suspensión es simplemente una función que se puede pausar y reanudar posteriormente. Pueden ejecutar una operación de larga duración y esperar a que se complete sin bloqueo.

Luego creamos nuestro presentador donde implementaremos la interfaz **MainController.Presenter**, instanciaremos nuestro **Service** y declararemos la interfaz **MainController.View**, donde nuestro presentador debería verse así.

```kotlin
class MainPresenter(private val context: Context) : MainController.Presenter {  
  
    private val restApi = RestApi.instanceClient()  
    lateinit var controller: MainController.View  
  
    override suspend fun loadUsers() {  
    }  
}
```

Ahora es el momento de hacer uso de nuestra llamada al servicio con la siguiente linea de código:
```kotlin
restApi.getUsers().await()
```

Lo que hace el método await es esperar la finalización de este valor sin bloquear el hilo y se reanuda cuando se completa el cálculo diferido, es decir que pasara a ejecutarse la siguiente linea de código cuando el servicio responda.
Pero ahora te estarás preguntando: ¿Como es que verifico si es que el código del response fue exitoso o no? 
Pues déjame decirte que es tan fácil como esto:
```kotlin
val response = restApi.getUsers().await()  
if (response.isSuccessful) {  

}
```
Se que este no es el mejor de los casos, por que en el desarrollo normal de un aplicativo nos pedirán que validemos todos los casos de errores, entonces contando con estos casos nuestro código final se vería así:
```kotlin
override suspend fun loadUsers() {  
    try {  
        val response = restApi.getUsers().await()  
        if (response.isSuccessful) {
	        // Your logic
        } else {
	        // Your logic
        }  
    } catch (ex: Exception) {  
        // Your logic
    }  
}
```
En el ejemplo que estamos siguiendo, implementando las salidas de la interfaz y agregando el método **evaluateFailure** que nos ayudara a evaluar las excepciones, nuestro presentador quedaría así:
```kotlin
class MainPresenter(private val context: Context) : MainController.Presenter {  
  
    private val restApi = RestApi.instanceClient()  
    lateinit var controller: MainController.View  
  
    override suspend fun loadUsers() {  
        try {  
            val response = restApi.getUsers().await()  
            if (response.isSuccessful) {  
                controller.onLoadUserSuccessful(response.body()!!)  
            } else {  
                controller.onLoadUserFailed(context.getString(R.string.server_error_message))  
            }  
        } catch (ex: Exception) {  
            controller.onLoadUserFailed(evaluateFailure(context, ex))  
        }  
    }  
  
    private fun evaluateFailure(context: Context, t: Throwable): String {  
        return when (t) {  
            is UnknownHostException -> context.getString(R.string.connection_message)  
            is SocketTimeoutException -> context.getString(R.string.time_out_message)  
            is SSLHandshakeException -> context.getString(R.string.connection_lost_message)  
            else -> context.getString(R.string.default_error_message)  
        }  
    }  
}
```
Ahora que dejamos todo listo en nuestro presentador, es el turno de nuestro Activity, donde inyectaremos nuestro presentador e implementaremos la interfaz **MainController.View** y se vería así:
```kotlin
class MainActivity : AppCompatActivity(), MainController.View { 
 
	private val presenter: MainPresenter by inject()

    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)
        presenter.controller = this
    }  
    
    override fun onLoadUserSuccessful(list: List<User>) {  
    }  
  
    override fun onLoadUserFailed(message: String) {  
    }  
}
```

Creare un diseño sencillo para este ejemplo que sera una lista de items, donde los xml serian los siguientes:

**item_user.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
  android:layout_width="match_parent"  
  android:layout_height="wrap_content"  
  android:orientation="vertical"  
  android:paddingStart="12dp"  
  android:paddingTop="12dp"  
  android:paddingEnd="16dp">  
  
	 <TextView  
		 android:layout_width="match_parent"  
		 android:layout_height="wrap_content"  
		 android:text="Nombre:"  
		 android:textColor="@color/black"  
		 android:textSize="16sp" />  
	  
	 <TextView  
		 android:id="@+id/nameText"  
		 android:layout_width="match_parent"  
		 android:layout_height="wrap_content"  
		 android:textColor="@color/black" />  
		  
	 <TextView  
		 android:layout_width="match_parent"  
		 android:layout_height="wrap_content"  
		 android:layout_marginTop="4dp"  
		 android:text="Apellido:"  
		 android:textColor="@color/black"  
		 android:textSize="16sp" />  
	  
	 <TextView  
		 android:id="@+id/lastNameText"  
		 android:layout_width="match_parent"  
		 android:layout_height="wrap_content"  
		 android:textColor="@color/black" />  
	  
	 <View  
		 android:layout_width="match_parent"  
		 android:layout_height="2dp"  
		 android:layout_marginTop="12dp"  
		 android:background="@color/black" />  
  
</LinearLayout>
```
**activity_main.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>  
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"  
  xmlns:app="http://schemas.android.com/apk/res-auto"  
  xmlns:tools="http://schemas.android.com/tools"  
  android:layout_width="match_parent"  
  android:layout_height="match_parent"  
  tools:context=".activity.MainActivity">  
  
	 <androidx.recyclerview.widget.RecyclerView
	  android:id="@+id/userRecycler"  
	  android:layout_width="0dp"  
	  android:paddingBottom="4dp"  
	  android:layout_height="0dp"  
	  app:layout_constraintBottom_toBottomOf="parent"  
	  app:layout_constraintLeft_toLeftOf="parent"  
	  app:layout_constraintRight_toRightOf="parent"  
	  app:layout_constraintTop_toTopOf="parent" />  
	  
	 <ProgressBar  
	  android:id="@+id/progressBar"  
	  android:layout_width="wrap_content"  
	  android:layout_height="wrap_content"  
	  app:layout_constraintBottom_toBottomOf="parent"  
	  app:layout_constraintLeft_toLeftOf="parent"  
	  app:layout_constraintRight_toRightOf="parent"  
	  app:layout_constraintTop_toTopOf="parent" />  
  
</androidx.constraintlayout.widget.ConstraintLayout>
```
Ahora que tenemos el diseño hecho, toca el ultimo paso para finalizar la implementación que seria

```kotlin

```
