# NAME

ZeroMQ::Poller::Timer - Simple timer for use with ZeroMQ::Poller

# SYNOPSIS

    use ZeroMQ::Poller::Timer;

    my $timer = ZeroMQ::Poller::Timer->new(
        name     => 'my_timer',    # Required
        after    => $seconds,      # Required
        interval => $seconds,
        pause    => [1|0],         # Defaults to 0
    );

# DESCRIPTION

ZeroMQ::Poller waits on ZeroMQ sockets for events, and if you're writing
a daemon you would usually have it do this in an infinite loop. However,
if nothing is happening on those sockets then ZeroMQ::Poller just blocks
on it's `poll()` method indefinitely. Daemons might periodically want to
do things, like reload configuration files, talk to databases, or process
jobs that didn't succeed the first time.

Currently, ZeroMQ::Poller has no built in functionality to let you
periodically break out of the the `poll()` and do work. So this is my
attempt at adding periodic timer functionality to ZeroMQ::Poller, using
ZeroMQ.

ZeroMQ::Poller::Timer is a simple, AnyEvent-like timer for use with
ZeroMQ::Poller. Like an AnyEvent timer you can set each timer to fire
off once, or at intervals. It currently does not support a callback
feature, and might never. The timer is simply a way to make it possible
to periodically break out of the blocking call to `poll` so you can
do other daemony stuff.

# FULL EXAMPLE

    use ZeroMQ::Poller::Timer;
    use ZeroMQ qw/ ZMQ_POLLIN /;

    my $timer = ZeroMQ::Poller::Timer->new(
        name     => 'my_timer',
        after    => $seconds,
        interval => $seconds,
    );

    my $poller = ZeroMQ::Poller->new(
        $timer->poll_hash,

        {
            # Another poll item
        },
    );

    while (1) {
        $poller->poll;

        if ($poller->has_event($timer->name)) {
            $timer->reset;  # This is important!

            # Do stuff...
        }
        if ($poller->has_event('other_item')) {
            # ...
        }
    }

# CONSTRUCTOR ARGUMENTS

- _name_     (__required__)

This is the unique name for this timer. It will be used in your poll loop
to identify which event block to execute. (See FULL EXAMPLE above and
METHODS below).

- _after_    (__required__)

Number of seconds after which to execute the timer. If you want to start
running the timer immediately set this value to 0 (zero).

- _interval_

To set up a periodic timer (which most of you will be wanting to do) use
this field. It is the number of seconds to wait before firing the timer.

- _pause_

By default ZeroMQ::Poller::Timer will create the thread timer at constructor
instantion. To delay this set the __pause__ field to a true value, like 1!

If you do this __you must__ also make sure to call the `start()` method before
you land in your poll loop. Otherwise this was all for not.

# METHODS

## name()

Return the name of the timer you passed into the constructor. You'll use
this when calling the `$poller->has_event()` method inside your polling
loop:

    if ($poller->has_event($timer->name)) {
        # ...
    }

or when you manually declare the poll item hash in the ZeroMQ::Poller
constructor (see `sock()` below).

## sock()

Return the ZeroMQ socket for the timer. This can be used if you manually
declare the poll item hash in the ZeroMQ::Poller constructor. (i.e. you
decide not to use the `poll_hash()` method):

    my $poller = ZeroMQ::Poller->new(
        {
            name   => $timer->name,
            sock   => $timer->sock,
            events => ZMQ_POLLIN,        
        },
    );

## start()

If you had passed a true value into the constructor for the 'pause' field
then you need to call `start()` to start your timer. The timer thread will
not be created until this is called, so make sure you do it before you enter
your infinite poll loop.

## reset()

When your timer fires off and you enter the `if ($poller->has_event(...))`
block inside your infinite loop you need to reset the timer. This is really
just a convience method and is the same as doing the following:

    $timer->socket->recv;

When you fall into a `has_event()` block you'd need to make a call to a
`revc()` anyways, so this doesn't add any overhead... just syntatic sugar.

## poll\_hash()

This is another convience method for you and is best explained by example.
The following two instantiations are identical:

    my $poller = ZeroMQ::Poller->new(
        $timer->poll_hash,
    );

and

    my $poller = ZeroMQ::Poller->new(
        {
            name   => $timer->name,
            socket => $timer->socket,
            events => ZMQ_POLLIN,
        },
    );

# NOTES

This module uses perl [threads](http://search.cpan.org/perldoc?threads). If you're using ZeroMQ to begin with then
threads shouldn't be a concern for you.

Also, it does not have any external module dependencies, other than ZeroMQ,
as I would like to keep it as lite and as simple as possible. 

# CAVEATS

ZeroMQ::Poller::Timer uses the ZMQ\_PAIR pattern, which for a long while
was considered experimental, though it seems to be stable now. It's main
use is for efficient multithreading communication, which is what we're
using it for here. ZMQ\_PAIR sockets are meant to be used in a controlled,
stable environment (i.e. not interprocess) and do not auto-reconnect.

If you are having any issues using ZeroMQ::Poller::Timer in your application
please file a bug report on github at:

[https://github.com/jconerly/ZeroMQ-Poller-Timer](https://github.com/jconerly/ZeroMQ-Poller-Timer)

# SEE ALSO

[ZeroMQ](http://search.cpan.org/perldoc?ZeroMQ)

# AUTHOR

James Conerly `<james at jamesconerly.com>`

# LICENSE

This module is free software; you can redistribute it and/or modify
it under the same terms as Perl itself. See [perlartistic](http://search.cpan.org/perldoc?perlartistic).
