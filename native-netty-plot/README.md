# Isolates for GraalVM Native Images

This application demonstrates the use of _isolates_ with [GraalVM Native Image](https://www.graalvm.org/latest/reference-manual/native-image/).
The code implements a web service that renders plots of mathematical functions, such as _sin(x)_.
For each requested plot, the web service spawns an isolate, which is an independent execution context.
The isolate generates the plot, and after it returns the result to the web service, it is discarded as a whole, along with any memory it used.

The code of this repository is based on an [earlier example](https://github.com/cstancu/netty-native-demo) for using `native-image` to build a web server using the [Netty](http://netty.io/) framework.
This example further uses the [exp4j](https://www.objecthunter.net/exp4j/) library to parse mathematical expressions, and [JFreeSVG](http://www.jfree.org/jfreesvg/) to generate SVG images.

## Preparation

1. Download and install the latest GraalVM JDK with Native Image using the [GraalVM JDK Downloader](https://github.com/graalvm/graalvm-jdk-downloader):
    ```bash
    bash <(curl -sL https://get.graalvm.org/jdk)
    ```
2. Download or clone the repository and navigate into the `native-netty-plot` directory:
    ```bash
    git clone https://github.com/graalvm/graalvm-demos
    ```
    ```bash
    cd graalvm-demos/native-netty-plot
    ```
For compilation, the `native-image` depends on the local toolchain.
Please make sure that `glibc-devel`, `zlib-devel` (header files for the C library and zlib), and `gcc` are available on your system. Some Linux distributions may additionally require `libstdc++-static`.
See [Prerequisites for Native Image](https://www.graalvm.org/latest/reference-manual/native-image/).

## Build the Project

The example is built with Maven:
  ```bash
  mvn package
  ```

This creates a JAR file with all dependencies embedded: `target/netty-plot-0.1-jar-with-dependencies.jar`.

The [pom.xml](pom.xml) specifies the expected dependencies to Netty, emp4j, and JFreeSVG.
It also specifies a dependency on Native Image, `org.graalvm.nativeimage`, - a technology to ahead-of-time compile Java code into a standalone executable.
This dependency is required for the [method substitutions that are necessary for Netty](https://github.com/cstancu/netty-native-demo).

## Generate a Native Executable

If the application is expected to use some dynamic features at run time (e.g., Reflection, Java Native Interface, Class Path Resources), they have to be provided to the `native-image` tool in the form of configuration files.
To avoid writing the configuration file yourself, apply the [tracing agent](https://www.graalvm.org/latest/reference-manual/native-image/metadata/AutomaticMetadataCollection/) when running on the Java HotSpot VM.
It will observe the application behavior and create configuration files (_jni-config.json_, _reflect-config.json_, _proxy-config.json_ and _resource-config.json_) in the _META-INF/native-image_ directory on the class path.
The _reflect-config.json_ file specifies classes which must be available via Java reflection at runtime.

1. Run the application on GraalVM JDK applying the tracing agent:
    ```bash
    java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image -jar target/netty-plot-0.1-jar-with-dependencies.jar
    ```

2. Terminate the application, `CTRL+C`.

3. Build a native executable. The `native-image` builder will automatically search for any configuration file under _META-INF/native-image_ and its subdirectories:
    ```bash
    native-image -jar target/netty-plot-0.1-jar-with-dependencies.jar
    ```
    The result is an executable file that is around 16 MByte in size:
    ```bash
    du -h netty-plot
    16M    netty-plot
    ```

4. You can now run the executable:
    ```bash
    ./netty-plot
    ```
    Open your web browser and navigate to http://127.0.0.1:8080/

5. Finally, you can open your browser and request rendering of a function, for example, by browsing to `http://127.0.0.1:8080/?function=abs((x-31.4)sin(x-pi/2))&xmin=0&xmax=31.4`.

### Background Information

Instead of specifying any additional parameters on the command line, they may provided in a properties file in the input JAR file.
The `native-image` builder automatically looks for files named `native-image.properties` and for any other configuration file under `META-INF/native-image` including subdirectories, and processes their contents.
The tracing agent writes the _reflect-config.json_ file specifying classes which must be available via Java reflection at run time.

With Maven projects, the path convention is `META-INF/native-image/${groupId}/${artifactId}/native-image.properties`.
In this example, the `META-INF/native-image/com.oracle.substratevm/netty-plot/native-image.properties` file contains the following:
```bash
ImageName = netty-plot
Args = --link-at-build-time
```
The `ImageName` property specifies the name of the resulting executable, while `Args` are treated like additional command-line arguments.

### A note about the application

This example cannot run as a regular Java application (on the JVM) and it cannot be profiled.
It will fail because the program tries to create an [isolate which is a Native Image specific feature](https://medium.com/graalvm/isolates-and-compressed-references-more-flexible-and-efficient-memory-management-for-graalvm-a044cc50b67e).
