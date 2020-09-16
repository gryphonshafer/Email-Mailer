# NAME

Email::Mailer - Multi-purpose emailer for HTML, auto-text, attachments, and templates

# VERSION

version 1.15

[![Build Status](https://travis-ci.org/gryphonshafer/Email-Mailer.svg)](https://travis-ci.org/gryphonshafer/Email-Mailer)
[![Coverage Status](https://coveralls.io/repos/gryphonshafer/Email-Mailer/badge.png)](https://coveralls.io/r/gryphonshafer/Email-Mailer)

# SYNOPSIS

    use Email::Mailer;
    my ( $to, $from, $subject, $text, $html );

    # send a simple text email
    Email::Mailer->send(
        to      => $to,
        from    => $from,
        subject => $subject,
        text    => $text,
    );

    # send multi-part HTML/text email with the text auto-generated from the HTML
    # and images and other resources embedded in the email
    my $mail = Email::Mailer->new;
    $mail->send(
        to      => $to,
        from    => $from,
        subject => $subject,
        html    => $html,
    );

    # send multi-part HTML/text email with the text auto-generated from the HTML
    # but skip embedding images and other resources
    Email::Mailer->new->send(
        to      => $to,
        from    => $from,
        subject => $subject,
        html    => $html,
        embed   => 0,
    );

    # send multi-part HTML/text email but supply the text explicitly
    Email::Mailer->new(
        to      => $to,
        from    => $from,
        subject => $subject,
        html    => $html,
        text    => $text,
    )->send;

    # send multi-part HTML/text email with a couple of attached files
    use IO::All 'io';
    Email::Mailer->send(
        to          => $to,
        from        => $from,
        subject     => $subject,
        html        => $html,
        text        => $text,
        attachments => [
            {
                ctype  => 'application/pdf',
                source => 'file.pdf',
            },
            {
                ctype    => 'application/pdf',
                content  => io('file.pdf')->binary->all,
                encoding => 'base64',
                name     => 'file.pdf',
            },
        ],
    );

    # build an email and iterate over a data set for sending
    Email::Mailer->new(
        from    => $from,
        subject => $subject,
        html    => $html,
    )->send(
        { to => 'person_0@example.com' },
        { to => 'person_1@example.com' },
        {
            to      => 'person_2@example.com',
            subject => 'Override $subject with this',
        },
    );

    # setup a second mail object based on the first but changing the "from"
    my $mail_0 = Email::Mailer->new(
        from    => $from,
        subject => $subject,
        html    => $html,
    );
    my $mail_1 = $mail_0->new( from => 'different_address@example.com' );
    $mail_0->send;
    $mail_1->send;

    # use a templating system for the HTML and subject
    use Template;
    my $tt    = Template->new;
    my $tmail = Email::Mailer->new(
        from    => $from,
        subject => \$subject,
        html    => \$html,
        process => sub {
            my ( $template, $data ) = @_;
            my $content;
            $tt->process( \$template, $data, \$content );
            return $content;
        },
    );
    $tmail->send($_) for (
        { to => 'person_0@example.com', data => { name => 'Person 0' } },
        { to => 'person_1@example.com', data => { name => 'Person 1' } },
    );

# DESCRIPTION

Following the charter and example of [Email::Simple](https://metacpan.org/pod/Email%3A%3ASimple), this module provides a
simple and flexible interface to sending various types of email including
plain text, HTML/text multi-part, attachment support, and template hooks.
The module depends on a series of great modules in the Email::\* and HTML::\*
namespaces.

# PRIMARY METHODS

There are 2 primary methods.

## new

This is an instantiator and a replicative instantiator. If passed nothing, it'll
return you a blank mail object. If you pass it anything, it'll use that data to
setup a more informed mail object for later sending.

    my $mail_blank = Email::Mailer->new;
    my $mail_to    = Email::Mailer->new( to => 'default_to@example.com');

If you call `new()` off an instantiated mail object, it'll make a copy of that
object, changing any internal data based on what you pass in to the `new()`.

    # create a new object with both a default "To" and "From"
    my $mail_to_from = $mail_to->new( from => 'default_from@example.com' );

## send

This method will attempt to send mail. Any parameters you can pass to `new()`
you can pass to `send()`. Any incoming parameters will override any existing
parameters in an instantiated object.

    $mail_to_from->send(
        subject => 'Example Subject Line',
        text    => 'Hello. This is example email content.',
    );

If `send()` succeeds, it'll return an instantiated object based on the combined
parameters. If it fails, it'll throw an exception.

    use Try::Tiny;

    my $mail_with_all_the_parameters;
    try {
        $mail_with_all_the_parameters = $mail_to_from->send(
            subject => 'Example Subject Line',
            text    => 'Hello. This is example email content.',
        );
    }
    catch {
        print "There was an error, but I'm going to ignore it and keep going.\n";
    };

You can also pass to `send()` a list of hashrefs. If you do that, `send()`
will assume you want each of the hashrefs to be like a set of data sent to an
independent call to `send()`. The method will attempt to send multiple emails
based on your data, and it'll return an array or arrayref (based on context)
of the mail objects ultimately created.

    my @emails_sent = $mail_with_all_the_parameters->send(
        { to => 'person_0@example.com' },
        { to => 'person_1@example.com' },
    );

    my $emails_sent = $mail_with_all_the_parameters->send(
        { to => 'person_0@example.com' },
        { to => 'person_1@example.com' },
    );

    $mail_with_all_the_parameters->send($_) for (
        { to => 'person_0@example.com' },
        { to => 'person_1@example.com' },
    );

# PARAMETERS

There are a bunch of parameters you can pass to the primary methods. First off,
anything not explicitly mentioned in this section, the methods will assume is
a mail header.

If any value of a key is a reference to scalar text, the value of that scalar
text will be assumed to be a template and processed through the subref defined
by the "process" parameter.

## html

This parameter should contain HTML content (or a reference to scalar text that
is the template that'll be used to generate HTML content).

## text

This parameter should contain plain text content (or a template reference). If
not provided then "text" will be automatically generated based on the "html"
content.

By default, the text generated will be wrapped at 72 characters width. However,
you can override that by setting width explicitly:

    Email::Mailer->new->send(
        to      => $to,
        from    => $from,
        subject => $subject,
        html    => $html,
        width   => 120,
    );

If you set a width to 0, this will be interpreted as meaning not to wrap text
lines.

## embed

By default, if your HTML has links to things like images or CSS, those resources
will be pulled in and embedded into the email message. If you don't want that
behavior, turn it off by explicitly setting "embed" to a false value.

    Email::Mailer->new->send(
        to      => $to,
        from    => $from,
        subject => $subject,
        html    => $html,
        embed   => 0,
    );

## attachments

This parameter if needed should be an arrayref of hashrefs that define the
attachments to add to an email. Each hashref should define a "ctype" for the
content type of the attachment and either a "source" or both a "name" and
"content" key. The "source" value should be a local relative path/file. The
"content" value should be binary data, and the "name" value should be the
filename of the attachment.

    use IO::All 'io';

    Email::Mailer->send(
        to          => $to,
        from        => $from,
        subject     => $subject,
        html        => $html,
        text        => $text,
        attachments => [
            {
                ctype  => 'application/pdf',
                source => 'file.pdf',
            },
            {
                ctype    => 'application/pdf',
                content  => io('file.pdf')->binary->all,
                encoding => 'base64',
                name     => 'file.pdf',
            },
        ],
    );

An optional parameter of "encoding" can be supplied in a hashref to
"attachments" to indicate what encoding the attachment should be encoded as.
If not specified, the default is "base64" encoding, which works in most cases.
Another popular choice is "quoted-printable".

## process

This parameter expects a subref that will be called to process any templates.
You can hook in any template system you'd like. The subref will be passed the
template text and a hashref of the data for the message.

    use Template;

    my $tt    = Template->new;
    my $tmail = Email::Mailer->new(
        from    => $from,
        subject => \$subject,
        html    => \$html,
        process => sub {
            my ( $template, $data ) = @_;
            my $content;
            $tt->process( \$template, $data, \$content );
            return $content;
        },
    );

## data

This parameter is the hashref of data that'll get passed to the "process"
subref.

    $tmail->send($_) for (
        { to => 'person_0@example.com', data => { name => 'Person 0' } },
        { to => 'person_1@example.com', data => { name => 'Person 1' } },
    );

## transport

By default, this module will try to pick an appropriate transport. (Well,
technically, [Email::Sender::Simple](https://metacpan.org/pod/Email%3A%3ASender%3A%3ASimple) does that for us.) If you want to override
that and set your own transport, use the "transport" parameter.

    use Email::Sender::Transport::SMTP;

    Email::Mailer->send(
        to        => $to,
        from      => $from,
        subject   => $subject,
        html      => $html,
        transport => Email::Sender::Transport::SMTP->new({
            host => 'smtp.example.com',
            port => 25,
        }),
    );

# AUTOMATIC HEADER-IFICATION

There are some automatic header-ification features to be aware of. Unless you
specify a value, the `Content-Type` and `Content-Transfer-Encoding` are
set as "text/plain; charset=us-ascii" and "quoted-printable" respectively, as
if you set the following:

    Email::Mailer->send(
        to        => $to,
        from      => $from,
        subject   => $subject,
        html      => $html,

        'Content-Type'              => 'text/plain; charset=us-ascii',
        'Content-Transfer-Encoding' => 'quoted-printable',
    );

Also, normally your `to`, `from`, and `subject` values are left untouched;
however, for any of these that contain non-ASCII characters, they will be
mimewords-encoded via [MIME::Words](https://metacpan.org/pod/MIME%3A%3AWords) using the character set defined in
`Content-Type`. If you don't like how that works, just encode them however
you'd like to ASCII.

# SEE ALSO

[Email::MIME](https://metacpan.org/pod/Email%3A%3AMIME), [Email::MIME::CreateHTML](https://metacpan.org/pod/Email%3A%3AMIME%3A%3ACreateHTML), [Email::Sender::Simple](https://metacpan.org/pod/Email%3A%3ASender%3A%3ASimple),
[Email::Sender::Transport](https://metacpan.org/pod/Email%3A%3ASender%3A%3ATransport), [HTML::FormatText](https://metacpan.org/pod/HTML%3A%3AFormatText), [HTML::TreeBuilder](https://metacpan.org/pod/HTML%3A%3ATreeBuilder).

You can also look for additional information at:

- [GitHub](https://github.com/gryphonshafer/Email-Mailer)
- [MetaCPAN](https://metacpan.org/pod/Email::Mailer)
- [Travis CI](https://travis-ci.org/gryphonshafer/Email-Mailer)
- [Coveralls](https://coveralls.io/r/gryphonshafer/Email-Mailer)
- [CPANTS](http://cpants.cpanauthors.org/dist/Email-Mailer)
- [CPAN Testers](http://www.cpantesters.org/distro/D/Email-Mailer.html)

# AUTHOR

Gryphon Shafer <gryphon@cpan.org>

# COPYRIGHT AND LICENSE

This software is copyright (c) 2020 by Gryphon Shafer.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
