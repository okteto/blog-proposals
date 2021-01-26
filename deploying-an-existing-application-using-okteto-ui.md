# Deploying An Existing Application Using Okteto UI

In the previous blog post, you learned how to build a FastAPI CRUD application and deployed it to Okteto using the Okteto CLI tool. In this tutorial, you will be learning how to deploy the application directly from your Okteto dashboard.

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

## Initial Deployment

In the previous post, you defined an Okteto stack file to facilitate the deployment of your application from the CLI tool. In this section, you will be deploying the application from the dashboard.

However, before deployment, you need to reassign the repository where Okteto will store your application images.

Your current `okteto-stack.yaml` file is similar to the code below:


```yaml
name: fastapi-crud
services:
  fastapi:
    public: true
    image: DOCKERHUBUSERNAME/fastapi-crud:latest
    build: .
    replicas: 1
    ports:
      - 8080
    resources:
      cpu: 100m
      memory: 128Mi
```

On line 5, modify the `image` value from:

```yaml
image: DOCKERHUBUSERNAME/fastapi-crud:latest
```

to: 

```yaml
image: okteto.dev/fastapi-crud:latest
```

In the code block above, you renamed the storage location of your application's image from your dockerhub to Okteto's repository.

### What's the Okteto.dev repository ?

The `okteto.dev` repository is the central repository for images of applications deployed from Okteto's UI. The need for the reassignment of the image repository in the stack file is because Okteto lacks the permission to push images into your docker repository from the dashboard as opposed to when deployed from the CLI, where you are required to have an active docker session on your machine.

Next, save the changes and commit the changes to GitHub:

```bash
(venv)$ git add okteto-stack.yaml
(venv)$ git commit -m "Update image repository"
(venv)$ git push
```

With the stack file updated, proceed to your Okteto dashboard to deploy the application:

![Okteto Dashboard](https://res.cloudinary.com/adeshina/image/upload/v1611604440/o5fyl3fhoywmc6auigli.png)

Click on the **Deploy** button. A popup will appear, fill in the **Repository URL** and optionally the branch name which in this case is *main*.

![Deploy Button](https://res.cloudinary.com/adeshina/image/upload/v1611604456/wdi3swnvoa2v2bhaaqpd.png)

Next, click on the deploy button. Okteto automatically retrieves, deploys and starts your application.

![Deployed Application](https://res.cloudinary.com/adeshina/image/upload/v1611604494/rscsvqxobd5dc5bt0baa.png)

## Adding A New Route

Let's update the application by adding a route for adding bulk recipes.

In `app/api.py`, just below the POST route, add another route. Start by importing the `List` wrapper from typing before proceeding to adding the new route:

```py
from typing import List

...

@app.post("/recipes", tags=["Recipe"])
def add_bulk_recipe(bulk_recipe: List[RecipeSchema] = Body(...)) -> dict:
    for recipe in bulk_recipe:
        recipe.id = len(recipes) + 1
        recipes.append(recipe.dict())

    return {            
        "message": "Recipes added successfully"    
    }

```

Test the new route from your terminal:

```bash
(venv)$ curl --request POST \
  --url http://0.0.0.0:8080/recipes \
  --header 'Content-Type: application/json' \
  --data '[
	{
		"name": "Donuts",
		"ingredients": [
			"Flour",
			"Milk",
			"Sugar",
			"Vegetable Oil"
		]
	},
	{
		"name": "Creamy Donuts",
		"ingredients": [
			"Flour",
			"Milk",
			"Sugar",
			"Vegetable Oil",
			"Irish whipped cream"
		]
	}
]'
```

You will get a response:

```json
{
  "message": "Recipes added successfully."
}
```

Commit the new change in the `app/api.py` file into a new branch:

```bash
(venv)$ git checkout -b bulk-recipes && git add app/api.py
(venv)$ git commit -m "Add bulk recipes"
(venv)$ git push -u origin bulk-recipes
```

Verify that the new branch has been created in your GitHub repo.

## Redeploying

Once again, login into your Okteto dashboard and click on the redeploy button:

![Redeploy Button](https://res.cloudinary.com/adeshina/image/upload/v1611604547/vhenrnydsenkb6a0xd3e.jpg)

Change the branch from `main` to `bulk-recipes` and deploy:

![Image 5](https://res.cloudinary.com/adeshina/image/upload/v1611604639/knpe6yiz4alvplgniszb.png)

The branch on which the redeployment can be seen:

![bulk-recipes branch](https://res.cloudinary.com/adeshina/image/upload/v1611604587/isccfvfifunqfmnevtp8.jpg)

You can go on to test the new route by replacing `localhost` from previous requests to the live application url.


## Conclusion

In this article, you deployed an application directly from the Okteto dashboard and redeployed another new branch application version. The deployment was done with minor changes to the Okteto stack file, demonstrating the adaptability of the Okteto stacks. 

Additionally, Okteto can perform the deployment from different branches from the CLI tool. You can find the code from this article on [GitHub]
(https://github.com/Okteto/fastapi-crud/tree/bulk-recipes)
