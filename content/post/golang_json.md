---
title: "Unmarshaling JSON in Golang"
date: 2018-05-30T16:35:28-07:00
categories: ["Golang"]
tags: ["Parsing"]
authors: ["Anna"]
draft: true
---

Unmarshaling JSON files in Golang can be tricky. Golang is a static language and does not allow dynamic JSON deserialization. In order to unmarshal a JSON blob into Golang it usually requires to know its structure beforehand. But, as the [internal documentation](https://golang.org/pkg/encoding/json/#Unmarshal) states, it is possible to unmarshal JSON into an empty interface.

An empty interface represents the empty set of methods and is satisfied by any value at all, since any value has zero or more methods. At run time the value stored in the empty interface variable may change type, but will always satisfy the interface.

The following post will provide a series of examples on how to unmarshal a JSON blob without specifying its entire structure in Golang.

Let's start with an easy example: unmarshaling a list of JSON with a predefined struct.

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

If you don't want to specify the entire JSON structure in your code, it is possible to unmarshal it into a map from string to an empty interface and then cast its value later on. This can be useful if you can't rely on the JSON structure or if it is too long to create your own.

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

So far so good. But what if things start getting more complicated?
In order to unmarshal a list of JSON that are inside a JSON field, you can always create the struct manually, or you can unmarshal the list into an empty map. More specifically, it is necessary to have a structure that maps the value of the field to a map of a string to an empty interface.

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

Another interesting feature is the possibility to nest data structures and use those to unmarshal the JSON blob. For example if you have a list of JSON inside another JSON, you may want to create a slice of the structure that unmarshals the inner JSON.

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

The example above works only if all the elements in the name list are strings. If the list has mixed types, it is possible to unmarshal it in an empty interface and then extract its values through reflection. If you are interested in knowing more about reflection, _[The Laws of Reflection](https://blog.golang.org/laws-of-reflection)_ explains how they work in Go.

```JSON
    {"ok": true,
    "users":[
		{"name": ["Alice", "Bob", 2], "role": "admin"},
    	{"name": ["Charlie", "Denise", 3],    "role": "user"}
    ]}
```

```Golang
	type jsonUsers struct {
		Ok    bool
		Users []map[string]interface{} `json:"users"`
	}

	var objmap jsonUsers

	err := json.Unmarshal(jsonBlob, &objmap)
	if err != nil {
		fmt.Println("error:", err)
	}
	for elem := range objmap.Users {
		usernames := objmap.Users[elem]["name"]
		name := reflect.ValueOf(usernames)

		userrole, ok := objmap.Users[elem]["role"].(string)
		if !ok {
			log.Fatal("Value conldn't be cast as a string")
		}

		for i := 0; i < name.Len(); i++ {
			fmt.Printf("%v is an %s\n", name.Index(i), userrole)
		}
	}
```

_[See this code in the playground.](https://play.golang.org/p/UT8HyEaa01q)_


