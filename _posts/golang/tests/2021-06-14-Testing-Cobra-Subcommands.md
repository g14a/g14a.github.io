---
title: Testing Cobra Subcommands
author: g14a
date: 2021-06-14 11:30:00 -0530
categories: [article]
tags: [golang, testing, cobra, cli]
toc: true
---

## **Just an Intro**

Go has been around for more than 10 years now. Recently more than a dozen tools and frameworks have come out making us create CLI apps easily than ever. Out of those countless tools, we know that [spf13/cobra](https://github.com/spf13/cobra) has stood out. It is completely tested and is readily usable.
And when we're designing CLI apps, we're usually refactoring our whole application to make it ready for testing purposes.

Now we know how cobra commands look like. The various parameters they take i.e `Use`, `Run`, `RunE`, `PreRun` etc. And we also know how child commands are tested. I mean they're on the internet. But testing subcommands is something I've seen the dev community ask for in the issues of the repo [spf13/cobra](https://github.com/spf13/cobra). 

So I thought I could share some of the things I learnt while doing them(or how I achieved them).

Recently I built an abstract migration CLI tool in Go called [metana](https://github.com/g14a/metana) for Go services. As usual I deal with configuration and settings of it. Ultimately I have a `config` command which looks like this.

```shell
$ metana config --help

Manage your local metana config in .metana.yml

Usage:
  metana config [flags]
  metana config [command]

Available Commands:
  set         Set your metana config

Flags:
  -h, --help   help for config

Global Flags:
      --config string   config gen (default is $HOME/.metana.yaml)

Use "metana config [command] --help" for more information about a command.
```

As you see in line 7, we have a subcommand called `set`.

```shell
Set your metana config

Usage:
  metana config set [flags]

Flags:
  -d, --dir string     Set your migrations directory
  -e, --env string     Set config for your environment
  -h, --help           help for set
  -s, --store string   Set your store

Global Flags:
      --config string   config gen (default is $HOME/.metana.yaml)
```

So to tell you a little bit about the usage, it goes like this:

```shell
metana config set --dir <dirname> --env <environment> --store <store> 
```

So lets start with how we go about testing the subcommand `set`.

## **Diving In**

Let's get the root command which is `metana` itself.

That's great, because we have a function which exactly that.

```go
func NewMetanaCommand() *cobra.Command {
    metanaCmd := cobra.Command{
        Use: "metana",
    }
    return &metanaCmd
}
```

We initialize a `config` command like this:

```go
configCmd := &cobra.Command{
    Use: "config",
    RunE: func(cmd *cobra.Command, args []string) error {
        return nil
    },
}
```

Our `config` command doesn't have any functionality, so we return nil in `RunE`.

Now we add the `config` command to `metana` command which is basically what happens when you hit `metana config ...`

```go
metanaCmd.AddCommand(configCmd)
```

Note the `Use` field inside the commands. They should be the same as the command you would be hitting on your CLI. They're there for a reason.

We've done the bare minimum. Let's see what our `set` command looks like.

```go
setCmd := &cobra.Command{
    Use:  "set",
    RunE: func(cmd *cobra.Command, args []string) error {
        FS := afero.NewMemMapFs()
        cmd.SetOut(&buf)
        afero.WriteFile(FS, "/Users/g14a/metana/.metana.yml", []byte("dir: schema-mig\nstore: \n"), 0644)
        err := RunSetConfig(cmd, FS, "/Users/g14a/metana")
        assert.NoError(t, err)
        file, err := afero.ReadFile(FS, "/Users/g14a/metana/.metana.yml")
        assert.NoError(t, err)
        assert.Equal(t, "dir: schema-mig\nstore: random\nenvironments: []\n", string(file))
        return nil
    },
}

setCmd.Flags().StringP("store", "s", "", "Set your store")
setCmd.Flags().StringP("dir", "d", "", "Set your migrations directory")
setCmd.Flags().StringP("env", "e", "", "Set config for your environment")
```

The `set` command needs the three flags mentioned below, so we add them to `setCmd` itself.

Let me explain what's going on in `RunE`. If you've written test cases in Go, we mostly mock our file systems or our database connections. I've found this fantastic library [spf13/afero](https://github.com/spf13/afero). It helps abstract file systems which is exactly what I need in this case.

You can check out what the function `RunSetConfig` does below

[https://github.com/g14a/metana/blob/main/pkg/cmd/config.go#L14](https://github.com/g14a/metana/blob/main/pkg/cmd/config.go#L14)

It writes some information into a config file called `.metana.yml`, which I read again in line no 9, and check if I get the expected result.

## **Making it testable**

We're ready to test it. But we need to find out a way to send in the arguments and flags as well. How do we do that?

We define a function which sends in all the arguments for us.

```go
func ExecuteCommandC(root *cobra.Command, args ...string) (c *cobra.Command, output string, err error) {
    buf := new(bytes.Buffer)
    root.SetOut(buf)
    root.SetErr(buf)
    root.SetArgs(args)

    c, err = root.ExecuteC()

    return c, buf.String(), err
}
```

But we can't send `setCmd` directly to this function because its a child command. To be precise its a grand child command of `metana`. So we add `setCmd` to `configCmd` and then add `configCmd` to `metana.`

```go
configCmd.AddCommand(setCmd)
metanaCmd.AddCommand(configCmd)
```

And now we send the whole of `metanaCmd` to `ExecuteCommandC`.

```go
c, out, err := pkg.ExecuteCommandC(metanaCmd, []string{"--dir=migrations", "--env=dev", "--store=random"})
```

Now the returned value `c` is basically the `set` command along with its properties. 

We can add a check whether we get `set` or any other command by checking its name like this:

```go
if c.Name() != "set" {
    t.Errorf(`invalid command returned from ExecuteC: expected "set"', got: %q`, c.Name())
}
```

So the above function i.e `ExecuteCommandC` simulates the following CLI command.

```shell
metana config set --dir migrations --env dev --store random
```

Pat yourselves because you're almost there.
I hope at this point you have an idea of what's going on here.

## **Making tests table driven**

Tests in Golang are known for being table driven which cover multiple scenarios at once.Let's see if we can make our subcommand testing table driven.

We declare an array of structs like always:

```go
tests := []struct {
    args     []string
    function func() func(cmd *cobra.Command, args []string) error
    output   string
}
```

* `args` are the values we send in while performing the operation.
* `function` is basically what would be assigned to `RunE` of the `setCmd` so we maintain the same signature of `RunE`
* `output` is a string representation of logs being printed out to the console(if any). 

Let's initialize an array like this:

```go
var buf bytes.Buffer

tests := []struct {
        args     []string
        function func() func(cmd *cobra.Command, args []string) error
        output   string
    }{
        {
            args: []string{"config", "set", "--store=random"},
            function: func() func(cmd *cobra.Command, args []string) error {
                return func(cmd *cobra.Command, args []string) error {
                    FS := afero.NewMemMapFs()
                    cmd.SetOut(&buf)
                    afero.WriteFile(FS, "/Users/g14a/metana/.metana.yml", []byte("dir: schema-mig\nstore: \n"), 0644)
                    err := RunSetConfig(cmd, FS, "/Users/g14a/metana")
                    assert.NoError(t, err)
                    file, err := afero.ReadFile(FS, "/Users/g14a/metana/.metana.yml")
                    assert.NoError(t, err)
                    assert.Equal(t, "dir: schema-mig\nstore: random\nenvironments: []\n", string(file))
                    return nil
            }
        },
            output: " ✓ Set config\n",
        },
        {
            args: []string{"config", "set", "--dir=migrations"},
            function: func() func(cmd *cobra.Command, args []string) error {
                return func(cmd *cobra.Command, args []string) error {
                    FS := afero.NewMemMapFs()
                    cmd.SetOut(&buf)
                    afero.WriteFile(FS, "/Users/g14a/metana/.metana.yml", []byte("dir: schema-mig\nstore: \n"), 0644)
                    err := RunSetConfig(cmd, FS, "/Users/g14a/metana")
                    assert.NoError(t, err)
                    file, err := afero.ReadFile(FS, "/Users/g14a/metana/.metana.yml")
                    assert.NoError(t, err)
                    assert.Equal(t, "dir: migrations\nstore: \"\"\nenvironments: []\n", string(file))
                    return nil
            }
        },
            output: " ! Make sure you rename your exising migrations directory to `migrations`\n ✓ Set config\n",
        },
        {
            args: []string{"config", "set", "--dir=migrations", "--env=dev"},
            function: func() func(cmd *cobra.Command, args []string) error {
                return func(cmd *cobra.Command, args []string) error {
                    FS := afero.NewMemMapFs()
                    cmd.SetOut(&buf)
                    afero.WriteFile(FS, "/Users/g14a/metana/.metana.yml", []byte("dir: schema-mig\nstore: \n"), 0644)
                    err := RunSetConfig(cmd, FS, "/Users/g14a/metana")
                    assert.NoError(t, err)
                    file, err := afero.ReadFile(FS, "/Users/g14a/metana/.metana.yml")
                    assert.NoError(t, err)
                    assert.Equal(t, "dir: schema-mig\nstore: \n", string(file))
                    return nil
            }
        },
            output: "No environment configured yet.\nTry initializing one with `metana init --env dev`\n",
        },
        {
            args: []string{"config", "set", "--dir=change-env-dir", "--env=dev"},
            function: func() func(cmd *cobra.Command, args []string) error {
                return func(cmd *cobra.Command, args []string) error {
                    FS := afero.NewMemMapFs()
                    cmd.SetOut(&buf)
                    afero.WriteFile(FS, "/Users/g14a/metana/.metana.yml", []byte("dir: schema-mig\nstore:\nenvironments:\n- name: dev\n  dir: dev\n  store: \"\"\n"), 0644)
                    err := RunSetConfig(cmd, FS, "/Users/g14a/metana")
                    assert.NoError(t, err)
                    file, err := afero.ReadFile(FS, "/Users/g14a/metana/.metana.yml")
                    assert.NoError(t, err)
                    assert.Equal(t, "dir: schema-mig\nstore: \"\"\nenvironments:\n- name: dev\n  dir: change-env-dir\n  store: \"\"\n", string(file))
                    return nil
                }
            },
            output: " ! Make sure you rename your exising environments directory to `change-env-dir`\n ✓ Set config\n",
        },
}
```

<p>
    <img src="https://media.giphy.com/media/ue1GO5swPdORq/giphy.gif" width="430" height="250"/>
    <em>Oh yeah!</em>
</p>


Now for each test, we initialize a new `metanaCmd`, `configCmd` and a `setCmd` like this and pass in the args:

```go
for _, tt := range tests {
    metanaCmd := NewMetanaCommand()

    configCmd := &cobra.Command{
        Use: "config",
        RunE: func(cmd *cobra.Command, args []string) error {
            return nil
        },
    }

    setCmd := &cobra.Command{
        Use:  "set",
        RunE: tt.function(),
    }
    setCmd.Flags().StringP("store", "s", "", "Set your store")
    setCmd.Flags().StringP("dir", "d", "", "Set your migrations directory")
    setCmd.Flags().StringP("env", "e", "", "Set config for your environment")
    configCmd.AddCommand(setCmd)
    metanaCmd.AddCommand(configCmd)
    c, out, err := pkg.ExecuteCommandC(metanaCmd, tt.args...)
    if out != "" {
        t.Errorf("Unexpected output: %v", out)
    }
    assert.NoError(t, err)
    assert.Equal(t, tt.output, buf.String())
    if c.Name() != "set" {
        t.Errorf(`invalid command returned from ExecuteC: expected "set"', got: %q`, c.Name())
    }
    buf.Reset()
}
```

Some points that I have missed are regarding the bytes.Buffer `buf`. This is the buffer which passed to each test's `setCmd` so that the console logs can be written into it. Notice the `cmd.SetOut(&buf)` in each `function()`. Also notice that the buffer is reset after every test, making it empty for the next test case.

## **Important parts to take home**

* You can directly use `ExecuteCommandC()` in your CLI app.
* Do not worry about the functionality of my application. It will obviously differ from your feature.   Write your own `function()` in the table tests.
* Add commands on top of another carefully i.e be clear about `metanaCmd.AddCommand(configCmd)`.
* Arguments and flags are passed as a slice of strings where each element is an argument/flag.
* Remember to set your buffers correctly to record logs on the console.

## **Conclusion**

Congrats! Good work! You have successfully tested sub commands in Cobra.

You can find the whole file here -> [https://github.com/g14a/metana/blob/b05ca789fee5fc7a501045d69b9666f294aeaa55/pkg/cmd/config_test.go](https://github.com/g14a/metana/blob/b05ca789fee5fc7a501045d69b9666f294aeaa55/pkg/cmd/config_test.go)

Feel free to check out how I implemented tests for other commands too. 

I hope you guys enjoyed this article. Please let me know if I missed anything or how I can do something better. Please also leave a star on the repo if you found this article useful or in general too. It doesn't cost a dime 😁

Together we all learn.

Cheers and good health!❤️❤️