.. cmd2 documentation for application and command lifecycle and the available hooks

cmd2 Application Lifecycle and Hooks
====================================

The typical way of starting a cmd2 application is as follows::

    import cmd2
    class App(cmd2.Cmd):
        # customized attributes and methods here

    if __name__ == '__main__':
        app = App()
        app.cmdloop()

There are several pre-existing methods and attributes which you can tweak to
control the overall behavior of your application before, during, and after the
command processing loop.

Application Lifecycle Hooks
---------------------------

You can register methods to be called at the beginning of the command loop::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_preloop_hook(self.myhookmethod)

        def myhookmethod(self):
            self.poutput("before the loop begins")

To retain backwards compatibility with `cmd.Cmd`, after all registered preloop
hooks have been called, the ``preloop()`` method is called.

A similar approach allows you to register functions to be called after the
command loop has finished::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_postloop_hook(self.myhookmethod)

        def myhookmethod(self):
            self.poutput("before the loop begins")

To retain backwards compatibility with `cmd.Cmd`, after all registered postloop
hooks have been called, the ``postloop()`` method is called.

Preloop and postloop hook methods are not passed any parameters and any return
value is ignored.


Application Lifecycle Attributes
--------------------------------

There are numerous attributes of and arguments to ``cmd2.Cmd`` which have
a significant effect on the application behavior upon entering or during the
main loop.  A partial list of some of the more important ones is presented here:

- **intro**: *str* - if provided this serves as the intro banner printed once
  at start of application, after ``preloop`` runs
- **allow_cli_args**: *bool* - if True (default), then searches for -t or
  --test at command line to invoke transcript testing mode instead of a normal
  main loop and also processes any commands provided as arguments on the
  command line just prior to entering the main loop
- **echo**: *bool* - if True, then the command line entered is echoed to the
  screen (most useful when running scripts)
- **prompt**: *str* - sets the prompt which is displayed, can be dynamically
  changed based on application state and/or command results


Command Processing Loop
-----------------------

When you call `.cmdloop()`, the following sequence of events are repeated until
the application exits:

#. Output the prompt
#. Accept user input
#. Parse user input into `Statement` object
#. Call methods registered with `register_postparsing_hook()`
#. Redirect output, if user asked for it and it's allowed
#. Start timer
#. Call methods registered with `register_precmd_hook()`
#. Call `precmd()` - for backwards compatibility with ``cmd.Cmd``
#. Add statement to history
#. Call `do_command` method
#. Call methods registered with `register_postcmd_hook()`
#. Call `postcmd(stop, statement)` - for backwards compatibility with ``cmd.Cmd``
#. Stop timer and display the elapsed time
#. Stop redirecting output if it was redirected
#. Call methods registered with `register_cmdfinalization_hook()`

By registering hook methods, steps 4, 8, 12, and 16 allow you to run code
during, and control the flow of the command processing loop. Be aware that
plugins also utilize these hooks, so there may be code running that is not part
of your application. Methods registered for a hook are called in the order they
were registered. You can register a function more than once, and it will be
called each time it was registered.

Postparsing, precommand, and postcommand hook methods share some common ways to
influence the command processing loop.

If a hook raises a ``cmd2.EmptyStatement`` exception:
- no more hooks (except command finalization hooks) of any kind will be called
- if the command has not yet been executed, it will not be executed
- no error message will be displayed to the user

If a hook raises any other exception:
- no more hooks (except command finalization hooks) of any kind will be called
- if the command has not yet been executed, it will not be executed
- the exception message will be displayed for the user.

Specific types of hook methods have additional options as described below.

Postparsing Hooks
^^^^^^^^^^^^^^^^^

Postparsing hooks are called after the user input has been parsed but before
execution of the command. These hooks can be used to:

- modify the user input
- run code before every command executes
- cancel execution of the current command
- exit the application

When postparsing hooks are called, output has not been redirected, nor has the
timer for command execution been started.

To define and register a postparsing hook, do the following::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_postparsing_hook(self.myhookmethod)

        def myhookmethod(self, params: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
            # the statement object created from the user input
            # is available as params.statement
            return params

``register_postparsing_hook()`` checks the method signature of the passed callable,
and raises a ``TypeError`` if it has the wrong number of parameters. It will
also raise a ``TypeError`` if the passed parameter and return value are not annotated
as ``PostparsingData``.

The hook method will be passed one parameter, a ``PostparsingData`` object
which we will refer to as ``params``. ``params`` contains two attributes.
``params.statement`` is a ``Statement`` object which describes the parsed
user input. There are many useful attributes in the ``Statement``
object, including ``.raw`` which contains exactly what the user typed.
``params.stop`` is set to ``False`` by default.

The hook method must return a ``PostparsingData`` object, and it is very
convenient to just return the object passed into the hook method. The hook
method may modify the attributes of the object to influece the behavior of
the application. If ``params.stop`` is set to true, a fatal failure is
triggered prior to execution of the command, and the application exits.

To modify the user input, you create a new ``Statement`` object and return it in
``params.statement``. Don't try and directly modify the contents of a
``Statement`` object, there be dragons. Instead, use the various attributes in a
``Statement`` object to construct a new string, and then parse that string to
create a new ``Statement`` object.

``cmd2.Cmd()`` uses an instance of ``cmd2.StatementParser`` to parse user input.
This instance has been configured with the proper command terminators, multiline
commands, and other parsing related settings. This instance is available as the
``self.statement_parser`` attribute. Here's a simple example which shows the
proper technique::

    def myhookmethod(self, params: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
        if not '|' in params.statement.raw:
            newinput = params.statement.raw + ' | less'
            params.statement = self.statement_parser.parse(newinput)
        return params

If a postparsing hook returns a ``PostparsingData`` object with the ``stop``
attribute set to ``True``:

- no more hooks of any kind (except command finalization hooks) will be called
- the command will not be executed
- no error message will be displayed to the user
- the application will exit


Precommand Hooks
^^^^^^^^^^^^^^^^^

Precommand hooks can modify the user input, but can not request the application
terminate. If your hook needs to be able to exit the application, you should
implement it as a postparsing hook.

Once output is redirected and the timer started, all the hooks registered with
``register_precmd_hook()`` are called. Here's how to do it::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_precmd_hook(self.myhookmethod)

        def myhookmethod(self, data: cmd2.plugin.PrecommandData) -> cmd2.plugin.PrecommandData:
            # the statement object created from the user input
            # is available as data.statement
            return data

``register_precmd_hook()`` checks the method signature of the passed callable,
and raises a ``TypeError`` if it has the wrong number of parameters. It will
also raise a ``TypeError`` if the parameters and return value are not annotated
as ``PrecommandData``.

You may choose to modify the user input by creating a new ``Statement`` with
different properties (see above). If you do so, assign your new ``Statement``
object to ``data.statement``.

The precommand hook must return a ``PrecommandData`` object. You don't have to
create this object from scratch, you can just return the one passed into the hook.

After all registered precommand hooks have been called,
``self.precmd(statement)`` will be called. To retain full backward compatibility
with ``cmd.Cmd``, this method is passed a ``Statement``, not a
``PrecommandData`` object.


Postcommand Hooks
^^^^^^^^^^^^^^^^^^

Once the command method has returned (i.e. the ``do_command(self, statement)
method`` has been called and returns, all postcommand hooks are called. If
output was redirected by the user, it is still redirected, and the command timer
is still running.

Here's how to define and register a postcommand hook::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_postcmd_hook(self.myhookmethod)

        def myhookmethod(self, data: cmd2.plugin.PostcommandData) -> cmd2.plugin.PostcommandData:
            return data

Your hook will be passed a ``PostcommandData`` object, which has a ``statement``
attribute that describes the command which was executed. If your postcommand
hook method gets called, you are guaranteed that the command method was called,
and that it didn't raise an exception.

If any postcommand hook raises an exception, the exception will be displayed to
the user, and no further postcommand hook methods will be called. Command
finalization hooks, if any, will be called.

After all registered postcommand hooks have been called,
``self.postcmd(statement)`` will be called to retain full backward compatibility
with ``cmd.Cmd``.

If any postcommand hook (registered or ``self.postcmd()``) returns a ``PostcommandData`` object
with the stop attribute set to ``True``, subsequent postcommand hooks will still be called, as
will the command finalization hooks, but once those hooks have all been called, the application
will terminate.  Likewise, if ``self.postcmd()`` returns ``True``, the command finalization hooks
will be called before the application terminates.

Any postcommand hook can change the value of the ``stop`` parameter before
returning it, and the modified value will be passed to the next postcommand
hook. The value returned by the final postcommand hook will be passed to the
command finalization hooks, which may further modify the value. If your hook
blindly returns ``False``, a prior hook's requst to exit the application will
not be honored. It's best to return the value you were passed unless you have a
compelling reason to do otherwise.


Command Finalization Hooks
^^^^^^^^^^^^^^^^^^^^^^^^^^

Command finalization hooks are called even if one of the other types of hooks or
the command method raise an exception. Here's how to create and register a
command finalization hook::

    class App(cmd2.Cmd):
        def __init__(self, *args, *kwargs):
            super().__init__(*args, **kwargs)
            self.register_cmdfinalization_hook(self.myhookmethod)

        def myhookmethod(self, stop, statement):
            return stop

Command Finalization hooks must check whether the statement object is ``None``. There are certain circumstances where these hooks may be called before the statement has been parsed, so you can't always rely on having a statement.

If any prior postparsing or precommand hook has requested the application to
terminate, the value of the ``stop`` parameter passed to the first command
finalization hook will be ``True``. Any command finalization hook can change the
value of the ``stop`` parameter before returning it, and the modified value will
be passed to the next command finalization hook. The value returned by the final
command finalization hook will determine whether the application terminates or
not.

This approach to command finalization hooks can be powerful, but it can also
cause problems. If your hook blindly returns ``False``, a prior hook's requst to
exit the application will not be honored. It's best to return the value you were
passed unless you have a compelling reason to do otherwise.

If any command finalization hook raises an exception, no more command
finalization hooks will be called. If the last hook to return a value returned
``True``, then the exception will be rendered, and the application will
terminate.
