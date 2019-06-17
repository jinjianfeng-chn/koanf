![koanf](https://user-images.githubusercontent.com/547147/59548139-fb2dd880-8f67-11e9-9af5-2fe2629a8fa6.png)

**koanf** (pronounced _conf_; a play on the Japanese _Koan_) is a library for reading configuration from different sources in different formats in Go applications. It is a cleaner, lighter [alternative to spf13/viper](#alternative-to-viper) with better abstractions and extensibility and fewer dependencies.

koanf comes with built in support for reading configuration from files, command line flags, and environment variables, and can parse JSON, YAML, TOML, and Hasicorp HCL.

### Installation

`go get -u github.com/knadh/koanf`

### Concepts

- `koanf.Provider` is a generic interface that provides configuration, for example, from files, enviornment variables, HTTP sources, or anywhere. The configuration can either be raw bytes that a parser can parse, or it can be a nested map[string]interface{} that can be directly loaded.
- `koanf.Parser` is a generic interface that takes raw bytes, parses, and returns a nested map[string]interface{} representation. For example, JSON and YAML parsers.
- Once loaded into koanf, configuration are values queried by a delimited key path syntax. eg: `app.server.port`. Any delimiter can be chosen.
- Configuration from multiple sources can be loaded and merged into a koanf instance, for example, load from a file first and override certain values with flags from the command line.

With these two interface implementations, koanf can obtain configuration from multiple sources and parse any format and make it available to an application.

### Reading config from files

```go
package main

import (
	"fmt"
	"log"

	"github.com/knadh/koanf"
	"github.com/knadh/koanf/parsers/json"
	"github.com/knadh/koanf/parsers/yaml"
	"github.com/knadh/koanf/providers/file"
)

// Global koanf instance. Use "." as the key path delimiter. This can be "/" or any character.
var k = koanf.New(".")

func main() {
	// Load JSON config.
	if err := k.Load(file.Provider("mock/mock.json"), json.Parser()); err != nil {
		log.Fatalf("error loading config: %v", err)
	}

	// Load YAML config and merge into the previously loaded config (because we can).
	k.Load(file.Provider("mock/mock.yaml"), yaml.Parser())

	fmt.Println("parent's name is = ", k.String("parent1.name"))
	fmt.Println("parent's ID is = ", k.Int("parent1.id"))
}

```

### Reading from command line

The following example shows the use of `posflag.Provider`, a wrapper over [spf13/pflag](/spf13/pflag) library, an advanced commandline lib. For Go's built in `flag` package, use `basicflag.Provider`.

```go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/knadh/koanf"
	"github.com/knadh/koanf/parsers/toml"
	"github.com/knadh/koanf/providers/file"
	"github.com/knadh/koanf/providers/posflag"
	flag "github.com/spf13/pflag"
)

// Global koanf instance. Use "." as the key path delimiter. This can be "/" or any character.
var k = koanf.New(".")

func main() {
	// Use the POSIX compliant pflag lib instead of Go's flag lib.
	f := flag.NewFlagSet("config", flag.ContinueOnError)
	f.Usage = func() {
		fmt.Println(f.FlagUsages())
		os.Exit(0)
	}
	// Path to one or more config files to load into koanf along with some config params.
	f.StringSlice("conf", []string{"mock/mock.toml"}, "path to one or more .toml config files")
	f.String("time", "2020-01-01", "a time string")
	f.String("type", "xxx", "type of the app")
	f.Parse(os.Args[1:])

	// Load the config files provided in the commandline.
	cFiles, _ := f.GetStringSlice("conf")
	for _, c := range cFiles {
		if err := k.Load(file.Provider(c), toml.Parser()); err != nil {
			log.Fatalf("error loading file: %v", err)
		}
	}

	// "time" and "type" may have been loaded from the config file, but
	// they can still be overridden with the values from the command line.
	// The bundled posflag.Provider takes a flagset from the spf13/pflag lib.
	if err := k.Load(posflag.Provider(f, "."), nil); err != nil {
		log.Fatalf("error loading config: %v", err)
	}

	fmt.Println("time is = ", k.String("time"))
}

```

### Reading environment variables

```go
package main

import (
	"fmt"
	"log"
	"strings"

	"github.com/knadh/koanf"
	"github.com/knadh/koanf/parsers/json"
	"github.com/knadh/koanf/providers/env"
	"github.com/knadh/koanf/providers/file"
)

// Global koanf instance. Use . as the key path delimiter. This can be / or anything.
var k = koanf.New(".")

func main() {
	// Load JSON config.
	if err := k.Load(file.Provider("mock/mock.json"), json.Parser()); err != nil {
		log.Fatalf("error loading config: %v", err)
	}

	// Load environment variables and merge into the loaded config.
	// "MYVAR" is the prefix to filter the env vars by.
	// "." is the delimiter used to represent the key hierarchy in env vars.
	// The (optional, or can be nil) function can be used to transform
	// the env var names, for instance, to lowercase them.
	//
	// For example, env vars: MYVAR_TYPE and MYVAR_PARENT1.CHILD1.NAME
	// will me merged into the "type" and the nested "parent1.child1.name"
	// keys in the config file.
	k.Load(env.Provider("MYVAR_", ".", func(s string) string {
		return strings.ToLower(s)
	}), nil)

	fmt.Println("name is = ", k.String("parent1.child1.name"))
}
```

### Reading arbitrary bytes

The bundled `rawbytes` Provider can be used to read arbitrary bytes from a source, like a database or an HTTP call.

```go
// Package rawbytes implements a koanf.Provider that takes a []byte slice
// and provides it to koanf to be parsed by a koanf.Parser.
package rawbytes

import (
	"errors"
)

// RawBytes implements a raw bytes provider.
type RawBytes struct {
	b []byte
}

// Provider returns a provider that takes a raw []byte slice to be parsed
// by a koanf.Parser parser. This should be a nested conf map, like the
// contents of a raw JSON config file.
func Provider(b []byte) *RawBytes {
	r := &RawBytes{b: make([]byte, len(b))}
	copy(r.b[:], b)
	return r
}

// ReadBytes is not supported by the env provider.
func (r *RawBytes) ReadBytes() ([]byte, error) {
	return r.b, nil
}

// Read returns the raw bytes for parsing.
func (r *RawBytes) Read() (map[string]interface{}, error) {
	return nil, errors.New("buf provider does not support this method")
}
```

### Unmarshalling

Unmarshalling is useful when you want to copy a nested config map into a struct (like unmarshalling JSON) instead of accessing individual config values using gettor methods.

```go
package main

import (
	"fmt"
	"log"

	"github.com/knadh/koanf"
	"github.com/knadh/koanf/parsers/json"
	"github.com/knadh/koanf/providers/file"
	"github.com/mitchellh/mapstructure"
)

// Global koanf instance. Use . as the key path delimiter. This can be / or anything.
var k = koanf.New(".")

func main() {
	// Load JSON config.
	if err := k.Load(file.Provider("mock/mock.json"), json.Parser()); err != nil {
		log.Fatalf("error loading config: %v", err)
	}

	// Structure to unmarshal nested conf to.
	type childStruct struct {
		Name       string            `koanf:"name"`
		Type       string            `koanf:"type"`
		Empty      map[string]string `koanf:"empty"`
		GrandChild struct {
			Ids []int `koanf:"ids"`
			On  bool  `koanf:"on"`
		} `koanf:"grandchild1"`
	}

	var out childStruct

	// Quick unmarshal.
	k.Unmarshal("parent1.child1", &out)
	fmt.Println(out)

	// Unmarshal with advanced config.
	out = childStruct{}
	k.UnmarshalWithConfig("parent1.child1", koanf.UnmarshalConf{
		DecoderConfig: &mapstructure.DecoderConfig{
			Metadata:         nil,
			Result:           &out,
			WeaklyTypedInput: true,

			// Customize the struct tag, for eg:, "json".
			TagName: "koanf",
		},
	})

}

```

### Setting default values.

The bundled `confmap` provider takes a `map[string]interface{}` that can be loaded into a koanf instance. Thus, koanf does not provide any special functions to set default values but uses the Provider interface to enable it.

```go
package main

import (
	"fmt"
	"log"

	"github.com/knadh/koanf"
	"github.com/knadh/koanf/providers/confmap"
	"github.com/knadh/koanf/providers/file"
	"github.com/knadh/koanf/parsers/json"
	"github.com/knadh/koanf/parsers/yaml"
)

// Global koanf instance. Use "." as the key path delimiter. This can be "/" or any character.
var k = koanf.New(".")

func main() {
	// Load default values using the confmap provider.
	// We provide a flat map with the "." delimiter.
	// A nested map can be loaded by setting the delimiter to an empty string "".
	k.Load(confmap.Provider(map[string]interface{}{
		"parent1.name": "Default Name",
		"parent3.name": "New name here",
	}, "."), nil)

	// Load JSON config on top of the default values.
	if err := k.Load(file.Provider("mock/mock.json"), json.Parser()); err != nil {
		log.Fatalf("error loading config: %v", err)
	}

	// Load YAML config and merge into the previously loaded config (because we can).
	k.Load(file.Provider("mock/mock.yaml"), yaml.Parser())

	fmt.Println("parent's name is = ", k.String("parent1.name"))
	fmt.Println("parent's ID is = ", k.Int("parent1.id"))
}
```

### Order of merge and key case senstivity

- Config keys are case sensitive in koanf. For example, `app.server.port` and `APP.SERVER.port` are not the same.
- koanf does not impose any ordering on loading config from various providers. Every successive `Load()` or `Load()` merges new config into existing config. That means it is possible to load environment variables first, then files on top of it, and then command line variables on top of it, or any such order.

### Custom Providers and Parsers

A Provider can provide a nested map[string]interface{} config that can be loaded into koanf with `koanf.Load()` or raw bytes that can be parsed with a Parser (loaded using `koanf.Load()`.

Writing Providers and Parsers are easy. See the bundled implementations in the `providers` and `parses` directory.

## API

### Bundled providers

| Package             | Provider                                                      | Description                                                                                                                                                                           |
| ------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| providers/file      | `file.Provider(filepath string)`                              | Reads a file and returns the raw bytes to be parsed.                                                                                                                                  |
| providers/basicflag | `basicflag.Provider(f *flag.FlagSet, delim string)`           | Takes an stdlib `flag.FlagSet`                                                                                                                                                        |
| providers/posflag   | `posflag.Provider(f *pflag.FlagSet, delim string)`            | Takes an `spft3/pflag.FlagSet` (advanced POSIX compatiable flags with multiple types) and provides a nested config map based on delim.                                                |
| providers/env       | `env.Provider(prefix, delim string, f func(s string) string)` | Takes an optional prefix to filter env variables by, an optional function that takes and returns a string to transform env variables, and returns a nested config map based on delim. |
| providers/confmap   | `confmap.Provider(mp map[string]interface{}, delim string)`   | Takes a premade `map[string]interface{}` conf map. If delim is provided, the keys are assumed to be flattened, thus unflattened using delim.                                          |
| providers/rawbytes  | `rawbytes.Provider(b []byte)`                                 | Takes a raw `[]byte` slice to be parsed with a koanf.Parser                                                                                                                           |

### Bundled parsers

| Package      | Parser                           | Description                                                                                                                                               |
| ------------ | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| parsers/json | `json.Parser()`                  | Parses JSON bytes into a nested map                                                                                                                       |
| parsers/yaml | `yaml.Parser()`                  | Parses YAML bytes into a nested map                                                                                                                       |
| parsers/toml | `toml.Parser()`                  | Parses TOML bytes into a nested map                                                                                                                       |
| parsers/hcl  | `hcl.Parser(flattenSlices bool)` | Parses Hashicorp HCL bytes into a nested map. `flattenSlices` is recommended to be set to true. [Read more](https://github.com/hashicorp/hcl/issues/162). |

### Instance methods

| Method                                                         | Description                                                                                                                            |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Load(p Provider, pa Parser) error`                            | Loads config from a Provider. If a koanf.Parser is provided, the config is assumed to be raw bytes that's then parsed with the Parser. |
| `Keys() []string`                                              | Returns the list of flattened key paths that can be used to access config values                                                       |
| `KeyMap() map[string][]string`                                 | Returns a map of all possible key path combinations possible in the loaded nested conf map                                             |
| `All() map[string]interface{}`                                 | Returns a flat map of flattened key paths and their corresponding config values                                                        |
| `Raw() map[string]interface{}`                                 | Returns a copy of the raw nested conf map                                                                                              |
| `Print()`                                                      | Prints a human readable copy of the flattened key paths and their values for debugging                                                 |
| `Sprint()`                                                     | Returns a human readable copy of the flattened key paths and their values for debugging                                                |
| `Cut(path string) *Koanf`                                      | Cuts the loaded nested conf map at the given path and returns a new Koanf instance with the children                                   |
| `Copy() *Koanf`                                                | Returns a copy of the Koanf instance                                                                                                   |
| `Merge(*Koanf)`                                                | Merges the config map of a Koanf instance into the current instance                                                                    |
| `Unmarshal(path string, o interface{}) error`                  | Scans the given nested key path into a given struct (like json.Unmarshal) where fields are denoted by the `koanf` tag                  |
| `Unmarshal(path string, o interface{}, c UnmarshalConf) error` | Like Unmarshal but with customizable options                                                                                           |

### Getter methods

|                                       |                                                                                                                                                                                            |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Get(path string) interface{}`        | Returns the value for the given key path, and if it doesn’t exist, returns nil                                                                                                             |
| `Exists(path string) bool`            | Returns true if the given key path exists in the conf map                                                                                                                                  |
| `Int64(path string) int64`            |                                                                                                                                                                                            |
| `Int64s(path string) []int64`         |                                                                                                                                                                                            |
| `Int(path string) int`                |                                                                                                                                                                                            |
| `Ints(path string) []int`             |                                                                                                                                                                                            |
| `Float64(path string) float64`        |                                                                                                                                                                                            |
| `Float64s(path string) []float64`     |                                                                                                                                                                                            |
| `Duration(path string) time.Duration` | Returns the time.Duration value of the given key path if it’s numeric (attempts a parse+convert if string)                                                                                 |
| `Time(path, layout string) time.Time` | Parses the string value of the the given key path with the given layout format and returns time.Time. If the key path is numeric, treats it as a UNIX timestamp and returns its time.Time. |
| `String(path string) string`          |                                                                                                                                                                                            |
| `Strings(path string) []string`       |                                                                                                                                                                                            |
| `Byte(path string) []byte`            |                                                                                                                                                                                            |
| `Bool(path string) bool`              |                                                                                                                                                                                            |
| `Bools(path string) []bool`           |                                                                                                                                                                                            |

### Alternative to viper

koanf is a light weight alternative to the popular [spf13/viper](https://github.com/spf13/viper). It does not aim to do everything viper does (such as mutating config maps and writing them back to files), but provides simpler primitives for reading and accessing configuration. It was written as a result of multiple stumbling blocks encountered with some of viper's fundamental flaws.

- viper breaks JSON, YAML, TOML, HCL language specs by [forcibly lowercasing keys](https://github.com/spf13/viper/pull/635).
- Tightly couples config parsing with file extensions.
- Has poor semantics and abstractions. Commandline, env, file etc. and various parses are hardcoded in the core. There are no primitives that can be extended.
- Pulls a large number of [third party dependencies](https://github.com/spf13/viper/issues/707) into the core package. For instance, even if you do not use YAML or flags, the dependencies are still pulled as a result of the coupling.
- Imposes arbitrary ordering conventions (eg: flag -> env -> config etc.)
- `Get()` returns references to slices and maps. Mutations made outside change the underlying values inside the conf map.
- Does non-idiomatic things such as [throwing away O(1) on flat maps](https://github.com/spf13/viper/blob/3b4aca75714a37276c4b1883630bd98c02498b73/viper.go#L1524).
- There are a large number of [open issues](https://github.com/spf13/viper/issues).