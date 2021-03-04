---
title: ParallelStream 的正确打开方式
tags: Java Coding 
# article_header:
#   type: cover
#   image:
#     src: /image/java/java-streams.jpg

comments: true

show_author_profile: true

footer: false

mathjax: true
mathjax_autoNumber: false
mermaid: true
chart: true

modify_date: 2021-01-09
---


# ParallelStream 是什么

**ParallelStream** 其实就是一个并行执行的流.它通过默认的ForkJoinPool,[^1]可能提高你的多线程任务的速度.

<!--more-->
<!-- more -->

<div class="card">
  <div class="card__content">
    <div class="card__header">
      <h4>Fork/Join</h4>
    </div>
    <p>The fork/join framework is an implementation of the ExecutorService interface that helps you take advantage of multiple processors. It is designed for work that can be broken into smaller pieces recursively. The goal is to use all the available processing power to enhance the performance of your application.</p>
  </div>
</div>


# ParallelStream 的作用

**ParallelStream** 
: 具有平行处理能力，处理的过程会分而治之，也就是将一个大任务切分成多个小任务，这表示每个任务都是一个操作。


{% highlight java linenos %}
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);

numbers.parallelStream()
       .forEach(out::println); 
{% endhighlight %}

上面的代码我们的到的不是~~顺序~~的1、2、3、4、5…，而是随机的顺序。

如果需要按照原来的顺序便利，可以使用 forEachOrdered() 方法：
{% highlight java linenos %}
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);

numbers.parallelStream()
       .forEachOrdered(out::println);
{% endhighlight %}

# ParallelStream 的实现
 
ParallelStream 是由 ForkJoin 与 ForkJoinPool 实现的。
ForkJoinPool 使用相对少的线程来处理大量的任务。
比如有一个任务需要处理100万的数据，那么这个任务会被分割成两个处理50万数据的任务，以此类推，直到任务单元被分割成一个预设的阈值大小的任务，直到这些子任务被执行完，这个任务才将完成。
这和ThreadPoolExecutor的区别是，ThreadPoolExecutor 处理这个100万的任务会需要100万的线程.

>ForkJoinPool 默认的大小等于当前机器的 CPU 核心数。
>可以使用 <br>
> ```-Djava.util.concurrent.ForkJoinPool.common.parallelism=N ``` 
>来设置 ForkJoinPool 的大小。
>
>**程序中的 ParallelStream 共用一个 commonForkJoinPool**。


# 关于使用 ParallelStream 的坑

**ParallelStream**固然有它强大的地方，但同样的，它也存在着陷阱。

{% highlight java linenos %}
 public Object query(List<Request> reqs) {
     Optional<String> result = reqs.stream().parallel().map( -> {
        return client.url(url).get();
      }).findAny();
      return result.get();
}
{% endhighlight %}

上面的代码用来获取第一个返回的请求，注意网络请求属于IO操作，这里面存在在被阻塞的风险，故而这段代码的设计是存在问题。
前面提到程序中的 ```ParallelStream``` 共用一个 commonForkJoinPool，试想如果这里调用的 API 响应比较慢，那么这段代码就会影响到其他使用 ```ParallelStream``` 的地方（**阻塞**其他任务）。这样的后果往往是不可估量的。

# 如何正确使用 ParallelStream

- 首先，我们需要考虑需不需要用```ParallelStream```。


    1. [x] 是否需要并行？  
    2. [x] 任务之间是否是独立的？是否会引起任何竞态条件？  
    3. [x] 结果是否取决于任务的调用顺序？ 


    考虑过这三个问题后再决定是否使用```ParallelStream```。


- 其次，使用指定的不与其他任务冲突的 ```ForkJoinPool```。

{% highlight java linenos %}
ForkJoinPool forkJoinPool = new ForkJoinPool(3);  
forkJoinPool.submit(() -> {  
    firstRange.parallelStream().forEach((number) -> {
        try {
            Thread.sleep(5);
        } catch (InterruptedException e) { }
    });
});

{% endhighlight %}




[^1]: [Fork/Join](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)

