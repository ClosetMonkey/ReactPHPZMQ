ZMQPoll based LoopInterface for ReactPHP EventLoop. Somewhat faster than default StreamSelect ReactPHP eventloop when operating on standard PHP stream sockets, but also supports first-class level triggered ZMQSockets (see `example.php`). 

Other ZMQ implementations such as [friends-of-reactphp/zmq](https://github.com/friends-of-reactphp/zmq) acquire a ZMQ_FD resource from zmq_getsockopt which is edge triggered and does not predictably represent the actual IO socket (eg TCP socket) used within ZMQ's IO loop. The result of it's edge triggered behavior is that events must be handled in a loop such as [here](https://github.com/friends-of-reactphp/zmq/blob/master/src/Buffer.php#L90-L110) and [here](https://github.com/friends-of-reactphp/zmq/blob/master/src/SocketWrapper.php#L63-L81) which blocks other events from being handled by ReactPHP's eventloop. This also interferes with the defined behavior of half ZMQ socket types. When polling on the ZMQ_FD such as with `stream_select()`, ZMQ send/recv calls must be done using the ZMQ::MODE_NOBLOCK flag on the ZMQ socket. ZMQ sockets are non-blocking by nature because it only writes onto a queue. Typically the only reason it wont let you take from the queue (zmq_recv) or add to the queue (zmq_send) is if the underlying queue model (req, rep, push, pull etc) is not satisfied, such as a zmq_req socket's read queue not yet having a response available from the zmq_rep socket. In other words, ZMQ::MODE_NOBLOCK dose not change the behavior of the actual IO (which is always non-blocking as well), but the behavior of the queues. This also lead to major shortcomings in zmq's Javascript implementation for the same reasons.

Ratchet Websocket benchmarks were performed using method found [here](https://github.com/matttomasetti/PHP-Ratchet_Websocket-Benchmark-Server) to demonstrate the ZMQPoll EventLoop's ability to handle standard PHP streams such as for HTTP and Websocket. Results provided in [StreamSelect.png](https://github.com/ClosetMonkey/ReactPHPZMQ/blob/main/StreamSelect.png) and [ZMQPoll.png](https://github.com/ClosetMonkey/ReactPHPZMQ/blob/main/ZMQPoll.png) .png files.

Also included is an example of using a ZMQPoll React eventloop to act as a reactor for parallel based PHP threads over ZMQ inproc:// transport within `react_parallel.php`. On my Ryzen 9 I'm able to communicate between 15 threads at once with round trips taking only 230.75 microseconds. That's 115 microsecond latency per event on each thread - which is 7.6 microseconds per event on the ReactPHP mainloop 😎 This was ran for an hour, during the PHP process only took 19% of my CPU and stayed at a constant 44.2 mb memory. This leaves 80% of my processor available for work if more is done within the threads than just echoing.

`parallel_ratchet.php` is a simple multi-thread websocket example for handling websocket messages in a parallel php threadpool.

I also added ZMQ socket based React\Stream and React\Socket implementation. All seems to run well but haven't thoroughly tested. Any and all input is welcome!

Bound socket using ReactPHP\Stream and ReactPHP\Socket implmentations.
```php
<?php

require __DIR__ . '/vendor/autoload.php';

$loop = new React\EventLoop\ZMQPollLoop();

React\EventLoop\Loop::set($loop);

$socket = new React\Socket\ZMQConnector($loop, ["type" => ZMQ::SOCKET_REP]);
$socket->bind("ipc:///tmp/reactphp")->then(
    function (React\Socket\ConnectionInterface $connection) 
    {
        $connection->on('data', function ($data) use ($connection)
        {
            echo "Got data: {$data[0]} \n";
            $connection->write(["HELLO BACK!!!"]);
        });
        
        $connection->on('error', function (Exception $e)
        {
            echo 'error: ' . $e->getMessage();
        });    
    },
    function (Exception $error) 
    {
        echo "failed to connect due to {$error} \n";
    }
);

React\EventLoop\Loop::run();
```

Client socket
```php
<?php

require __DIR__ . '/vendor/autoload.php';

$loop = new React\EventLoop\ZMQPollLoop();

React\EventLoop\Loop::set($loop);

$socket = new React\Socket\ZMQConnector($loop, ["type" => ZMQ::SOCKET_REQ]);
$socket->connect("ipc:///tmp/reactphp")->then(
    function (React\Socket\ConnectionInterface $connection)
    {
        $connection->on('data', function ($data) use ($connection)
        {
            echo "Got data: {$data[0]} \n";
        });
        
        $connection->on('error', function (Exception $e)
        {
            echo 'error: ' . $e->getMessage();
        });    
    
        $connection->write(["hello"]);
    },
    function (Exception $error) {
        echo "failed to connect due to {$error} \n";
    }
);

React\EventLoop\Loop::run();
```
