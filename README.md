# <img src="zonky.jpg" height="100"> Embedded Postgres Binaries

## Introduction

This project provides lightweight bundles of PostgreSQL binaries with reduced size that are intended for testing purposes.
It is a supporting project for the primary [io.zonky.test:embedded-database-spring-test](https://github.com/zonkyio/embedded-database-spring-test) and [io.zonky.test:embedded-postgres](https://github.com/zonkyio/embedded-postgres) projects.
However, with a little effort it can be also applicable with [com.opentable:otj-pg-embedded](https://github.com/opentable/otj-pg-embedded) and maybe some other projects. 

## Provides

* Lightweight bundles of PostgreSQL binaries with reduced size (~10MB)
* Embedded PostgreSQL 11+ binaries even for Linux platform
* Configurable version of PostgreSQL binaries

## Use with [embedded-database-spring-test](https://github.com/zonkyio/embedded-database-spring-test) or [embedded-postgres](https://github.com/zonkyio/embedded-postgres) project

All necessary dependencies are already included in these projects, so no further action is required.
However, you can change the version of the postgres binaries by following the instructions described in [Postgres version](#postgres-version).

## Use with [com.opentable:otj-pg-embedded](https://github.com/opentable/otj-pg-embedded) project

First, you have to add any of the [available dependencies](https://mvnrepository.com/artifact/io.zonky.test.postgres) to your Maven configuration:

```xml
<dependency>
    <groupId>io.zonky.test.postgres</groupId>
    <artifactId>embedded-postgres-binaries-linux-amd64</artifactId>
    <version>11.11.0</version>
    <scope>test</scope>
</dependency>
```

Then you need to implement a custom [PgBinaryResolver](https://github.com/opentable/otj-pg-embedded/blob/master/src/main/java/com/opentable/db/postgres/embedded/PgBinaryResolver.java): 
```java
public class CustomPostgresBinaryResolver implements PgBinaryResolver {
    public InputStream getPgBinary(String system, String architecture) throws IOException {
        ClassPathResource resource = new ClassPathResource(format("postgres-%s-%s.txz", system, architecture));
        return resource.getInputStream();
    }
}
```

<details>
  <summary>Alpine variant</summary>
  
  ```java
  public class CustomPostgresBinaryResolver implements PgBinaryResolver {
      public InputStream getPgBinary(String system, String architecture) throws IOException {
          ClassPathResource resource = new ClassPathResource(format("postgres-%s-%s-alpine_linux.txz", system, architecture));
          return resource.getInputStream();
      }
  }
  ```

</details><br/>

And finally register it to the junit rule.

```java
@Rule
public SingleInstancePostgresRule pg = EmbeddedPostgresRules.singleInstance()
        .customize(builder -> builder.setPgBinaryResolver(new CustomPostgresBinaryResolver()));
```

## Use with [pg-embedded](https://pg-embedded.softwareforge.de/)

`pg-embedded` is a Java 11+ library that uses zonky.io Postgres binaries to spin up embedded PostgreSQL instances. 

It can be used [with any build tool](https://pg-embedded.softwareforge.de/4.0/dependency-info.html) and supports automatic binary selection through properties or included builders:

```java
try (DatabaseManager manager = DatabaseManager.multiDatabases()
    .withInstancePreparer(b -> b.setServerVersion("14"))
    .build()
    .start()) {
    DatabaseInfo databaseInfo = manager.getDatabaseInfo();
    // connected to PostgreSQL 14.x
    try (Connection c = databaseInfo.asDataSource().getConnection();
        Statement s = c.createStatement()) {
        try (ResultSet rs = s.executeQuery("SELECT 1")) {
            assertTrue(rs.next());
            assertEquals(1, rs.getInt(1));
            assertFalse(rs.next());
        }
    }
}
```

`pg-embedded` supports JUnit 5:

```java
@RegisterExtension
public static EmbeddedPgExtension multiDatabase = MultiDatabaseBuilder.instanceWithDefaults().build();
```

## Postgres version

The version of the postgres binaries can be managed by importing `embedded-postgres-binaries-bom` in a required version into your dependency management section.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.zonky.test.postgres</groupId>
            <artifactId>embedded-postgres-binaries-bom</artifactId>
            <version>13.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

A list of all available versions of postgres binaries is here: https://mvnrepository.com/artifact/io.zonky.test.postgres/embedded-postgres-binaries-bom

## Supported architectures

By default, only dependencies for `amd64` architecture, in the [io.zonky.test:embedded-database-spring-test](https://github.com/zonkyio/embedded-database-spring-test) and [io.zonky.test:embedded-postgres](https://github.com/zonkyio/embedded-postgres) projects, are included.
Support for other architectures can be enabled by adding the corresponding Maven dependencies as shown in the example below.

```xml
<dependency>
    <groupId>io.zonky.test.postgres</groupId>
    <artifactId>embedded-postgres-binaries-linux-i386</artifactId>
    <scope>test</scope>
</dependency>
```

**Supported platforms:** `Darwin`, `Windows`, `Linux`, `Alpine Linux`  
**Supported architectures:** `amd64`, `i386`, `arm32v6`, `arm32v7`, `arm64v8`, `ppc64le`

Note that not all architectures are supported by all platforms, look here for an exhaustive list of all available artifacts: https://mvnrepository.com/artifact/io.zonky.test.postgres
  
Since `PostgreSQL 10.0`, there are additional artifacts with `alpine-lite` suffix. These artifacts contain postgres binaries for Alpine Linux with disabled [ICU support](https://blog.2ndquadrant.com/icu-support-postgresql-10/) for further size reduction.

## Building from Source
The project uses a [Gradle](http://gradle.org)-based build system. In the instructions
below, [`./gradlew`](http://vimeo.com/34436402) is invoked from the root of the source tree and serves as
a cross-platform, self-contained bootstrap mechanism for the build.

### Prerequisites

[Git](http://help.github.com/set-up-git-redirect), [JDK 6 or later](http://www.oracle.com/technetwork/java/javase/downloads) and [Docker](https://www.docker.com/get-started)

Be sure that your `JAVA_HOME` environment variable points to the `jdk1.6.0` folder
extracted from the JDK download.

Compiling non-native architectures rely on emulation, so it is necessary to register `qemu-*-static` executables:
   
`docker run --rm --privileged multiarch/qemu-user-static:register --reset`

**Note that the complete build of all supported architectures is now supported only on Linux platform.**

### Check out sources
`git clone git@github.com:zonkyio/embedded-postgres-binaries.git`

### Make complete build

Builds all supported artifacts for all supported platforms and architectures, and also builds a BOM to control the versions of postgres binaries.

`./gradlew clean install --parallel -Pversion=10.6.0 -PpgVersion=10.6`

Note that the complete build can take a very long time, even a few hours, depending on the performance of the machine on which the build is running.

### Make partial build

Builds only binaries for a specified platform/submodule.

`./gradlew clean :repacked-platforms:install -Pversion=10.6.0 -PpgVersion=10.6`

### Build only a single binary

Builds only a single binary for a specified platform and architecture.

`./gradlew clean install -Pversion=10.6.0 -PpgVersion=10.6 -ParchName=arm64v8 -PdistName=alpine`

It is also possible to include the PostGIS extension by passing the `postgisVersion` parameter, e.g. `-PpostgisVersion=2.5.2`. Note that this option is not (yet) available for Windows and Mac OS platforms.

Optional parameters:
- *postgisVersion*
  - default value: unset
  - supported values: a postgis version number (only 2.5.2+, 2.4.7+, 2.3.9+ versions are supported)
- *archName*
  - default value: `amd64`
  - supported values: `amd64`, `i386`, `arm32v6`, `arm32v7`, `arm64v8`, `ppc64le`
- *distName*
  - default value: debian-like distribution
  - supported values: the default value or `alpine`
- *dockerImage*
  - default value: resolved based on the platform
  - supported values: any supported docker image
- *qemuPath*
  - default value: executables are resolved from `/usr/bin` directory or downloaded from https://github.com/multiarch/qemu-user-static/releases/download/v2.12.0
  - supported values: a path to a directory containing qemu executables

### MacOS arm64/amd64
We've been unable to get cross compiling to work nicely for this platform so generation of these is a manual process.
- To generate for arm64 you need to do the compling on an M1 mac
- To generate for amd64 you need to be on an intel mac
- You need brew installed (in order to run cmake, which timescale requires)
- Simply run the script in scripts/build-postgres-darwin.sh
- It should compile everything, remove baked in hard coded paths in the libraries, and make a file, e.g.
`scripts/darwin_build/embedded-postgres-binaries-darwin-arm64-14.1.0.zip` suitable for upload to the release page on github.
- For simplicity and because we aren't using it in vega right now, this omits a SSL support and a few other optional bits.

## License
The project is released under version 2.0 of the [Apache License](http://www.apache.org/licenses/LICENSE-2.0.html).
