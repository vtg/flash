flash
====
HTTP routing package that helps to create restfull json api for Go applications.

what it does:

 - dispatching actions to controllers
 - rendering JSON response
 - extracting JSON request data by key
 - handling file uploads
 - sending gzipped JSON responses when applicable
 - sending gzipped versions of static files if any

Routing:
```go
r := flash.NewRouter()

// route to function(*Ctx)
r.Route("/pages/:id", ShowPage)

// auto generates controller routes
r.Resource("/pages", &PagesController{})

// standard http handler
r.HandleFunc("/", IndexHandler)
```

URL Parameters:
```go
// prefixed with ':' are strict params. all parts should be present in request
// strict params can't be used after optional or global params
// Request: '/pages/1/act' Returns: [id:1, action:act]
// Request: '/pages/1' Returns: not found
"/pages/:id/:action"

// prefixed with '&' are optional params. any or non can be present in request
// Request: '/pages/1/act' Returns: [id:1, action:act]
// Request: '/pages/1' Returns: [id:1]
// Request: '/pages' Returns: []
"/pages/&id/&action"

// prefixed with '@' are global params. global param returns the rest of request
// global param can only be used as last param
// Request: '/files/path_to/file.go' Returns: [name:"path_to/file.go"]
"/files/@name"
```


standard REST usage example:

```go
package main

import (
	"net/http"

	"github.com/vtg/flash"
)

var pages map[int64]*Page

func main() {
	pages = make(map[int64]*Page)
	pages[1] = &Page{Id: 1, Name: "Page 1"}
	pages[2] = &Page{Id: 2, Name: "Page 2"}

	r := flash.NewRouter()
	a := r.PathPrefix("/api/v1")

	a.Resource("/pages", &Pages{}, auth)
	r.PathPrefix("/images/").FileServer("./public/")
	r.HandleFunc("/", indexHandler)
	http.ListenAndServe(":8080", r)
}

// simple authentication implementation
func auth(c *flash.Ctx) bool {
	key := c.QueryParam("key")
	if key == "correct-password" {
		return true
	} else {
		c.RenderJSONError(http.StatusUnauthorized, "unauthorized")
	}
	return false
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
}

type Page struct {
	Id      int64  `json:"id"`
	Name    string `json:"name"`
	Content string `json:"content"`
	Visible bool   `json:"visible"`
}

func findPage(id int64) *Page {
	p := pages[id]
	return p
}
func insertPage(p Page) *Page {
	id := int64(len(pages) + 1)
	p.Id = id
	pages[id] = &p
	return pages[id]
}

// Pages used as controller
type Pages struct {
	flash.Ctx
}

// Index processed on GET /pages
func (p *Pages) Index() {
	var res []*Page

	for _, v := range pages {
		res = append(res, v)
	}

	p.RenderJSON(200, flash.JSON{"pages": res})
}

// Show processed on GET /pages/1
func (p *Pages) Show() {
	page := findPage(p.ID64())

	if page == nil {
		p.RenderJSONError(404, "record not found")
		return
	}

	p.RenderJSON(200, flash.JSON{"page": page})
}

// Create processed on POST /pages
// with input data provided {"page":{"name":"New Page","content":"some content"}}
func (p *Pages) Create() {
	m := Page{}
	if m.Name == "" {
		// see Request.LoadJSONRequest for more info
		p.LoadJSONRequest("page", &m)
		p.RenderJSONError(422, "name required")
	} else {
		insertPage(m)
		p.RenderJSON(200, flash.JSON{"page": m})
	}
}

// Update processed on PUT /pages/1
// with input data provided {"page":{"name":"Page 1","content":"updated content"}}
func (p *Pages) Update() {
	page := findPage(p.ID64())

	if page == nil {
		p.RenderJSONError(404, "record not found")
		return
	}

	m := Page{}
	p.LoadJSONRequest("page", &m)
	page.Content = m.Content
	p.RenderJSON(200, flash.JSON{"page": page})
}

// Destroy processed on DELETE /pages/1
func (p *Pages) Destroy() {
	page := findPage(p.ID64())

	if page == nil {
		p.RenderJSONError(404, "record not found")
		return
	}

	delete(pages, page.Id)
	p.RenderJSON(203, flash.JSON{})
}

// POSTActivate custom non crud action activates/deactivated page. processed on POST /pages/1/activate
func (p *Pages) POSTActivate() {
	page := findPage(p.ID64())
	if page == nil {
		p.RenderJSONError(404, "record not found")
		return
	}

	page.Visible = !page.Visible
	p.RenderJSON(200, flash.JSON{"page": page})
}
```

Its possible to serve custom actions.
To add custom action to controller prefix action name with HTTP method:

```go
 // POST /pages/clean or POST /pages/1/clean
 func (p *Pages) POSTClean {
   // do some work here
 }
 // DELETE /pages/clean or DELETE /pages/1/clean
 func (p *Pages) DELETEClean {
   // do some work here
 }
 // GET /pages/stat or GET /pages/1/stat
 func (p *Pages) GETStat {
   // do some work here
 }
 ...
```

#####Author

VTG - http://github.com/vtg

##### License

Released under the [MIT License](http://www.opensource.org/licenses/MIT).

[![GoDoc](https://godoc.org/github.com/vtg/flash?status.png)](http://godoc.org/github.com/vtg/flash)
