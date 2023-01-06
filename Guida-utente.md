### Guida Utente

L'applicativo, al momento, è disponibile unicamente per sistemi desktop (Windows, macOS, UNIX-based).

Avviando l'applicazione //TODO
Spiegare il funzionamento //TODO

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