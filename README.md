# NAME

DBIx::Connector::Retry - DBIx::Connector with block retry support

# SYNOPSIS

```perl
my $conn = DBIx::Connector::Retry->new(
    connect_info  => [ 'dbi:Driver:database=foobar', $user, $pass, \%args ],
    retry_debug   => 1,
    max_attempts  => 5,
);

# Keep retrying/reconnecting on errors
my ($count) = $conn->run(ping => sub {
    $_->do('UPDATE foobar SET updated = 1 WHERE active = ?', undef, 'on');
    $_->selectrow_array('SELECT COUNT(*) FROM foobar WHERE updated = 1');
});

# Add a simple retry_handler for a manual timeout
my $start_time = time;
$conn->retry_handler(sub { time <= $start_time + 60 });

my ($count) = $conn->txn(fixup => sub {
    $_->selectrow_array('SELECT COUNT(*) FROM barbaz');
});
$conn->clear_retry_handler;

# Plus everything else in DBIx::Connector
```

# DESCRIPTION

DBIx::Connector::Retry is a Moo-based subclass of [DBIx::Connector](https://metacpan.org/pod/DBIx::Connector) that will retry on
failures.  Most of the interface was modeled after [DBIx::Class::Storage::BlockRunner](https://metacpan.org/pod/DBIx::Class::Storage::BlockRunner)
and adapted for use in DBIx::Connector.

# ATTRIBUTES

## connect\_info

An arrayref that contains all of the connection details normally found in the [DBI](https://metacpan.org/pod/DBI) or
[DBIx::Connector](https://metacpan.org/pod/DBIx::Connector) call.  This data can be changed, but won't take affect until the next
`$dbh` re-connection cycle.

Obviously, this is required.

## mode

This is just like ["mode" in DBIx::Connector](https://metacpan.org/pod/DBIx::Connector#mode) except that it can be set from within the
constructor.

Unlike DBIx::Connector, the default is `ping`, not `no_ping`.

## disconnect\_on\_destroy

This is just like ["disconnect\_on\_destroy" in DBIx::Connector](https://metacpan.org/pod/DBIx::Connector#disconnect_on_destroy) except that it can be set
from within the constructor.

Default is on.

## max\_attempts

The maximum amount of block running attempts before the Connector gives up and dies.

Default is 10.

## retry\_debug

If enabled, any retries will output a debug warning with the error message and number
of retries.

## retry\_handler

An optional handler that will be checked on each retry.  It will be passed the Connector
object as its only input.  If the handler returns a true value, retries will continue.
A false value will cause the retry loop to immediately rethrow the exception.  You can
also throw your own, if you prefer.

This check is independent of checks for ["max\_attempts"](#max_attempts).

The last exception can be inspected as part of the check by looking at ["last\_exception"](#last_exception).
This is recommended to make sure the failure is actually what you expect it to be.
For example:

```perl
$conn->retry_handler(sub {
    my $c = shift;
    my $err = $c->last_exception;
    $err = $err->error if blessed $err && $err->isa('DBIx::Connector::RollbackError');

    $err =~ /deadlock|timeout/i;  # only retry on deadlocks or timeouts
});
```

Default is an always-true coderef.

This attribute has the following handles:

### clear\_retry\_handler

Sets it back to the always-true default.

## failed\_attempt\_count

The number of failed attempts so far.  This can be used in the ["retry\_handler"](#retry_handler) or
checked afterwards.  It will be reset on each block run.

Not available for initialization.

## exception\_stack

The stack of exceptions received so far, as an arrayref.  This can be used in the
["retry\_handler"](#retry_handler) or checked afterwards.  It will be reset on each block run.

Not available for initialization.

This attribute has the following handles:

### last\_exception

The last exception on the stack.

# CONSTRUCTORS

## new

```perl
my $conn = DBIx::Connector::Retry->new(
    connect_info => [ 'dbi:Driver:database=foobar', $user, $pass, \%args ],
    max_attempts => 5,
    # ...etc...
);

# Old-DBI syntax
my $conn = DBIx::Connector::Retry->new(
    'dbi:Driver:database=foobar', $user, $pass, \%dbi_args,
    max_attempts => 5,
    # ...etc...
);
```

As this is a [Moo](https://metacpan.org/pod/Moo) class, it uses the standard Moo constructor.  The ["connect\_info"](#connect_info)
should be specified as its own key.  The [DBI](https://metacpan.org/pod/DBI)/[DBIx::Connector](https://metacpan.org/pod/DBIx::Connector) syntax is available,
but only as a nicety for compatibility.

# MODIFIED METHODS

## run / txn

```perl
my @result = $conn->run($mode => $coderef);
my $result = $conn->run($mode => $coderef);
$conn->run($mode => $coderef);

my @result = $conn->txn($mode => $coderef);
my $result = $conn->txn($mode => $coderef);
$conn->txn($mode => $coderef);
```

Both [run](https://metacpan.org/pod/DBIx::Connector#run) and [txn](https://metacpan.org/pod/DBIx::Connector#txn) are modified to run inside
a retry loop.  If the original Connector action dies, the exception is caught, and if
["retry\_handler"](#retry_handler) and ["max\_attempts"](#max_attempts) allows it, the action is retried.  The database
handle may be reset by the Connector action, according to its connection mode.

See ["CAVEATS"](#caveats) for important behaviors/limitations.

# CAVEATS

## $dbh settings

Like [DBIx::Connector](https://metacpan.org/pod/DBIx::Connector), it's important that ["connect\_info"](#connect_info) have sane connection
settings.

[AutoCommit](https://metacpan.org/pod/DBI#AutoCommit) should be turned on.  Otherwise, the connection is
considered to be already in a transaction, and no retries will be attempted.  Instead,
use transactions via [txn](https://metacpan.org/pod/DBIx::Connector#txn).

[RaiseError](https://metacpan.org/pod/DBI#RaiseError) should also be turned on, since exceptions are captured,
and both Retry and Connector use them to determine if any of the &lt;$dbh> calls failed.

## Savepoints and nested transactions

[The svp method](https://metacpan.org/pod/DBIx::Connector#svp) is NOT modified to work inside of a retry loop,
because retries are generally not possible for savepoints, and a disconnected connection
will rollback any uncommited statements in most RDBMS.  The same goes for any `run`/`txn`
calls attempted inside of a transaction.

Consider the following:

```perl
# If this dies, sub will retry
$conn->txn(ping => sub {
    shift->do('UPDATE foobar SET updated = 1 WHERE active = ?', undef, 'on');

    # If this dies, it will not retry
    $conn->svp(sub {
        my $c = shift;
        $c->do('INSERT foobar (name, updated, active) VALUES (?, ?)', undef, 'barbaz', 0, 'off');
    });
});
```

If the savepoint actually tried to retry, the `UPDATE` statement would get rolled back by
virtue of database disconnection.  However, the savepoint code would continue, possibly
even succeeding.  You would never know that the `UPDATE` statement was rolled back.

However, without savepoint retry support, as it is currently designed, the statements
will work as expected.  If the savepoint code dies, and if `$conn` is set up for
retries, the transaction code is restarted, after a rollback or reconnection.  Thus, the
`UPDATE` and `INSERT` statements are both ran properly if they now succeed.

Obviously, this will not work if transactions are manually started outside of the main
Connector interface:

```perl
# Don't do this!  The whole transaction isn't compartmentalized properly!
$conn->run(ping => sub {
    $_->begin_work;  # don't ever call this!
    $_->do('UPDATE foobar SET updated = 1 WHERE active = ?', undef, 'on');
});

# If this dies, the whole app will probably crash
$conn->svp(sub {
    my $c = shift;
    $c->do('INSERT foobar (name, updated, active) VALUES (?, ?)', undef, 'barbaz', 0, 'off');
});

# Don't do this!
$conn->run(ping => sub {
    $_->commit;  # no, let Connector handle this process!
});
```

## Fixup mode

Because of the nature of [fixup mode](https://metacpan.org/pod/DBIx::Connector#Connection-Modes), the block may be
executed twice as often.  Functionally, the code looks like this:

```perl
# Very simplified example
sub fixup_run {
    my ($self, $code) = @_;

    my (@ret, $run_err);
    do {
        eval {
            @ret = eval { $code->($dbh) };
            my $err = $@;

            if ($err) {
                die $err if $self->connected;
                # Not connected. Try again.
                return $code->($dbh);
            }
        };
        $run_err = $@;

        if ($run_err) {
            # Push exception_stack, set/check attempts, check retry_handler
        }
    } while ($run_err);
    return @ret;
}
```

If the first eval dies because of a connection failure, the code is ran twice before the
retry loop finds it.  This is only considered to be one attempt.  If it dies because of
some other fault, it's only ran once and continues the retry loop.

If this is behavior is undesirable, this can be worked around by using the ["retry\_handler"](#retry_handler)
to change the [mode](https://metacpan.org/pod/DBIx::Connector#mode) after the first attempt:

```perl
$conn->retry_handler(sub {
    my $c = shift;
    $c->mode('ping') if $c->mode eq 'fixup';
    1;
});
```

Mode is localized outside of the retry loop, so even `$conn->run(fixup => $code)`
calls work, and the default mode will return to normal after the block run.

# SEE ALSO

[DBIx::Connector](https://metacpan.org/pod/DBIx::Connector), [DBIx::Class](https://metacpan.org/pod/DBIx::Class)

# AUTHOR

Grant Street Group <developers@grantstreet.com>

# LICENSE AND COPYRIGHT

Copyright 2018 Grant Street Group.

This program is free software; you can redistribute it and/or modify it
under the terms of the the Artistic License (2.0). You may obtain a
copy of the full license at:

[http://www.perlfoundation.org/artistic\_license\_2\_0](http://www.perlfoundation.org/artistic_license_2_0)