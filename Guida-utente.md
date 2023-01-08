### Guida Utente

L'applicativo, al momento, è disponibile unicamente per sistemi desktop (Windows, macOS, UNIX-based).

### Guida per lo sviluppatore

Il framework è stato caricato su [Jitpack](https://jitpack.io/#org.bitbucket.Cava99/ddos-framework) in modo da poter essere aggiunto più facilmente come dipendenza in sistemi come Maven, Gradle e SBT.

Esempio di uso su Gradle:

```kotlin
allprojects {
		repositories {
			...
			maven { url 'https://jitpack.io' }
		}
	}
```

```kotlin
dependencies {
	implementation 'org.bitbucket.Cava99:ddos-framework:Tag'
}
```

### Eseguire la demo
Una volta scaricato il progetto sarà sufficiente buildarlo ed assicurarsi che tutte le dipedenze all'interno del file build.sbt siano state importate correttamente; successivamente eseguire il commando compile all'interno della sbt shell.
Per avviare l'interfaccia grafica è sufficiente eseguire il metodo main dentro l'oggetto Main
È possibile personalizzare la demo modificando il contenuto di quest'ultimo metodo
