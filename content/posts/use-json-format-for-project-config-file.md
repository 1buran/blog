---
title: "Use JSON format for project config file"
date: 2024-04-22T00:59:58+03:00
draft: false
toc: true
images:
tags:
  - Golang
  - rHttp
  - JSON
---

# Existing solutions of configuration management

There are so many libs for config parsing that hard to choose the one, for example Awesome Go,
configration section[^2] has 50+ libs!

When i wanted to add config to one small pet project, I faced to problem that many of those libs
were abandoned or look like _silver bullet_ sulution: it have a lot of extra features, that
actually no need to me.

I think I found a good way to create simple lightweight config system for small projects, basically
for pet projects. If you are interested, keep reading!

## .env, INI, YAML, TOML, HCL, language specific format... what's more?

All of these popular formats look like a good solution, but all they are require adding one more
project dependency - library for parsing that type of config. Often such library is very big and
not fast - it has lexers, parsers, tokenizers, some of them are not even stable. I'd not like to
add one more lib only for such utilitary things like config parsing: it is need to spent a time for
read its documentation, sometimes spent extra for find out workaroud of bugs, continiusly follow
the new releases and update it version in project... uh...

Thinking a lot... considering all type of configs, I've come up with an answer for a question in
topic: JSON. Yeah, simple, old fasioned json.

It is most widespread format in the web, but not only for web, some language package managers
create lock file of project dependencies in json format, i think every programming language has
json lib for encoding/decoding native data sturctures to/from json, so the config file in json
format, in this context, would be even cross platformed!

For example you even may to create a little web service for creation of custom config of your
softaware, to help users make customization of settings is less painful especially if your software
has a lot of settings.

Json is also has most spread support ever in different kind of IDE/editors, with validation and
autoformatting, that helps to reduce the main one of drawbacks of format: it is not human friendly
compared to others.

Let's try to create simple configuration management system based on json.

# Code

Fortunately Golang has a good way to decode/encode json to/from native data types.

Let's suppose our project is web app, it has a group of base parameters like timeouts, db
connections etc. They do not change very often, let's named them `static parameters`. And there are
some parameters that may change very often: add new ones, remove olds or change their names. Let's
call them `dynamic parameters` of project configuration.

Obviously for `static parameters` we can use `struct` which reflects the names of parameters in
config e.g.:

```go
type ServerSettings struct {
    Listen     string `json:"Listen"` // format host:port
    MaxWorkers int    `json:"MaxWorkers"`
}

type DatabaseSettings struct {
    Timeout         int    `json:"Timeout"` // in seconds
    MaxConnections  int    `json:"MaxConnections"`
    Uri             string `json:"dbUri"`    // contains db uri with credentials for connect
}

type Config struct {
    DatabaseSettings `json:"Database"`
    ServerSettings   `json:"Server"`
}
```

in the terms of json our config would like this:

```json
{
  "Server": {
    "Listen": "127.0.0.1:8080",
    "MaxWorkers": 100
  },
  "Database": {
    "Timeout": 300,
    "MaxConnections": 100,
    "dbUri": "mongodb+srv://[username:password@]host[/[defaultauthdb][?options]]"
  }
}
```

it looks very nice: sensible name of params, sections, it would be easy support and expand, even
though JSON is not a human friendly format, everyone can understand setting meaning at first look.

But what about `dynamic parameters`? Well.. we can use `maps` for them: `map[string]any` - map name
of option to any value, then add more logic about converting the value from JSON to appropriate
golang type, but in fact, such kind of options usually have identical types of values, I mean if
you need something add / remove very often then usually this is same type of data.

Let's suppose our project is a TUI app (terminal user interface) or desktop app (GUI app) which has
color themes with very dynamic values, we can say these are typical `dynamic parameters`: they may
change very often, some colors removed, some added, override the default values for some, in terms
of config it would be look like this:

```go
type Theme struct {
	Colors map[string]lipgloss.Color `json:"Colors"`
	Emojis map[string]string         `json:"Emojis"`
}

type Config struct {
    Theme   `json:"Theme"`
}
```

then we can add little bit logic for getting the colors from our config:

```go
func (c *Config) Color(s string) lipgloss.Color {
	color, ok := c.Colors[s]
	if !ok {
		fallbackColor := lipgloss.Color("11")
		return fallbackColor
	}
	return color
}
```

in term of json such config may looks like:

```json
{
  "Theme": {
    "Colors": {
      "checkboxOn": "42",
      "checkboxOff": "8",
      "fileinputPrompt": "177",
      "fileinputPlaceholder": "243",
      "fileinputText": "219",
      "statusbarFg": "#C1C6B2",
      "statusbarBg": "#353533",
      "statusbarTextWarning": "220",
      "statusbarTextError": "225",
      "statusbarIndicator": "#6124DF",
      "statusbarNugget": "#FFFDF5",
      "statusbarBadgeBg": "#59A8C9",
      "statusbarBadgeFg": "#FFFDF5",
      "statusbarBadgeError": "#FF5F87",
      "statusbarBadgeOk": "#2e8048",
      "statusbarBadgeWarning": "130",
      "statusbarReqCount": "#A550DF"
    }
  }
}
```

(the demonstrated config is the part of rHttp[^1] configuration)

Again it looks pretty nice, it is easily expandable, maintainable.

Ok, the next thing, that needs to be implemented, that every normal configuration system has, is
the overriding configuration.

## Configuration override system

First of all let's implement default configuration. There are many ways to do this, but I found the
next approach is more simple and better maintainable than others: _embed default config.json and
ship the app with it_. This approach is not require write extra code to setting default settings
values, for instance check the values of settings after read the user config and if some is missed
used the default value. Instead of all of this we just read default config file and apply the
settings performing the same actions like for user config file.

```go
//go:embed config.json
var defaultConfig []byte

func defaultSettings(c *Config) error {
	return json.Unmarshal(defaultConfig, c)
}

func overrideSettings(path, c *Config) error {
	b, err := os.ReadFile(path)
	if err != nil {
		return err
	}

    if err = json.Unmarshal(b, c); err != nil {
		return err
	}

	return nil
}

func ReadConfig(configPath string) (*Config, error) {
	c := &Config{}

	if err := defaultSettings(c); err != nil {
		return nil, err
	}

	if err := overrideSettings(configPath, c); err != nil {
		return nil, err
	}

	return c, nil
}
```

Firstly `ReadConfig` loads default settings values to instance of `Config`, then overrides the
default settings by the user defined. That looks pretty organic: created instance of config is
mutated by the overridings. We even might add more than one source of config to create complex
system of lookup pathes e.g.:

- firstly load the default settings which is ship together with app (config.json is embedded into
  binary file) (lowest precedence), then
- lookup the config in current work dir `./app-config.json`, then
- lookup the config in user home `~/.config/app/app-config.json`, then
- load config from path given from cmd arg `app -c /opt/etc/app/app-config.json` (highest
  precendence)

We can continue to add more and more sources if we needed, for example we might add support of
`.env` or loading the config from web, whatever we want. Cool.

# Conclusion

The configuration management system we have created is flexible and reliable, expandable and
maintainable. That is what i wanted from configuration management system! Hope you had fun and
learned something useful for yourself! Any feedback is welcome!

<!-- prettier-ignore-start -->

[^1]: rHttp - REPL for http request in terminal, lightweight, simple and fast, without black magic, [see demo on GitHub](https://github.com/1buran/rHttp)
[^2]: see libs on [Awesome Go / Configuration section](https://awesome-go.com/configuration/) or the same on the [GitHub](https://github.com/avelino/awesome-go?tab=readme-ov-file#configuration)

<!-- prettier-ignore-end -->
