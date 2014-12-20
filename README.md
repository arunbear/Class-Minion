# NAME

Minions - What is _your_ API?

# SYNOPSIS

    package Example::Synopsis::Counter;

    use Minions
        interface => [ qw( next ) ],
        implementation => 'Example::Synopsis::Acme::Counter';

    1;
    
    # In a script near by ...
    
    use Test::Most tests => 5;
    use Example::Synopsis::Counter;

    my $counter = Example::Synopsis::Counter->new;

    is $counter->next => 0;
    is $counter->next => 1;
    is $counter->next => 2;

    throws_ok { $counter->new } qr/Can't locate object method "new"/;
    
    throws_ok { Example::Synopsis::Counter->next } 
              qr/Can't locate object method "next" via package "Example::Synopsis::Counter"/;

    
    # And the implementation for this class:
    
    package Example::Synopsis::Acme::Counter;
    
    use strict;
    
    our %__Meta = (
        has  => {
            count => { default => 0 },
        }, 
    );
    
    sub next {
        my ($self) = @_;
    
        $self->{-count}++;
    }
    
    1;    
    

# STATUS

This is an early release available for testing and feedback and as such is subject to change.

# DESCRIPTION

Minions is a class builder that makes it easy to create classes that are [modular](http://en.wikipedia.org/wiki/Modular_programming).

Classes are built from a specification that declares the interface of the class (i.e. what commands minions of the classs respond to),
as well as a package that provide the implementation of these commands.

This separation of interface from implementation details is an important aspect of modular design, as it enables modules to be interchangeable (so long as they have the same interface).

It is not a coincidence that the Object Oriented way as it was originally envisioned was mainly concerned with messaging,
where in the words of Alan Kay (who coined the term "Object Oriented Programming") objects are "like biological cells and/or individual computers on a network, only able to communicate with messages"
and "OOP to me means only messaging, local retention and protection and hiding of state-process, and extreme late-binding of all things."
(see [The Deep Insights of Alan Kay](http://mythz.servicestack.net/blog/2013/02/27/the-deep-insights-of-alan-kay/)).

# USAGE

## Via Import

A class can be defined when importing Minions e.g.

    package Foo;

    use Minions
        interface => [ qw( list of methods ) ],

        construct_with => {
            arg_name => {
                assert => {
                    desc => sub {
                        # return true if arg is valid
                        # or false otherwise
                    }
                },
                optional => $boolean,
            },
            # ... other args
        },

        implementation => 'An::Implementation::Package',
        ;
    1;

## Minions->minionize(\[HASHREF\])

A class can also be defined by calling the `minionize()` class method, with an optional hashref that 
specifies the class.

If the hashref is not given, the specification is read from a package variable named `%__Meta` in the package
from which `minionize()` was called.

The class defined in the SYNOPSIS could also be defined like this

    use Test::Most tests => 4;
    use Minions ();

    my %Class = (
        name => 'Counter',
        interface => [qw( next )],
        implementation => {
            methods => {
                next => sub {
                    my ($self) = @_;

                    $self->{-count}++;
                }
            },
            has  => {
                count => { default => 0 },
            }, 
        },
    );

    Minions->minionize(\%Class);
    my $counter = Counter->new;

    is $counter->next => 0;
    is $counter->next => 1;

    throws_ok { $counter->new } qr/Can't locate object method "new"/;
    throws_ok { Counter->next } qr/Can't locate object method "next" via package "Counter"/;

## Examples

Further examples of usage can be found in the following documents

- [Minions::Construction](https://metacpan.org/pod/Minions::Construction)

## Specification

The meaning of the keys in the specification hash are described next.

### interface => ARRAYREF

A reference to an array containing the messages that minions belonging to this class should respond to.
An exception is raised if this is empty or missing.

The messages named in this array must have corresponding subroutine definitions in a declared implementation,
otherwise an exception is raised.

### construct\_with => HASHREF

An optional reference to a hash whose keys are the names of keyword parameters that are passed to the default constructor.

The values these keys are mapped to are themselves hash refs which can have the following keys.

#### optional => BOOLEAN (Default: false)

If this is set to a true value, then the corresponding key/value pair need not be passed to the constructor.

#### assert => HASHREF

A hash that maps a description to a unary predicate (i.e. a sub ref that takes one value and returns true or false).
The default constructor will call these predicates to validate the parameters passed to it.

### implementation => STRING | HASHREF

The name of a package that defines the subroutines declared in the interface.

The package may also contain other subroutines not declared in the interface that are for internal use in the package.
These won't be callable using the `$minion->command(...)` syntax.

Alternatively an implementation can be hashref as shown in the synopsis above.

## Configuring an implementation package

An implementation package can also be configured with a package variable `%__Meta` with the following keys:

### has => HASHREF

This declares attributes of the implementation, mapping the name of an attribute to a hash with keys described in
the following sub sections.

An attribute called "foo" can be accessed via it's object like this:

    $self->{-foo}

Objects created by Minions are hashes,
and are locked down to allow only keys declared in the "has" (implementation or role level)
declarations. This is done to prevent accidents like mis-spelling an attribute name.

#### default => SCALAR | CODEREF

The default value assigned to the attribute when the object is created. This can be an anonymous sub,
which will be excecuted to build the the default value (this would be needed if the default value is a reference,
to prevent all objects from sharing the same reference).

#### assert => HASHREF

This is like the `assert` declared in a class package, except that these assertions are not run at
construction time. Rather they are invoked by calling the semiprivate ASSERT routine.

#### handles => ARRAYREF | HASHREF | SCALAR

This declares that methods can be forwarded from the object to this attribute in one of three ways
described below. These forwarding methods are generated as public methods if they are declared in
the interface, and as semiprivate routines otherwise.

#### handles => ARRAYREF

All methods in the given array will be forwarded.

#### handles => HASHREF

Method forwarding will be set up such that a method whose name is a key in the given hash will be
forwarded to a method whose name is the corresponding value in the hash.

#### handles => SCALAR

The scalar is assumed to be a role, and methods provided directly (i.e. not including methods in sub-roles) by the role will be forwarded.

#### reader => SCALAR

This can be a string which if present will be the name of a generated reader method.

This can also be the numerical value 1 in which case the generated reader method will have the same name as the key.

Readers should only be created if they are logically part of the class API.

### semiprivate => ARRAYREF

Any subroutines in this list will be semiprivate, i.e. they will not be callable as regular object methods but
can be called using the syntax:

    $obj->{'!'}->do_something(...)

### roles => ARRAYREF

A reference to an array containing the names of one or more Role packages that define the subroutines declared in the interface.

The packages may also contain other subroutines not declared in the interface that are for internal use in the package.
These won't be callable using the `$minion->command(...)` syntax.

## Configuring a role package

A role package must be configured with a package variable `%__Meta` with the following keys (of which only "role"
is mandatory):

### role => 1 (Mandatory)

This indicates that the package is a Role.

### has => HASHREF

This works the same way as in an implementation package.

### semiprivate => ARRAYREF

This works the same way as in an implementation package.

### requires => HASHREF

A hash with keys:

#### methods => ARRAYREF

Any methods listed here must be provided by an implementation package or a role.

#### attributes => ARRAYREF

Any attributes listed here must be provided by an implementation package or a role, or by the "requires"
definition in the class.

# BUGS

Please report any bugs or feature requests via the GitHub web interface at 
[https://github.com/arunbear/perl5-minion/issues](https://github.com/arunbear/perl5-minion/issues).

# AUTHOR

Arun Prasaad <arunbear@cpan.org>

# COPYRIGHT

Copyright 2014- Arun Prasaad

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the terms of the GNU public license, version 3.

# SEE ALSO
