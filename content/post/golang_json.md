---
title: "Unmarshaling JSON in Golang"
date: 2018-05-30T16:35:28-07:00
categories: ["Golang"]
tags: ["Parsing"]
authors: ["Anna"]
draft: true
---

Unmarshaling JSON files in Golang can be tricky. To be fair the [internal documentation](https://golang.org/pkg/encoding/json/#Unmarshal) is pretty explicit about the possibility to unmarshal JSON into an interface, but doesn't offer a lot of examples.

Of course, if you already know the JSON structure *and* it is not too big, it's just a matter of verbosity. For example, unmarshaling a small JSON like this, requires a few lines of code.

```JSON
    [{"name": "Alice", "role": "admin"},
    {"name": "Bob",    "role": "user"}]
```

```Golang
	type Users struct {
		Name string `json:"Name"`
		Role string `json:"Role"`
	}
	var users []Users
	err := json.Unmarshal(jsonBlob, &users)
	if err != nil {
		fmt.Println("error:", err)
	}
	for _, v := range users {
		fmt.Printf("%s is an %s\n", v.Name, v.Role)
	}
```
_[See this code in the playground.](https://play.golang.org/p/8lPmI6lmtQo)_

If you don't want to specify the entire JSON structure in your code, it is possible to unmarshal it into a map from string to an empty interface and then cast its value later on. This can be useful if you can't rely on the JSON structure or if it is too long to create your own struct.

```Golang
	var users []map[string]interface{}

	err := json.Unmarshal(jsonBlob, &users)
	if err != nil {
		fmt.Println("error:", err)
	}
	for _, v := range users {
		username, ok := v["name"].(string)
		if !ok {
			log.Fatal("Value couldn't be cast as a string")
		}

		userrole, ok := v["role"].(string)
		if !ok {
			log.Fatal("Value conldn't be cast as a string")
		}
		fmt.Printf("%s is an %s\n", username, userrole)
	}

```
_[See this code in the playground.](https://play.golang.org/p/A9jOJq0ddwP)_

So far so good. But what if things starts getting more complicated?
In order to unmarshal a list of JSON that are inside a JSON field, it is necessary to set up a struct that maps the value of the field to a map of a string to an empty interface.

```JSON
    {"ok": true,
	"users":[
		{"name": "Alice", "role": "admin"},
		{"name": "Bob", "role": "user"}
    ]}
```

```Golang
	type jsonUsers struct {
		Ok    bool `json:"ok"`
		Users []map[string]interface{} `json:"users"`
	}

	var objmap jsonUsers

	err := json.Unmarshal(jsonBlob, &objmap)
	if err != nil {
		fmt.Println("error:", err)
	}
	for elem := range objmap.Users {
		username, ok := objmap.Users[elem]["name"].(string)
		if !ok {
			log.Fatal("Value couldn't be cast as a string")
		}

		userrole, ok := objmap.Users[elem]["role"].(string)
		if !ok {
			log.Fatal("Value conldn't be cast as a string")
		}
		fmt.Printf("%s is an %s\n", username, userrole)
	}
```
_[See this code in the playground.](https://play.golang.org/p/yUPl0EAfT-y)_

It is also possible to nest struct for nested JSON.

```JSON
    {"ok": true,
    "users":[
		{"name": ["Alice","Charlie"], "role": "admin"},
    	{"name": ["Bob","Denise"],    "role": "user"}
    ]}
```

```Golang
	type dataUsers struct {
		Name []string `json:"name"`
		Role string   `json:"role"`
	}

	type jsonUsers struct {
		Ok    bool        `json:"ok"`
		Users []dataUsers `json:"users"`
	}

	var objmap jsonUsers

	err := json.Unmarshal(jsonBlob, &objmap)
	if err != nil {
		fmt.Println("error:", err)
	}

	for index, _ := range objmap.Users {
		for _, username := range objmap.Users[index].Name {
			userrole := objmap.Users[index].Role
			fmt.Printf("%s is an %s\n", username, userrole)
		}
	}
```
_[See this code in the playground.](https://play.golang.org/p/SdGfyaNId6V)_

5. [{...},{...}],[{...},{...}]
