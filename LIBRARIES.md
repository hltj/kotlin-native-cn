# Kotlin/Native 库

## Kotlin compiler specifics

To produce a library with the Kotlin/Native compiler use the `-produce library` or `-p library` flag. For example:



```bash
$ kotlinc foo.kt -p library -o bar
```



the above command will produce a `bar.klib` with the compiled contents of `foo.kt`.

To link to a library use the `-library <name>` or `-l <name>` flag. For example:



```bash
$ kotlinc qux.kt -l bar
```




the above command will produce a `program.kexe` out of `qux.kt` and `bar.klib`


## cinterop tool specifics

The **cinterop** tool produces `.klib` wrappers for native libraries as its main output. 
For example, using the simple `libgit2.def` native library definition file provided in your Kotlin/Native distribution



```bash
$ cinterop -def  samples/gitchurn/src/main/c_interop/libgit2.def -compilerOpts -I/usr/local/include -o libgit2
```



we will obtain `libgit2.klib`.

See more details in [INTEROP.md](INTEROP.md)


## klib utility

The **klib** library management utility allows you to inspect and install the libraries.

The following commands are available.

To list library contents:



```bash
$ klib contents <name>
```



To inspect the bookkeeping details of the library 



```bash
$ klib info <name>
```



To install the library to the default location use



```bash
$ klib install <name>
```



To remove the library from the default repository use 



```bash
$ klib remove <name>
```



All of the above commands accept an additional `-repository <directory>` argument for specifying a repository different to the default one.



```bash
$ klib <command> <name> -repository <directory>
```




## Several examples

First let's create a library.
Place the tiny library source code into `kotlinizer.kt`:



```kotlin
package kotlinizer
val String.kotlinized
    get() = "Kotlin $this"
```

```bash
$ kotlinc kotlinizer.kt -p library -o kotlinizer
```



The library has been created in the current directory:



```bash
$ ls kotlinizer.klib
kotlinizer.klib
```



Now let's check out the contents of the library:



```bash
$ klib contents kotlinizer
```



We can install `kotlinizer` to the default repository:



```bash
$ klib install kotlinizer
```



Remove any traces of it from the current directory:



```bash
$ rm kotlinizer.klib
```



Create a very short program and place it into a `use.kt` :



```kotlin
import kotlinizer.*

fun main(args: Array<String>) {
    println("Hello, ${"world".kotlinized}!")
}
```



Now compile the program linking with the library we have just created:



```bash
$ kotlinc use.kt -l kotlinizer -o kohello
```



And run the program:



```bash
$ ./kohello.kexe
Hello, Kotlin world!
```



Have fun!

# Advanced topics

## Library search sequence

When given a `-library foo` flag, the compiler searches the `foo` library in the following order:

    * Current compilation directory or an absolute path.

    * All repositories specified with `-repo` flag.

    * Libraries installed in the default repository (For now the default is  `~/.konan`, however it could be changed by setting **KONAN_DATA_DIR** environment variable).

    * Libraries installed in `$installation/klib` directory.

## The library format

Kotlin/Native libraries are zip files containing a predefined 
directory structure, with the following layout:

**foo.klib** when unpacked as **foo/** gives us:

```yaml
  - foo/
    - targets/
      - $platform/
        - kotlin/
          - Kotlin compiled to LLVM bitcode.
        - native/
          - Bitcode files of additional native objects.
      - $another_platform/
        - There can be several platform specific kotlin and native pairs.
    - linkdata/
      - A set of ProtoBuf files with serialized linkage metadata.
    - resources/
      - General resources such as images. (Not used yet).
    - manifest - A file in *java property* format describing the library.
```

An example layout can be found in `klib/stdlib` directory of your installation.

