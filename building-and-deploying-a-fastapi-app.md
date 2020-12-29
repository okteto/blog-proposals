# Building a FastAPI Application and Deploying to Okteto


In this tutorial, you'll learn how to develop a CRUD API with [FastAPI](https://fastapi.tiangolo.com) and deploy the application to [Okteto](https://okteto.com).

You'll start by building the application, after which you will now deploy it to Okteto.

## What is FastAPI

FastAPI is a modern Python web framework designed for building fast and efficient backend applications. It comes with built-in support for data validation, authentication, and interactive API documentation powered by OpenAPI and Swagger.

## What is Okteto?

Okteto is a tool used to develop and accelerate the development workflow of Kubernetes applications.

## Initial Setup

Start by creating a new folder to hold your project called 'fastapi-crud':

```sh
$ mkdir fastapi-crud
$ cd fastapi-crud
```

Next, create and activate a virtual environment in the project folder:

```
$ python3.9 -m venv venv
$ source venv/bin/activate
$ export PYTHONPATH=$PWD
```

Virtual environment provides an isolated environment to run our Python applications. FastAPI applications require an isolated environment to manage it's dependencies such as Uvicorn which is an Asynchronous GateWay Server Interface server.

Next, create the following files and folder:

```
├── app
│       ├── __init__.py
│       ├── model.py
│       ├── api.py
├──main.py
└── requirements.txt
```

In your *requirements.txt* file, add the following dependencies:

```
fastapi==0.62.0
uvicorn==0.13.1
```

The first dependency, `fastapi,` is the framework on which your application will be built. The second dependency, [`uivorn`](http://uvicorn.org), is an Asynchronous Gateway Server Interface (ASGI) that enables us to run our FastAPI application.

Install the dependencies:

```sh
(venv)$ pip install -r requirements.txt
```

With the installation complete, define a base route in `app/api.py`:

```py
from fastapi import FastAPI

app = FastAPI()

@app.get("/", tags=["Home"])
def get_root() -> dict:
    returrn {
        "message": "Welcome to the okteto's app."
    }
```

You started by importing the FastAPI class from the `fastapi` package in the code block above. Then you created an instance of the class in the variable `app`. 

Next, you defined a `GET` route on __"/"__ which is handled by the `get_root()` function.


In the `main.py` file, define an entry point for running the application:

```py
import uvicorn

if __name__ == "__main__":
    uvicorn.run("app.api:app", host="0.0.0.0", port=8080, reload=True)
```

In the code block above, you imported the uvicorn package itself. Under the initializer block, you invoked the `run` method, which takes the location of FastAPI's instance, the host, port, and the reload boolean value.

In this application, the location of the FastAPI instance, `app = FastAPI()` is in the file `app/api.py`. The host value can be set to a valud IP address within your local macbine's configured IP scope, it can also be left blank. Likewise the port and reload value. The port `8080` is chosen since Okteto applications run on port 8080 by default, the reload value is set to True to avoid restarting the application on every change made.

Next, start the application:

```
python main.py
```


You should get a response like this:

```
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
INFO:     Started reloader process [24513] using statreload
INFO:     Started server process [24515]
INFO:     Waiting for application startup.
INFO:     Application startup complete.

```

Navigate to http://localhost:8080 in your browser. You should see:

```
{
    "message": "Welcome to the okteto's app."
}
```

## Routes

You'll be building a recipe application where you can store, remove, update and delete recipes by sending HTTP requests.

Before you begin writing the routes, define the model schema for the application:

In `app/model.py`, write the following:

```
from pydantic import BaseModel, Field
from typing import Optional, List


class RecipeSchema(BaseModel):
    id: Optional[int]
    name: str = Field(...)
    ingredients: List[str] = Field(...)

    class Config:
        schema_extra = {
            "example": {
                "name": "Donuts",
                "ingredients": ["Flour", "Milk", "Sugar", "Vegetable Oil"]
            }
        }


class UpdateRecipeSchema(BaseModel):
    name: Optional[str]
    ingredients: Optional[List[str]]

    class Config:
        schema_extra = {
            "example": {
                "name": "Buns",
                "ingredients": ["Flour", "Milk", "Sugar", "Vegetable Oil"]
            }
        }


```

In the block of code above, you defined how the recipe would be represented, sent, and retrieved in your in-app database. Each recipe will have an ID automatically generated once a POST request is sent. Each recipe will also comprise of a name and an array of ingredients. If any other type of data not listed in the schema is sent in the request body, an error will be returned.

The `RecipeSchema` has a subclass `Config`. The subclass contains an object variable, `schema_extra`, which includes a key `example` used as the mock data in the interactive documentation.

The `UpdateRecipeSchema`, on the other hand, is the model for guiding request body sent through the UPDATE route. The difference between this model and the `RecipeSchema` is that the schema values can be passed optionally.

With the model in place, let's write the code for the application's routes. In `app/api.py`, add the recipes database before the base route __"/"__:

```py
recipes = [
    {
        "id": 1,
        "name": "Donuts",
        "ingredients": ["Flour", "Milk", "Sugar", "Vegetable Oil"]
    }
]
```

Below the base route, add the GET routes:

```
@app.get("/recipe", tags=["Recipe"])
def get_recipes() -> dict:
    return {
        "data": recipes
    }

@app.get("/recipe/{id}", tags=["Recipe"])
def get_recipe(id: int) -> dict:
    if id > len(recipes) or id < 1:
        return {
            "error": "Invalid ID passed."
        }

    for recipe in recipes:
        if recipe['id'] == id:
            return {
                "data": [
                    recipe
                ]
            }

    return {
        "error": "No such recipe with ID {} exist".format(id)
    }
```

You defined GET routes for retrieving all the recipes and a single recipe using its ID in the code block above. An error message is returned if an invalid ID is passed as a parameter in the __/recipe/{id}__ route. Test the routes by visiting http://localhost:8080/recipe and http://localhost:8080/recipe/1:

![Recipes](https://res.cloudinary.com/adeshina/image/upload/v1608635876/zree0awqm8co1g2kf339.png)
    

![First Recipe](https://res.cloudinary.com/adeshina/image/upload/v1608635909/gdted1ywrnnkjupy9x2o.png)

Define the POST route for adding new recipes:

Start by updating the imports:

```py
from fastapi import FastAPI, Body

from app.model import RecipeSchema, UpdateRecipeSchema
from fastapi.encoders import jsonable_encoder
```

Next, add the POST route beneath the GET routes:

```py
@app.post("/recipe", tags=["Recipe"])
def add_recipe(recipe: RecipeSchema = Body(...)) -> dict:
    recipe.id = len(recipes) + 1
    recipes.append(recipe.dict())
    return {
        "message": "Recipe added successfully."
    }
```

In the code block above, we made sure the request body is modeled in our `RecipeSchema` by setting the type `recipe` to `RecipeSchema` in the `add_recipe` function. The `Body(...)` imported from FastAPI ensures that the request body is passed.
In an event where the content of the request doesn't match the specified schema type, an error message automatically generated from Pydantic will be generated and then returned.

In the `add_recipe` function, the recipe ID is computed by incrementing the length of the recipes database by 1. Test the POST route using curl:

```sh
curl -X POST http://localhost:8080/recipe -d \
'{"name": "Donut", "ingredients": ["Flour", "Milk", "Butter"]}' \
-H 'Content-Type: application/json'
```

You will get a response:

```json
{
    "message": "Recipe added successfully."
}
```

To verify that the recipe has been added, retrieve all the recipes:

```
curl -X GET http://localhost:8080/recipe/2 -H 'Content-Type: application/json'
```

You will get this response:

```json
{
    "data": [
                {"id":1,"name":"Donuts","ingredients":["Flour","Milk","Sugar","Vegetable Oil"]},
                {"id":2,"name":"Donut","ingredients":["Flour","Milk","Butter"]},
            ]
}
```

You can also retrieve the recipe from its ID:

```
curl -X GET http://localhost:8080/recipe/2 -H 'Content-Type: application/json'
```

You will get the response:

```json
{
    "data": [
                {"id":2,"name":"Donut","ingredients":["Flour","Milk","Butter"]}
            ]
}
```


With the POST route in place, define the UPDATE route:



```py
@app.put("/recipe", tags=["Recipe"])
def update_recipe(id: int, recipe_data: UpdateRecipeSchema) -> dict:
    stored_recipe = {}
    for recipe in recipes:
        if recipe["id"] == id:
            stored_recipe = recipe
        else:
            return {
                "error": "No such recipe exist"
            }

    stored_recipe_model = RecipeSchema(**stored_recipe)
    update_recipe = recipe_data.dict(exclude_unset=True)
    updated_recipe = stored_recipe_model.copy(update=update_recipe)
    recipes[recipes.index(stored_recipe_model)] = jsonable_encoder(updated_recipe)

    return {
        "message": "Recipe updated successfully."
    }

```

In the code block above, you find a recipe by the ID supplied and perform a partial or full update depending on the content of the request body passed. If no such ID exists, it returns an error message. 

Test the UPDATE route by changing thw title of the first recipe to __Buns__ using curl:

```
curl -X PUT "http://0.0.0.0:8080/recipe?id=1" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{  \"name\": \"Buns\",}"
```

The UPDATE request returns this response:

```json
{
  "message": "Recipe updated successfully."
}
```

Let's define the final route, DELETE. The DELETE route is used to remove recipe in your application using their ID. Beneath the UPDATE route, add the following:

```py
@app.delete("/recipe/{id}", tags=["Recipe"])
def delete_recipe(id: int) -> dict:
    if id > len(recipes) or id < 1:
        return {
            "error": "Invalid ID passed"
        }

    for recipe in recipes:
        if recipe['id'] == id:
            recipes.remove(recipe)
            return {
                "message": "Recipe deleted successfully."
            }

    return {
        "error": "No such recipe with ID {} exist".format(id)
    }

```

In the code block above, you first check if the recipe whose ID is passed exists. If it does, the recipe is removed. Otherwise, an error message is returned. Test the DELETE route:

```sh
curl -X DELETE "http://0.0.0.0:8080/recipe/1" -H  "accept: application/json"
```

The request above return this response:

```json
{
  "message": "Recipe deleted successfully."
}
```

You have successfully built the CRUD application. 

## Deploying to Okteto

In this section, you will be deploying the application to Okteto.

### Why Deployment?

Deploying your application enable you and other people access it through it's unique URL address. Your application is inaccessible by other people since it runs only on your machine's localhost. Also, deploying it on a cloud server like Okteto enables your application to run on the HTTPs service.

Create an account on [Okteto](https://cloud.okteto.com) if you do not have one setup. Additionally, [Install the Okteto CLI](https://okteto.com/docs/getting-started#step-2-install-the-okteto-cli) on your machine.


Before proceeding, create a _.gitignore_ file in the project folder to prevent checking in the "venv" folder to git:


```
$ touch .gitignore
```

Add the following:


```
venv
__pycache__
```

With the necessary installations in place, log on to Okteto from your terminal using the command:

```sh
$ okteto login
```

The next step is to create a stack manifest file for Okteto. The manifest file removes the need to deal with the complexities of Kubernetes manifests.

```
$ touch okteto-stack.yaml
```

In the `okteto-stack.yaml` file, define the application using a format that's very similar to docker-compose:

```yaml
name: fastapi-crud
services:
  fastapi:
    public: true
    image: fastapi-crud:latest
    build: .
    replicas: 1
    ports:
      - 8080
    resources:
      cpu: 100m
      memory: 128Mi
```

In the manifest file above, you defined your application's name and the services your app is made up of, fastapi. The fastapi service exposes your application via the port `8080` to a public HTTPs endpoint with a valid certificate. The service also allocates itself with 100 mili CPUs and 128Mi of memory. 

> Okteto has a reference page on [https://okteto.com/docs/reference/stacks](Okteto Stacks Manifest).

With the manifest in place, create a Dockerfile to house the build instruction for the application. Dockerfile contains build instruction for images to be deployed on Okteto.

```sh
$ touch Dockerfile
```

Add the following:

```Dockerfile
FROM python:3.8

ADD requirements.txt /requirements.txt

ADD main.py /main.py

ADD okteto-stack.yaml /okteto-stack.yaml

ADD okteto.yml /okteto.yml

RUN pip install -r requirements.txt

EXPOSE 8080

COPY ./app app

CMD ["python3", "main.py"]
```

Before proceeding to deployment on Okteto, download and set up your Kubernetes credentials on your local machine:

![Okteto Dashboard](https://res.cloudinary.com/adeshina/image/upload/v1608635743/x9tmv4styid2hffxilxx.png)

Let's deploy your application to Okteto. Start by running the command: 

```
okteto namespace
```

The command above setups a namespace on the Okteto cloud. Next, run the command to deploy the application:

```
okteto stack deploy --build
```

The command above builds the service you indicated in the `okteto-stack.yml` file and then deploys the application. This command eases the stress of having to build, then manually configuring the application after deployment.

The command above starts the application. Navigate to your dashboard and click the link under the "Endpoints" heading:

![Dashboard](https://res.cloudinary.com/adeshina/image/upload/v1608638743/q8xcjhl2gdnh0tdzr573.png).

Go on and test the endpoints.

You can exit the development console by running the Ctrl + D command twice.

## Conclusion

In this article, you built a CRUD recipe application and deployed it to Okteto. The code used in this article can be found in [GitHub](https://github.com/okteto/fastapi-crud).
