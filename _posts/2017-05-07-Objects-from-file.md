---
layout: post
title: Java Objects to and from a File
headerImage: beans/beans-to-file.png
---

There will come a time when the need arises to have Java objects persist without the use of a relational or NoSQL database.
Simply put, reading and writing an object to a local file.
Let us explore how this can be accomplished and find some reason this is helpful in the first place.

#### What can expected from this post:
1. How to read and write Serializable objects to a file.
1. How to read and write Non-Serializable objects to a file.
1. Why would we want to read and write objects to a file.

**Writing Serializable objects to a file.**

The first thing that is needed is an object to write to a file. 
It will need to implement the `java.io.Serializable` or `java.io.Externalizable` interface.
For more information about serializable objects, checkout [this post on java serialization performance!]({% post_url 2017-05-01-Hazelcast-Performance-Serialization %})

Here is the how.

{% highlight java %}
package io.acari;

import io.acari.pojo.ExternalizableProgrammer;
import io.acari.pojo.Programmer;
import io.acari.repositories.ProgrammerRepository;

import java.io.*;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;
import java.util.LinkedList;
import java.util.List;
import java.util.stream.Stream;

public class Main {

    public static void main(String[] args) throws IOException {
        ProgrammerRepository programmerRepository = ProgrammerRepository.newProgrammerRepository();
        readWriteObject(programmerRepository.getProgrammers(), Programmer.class);
        readWriteObject(programmerRepository.getProgrammers().map(ExternalizableProgrammer::new), ExternalizableProgrammer.class);
    }

    /**
     * Takes a stream of objects and writes and reads from a file
     * created in the directory the main method is executed in.
     * <p>
     * Creates a file named after the class provided's simple name
     * post-fixed by .data
     *
     * @param objectStream a open stream of objects to be serialized.
     * @param tClass       Class of the object of the to be serialized
     * @param <T>          Any class that extends Serializable
     * @throws IOException if user has unsufficent privledges to write in
     *                     current working directory.
     */
    private static <T extends Serializable> void readWriteObject(Stream<T> objectStream, Class<T> tClass) throws IOException {
        String simpleName = tClass.getSimpleName();
        Path fileToWrite = Paths.get(simpleName + ".data");
        //Create File (if needed) to write to.
        if (Files.notExists(fileToWrite)) {
            Files.createFile(fileToWrite);
        }

        //Write stream of objects to file.
        try (ObjectOutputStream out = new ObjectOutputStream(
                Files.newOutputStream(fileToWrite, StandardOpenOption.TRUNCATE_EXISTING))) {
            objectStream.forEach(object -> {
                try {
                    out.writeObject(object);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }

        //Read objects from file.
        try (ObjectInputStream in = new ObjectInputStream(Files.newInputStream(fileToWrite, StandardOpenOption.READ))) {
            List<T> programmers = new LinkedList<>();
            try {
                while (true) {
                    programmers.add((T) in.readObject());
                }
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            } catch (EOFException ignored) {
                /*
                 * Reached the end of the file.
                 * No more objects to read
                 */
            }
            System.out.format("%d %s read from file!\n", programmers.size(), simpleName);
        }
    }
}
{% endhighlight %}

One of the most important parts to know is that any object written to a file _must_ implement the `Serializable` interface.
Otherwise a pretty little `NotSerializableException` gets throw when writing to the `ObjectOutputStream`.

Another handy fact is that the `ObjectInputStream` will throw an `EOFException` (End-of-File) when the data stream has no more bytes to read.
Which will allows the infinite loop, created in the method, to break.

A complete functioning project using this code can be found at the following repository listed below.
The repository contains a README file that will help get the project up and running!

[https://github.com/cyclic-reference/object-to-file](https://github.com/cyclic-reference/object-to-file)

**Writing Non-Serializable objects to a file.**

**Disclamer**: This does not mean, SERIALIZE ALL THE THINGS!
Some classes are not serializable for a good reason. 
Such as FutureTask, Thread, and Executor. 
These are very complex and stateful objects that have no use becoming stagnate in a file.

An important question to ask is: why in the world would this ever be a thing in the first place?

Which is a really good question. 
There are some things that are out of our control.
Which really chaffs my chaps sometimes. I mean come on, we are programmers! We bend computers to our will such that nothing should be impossible,

Lets say that we have to consume a third-party (someone else other than us) library to consume a web-service. 
This library contains a domain object that is vital to the core application logic.
The object is _very_ large and complex. 
In order to unit-test the classes that interface with the third party library, these domain objects need to be created.
Since they are so large and complex that creating different permutations these objects from hand is time consuming and error prone.
Suppose that is far easier to create this domain object by using dedicated application.
However, the issue here is that the object is not serializable or easily serializable (has complex types).

One way around this issue is mapping the object to a JSON String. 
This is viable if have access to a Javascript Object Notation object mapper library is permitted.
Such libraries include but are not limited to:

- [Google GSON](https://github.com/google/gson)
- [Jackson FasterXML](https://www.mkyong.com/java/jackson-2-convert-java-object-to-from-json/)

The following example will be done using GSON, Google's POJO to JSON mapping tool.

{% highlight java %}
package io.acari;

import com.google.gson.Gson;
import io.acari.pojo.NonSerializableProgrammer;
import io.acari.repositories.ProgrammerRepository;

import java.io.BufferedWriter;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.function.Consumer;

import static io.acari.repositories.ProgrammerRepository.newProgrammerRepository;

public class TestDataCreator {
    private static final Path TEST_JSON_DATA_FILE = Paths.get("src", "test", "resources", "programmers.json").toAbsolutePath();
    private static final Gson GSON = new Gson();

    public Path fetchJSONFile(){
        if (needToWriteToFile(TEST_JSON_DATA_FILE)) {
            Consumer<Path> jsonWriter = path -> {
                try (BufferedWriter out = Files.newBufferedWriter(path)) {
                    newProgrammerRepository().getProgrammers()
                            .map(NonSerializableProgrammer::new)
                            .forEach(programmer -> {
                                try {
                                    out.write(GSON.toJson(programmer));
                                    out.newLine();
                                } catch (IOException e) {
                                    e.printStackTrace();
                                }
                            });
                } catch (IOException e) {
                    e.printStackTrace();
                }
            };
            createData(TEST_JSON_DATA_FILE, jsonWriter);
        }
        return TEST_JSON_DATA_FILE;
    }

    private void createData(Path testDataFile, Consumer<Path> consumer) {
        try {
            if (Files.size(testDataFile) == 0) {
                consumer.accept(testDataFile);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private boolean needToWriteToFile(Path testDataFile) {
        try {
            if (Files.notExists(testDataFile)) {
                Files.createFile(testDataFile);
            }
        } catch (IOException e) {
            e.printStackTrace();
            return false;
        }
        return true;
    }
}

{% endhighlight %}

Running `fetchJSONFile()`, provided proper permissions, will create a file whose relative path is test-objects-from-file/src/test/resources/programmers.json.
Having the file in the resources directory will allow the build to put it directly into the classpath, next time the build happens.
Meaning that this class is really just good for one run, after that the file it created will be package into the executable .jar file.
This can change how the test data resource is accessed in the created program.

Here is how to use the resource via absolute path, using TestDataCreator:

{% highlight java %}
package io.acari;

import com.google.gson.Gson;
import io.acari.pojo.NonSerializableProgrammer;
import io.acari.pojo.Programmer;

import java.io.EOFException;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class TestDataProvider {
    private static Map<String, NonSerializableProgrammer> nonSerialProgrammers = new LinkedHashMap<>();
    private static final Gson GSON = new Gson();

    static {
        TestDataCreator testDataCreator = new TestDataCreator();
        Path path = testDataCreator.fetchSerializableObjectFile();
        nonSerialProgrammers = createProgrammersFromJSON(testDataCreator.fetchJSONFile());
    }

    private static Map<String, NonSerializableProgrammer> createProgrammersFromJSON(Path path) {
        if (Files.exists(path)) {
            try {
                return Files.lines(path)
                        .map(programmerJson -> GSON.fromJson(programmerJson, NonSerializableProgrammer.class))
                        .collect(Collectors.toMap(NonSerializableProgrammer::getName, Function.identity()));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return Collections.emptyMap();
    }

    public static Map<String, NonSerializableProgrammer> getNonSerialProgrammers() {
        return nonSerialProgrammers;
    }
}
{% endhiglight %}

Before delving into the how and why of writing objects to a file, it would be nice to mention some more concrete persistence APIs.

As relation databases go there are many to choose from, here is a small sample:

- [MySQL](https://www.mysql.com/)
- [PostgreSQL](https://www.postgresql.org/about/)
- [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-2016)

On the other hand, there are also quite a few NoSQL databases to choose from as well.
The primary difference between SQL and NoSQL databases is that there is no relational setup between objects stored in NoSQL databases.
Items in a NoSQL database are stored as Key-Value pairs. 
Here is a small subset of NoSQL databases:

- [MongoDB](https://www.mongodb.com/what-is-mongodb)
- [Redis](https://redis.io/topics/introduction)
- [Cassandra](http://cassandra.apache.org/)