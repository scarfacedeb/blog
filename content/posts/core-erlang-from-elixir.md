---
title: "Getting Core Erlang from Elixir"
date: 2019-01-07T18:02:57+03:00
tags: ["elixir"]
---


It's mentioned in [Erlang/Elixir Syntax: A Crash Course](https://elixir-lang.org/crash-course.html#adding-elixir-to-existing-erlang-programs) that Elixir compiles into BEAM byte code via [Erlang Abstract Format](http://erlang.org/doc/apps/erts/absform.html). But actually, there's an extra step between the Erlang Abstract Format and the BEAM byte code – [Core Erlang](https://www.it.uu.se/research/group/hipe/cerl/doc/core_erlang-1.0.3.pdf).

It's a neat little language with strict syntax that isn't suited for writing by hand, but is used as an intermediary format to run optimizations, checks and simplifications.  
[Dialyzer](http://erlang.org/doc/man/dialyzer.html) also works on the Core Erlang level.

<!--more-->

## Elixir/Erlang compilation steps

![Elixir compilation steps](/img/elixir_compilation.png)
[Source](https://elixirforum.com/t/getting-each-stage-of-elixirs-compilation-all-the-way-to-the-beam-bytecode/1873/5)

## Elixir -> Core Erlang transformation

1. Write an elixir module and save it into `test.ex`:

    ```elixir
    # test.ex
    defmodule Test do
      @num 10

      def calc(42), do: :ok

      def calc(list) when is_list(list) do
        [h | _] = list
        h + @num + :rand.uniform()
      end
    end
    ```

2. Compile it to produce `Elixir.Test.beam` file:

    ```bash
    $> elixirc test.ex
    ```

3. Get Erlang Abstract Format from compiled BEAM module:

    ```elixir
    {:ok, {_, [abstract_code: {_, ac}]}} = 
      Test |> :code.which() |> :beam_lib.chunks([:abstract_code])

    ac # erlang abstract format
    ```

    It'll return the following structure:

    ```elixir
    [
      {:attribute, 1, :file, {'lib/test.ex', 1}},
      {:attribute, 1, :module, Test},
      {:attribute, 1, :compile, [:no_auto_import]},
      {:attribute, 1, :export, [__info__: 1, calc: 1]},

      #... __info__ omitted

      {:function, 6, :calc, 1,
      [
        {:clause, 4, [{:integer, 0, 42}], [], [{:atom, 0, :ok}]},
        {:clause, 6, [{:var, 6, :_list@1}],
          [
            [
              {:call, 6, {:remote, 6, {:atom, 0, :erlang}, {:atom, 6, :is_list}},
              [{:var, 6, :_list@1}]}
            ]
          ],
          [
            {:match, 7, {:cons, 0, {:var, 7, :_h@1}, {:var, 7, :_}},
            {:var, 7, :_list@1}},
            {:op, 8, :+, {:op, 8, :+, {:var, 8, :_h@1}, {:integer, 0, 10}},
            {:call, 8, {:remote, 8, {:atom, 0, :rand}, {:atom, 8, :uniform}}, []}}
          ]}
      ]}
    ]}
    ```

4. Get Erlang module out of the abstract code and save it into a file:

    ```elixir
    erl = :erl_prettypr.format(:erl_syntax.form_list(ac))
    File.write!("Elixir.Test.erl", erl)
    ```

    It'll return an ordinary erlang module:

    ```erlang
    -file("test.ex", 1).

    -module('Elixir.Test').

    -compile([no_auto_import]).

    -export(['__info__'/1, calc/1]).

    % ... __info__ omitted ...

    calc(42) -> ok;
    calc(_list@1) when erlang:is_list(_list@1) ->
    [_h@1 | _] = _list@1, _h@1 + 10 + rand:uniform().
    ```

    Now you can get a clear picture how Elixir modules map to Erlang code.

    But we're not finished just yet.

4. Finally, compile erlang module to get Core Erlang representation:

    ```elixir
    # Compile erl
    $> erlc +to_core Elixir.Test.erl
    ```

    It'll produce `Elixir.Test.core` file:

    ```shell
    module 'Elixir.Test' ['__info__'/1,
          'calc'/1,
          'module_info'/0,
          'module_info'/1]
    attributes [%% Line 1
    'file' =
        %% Line 1
        [{[69|[108|[105|[120|[105|[114|[46|[84|[101|[115|[116|[46|[101|[114|[108]]]]]]]]]]]]]]],1}],
    %% Line 1
    'file' =
        %% Line 1
        [{[116|[101|[115|[116|[46|[101|[120]]]]]]],1}],

    %% Line 5
    'compile' =
        %% Line 5
        ['no_auto_import'],

    %% ... '__info__'/1 omitted ...

    'calc'/1 =
        %% Line 23
        fun (_0) ->
      case _0 of
        <42> when 'true' ->
            'ok'
        %% Line 24
        <_X_list@1>
            when call 'erlang':'is_list'
            (_0) ->
            %% Line 25
            case _X_list@1 of
        <[_X_h@1|_5]> when 'true' ->
            let <_3> =
          call 'erlang':'+'
              (_X_h@1, 10)
            in  let <_2> =
              call 'rand':'uniform'
            ()
          in  call 'erlang':'+'
            (_3, _2)
        ( <_1> when 'true' ->
              primop 'match_fail'
            ({'badmatch',_1})
          -| ['compiler_generated'] )
            end
        ( <_4> when 'true' ->
        ( primop 'match_fail'
              ({'function_clause',_4})
          -| [{'function_name',{'calc',1}}] )
          -| ['compiler_generated'] )
      end

    'module_info'/0 =
        fun () ->
      call 'erlang':'get_module_info'
          ('Elixir.Test')
    'module_info'/1 =
        fun (_0) ->
      call 'erlang':'get_module_info'
          ('Elixir.Test', _0)
          end
    ```


## P.S: Further experiments with debug_info.

Prior to OTP 20, BEAM had a special chunk - `Abst` - that kept Erlang Abstract Code inside the BEAM files.

OTP 20 introduced new BEAM chunk: `Dbgi` that allowed to store Elixir Abstract Code in the BEAM files, instead of Erlang.

It didn't break any of the old code, because `:beam_lib.chunks([:abstract_code])` converts Elixir Abstract Code into Erlang on the fly.

We can compare `debug_info` metadata for Elixir and Erlang modules to get better idea how it works.

### Debug_info chunk for Elixir module

Compile Elixir module and get `debug_info` chunk:

```elixir
# compile Elixir module
$> elixirc test.ex

'Elixir.Test.beam' |> :beam_lib.chunks([:debug_info])
```

You'll get `elixir_erl` abstract code in the debug_info:

```
[
  debug_info: {:debug_info_v1, :elixir_erl,
    {:elixir_v1,
    %{
      attributes: [],
      compile_opts: [],
      definitions: [
        {{:calc, 1}, :def, [line: 6],
          [
            {[line: 4], '*', [], :ok},
            {[line: 6], [{:list, [line: 6], nil}],
            [
              {{:., [], [:erlang, :is_list]}, [line: 6],
                [{:list, [line: 6], nil}]}
            ],
            {:__block__, [line: 6],
              [
                {:=, [line: 7],
                [
                  [
                    {:|, [line: 7],
                      [{:h, [line: 7], nil}, {:_, [line: 7], nil}]}
                  ],
                  {:list, [line: 7], nil}
                ]},
                {{:., [], [:erlang, :+]}, [line: 8],
                [
                  {{:., [], [:erlang, :+]}, [line: 8],
                    [{:h, [line: 8], nil}, 10]},
                  {{:., [line: 8], [:rand, :uniform]}, [line: 8], []}
                ]}
              ]}}
          ]}
      ],
      deprecated: [],
      file: "**/test.ex",
      line: 1,
      module: Test,
      unreachable: []
    }, []}}
]
```


### Debug_info chunk for Erlang module

Compile previous Erlang module and get `debug_info` chunk:

```elixir
# compile Erlang module
$> erlc +debug_info Elixir.Test.erl

'Elixir.Test.beam' |> :beam_lib.chunks([:debug_info])
```

Now you'll get `erlang_abstract_code` directly instead:

```
 [
    debug_info: {:debug_info_v1, :erl_abstract_code,
     {[
        {:attribute, 1, :file, {'Elixir.Test.erl', 1}},
        {:attribute, [generated: true, location: 1], :file, {'test.ex', 1}},
        {:attribute, 3, :module, Test},

        %% ... the same Erlang Abstract Code as above ...
```


### Debug_info chunk for Core Erlang

As final experiment, compile `Elixir.Test.core` file:

```elixir
# Shell
$> erlc +debug_info +to_core Elixir.Test.erl
$> erlc +debug_info Elixir.Test.core

'Elixir.Test.beam' |> :beam_lib.chunks([:debug_info])
```

There's no abstract_code metadata in debug_info:

```
[
  debug_info: {:debug_info_v1, :erl_abstract_code,
    {[], [{:cwd, '/**'}, :debug_info]}}
]
```

## Sources:

* [The Feature That No One Knew About in Elixir 1.5 - José Valim - Elixir.LDN 2017](https://youtu.be/p4uE-jTB_Uk)
* [The Core of Erlang | 8th Light](https://8thlight.com/blog/kofi-gumbs/2017/05/02/core-erlang.html)
* [Core Erlang by Example – A Blog from the Erlang/OTP team.](http://blog.erlang.org/core-erlang-by-example/)
* [Disassemble Elixir code – Learn Elixir](https://medium.com/learn-elixir/disassemble-elixir-code-1bca5fe15dd1)
