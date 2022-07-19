## Endpoint definition

```go
package main

import "github.com/cifrazia/cats-go/cats"

func Users(api *cats.Api) {
	api.On("find-profiles", func(c *cats.Ctx) error {
		return c.SendString("Hello world!")
	})
}

func main() {
	app := cats.New("minecraft")
	app.Api("users", Users)
}
```