---
layout: post
tags: [Java, Spring Boot, Reactor, WebFlux, Mongodb, Reactive Streams, ReactiveX, Node.js, Angular, Gridfs, Webpack]
---

Have you bought into the whole [Reactive Programming](https://en.wikipedia.org/wiki/Reactive_programming) bit yet? 
Well, I have participated in consuming the "_Reactive Flavor Aid_" and there is one issue at the moment.
Currently, there is a supreme lack of helpful documentation to assist in a seamless development experience.
Fear not fellow reactive enthusiast, this post should help you save images in MongoDB utilizing reactive streams!

Recently, I wanted to create a project that is a full slice reactive application.
Meaning that the code and communication between the Frontend, Backend, and Data Persistence Layers are _reactive_.
When reactive is stated, the following can be assumed:

- All code is no blocking
- Code is processed in an [event loop](http://vertx.io/docs/guide-for-java-devs/#_core_vert_x_concepts).
- [Push based programming](https://en.wikipedia.org/wiki/Push_technology).
- Publisher/Subscriber or [Observable/Observer](http://reactivex.io/intro.html) pattern. 

With all of this criterion set, here is the following application stack that I came up with:

- [Angular 5](https://angular.io/) paired with [RxJS](http://reactivex.io/rxjs/)
- [Spring 5](https://spring.io/) paired with WebFlux powered by [Reactor](https://projectreactor.io/)
- [MongoDB](https://www.mongodb.com/) which supports reactive streams!

At a high level, the project consists of a Spring Boot server.
Which serves static content to the user, and provides a REST API to save, update, retrieve, and delete persisted images.

The images are stored in a Mongo database. 
The server takes advantage of GridFS which breaks the image binary into chunks which allows for storage of images greater than 16MB. 

This all sounds straight forward, right? As it turns out, there is very little documentation and examples on how to do anything reactive!

The rest of this post will show you how you can choose, upload, and receive images! 

![Warning]({{site.imageDir}}mongo_images/trigger warnings.png)  

Attention! The remainder of this post contains _a lot_ of code and corresponding explanations of code.
If the reader feels so inclined, you can find the  *[Source Code to the project described above.](https://github.com/Unthrottled/mongo-images)*  

## Backend Setup

Spring Boot is an awesome framework, the following will show you how to create a WebFlux capable Tomcat Server.

---

### Rest Controller

One of the things that will be needed is a non-blocking REST API that will allow our frontend to persist images.
The following component provides a WebFlux API that is a simple CRUD (create, update, delete) routing function.

- `/api/images` is a GET endpoint that returns a non-blocking `Flux<String>` of all of the current images that have been saved.
- `/api/image/save` is a POST endpoint that consumes the form multipart data as a   `Flux<Part>` and returns the image identifiers as `Flux<String>`.
- `/api/image/get/{id}` is a GET endpoint that takes a path variable as the Hexadecimal Identifier given by the save or images endpoint.
- `/api/image/delete/{id}` is a DELETE endpoint that takes a path variable as the Hexadecimal Identifier given by the save or images endpoint.

Side note, the other routing function is to serve the static resources defined in src/main/resources/static.

#### [Webflux Rest Controller.](/code/rmip/rest_controller.html)

### GridFS Components

Now that we know how to consume and produce images using a REST API powered by Spring, here is how we satisfy the implementation.
GridFS has a reactive client so that means that it will need to read and write bytes in asynchronous streams.

The first hurdle that needs to be crossed is, "how to create the respective asynchronous streams?"

The `Part` class only exposes a `Flux<DataBuffer` which is basically just a Spring wrapped ByteBuffer.

Unfortunately, there is no direct api support for converting a `Flux<Part>` to a `AsycInputStream`, so I had to make my own.
All in the name of non-blocking asynchronous code! I will address my implementation later. 

This works fine and dandy when an image file needs to be saved; however, returning an image is a bit different.

The REST API is designed to to stream the buffered bytes to the client, so that the server to keep everything in memory as it is reading the image.
So this means that our API for retrieving images has a return signature of `Flux<byte[]>`, which will stream the chunked image data to http clients.

#### [Reactive GridFS Component.](/code/rmip/gridfs_component.html)

### Converting Flux<Part> to AsyncInputStream

This was a very interesting problem that I had to solve.
The primary issue between the two classes is one is purely push based, while the other is pull then push.

The Flux, in my opinion is purely push. It is a direct source of whatever it is labeled as.
One subscribed to the stream of objects, then they will be pushed to your code as they come.

While the Async stream is pull then push. What happens here is that you have to ask (pull) for the stream to fill your buffer with bytes.
Then it will push the result of the reading once it is done. Then it is up to you again to ask. Which is the primary differences between these two APIs.

The Flux will give you all of the data one you ask once. While it it up to the user to fetch all of the data.
Which leaves us with a window of time where the Flux is producing data, while the user is doing stuff other than asking to fill another buffer.

So in order to make a successfully conversion, we need to return the data from the `Flux<DataBuffer>` in the same sequence as we received it, to each sequential read call of the Async Input Stream.

There is a blocking way to accomplish this and that would be to turn the Flux into a Iterable, which is natively supported by the Reactor library.
Unfortunately, again, there is no native support for non-blocking Flux iteration. 

Again, I had to make my own implementation. which I will cover a little bit later.

#### Part One

#### [Flux\<DataBuffer\> \-\> AysncInputStream.](/code/rmip/flux_to_asyncinput.html)

#### Part Two: Sequentially iterating a `Flux` without blocking and returning a `Publisher`.

####[Here is how to make a sequential interable non-blocking Flux\!](/code/rmip/sequential_flux.html)


#### Part Three: The helper.

I need a class to let me know if any subscriber has unsubscribed, while waiting for the next element to be returned from the Flux.
This way the sequence is still preserved, and the data goes to a subscriber that is actually listening!

#### [Subcription helper code.](/code/rmip/sink_helper.html)

#### Part Four: Converting AsyncInputStream into Flux<DataBuffer>

Fortunately, this process is much easier! Plus there is native library support!! 

#### [Code to covert InputStream into Flux.](/code/rmip/async_to_flux.html)

### Spring Configurations

All of the proper framework is set to start persisting images. 
The next step is to wire up all of the correct configuration.

#### Spring Boot Configuration

Here we need to let Spring be less helpful. We want to exclude the automatic Mongo configurations.

[Link To File](https://github.com/Unthrottled/mongo-images/blob/master/web-service/src/main/java/io/acari/images/ReactiveImageApplication.java)

{% highlight java %}
    
    //...
    @EnableReactiveMongoRepositories
    @SpringBootApplication(exclude = {MongoAutoConfiguration.class, MongoDataAutoConfiguration.class})
    public class ReactiveImageApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(ReactiveImageApplication.class, args);
        }
    }

{% endhighlight %}

#### Mongo Client Configuration

A mongo client will have to be created so that the application can actually save files!
Setting up the Mongo configuration start of by extending the Abstract ReactiveMongoConfiguration class.
Which will require the implementation of a MongoClient and Database Name factory method.

There are a couple of "gotchas" when creating a reactive Mongo client.
The first being tha the Stream Factory needs to be a NettyStream, otherwise are runtime exception gets thrown when it is omitted!
A required dependency of the stream factory is to specify a NioEventLoopGroup (which is fun to say a loud, Loop Group).
The event loop will need to be shut down when the Spring component is shutdown.

The database connection string and database name can be specified in the application properties file.
Which will be mentioned soon enough!

While the Mongo client is never directly used in the code mentioned above, it is used transitively by the GridFSBucket.
The reactive GridFSBucket is provided as a Spring Bean in this configuration component as well.

#### [Spring configuration for Reactive Mongo.](/code/rmip/mongo_config.html)

#### WebFlux Configuration

This configuration component is necessary to use WebFlux.

[Link To File](https://github.com/Unthrottled/mongo-images/blob/master/web-service/src/main/java/io/acari/images/configuration/WebConfig.java) 

{% highlight java %}

    //....
    import org.springframework.context.annotation.Configuration;
    import org.springframework.web.reactive.config.EnableWebFlux;
    import org.springframework.web.reactive.config.WebFluxConfigurer;
    
    @Configuration
    @EnableWebFlux
    public class WebConfig implements WebFluxConfigurer {
    
    }


{% endhighlight %}

#### Spring Application Property Configuration

Lastly, this is the applications properties file that will allow for the configuration of both the database and Mongo instance used!

[Link to File](https://github.com/Unthrottled/mongo-images/blob/master/web-service/src/main/resources/application.yml)

### application.yml

        acari:
          mongo:
            connectionString: mongodb://mongo
            landingDatabase: images
        
        server:
          port: 80
          
          
---
          
## Frontend Setup

There is a ton of Angular 5 code that I had to write to get this project to be minimally viable.
I wanted to avoid the fluff and just get to the things (I think) that matter.

The following will be covered:

- How to let the user to Select an image.
- How to Display a file, represented as binary, in the browser.
- How to Upload a local user file (POST the image to the web-service created).
- How to retrieve an image from a Byte Stream, and covert it into browser accepted binary.

### Image File Selection

It is hard to save an image that has not been chosen by the user.
This Angular 5 component is dedicated to just that, selecting a file!

When the user chooses an image an event is fired. 
This component finds the selected file and emits it as output.

#### [Image selection component.](/code/rmip/image_selection.html)

## Image Visualization 

Now that the user can choose a file, and the corresponding component emits selected file, image visualization can happen!

### Model Translation Component

This project has a model called `ProjectFile` that is an abstraction of our Image file.

{% highlight javascript %}
    export interface ProjectFile {
        imageBinary(): Observable<any>;
        getIdentifier(): string;
        setNewFile(file: File): void;
    }
{% endhighlight %}

The `ProjectFileChooseComponent` is a part of this component which subscribes to the "File Selected" event that is emitted by the file choosing component.
When the event is fired, it is this component's job to set the current project file to the newly emitted file.

How the _ProjectFile_ model handles a file being set will be discussed later when the implementations of `ProjectFile` are covered.

#### [Project file emitter.](/code/rmip/project_emitter.html)

### Image Visualization Component

An important concept to understand is that the `ProjectFile` has a method that should provide an image's binary.
Given raw binary, then it is expected that there is some way to display that on the browser.

Great expectation! 

Two important bits in this component are the `get imageBinary()` function and
 
    <img [src]="imageBinary | async" alt="SOME IMAGE"/>

The first function supplies the reference to the Observable image binary property. 
While the HTML uses the supplied image binary property and runs it through the asynchronous angular pipe.
    

#### [Image visualization component.](/code/rmip/image_visualization.html)

### Image Model

There are currently two implementations of the ProjectFile interface.
The LocalProjectFile, which is a representation of an image that has not be stored in MongoDB yet.
Local meaning that it has not left your machine yet!

When a file is set on a LocalProject file, the file is converted into binary using FileReader.
The results of the FileReader result are put into a `ReplaySubject`.
A replay subject is a special observable that, sort of, has a current value. 
Meaning that whenever any observer subscribes, then at most one image binary is emitted.
This prevents the need to keep converting the file over and over again. 
Just store the binary in memory from then on return that. 

#### [ProjectFile local implementation.](/code/rmip/local_project_impl.html)

## Uploading an Image 

### Image Upload Service

Being able to choose and display the chosen image is nice, but what about uploading?
Fear not, our REST server takes a MultiPartFile as input.
Here is how to convert a file into FormData which is a MultiPartFile!

#### [How wrap an image to be sent as a multipart.](/code/rmip/image_upload.html)

### HTTP Client

Most of the work above was done by the `BackendAPIService` which is a wrapper around the HttpClient, which is the class responsible for running HttpRequests.
The `postImage` function takes FormData and uses that as the POST request body. 
The return is an observable of key that can be used to fetch and delete the image in persistence.

#### [HTTP client implementation.](/code/rmip/http_client.html)

## Retrieve A Remote Image File

### Remote Image File Model

The second implementation of the `ProjectFile` interface is the RemoteProjectFile.
Which is the abstraction of an image that comes from persistence.
The key difference between the Local and the Remote project is where the binary source is located.

In the case of the local file the binary come directly from your disk, while the binary for the remote file comes from the server (which is probably still your disk :) ).

Much like the `LocalFile`, the remote file takes advantage of the `ReplaySubject` which prevents the need to make more than one REST call to retrieve the image binary while still using the Observable abstraction.

#### [Remote ProjectFile implementation.](/code/rmip/remote_project_impl.html)

### Remote Image Factory

The great thing about the `RemoteProjectFileService` is that when provided a File Key, it creates a ImageBinary Observable (which is an HTTP call waiting to happen), and wraps the key and binary in the RemoteProjectFile model.
As saw above, the model subscribes to the observable, which in turn eventually makes an HTTP POST.

Since the binary coming from the server is raw it needs to be converted into a format the browser should be able to understand.

#### [The remote project file service.](/code/rmip/remote_project_factory.html)

That should be enough to make you armed and dangerous, when it comes to reactive programming!

#### -Alex

# [Source Code!](https://github.com/Unthrottled/mongo-images)
