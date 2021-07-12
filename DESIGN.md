Some rambling thoughts that don't deserve a place in the README.

# Cleanness
The API consists of two clean simple powerful methods with no arguments and no configurability, plus some more junk to make them convenient to use.

I don't really like the `ValueExt` extension trait, but I can't think of a nicer way to parse values.

In my ideal workflow you would call `.into_string()?.parse()?` to parse a value, all built-in methods. But I don't think it's possible to have an error type that can be transformed from both methods' error types, `into_string` returns `OsString` and there are annoying rules around overlapping trait implementations. The error messages would also suffer.

Keeping the core API clean and generic means this could perhaps be used as the basis of a more complete parser.

# Possible enhancements
It is sometimes useful to look at argv[0], the command name. `Parser::from_env` currently discards it. Two use cases are help and error messages, where a decoded string is appropriate, but an `&OsStr` would be lossless. For now it can be extracted manually before calling `Parser::from_args`.

Some programs have flags with optional arguments. `-fvalue` counts, `-f value` does not. There's a private method that supports exactly this behavior but I don't know if exposing it is a good idea.

It is sometimes useful to get argv[0], the command name. `Parser::from_env` currently discards it. Two use cases are help and error messages, where a decoded string is appropriate, but an `&OsStr` would be lossless. For now it can be extracted manually before calling `Parser::from_args`.

POSIX [recommends](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html#tag_12_02) that no more options are parsed after the first free-standing argument. GNU getopt does this if the $POSIXLY_CORRECT environment variable is set. It could be supported by setting `finished_opts` when the first `Arg::Value` is found.

POSIX also has a notion of subarguments, combining multiple values in a single option-argument by separating them with commas or spaces. This is easy enough to hand-roll for valid unicode (`.into_string()?.split(...)`) but we could provide a function that does it on `OsString`s. I can't think of a case where values may not be valid unicode but definitely don't contain commas or spaces, though.

I'm not convinced any of these are good ideas.

# Language quirks
Sometimes Rust is a bother.

`Arg::Long` contains a borrowed string instead of an owned string because you can't match owned strings against string literals. That means `Arg` needs a lifetime, the iterator protocol cannot be used (it would also be a bad fit for other reasons), and error messages can be slightly better.

Arguments on Windows sometimes have to be transcoded three times: from UTF-16 to WTF-8 by `args_os`, then back to UTF-16 to parse them, then to WTF-8 again to be used. This ensures we see the original invalid code unit if there's a problem, but it's a bit sad.

# Errors
There's not always enough information for a good error message. A plain `OsString` doesn't remember what the parser knows, like what the last flag was.

`ValueExt::parse` exists to include the original string in an error message and to wrap all errors inside a uniform type. It's unclear if it earns its upkeep.

# Problems in other libraries
These are all defensible design choices, they're just a bad fit for some of the programs I want to write. All of them make some other kind of program easier to write.

## pico-args
- Results can be erratic in edge cases: option arguments may be interpreted as flags, the order in which you request flags matters, and arguments may get treated as if they're next to each other if the arguments inbetween get parsed first.
- `--` as a separator is not built in.
- Arguments that are not valid unicode are not recognized as flags, even if they start with a dash.
- Left-over arguments are ignored by default. I prefer when the path of least resistance is strict.
- It uses `Vec::remove`, so it's potentially slow if you pass many thousands of flags. (This is a bit academic, there's no problem for realistic workloads.)

These make the library simpler and smaller, which is the whole point.

## clap/structopt
- structopt nudges the user toward needlessly panicking on invalid unicode: even if a field has type `OsString` or `PathBuf` it'll round-trip through a unicode string and panic unless `from_os_str` is used. (I don't know if this is fixable even in theory while keeping the API ergonomic.)
- Invalid unicode causes a panic instead of a soft error.
- Options with a variable number of arguments are supported, even though they're ambiguous. In structopt you need to take care not to enable this if you want a flag that can occur multiple times with a single argument each time.
- They're large, both in API surface and in code size.

That said, it's still my first choice for complicated interfaces.