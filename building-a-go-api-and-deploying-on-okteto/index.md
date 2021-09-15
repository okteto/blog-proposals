Go or Golang is an open-source, robust-system-level programming language used to build large-scale network servers and distributed systems. Go was developed by Google in 2007 as a statically-typed compiled language. 

Considering Go is a member of the C-family, it has similar syntax with languages like Java and C++, but it has a more succinct syntax that makes it simpler to learn and understand. Making it easy to develop software that is fast, dependable, and efficient. Go also supports concurrent programming, i.e. it allows running multiple processes simultaneously.

This tutorial focuses on building a simple RESTAPI application with Golang and Mux (A powerful URL router and dispatcher) on Okteto using the Okteto development container. 

Okteto is a cloud-based platform that gives developers rapid access to secure Kubernetes namespaces and allows them to code, build, and run Kubernetes applications. Okteto also speeds up the Kubernetes application development process.

Your Kubernetes deployment is replaced by a development container that contains your development tools when you start Okteto.  To read more, see the full [documentation](https://okteto.com/docs/getting-started).

### Prerequisite

- A basic understanding of Go syntax
- Good understanding of PostgreSQL
- Basic understanding of Docker

### Setup project

Before we begin setting up our project and installing these packages, ensure you have Go installed if you don't simply click [here](https://golang.org/doc/install). Once that is done, create a folder in your **$GOPATH** by adding the following: 

```bash
mkdir movie_store
cd movie_store
```

```bash
touch main.go
touch .env
```

### Installations

Hey, let's set up Postgresql. Firstly, download Postgres through the PgAdmin app. Click [here](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads) to download. Then, using the SQL shell terminal, we'll create our database.

```bash
CREATE DATABASE movie_store;
```

Next, we'll be installing the packages required for our project in our terminal. To install these packages run the following lines of code: 

```go
go get github.com/gorilla/mux 
go get github.com/jinzhu/gorm
go get github.com/joho/godotenv
go get github.com/jinzhu/gorm/dialects/postgres
```

Next, we will be building our Docker image.  Create a `Dockerfile` that provides instructions on building images for deployment on Okteto. In the  `Dockerfile`  add the following:

```go
FROM golang:1.16-alpine

WORKDIR /app

COPY go.mod ./
COPY go.sum ./

RUN go mod download

COPY *.go ./

RUN go build -o /go-moviestore

EXPOSE 8080
```

 

Create a `K8s.yml` file that will contain the Kubernetes manifests.

Add the following to your `k8s.yml` manifest:

```yaml
apiVersion: v1
kind: Secret
metadata:
 name: db-creds
type: Opaque
data:
 connection: 
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: moviestore
spec:
  replicas: 1
  selector:
    matchLabels:
      app: moviestore
  template:
    metadata:
      labels:
        app: moviestore
    spec:
      containers:
      - image: registry.cloud.okteto.net/anitaachu/go-moviestore:golang
        name: moviestore
        env:
        - name: POSTGRES_CONNECTION
          valueFrom:
            secretKeyRef:
              name: db-creds
              key: connection

---

apiVersion: v1
kind: Service
metadata:
  name: moviestore
  annotations:
    dev.okteto.com/auto-ingress: "true"
spec:
  type: ClusterIP  
  ports:
  - name: "moviestore"
    port: 8080
  selector:
    app: moviestore
```

*Now let's get to work!!!* 

### Building application on Okteto

We will be building our application in the Okteto development environment. To do this, first, you have to install [Okteto CLI](https://okteto.com/docs/getting-started/installation/). 

Once Okteto has been successfully installed. Log in to your Okteto Cloud from your CLI:

```go
okteto login
```

If this is successful, you'll be asked to open your browser.

Next, download your kubernetes credentials. To this by running this command 

```bash
okteto namespace
```

### Setting up Okteto Development container

Firstly, in our terminal let's create an Okteto manifest. To this by running the line of code below:

```bash
okteto init
```

![Okteto init command](/blog-proposals/building-a-go-api-and-deploying-on-okteto/init.png)

`okteto init` command checks your Kubernetes namespace for available deployments. From the options, select "*moviestore*".

![Okteto yaml](/blog-proposals/building-a-go-api-and-deploying-on-okteto/oktetoup.png)

`okteto init` command also creates an `okteto.yml` file:

```yaml
name: moviestore
image: anitaachu/go-moviestore:golang
command: bash
securityContext:
  capabilities:
    add:
    - SYS_PTRACE
volumes:
  - /go/pkg/
  - /root/.cache/go-build/
sync:
  - .:/usr/src/app
forward:
  - 8080:8080
  - 2345:2345
```

Next, let's activate our development container. We will do this by running this line of code:

```bash
okteto up
```

![Okteto development](/blog-proposals/building-a-go-api-and-deploying-on-okteto/terminal.png)

In your `main.go` file which we created earlier add the following: 

```go

package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"github.com/gorilla/mux"
	"github.com/jinzhu/gorm"
	"github.com/joho/godotenv"
	_ "github.com/jinzhu/gorm/dialects/postgres"
)

type User struct {
	gorm.Model

	Name  string
	Email string

	Movies []Movie
}

type Movie struct {
	gorm.Model

	Title    string
	Rating   int
	Year     int
	PersonID int
}
```

We successfully defined our models.

**Connecting Database**

The first step is connecting data to our database. This will have the information our app needs to access the database. However, saving these data in our code is not secure. Therefore, we will store our database credentials in an `env` file which we created earlier. 

```go
DIALECT ="postgres"
USER="postgres"
HOST="localhost"
DBPORT="5432"
DBNAME="ecommerce" 
PASSWORD="*****"
```

 Back to our `main.go`  load the environment variables by adding these lines of code: 

```go
var db *gorm.DB
var err error

func main() {
	err = godotenv.Load(".env")
	if err != nil {
		log.Fatalf("error loading %s", err)
	} else {
		log.Println("env loaded!")
	}
	//Loading environment variables

	dialect := os.Getenv("DIALECT")
	user := os.Getenv("USER")
	password := os.Getenv("PASSWORD")
	host := os.Getenv("HOST")
	dbName := os.Getenv("NAME")
	dbPort := os.Getenv("DBPORT")

	// Database connection
	dbURI := fmt.Sprintf("host=%s user=%s dbName=%s sslmode=disable password=%s dbPort=%s", host, user, dbName, password, dbPort)

	db, err = gorm.Open("postgres", dbURI)
	if err != nil {
		log.Fatal(err)
	} else {
		fmt.Println("Successfully connected to database!")

	}

	defer db.Close()

	db.AutoMigrate(&User{})
	db.AutoMigrate(&Movie{})
```

 We created a function to handle our environment variables and connected the data to our database. Lastly, we migrated our models to the database. 

Now, let us test this out in our terminal. Save code and run `go run main.go`

 *Successfully connected to database!*

Hope it worked for you? 

**Creating API routes**

We proceed to the API part, this part would handle the routes, handlers, and controllers. 

Ensure you have gorilla mux imported in your code in this manner:

```go
package main

import (
"github.com/gorilla/mux"
)
```

Gorilla mux would handle our router and incoming requests. Back in our `main.go` ******file, add these lines of code for the routes 

Next, we proceed to creating each route for each function and building the router function.

```go
	// API routes

	router := mux.NewRouter()

	router.HandleFunc("/users", getUsers).Methods("GET")
	router.HandleFunc("/user/{id}", getUser).Methods("GET")
	router.HandleFunc("/movie/{id}", getMovie).Methods("GET")
	router.HandleFunc("/movies", getMovies).Methods("GET")
	router.HandleFunc("/create/user", createUser).Methods("POST")
	router.HandleFunc("/create/movie", createMovie).Methods("POST")
	router.HandleFunc("/delete/user/{id}", deleteUser).Methods("DELETE")
	router.HandleFunc("/delete/movie/{id}", deleteMovie).Methods("DELETE")

	log.Fatal(http.ListenAndServe(":8080", router))

}
```

Successfully created our routes, we will proceed to create the handlers for each of these routes.

This will handle eight key operations on our database. These operations include:

1. Get all users
2. Get a user by ID
3. Create a user
4. Delete a user
5. Get all movies 
6. Get a movie by ID 
7. Create a movie
8. Delete a movie

**Getting all users and movies**

```go
// Model controllers
func getUsers(w http.ResponseWriter, r *http.Request) {
	var users []User
	db.Find(&users)

	json.NewEncoder(w).Encode(&users)
}

func getMovies(w http.ResponseWriter, r *http.Request) {
	var movies []Movie

	db.Find(&movies)

	json.NewEncoder(w).Encode(&movies)
}

```

In the `getUsers` and `getMovies` function, we pass our request and response to get a variable, an array of users and movies. 

`db.Find` finds an array of users and movies in the database and sends it back in *JSON.* Get `"Encoding/json"` imported into your code. 

**Get a specific user and movie**

```go
func getUser(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)

	var users User
	var movies []Movie

	db.First(&user, params["id"])
	db.Model(&user).Related(&movies)

	user.Movies = movies

	json.NewEncoder(w).Encode(&user)
}

func getMovie(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)

	var movie Movie

	db.First(&movie, params["id"])

	json.NewEncoder(w).Encode(&movie)
}
```

This takes in request and response and stores the data in `params` array, which is used to get the params in our route.  Then, we declared a variable of type `user` and `movie` struct. 

`db.First` finds the first user and movie with the query you give it. You can get the `id` from the URL by passing `params["id"]` Lastly, we send back the data in JSON*.* 

In the `getUser`  function, we created a `db.Model` to get all  the movies related to a user. 

**Creating a user and movie**

```go
func createUser(w http.ResponseWriter, r *http.Request) {
	var user User
	json.NewDecoder(r.Body).Decode(&user)

	createdUser := db.Create(&user)
	err = createdUser.Error
	if err != nil {
		json.NewEncoder(w).Encode(err)
	} else {
		json.NewEncoder(w).Encode(&user)
	}
}

func createMovie(w http.ResponseWriter, r *http.Request) {
	var movie Movie
	json.NewDecoder(r.Body).Decode(&movie)

	createdMovie := db.Create(&movie)
	err = createdMovie.Error
	if err != nil {
		json.NewEncoder(w).Encode(err)
	} else {
		json.NewEncoder(w).Encode(&movie)
	}
}
```

In creating a user and movie to the database, we would get `JSON` ** input from and outside users using `json.NewDecoder(r.Body).Decode(&user)` to turn to a user and movie struct then we check for errors and pass our response. 

**Deleting a user and movie**

```go
func deleteUser(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)

	var user User

	db.First(&user, params["id"])
	db.Delete(&user)

	json.NewEncoder(w).Encode(&user)
}

func deleteMovie(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)

	var movie Movie

	db.First(&movie, params["id"])
	db.Delete(&movie)

	json.NewEncoder(w).Encode(&movie)
}
```

Lastly, the delete functionality. We begin by getting an `id` of the user or movie in the URL using `params`

`db.First` gets specific user or movie then deletes the user or movie from the database.

  

***Hey! we are done building.***


Copy the endpoint on your Okteto dashboard.
![Okteto dashboard](/blog-proposals/building-a-go-api-and-deploying-on-okteto/namespace.png)


 ***Our API is live!***

**Testing out on postman**

Our server is up. we will proceed to Postman to test our endpoint which was provided by Okteto.

![Create movie](/blog-proposals/building-a-go-api-and-deploying-on-okteto/createmovie.png)

![Get movies](/blog-proposals/building-a-go-api-and-deploying-on-okteto/movies.png)

### Conclusion

Developers can code, build, and run Kubernetes applications entirely in the cloud thanks to Okteto Cloud's rapid access to secure Kubernetes namespaces.

In this tutorial, we built a CRUD API in Golang directly on Okteto using the Okteto development environment. We also deployed the application to Okteto. You can also deploy your application to a production-like development environment on an external Kubernetes cluster.

I hope you have good time building and deploying on Okteto with ease. 

Happy coding!
