Preview environments serve an important role when performing code reviews for an incoming change or addition to an application's codebase. It is a great way for technical and non-technical members of your team to assess and give feedback on the changes made to your application.

In the last couple of posts; [Building a FastAPI Application and Deploying it with Okteto Stacks](https://okteto.com/blog/building-and-deploying-a-fastapi-app-in-okteto-cloud/), [Deploying An Existing Application Using Okteto Cloud](https://okteto.com/blog/deploying-an-existing-application-with-okteto-cloud/) and [Adding A Database to Your Application Using Okteto Stacks](https://okteto.com/blog/adding-a-database-to-your-app-using-okteto-stack/), you have learned how to build a FastAPI application, deploy the application to Okteto and add a database using Okteto stacks. This article will be explaining how to configure preview environments for incoming changes from pull requests using GitHub Actions and Okteto.

## What Are Github Actions

[GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions) are event-driven task automation tools that runs a series of command after an event has occurred. These events and commands are contained in a workflow file. The workflow file is responsible for the orderly execution by GitHub Actions.


## Creating A GitHub Action Workflow

GitHub Actions are created and managed from the **Actions** tab in a repository. Create a new workflow from the Actions tab:

![Create A New Workflow](https://res.cloudinary.com/laisi/image/upload/v1615041975/emgx1lf6c9qe0snrxq9a.png)

There are a number of available templates in the GitHub Actions page. However, click on the **set up a workflow yourself** link. This opens a new page with a default workflow, let's rewrite the workflow file:

Start by defining the name of the workflow, and the event which should trigger the execution of the workflow file:

```yaml
name: CRUD App Preview Deployment

on: 
  pull_request:
    branches:
      - main

```

In the code block above, we have set the name of the workflow, and the event to trigger the workflow. The workflow file is executed when a pull request is made to the `main` branch. Next, let's define the series of tasks to be executed. These tasks are called **jobs** in GitHub Actions.

Add the following below the existing code:

```yaml
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
    - name: checkout
      uses: actions/checkout@master

    - name: Login
      uses: okteto/login@master
      with:
        token: ${{ secrets.OKTETO_TOKEN }}

    - name: Create namespace
      uses: okteto/create-namespace@master
      with:
        namespace: pr-${{ github.event.number }}-namespace
   
    - name: Deploy application
      uses: okteto/deploy-stack@master
      with:
        build: "true"
        namespace: pr-${{ github.event.number }}-namespace

    - name: add comment to PR
      uses: unsplash/comment-on-pr@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        msg: "Preview environment available at https://fastapi-pr-${{ github.event.number }}-namespace.cloud.okteto.net"  
```

In the code block above, we defined the job `preview` which runs on the `ubuntu-latest` environment. The steps to be executed under this job is defined under the `steps` heading. Let's go over these steps.

Replace `namespace` in the code block above with your Okteto default namespace.

### Steps

In our `preview` job, we have five tasks to be executed. 

1. `checkout`: This step checks out the repository using the `actions/checkout@master` actions, so that the repository is accessible by the entire workflow. 

2. `Login`: The step logs in into your Okteto account as a prerequisite to creating a new namespace where the preview environment will be deployed. This task makes use of` secrets.OKTETO_TOKEN`,  which holds the value of your Okteto token. You'll learn how to create the secret in the next section.

3. `Create Namespace`: This step creates a new namespace using the `okteto/create-namespace@master` action.
4. `Deploy Application`: This step uses the action `okteto/deploy-stack@master` to deploy the application using the okteto stack file. Once again, we have witnessed the adaptability of Okteto stacks.

5. `add comment to PR`: This step adds a comment on the pull request discussion indicating a link to the deployed application. It makes use of the `unsplash/comment-on-pr@master` action.

### Adding Your Okteto Token To GitHub

Before adding your Okteto token to GitHub, retrieve your OKteto token from your dashboard settings:

![Okteto dashboard - Settings](https://res.cloudinary.com/laisi/image/upload/v1615047658/pbkydburxszrz8hjgmgg.png)

The workflow file needs your [Okteto account](https://cloud.okteto.com) token to function properly. Follow the steps below to add your okteto token to your GitHub repository:

1. In your repository, navigate to the **Settings** tab and click on the **Secrets** link on the menu by the left:

![Secrets](https://res.cloudinary.com/laisi/image/upload/v1615045117/cskayu5fymd9pfkcnvvn.png)

2. Click on **New Repository Secret**

![Add A New Secret](https://res.cloudinary.com/laisi/image/upload/v1615045246/vvavcscqrww9dnxm6pyh.png)

3. Create a new secret `OKTETO_TOKEN` and set the value to the token you copied from your dashboard.

![Create New Secret](https://res.cloudinary.com/laisi/image/upload/v1615047981/pqfyim20zfw8gzoycwma.png)

You can learn more on how to store and manage GitHub secrets on their official [documentation](https://docs.github.com/en/actions/reference/encrypted-secrets).

## Building The Preview Environment

With the secret in place, let's go on to create a pull request from the `mongodb-crud` branch into the `main` branch to see the workflow in action:

![New Pull Request](https://res.cloudinary.com/laisi/image/upload/v1615049077/llf2krz4caqvfjcvcrx7.png)

Immediately the pull request is created, the workflow kicks into action:

![Workflow In Action](https://res.cloudinary.com/laisi/image/upload/v1615049480/ed3myn0hqtq0lg04rrnr.png)

To monitor the progress of the workflow in action, navigate to the **Actions** tab. The current workflow is listed like in the image below:

![Actions Tab](https://res.cloudinary.com/laisi/image/upload/v1615049702/ezywz076l9q0ml535doc.png)

Click on the workflow currently under execution. In the workflow page, click on the `preview` job to view the execution and logs of each step:

![Preview Job](https://res.cloudinary.com/laisi/image/upload/v1615049878/vrdlqon5qg5y1xfs3bqy.png)

The tasks undergoing execution are listed:

![Tasks undergoing execution](https://res.cloudinary.com/laisi/image/upload/v1615049920/d5u6ipcpajgwfjhxdy5x.png)

With all the necessary details put in place, the execution completes successfully:

![Workflow complete](https://res.cloudinary.com/laisi/image/upload/v1615050278/r1v2m4cihht4sxscqxit.png)

Return to the pull request thread to view the comment made by `github-actions` bot:

![Comment](https://res.cloudinary.com/laisi/image/upload/v1615051302/annyi2niixaeyloro1kk.png)

### Deleting Preview Environment Namespace

The preview environment ceases to be of use after the pull request has been merged. As such, there is no point in keeping the namespace created by the workflow earlier as newer pull requests will create new namespaces. Okteto provides an action file that allows us destroy namespaces from a workflow.

Repeating the steps followed in creating a workflow above, create a new workflow `preview-close` to automatically destroy the namespace after the pull request has either been merged or close:

```yaml
name: Delete Preview Namespace

on:
  pull_request:
    types:
      - closed

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@master

      - name: Login
        uses: okteto/login@master
        with:
          token: ${{ secrets.OKTETO_TOKEN }}

      - name: Delete namespace
        uses: okteto/delete-namespace@master
        with:
          namespace: pr-${{ github.event.number }}-namespace
```

Remember to change `namespace` to your Okteto namespace.

With the workflow in place, let's merge the pull request and verify that the delete namespace workflow was executed:

![Merge Pull Request](https://res.cloudinary.com/laisi/image/upload/v1615052185/zvflrb4o3xvcyhhh1q8w.png)

Navigate to the **Actions** tab and click on the workflow undergoing execution:

![Actions Tab](https://res.cloudinary.com/laisi/image/upload/v1615052200/disjxfoomldoc6l1aahm.png)

The namespace has been deleted successfully:

![Workflow Execution Complete](https://res.cloudinary.com/laisi/image/upload/v1615052635/doa7rgdjuyyf6fhzcerv.png)

## Conclusion

In this article, you built a workflow for creating a preview environment when a pull request is made to the main branch and another workflow to delete the namespace containing the preview environment after the pull request has either been closed or merged.

Once again, the deployment of the preview environment was made possible by the stack manifest we have been using from the [first post](https://okteto.com/blog/building-and-deploying-a-fastapi-app-in-okteto-cloud/) in the series. The complete code used in this article can be found in our [GitHub repository](https://github.com/okteto/fastapi-crud)
