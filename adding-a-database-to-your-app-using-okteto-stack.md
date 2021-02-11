# Adding A Database to Your Application Using Okteto Stacks

Temporary in-app databases are not ideal for web applications. Any unfortunate event such as an abrupt shutdown or restarting of the application will lead to the total loss of stored data.

In previous posts, you have learned how to [deploy applications directly from your console using Okteto Stacks](https://okteto.com/blog/building-and-deploying-a-fastapi-app-in-okteto-cloud/) and [how to deploy existing applications from Okteto's UI](https://okteto.com/blog/deploying-an-existing-application-with-okteto-cloud/) thereby seeing the immense adaptability of Okteto Stacks.

In this tutorial, you will be adding a database to store better your application data after which you will deploy it to your Okteto namespace from your command line.

## Initial Setup

Start by creating a fork of the application from [GitHub](https://github.com/okteto/fastapi-crud) and then clone it locally. In your local clone, set it up using the following commands:

```bash
$ cd fastapi-crud
$ python3 -m venv venv && source venv/bin/activate
(venv)$ pip install -r requirements.txt
```

Verify that the setup is complete by running:

```bash
python3 main.py
```

After verifying the installation, update the `DOCKERHUBUSERNAME` variable in your stack file to your namespace name. Deploy the application with the command:


```bash
(venv)$ okteto stack deploy --build
```

## Adding A Database
Before proceeding, verify that you have MongoDB installed or proceed to the [MongoDB's Installation page](https://docs.mongodb.com/manual/installation/) to install MongoDB.

To avoid data loss, you're going to rewrite the application logic to use a MongoDB database, which will be mounted as a container in your Okteto namespace.

Start by installing `pymongo`, a MongoDB driver for Python application, `python-decouple` for reading environment secrets and update the `requirements.txt` file:

```bash
(venv)$ pip install pymongo
```

Update your `requirements.txt` file:

```txt
...
pymongo
python-decouple
```

In the **api** folder, create a new file, `database.py`, where you'll write the database CRUD operations' functions.

Start by importing `MongoClient`, `ObjectID` and `config`:

```py
from pymongo import MongoClient
from bson import ObjectId
from decouple import config
```

MongoClient is responsible for the connection from our application to the database, ObjectId, on the other hand, is used to pass `id` values in MongoDB properly, and `config` is responsible for reading application secrets from `.env` files.

Next, define the connection, database and database collection details:

```py
connection_details = config("DB_HOST")

client = MongoClient(connection_details)

database = client.recipes

recipe_collection = database.get_collection('recipes_collection')
```

On the first line above, you are using the `decouple` library to read the environment variable `DB_HOST`. Create a `.env` file in the root folder containing the connection detail:

```dotenv
DB_HOST=mongodb://localhost:27017
```

Documents in MongoDB are stored in JSON format and the `_id` in `ObjectId` format. Write a function to parse the result from a query:

```py
def parse_recipe_data(recipe) -> dict:
    return {
        "id": str(recipe["_id"]),
        "name": recipe["name"],
        "ingredients": recipe["ingredients"]
    }
```

### CRUD functions

The next step is to write the functions responsible for saving, removing, updating and deleting recipes. Start by implementing the `save_recipe`:

```py
def save_recipe(recipe_data: dict) -> dict:
    recipe = recipe_collection.insert_one(recipe_data).inserted_id
    return {
        "id": str(recipe)
    }
```

The function above inserts the recipe data into the database and returns the newly created recipe's ID.

Next, the function for retrieving a single recipe and all the recipes from the database:

```py
def get_single_recipe(id: str) -> dict:
    recipe = recipe_collection.find_one({"_id": ObjectId(id)})
    if recipe:
        return parse_recipe_data(recipe)

def get_all_recipes() -> list:
    recipes = []
    for recipe in recipe_collection.find():
        recipes.append(parse_recipe_data(recipe))

    return recipes
```

The first function above returns a single recipe whose ID matches the supplied one and an error message if it doesn't exist, while the second function returns all the contained recipes in the database.

Next, write the `update_recipe_data` function responsible for updating recipe data:

```py
def update_recipe_data(id: str, data: dict):
    recipe = recipe_collection.find_one({"_id": ObjectId(id)})
    if recipe:
        recipe_collection.update_one({"_id": ObjectId(id)}, {"$set": data})
        return True
```

Lastly, write the function for deleting a recipe:

```py
def remove_recipe(id: str):
    recipe = recipe_collection.find_one({"_id": ObjectId(id)})
    if recipe:
        recipe_collection.delete_one({"_id": ObjectId(id)})
        return True
```

With the database CRUD functions in place, replace the content of `app/api.py` with:

```py
from fastapi import FastAPI, Body
from fastapi.encoders import jsonable_encoder

from app.model import RecipeSchema, UpdateRecipeSchema
from app.database import save_recipe, get_all_recipes, get_single_recipe, update_recipe_data, remove_recipe

app = FastAPI()

@app.get("/", tags=["Root"])
def get_root() -> dict:
    return {
        "message": "Welcome to the okteto's app.",
    }

@app.get("/recipe", tags=["Recipe"])
def get_recipes() -> dict:
    recipes = get_all_recipes()
    return {
        "data": recipes
    }

@app.get("/recipe/{id}", tags=["Recipe"])
def get_recipe(id: str) -> dict:
    recipe = get_single_recipe(id)
    if recipe:
        return {
            "data": recipe
        }
    return {
        "error": "No such recipe with ID {} exist".format(id)
    }

@app.post("/recipe", tags=["Recipe"])
def add_recipe(recipe: RecipeSchema = Body(...)) -> dict:
    new_recipe = save_recipe(recipe.dict())
    return new_recipe

@app.put("/recipe", tags=["Recipe"])
def update_recipe(id: str, recipe_data: UpdateRecipeSchema)  -> dict:
    if not get_single_recipe(id):
        return {
            "error": "No such recipe exist"
        }

    update_recipe_data(id, recipe_data.dict())

    return {
        "message": "Recipe updated successfully."
    }

@app.delete("/recipe/{id}", tags=["Recipe"])
def delete_recipe(id: str) -> dict:
    if not get_single_recipe(id):
        return {
            "error": "Invalid ID passed"
        }


    remove_recipe(id)
    return {
        "message": "Recipe deleted successfully."
    }
```

Update the `api/model.py` by removing the `id` field in the `RecipeSchema` model class.

## Testing The Database

With the database connection in place, start a `mongod` server to allow interactions with the database:

```
mongod --port 27017
```

Next, test the POST route:

```bash
(venv)$ curl -X POST http://localhost:8080/recipe -d \
'{"name": "Donut", "ingredients": ["Flour", "Milk", "Butter"]}' \
-H 'Content-Type: application/json'
```

Response:

```json
{
  "id": "601fdcd82fbbf462d33a6e34"
}
```

Test the GET routes:

1. Return all recipes

```bash
(venv)$  curl -X GET http://localhost:8080/recipe/2 -H 'Content-Type: application/json'
```

Response:

```json
{
  "data": [
    {
      "id": "601fdcd82fbbf462d33a6e34",
      "name": "Donut",
      "ingredients": [
        "Flour",
        "Milk",
        "Butter"
      ]
    }
  ]
}
```

2. Return a single recipe

```bash
(venv)$ curl -X GET http://localhost:8080/recipe/601fdcd82fbbf462d33a6e34 -H 'Content-Type: application/json'
```

Response:

```json
{
  "data": {
    "id": "601fdcd82fbbf462d33a6e34",
    "name": "Donut",
    "ingredients": [
      "Flour",
      "Milk",
      "Butter"
    ]
  }
}
```

Test the UPDATE route:

```bash
(venv)$ curl -X PUT "http://0.0.0.0:8080/recipe?id=601fdcd82fbbf462d33a6e34" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"name\":\"Buns\",\"ingredients\":[\"Flour\",\"Milk\",\"Sugar\",\"Vegetable Oil\"]}"
```

Response:

```json
{
  "message": "Recipe updated successfully."
}
```

Lastly, test the DELETE route:

```bash
(venv)$ curl -X DELETE "http://0.0.0.0:8080/recipe/601fdcd82fbbf462d33a6e34" -H  "accept: application/json"
```

Response:

```json
{
  "message": "Recipe deleted successfully."
}
```

## Redeploying to Okteto

Okteto eases the stress of deployment and subsequent redeployment by allowing us update and upgrade existing applications from the stack file. In the previous post, we created a okteto stack manifest to deploy our fastAPI service.  We are now going to update it also include a mongodb service:

```yaml
  mongodb:
    image: bitnami/mongodb:latest
    ports:
      - 27017
    resources:
      cpu: 100m
      memory: 128Mi
      storage: 1Gi
    volumes:
      - /bitnami/mongodb
```

In the code above, you added another service, `mongodb`, to house a MongoDB container from bitnami; the default port **27017** is exposed under the ports heading.

Under the `fastapi` service, add an environment heading containing the `DB_HOST` the database file reads using the `decouple` library:

```yaml
environment:
  - DB_HOST=mongodb://mongodb:27017
  - secret=dev 
```

With the stack file updated, deploy it using the command:

```bash
(venv)$ okteto stack deploy --build
```

Log on to your [Okteto Dashboard](https://cloud.okteto.com), a MongoDB container alongside your application is mounted:

![Dashboard](https://res.cloudinary.com/adeshina/image/upload/v1612703043/rlkchjwvcomtq3zuhfl9.png)

Test the newly deployed route by replacing `deployedapp` from previous requests with the live application URL. From your terminal, run the command:

```bash
(venv)$ curl -X POST "https://fastapi-youngestdev.cloud.okteto.net/recipe" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"name\":\"Donuts\",\"ingredients\":[\"Flour\",\"Milk\",\"Sugar\",\"Vegetable Oil\"]}"
```

The response sent out is:

```json
{
  "id": "601fe6deaa1a27fbcb9a60fb"
}
```

## Conclusion

In this article, you added a database to an existing application and deployed it to Okteto using Okteto stacks. The deployment was done with minor upgrades to the Okteto stack file, demonstrating the adaptability of the Okteto stacks. You can find the code from this article on [GitHub]
(https://github.com/Okteto/fastapi-crud/tree/mongodb-crud)

Get started with [Okteto](https://okteto.com) for free today to begin deploying your application in one click.
