# pegviz

![](https://img.shields.io/badge/quick%20hack%3F-yes!-green)
![](https://img.shields.io/badge/maintained%3F-not%20really-red)
![](https://img.shields.io/badge/license-mit-blue)

Visualizer for https://crates.io/crates/peg parsers.

## Screenshot

`pegviz` reads peg's tracing markers and generates a collapsible HTML tree.

![](https://user-images.githubusercontent.com/7998310/80628077-1fe05100-8a51-11ea-87aa-4b8362adf56c.png) 

Left side:

  * Green: matched rule
  * Red: failed rule

Right side:

  * Gray: previous input, for context
  * Blue background: input matched by this rule
  * White text: rest of input after matching

## Format

`pegviz` expects input in the following format:

```
[PEG_INPUT_START]
int a = 12 + 45;
[PEG_TRACE_START]
[PEG_TRACE] Attempting to match rule `translation_unit0` at 1:1
[PEG_TRACE] Attempting to match rule `list0` at 1:1
[PEG_TRACE] Attempting to match rule `node` at 1:1
[PEG_TRACE] Attempting to match rule `external_declaration` at 1:1
[PEG_TRACE] Attempting to match rule `declaration` at 1:1
[PEG_TRACE] Attempting to match rule `node` at 1:1
[PEG_TRACE] Attempting to match rule `declaration0` at 1:1
[PEG_TRACE] Attempting to match rule `gnu` at 1:1
[PEG_TRACE] Attempting to match rule `gnu_guard` at 1:1
[PEG_TRACE] Failed to match rule `gnu_guard` at 1:1
[PEG_TRACE] Failed to match rule `gnu` at 1:1
[PEG_TRACE] Attempting to match rule `_` at 1:1
[PEG_TRACE] Matched rule `_` at 1:1 to 1:1
```

The `_START` and `_STOP` marker are pegviz-specific, you'll need to add
them to your program. See the **Integration** section for more information.

Multiple traces may be processed, they'll all show up in the output file.
Output that occurs *between* traces is ignored.

## Compatibility

pegviz has been used with:

  * peg 0.5.7
  * peg 0.6.2

There are no tests. It's quickly thrown together.

## Integration

In your crate, re-export the `trace` feature:

```
# in Cargo.toml

[features]
trace = ["peg/trace"]
```

Then, in your parser, add a `tracing` rule that captures all the input
and outputs the markers `pegviz` is looking for:

```rust
peg::parser! { pub grammar example() for str {

rule traced<T>(e: rule<T>) -> T =
    &(input:$([_]*) {
        #[cfg(feature = "trace")]
        println!("[PEG_INPUT_START]\n{}\n[PEG_TRACE_START]", input);
    })
    e:e()? {?
        #[cfg(feature = "trace")]
        println!("[PEG_TRACE_STOP]");
        e.ok_or("")
    }

pub rule toplevel() -> Toplevel = traced(<toplevel0()>)

}}
```

If your parser uses slices (such as `&[u8]`, `&[T]`), then each character or token must be on a new line.

```rust
peg::parser! { pub grammar example() for str {

rule traced<T>(e: rule<T>) -> T =
    &(input:$([_]*) {
        #[cfg(feature = "trace")]
        println!(
            "[PEG_INPUT_START]\n{}\n[PEG_TRACE_START]",
            input.iter().fold(
                String::new(),
                |s1, s2| s1 + "\n" + s2.to_string().as_str()
            ).trim_start().to_string()
        );
    })
    e:e()? {?
        #[cfg(feature = "trace")]
        println!("[PEG_TRACE_STOP]");
        e.ok_or("")
    }

pub rule toplevel() -> Toplevel = traced(<toplevel0()>)

}}
```

The above is the recommended way *if you're maintaining the grammar* and want
to be able to turn on pegviz support anytime.

If you're debugging someone else's parser, you may want to print the start/stop
markers and the source yourself, around the parser invocation, like so:

```rust
    let source = std::fs::read_to_string(&source).unwrap();
    println!("[PEG_INPUT_START]\n{}\n[PEG_TRACE_START]", source);
    let res = lang_c::driver::parse_preprocessed(&config, source);
    println!("[PEG_TRACE_STOP]");
```

Make sure you've installed `pegviz` into your `$PATH`:

```shell
cd pegviz/
cargo install --force --path .
```

> While installing it, you may notice `pegviz` depends on `peg`.
> That's right! It's using a PEG to analyze PEG traces.

Then, simply run your program with the `trace` Cargo feature enabled, and
pipe its standard output to `pegviz`.

```shell
cd example/
cargo run --features trace | pegviz --output ./pegviz.html
```

Note that the `--output` argument is mandatory.

The last step is to open the resulting HTML file in a browser and click around!

## License

pegviz is released under the MIT License. See the LICENSE file for details.
