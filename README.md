# Web Scraper Workshop
Welcome by the workshop where you are going to build a web scraper that is going to use:

- Platform threads
- Virtual threads
- Structured task scope
- Scoped values

The workshop start with just a simple single threaded web scraper that only scrapes a single page. You are to improve 
this web scraper by first making it multithreaded followed by using virtual threads. These new kind of threads are a great 
new addition to the Java languages but don't work the same as the old threads in every situation. During this workshop you
are going to experience when virtual threads work best, and when they work just oke.

To follow along with the workshop you need to also check out [this repository](https://github.com/davidtos/workshop_server).
The repo contains a Spring application that is going to act as the server where the scraper is going to be talking to.

## How to follow along with the workshop
Below you will find the steps of the workshop. The best way to following along is to start with step 1 and building 
inside this branch. If you want to start every step of the workshop with a clean branch you can also check out the branch belonging to
that step. Each step has a branch inside this git repo with the same name. If you are falling behind please say so, then we
can adjust the speed of the workshop :-) or you can also check at the branch of the next step.

# TL;DR
- Let's build a web scraper!
- Run the Spring project inside [this repository](https://github.com/davidtos/workshop_server), it has the web server we scrape inside it
- Follow along with the steps below (Ron and David give some theory, hints and background info between steps)
- Already done and want to start the next step? go head! :-)
- Any questions? Feel free to ask!

# The steps of the workshop:
Just follow along with the following steps. If you have any questions feel free to ask Ron and I are there to answer them.
We will you give some needed information between steps, so you can focus on solving one problem at a time. :-)  

## (Step 1) - check out the code
You need to check out these two repositories:

### The scraper basis https://github.com/davidtos/virtual_thread_workshop
This is the repository you are looking at. It contains all the steps/branches that you will need to build the scraper.

### The web server https://github.com/davidtos/workshop_server.
This is the webserver that the scraper is going to scape. You can run the Spring project and let it run in the background.
You don't have to make any changes inside this project.

To check if everything works you can try and run the WebScraper class; It should scrape a single page.

## (Step 2) - add platform threads
If you didn't do it already check out the following branch "add platform threads" this is the basis of the web scraper.
you can already run it, and it will scrape a single page from the web server.

The first step is to make the **Scrape** run in concurrently using platform threads. The goal is to be able to create any number of 
Scrape instances that each can scrape a single page.

## (Step 3) - Start using virtual threads
Now you can scrape webpages using multiple Scrape instances that each run on a Platform thread. The next step is to implement
the same thing, but now it should use Virtual Threads instead.

**Hint**: Make it easy to switch between Virtual and Platform threads, so you can switch between the two to see the difference in performance.

# REMOVE
- show that it doesn't make a difference in performance (you need to measure it, to improve it)
- tell about the VT and their carrier threads

## (Step 4) - Difference between virtual and platform threads
The scraper is now able to run on either Virtual Threads or on Platform threads. To see the impact these Thread have on the Scraper
you can play with the following two variables:

1. The URL of the scraper
2. The number of threads you create.

The URLs you can use are:
- http://localhost:8080/v1/crawl/delay/330/57
- http://localhost:8080/v1/crawl/330/57

The first end point has a delay between 10 and 200 milliseconds, forcing the threads to block and wait for a number of seconds. 
The other URL without the delay returns immediately without waiting, using this URL the thread doesn't have to wait that long.

The second thing you can change is the number of the scrape jobs you start. Try out what differance it makes when you submit 
200 or 400 scrape task to a pool of platform threads or create as many virtual threads you have jobs.

The results may surprise you :-)

**Hint**: Try out lots of task with the delay endpoint.

# REMOVE
- Let them use a webserver that returns without delay
- Show that VT are not faster than PT
- Switch to "normal" behaving end-points  that check the performance improvement

## (Step 5) - find the pinned virtual thread
Virtual are unmounted are when they are blocked and waiting for a result to come return. This doesn't always happen though...
It's up to you to fix the scraper and replace the functionality that causes virtual threads to be pinned.

To help you find the method causing issues you can use the following VM options:
```text
-Djdk.tracePinnedThreads=short

-Djdk.tracePinnedThreads=full
```
Run the web scraper with one of these two options and replace the functionality with one that does not cause the virtual threads
to be pinned.

**Hint**: Java 9 added a http client that does not block

## (Step 6) - Set carrier threads (Improve performance branch)
By default, you get as many carrier thread as there are cores available inside you system. There are two ways to tweak the 
number of carrier threads that get created. 

Use the following options and see what impact it has on your scraper.
```text
jdk.virtualThreadScheduler.parallelism=5

jdk.virtualThreadScheduler.maxPoolSize=10
```

You can remove these options when you continue to the next step.

## (Step 7) - Improve performance
The next step is to improve the performance of the scraper. Make it so that the following operations run in their own virtual thread.

```java
visited.add(url);
for (Element link : linksOnPage) {
    String nextUrl = link.attr("abs:href");
    if (nextUrl.contains("http")) {
        pageQueue.add(nextUrl);
    }
}
```
Run the Scraper a few times with and without the improvement to see the differance it makes.

**Note**: Not seeing a lot of differance or lower performance? Those things happen sadly, and depend on lot different factors. 
One big thing that has a lot of impact on the performance is the Spring boot project you started in the beginning. You can
make changes to the TestForWorkshopController.java class to make the server behave differently. One big thing that impact this
part of the code is the number of URls on a page.

## (Step 8) - Use StructuredTaskScope
introduce structured concurrency and improve it by adding the StructuredTaskScope

```java
 try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
                scope.fork(() -> visited.add(url));
                scope.fork(() -> {
                            for (Element link : linksOnPage) {
                                String nextUrl = link.attr("abs:href");
                                if (nextUrl.contains("http")) {
                                    pageQueue.add(nextUrl);
                                }
                            }
                            return null;
                        }
                );
            }
```


## (Step 9) - Implement ShutdownOnSuccess
introduce the ShutdownOnSuccess scope

```java
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<>()) {

    scope.fork(() -> post("http://localhost:8080/v1/VisitedService/1", url));
    scope.fork(() -> post("http://localhost:8080/v1/VisitedService/2", url));
    scope.fork(() -> post("http://localhost:8080/v1/VisitedService/3", url));

    scope.join();

}

private Object post(String serviceUrl, String url) throws IOException, InterruptedException {
    HttpRequest request = HttpRequest.newBuilder().POST(HttpRequest.BodyPublishers.ofString(url)).uri(URI.create(serviceUrl)).build();
    client.send(request, HttpResponse.BodyHandlers.ofString());
    return null;
}
```

## (Step 10) - Use scoped values 
use scoped values for the url lue

```java
    final static ScopedValue<Supplier<String>> URL = ScopedValue.newInstance();

        Scrape.URL.get();

executor.submit(() -> ScopedValue.runWhere(Scrape.URL,
        () -> {
            try {
                return queue.take();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        },
        new Scrape(queue, visited, client)));
```

- show the performance impact
- show the inheritance when using structured task scope



